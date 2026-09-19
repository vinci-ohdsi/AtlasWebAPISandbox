# Study Agent concept-search assistant: integration design

**Status:** agreed design contract; no runtime endpoint is implemented by this document.

## Objective

Enable an authenticated Atlas3 user to enter an `/ohdsi` narrative in the
Concepts → Search view, receive a Study Agent-guided strategy, and—only after
explicit review and confirmation—create an ordinary draft Atlas concept set.

Example initial request:

> `/ohdsi help me find all RxNorm codes for bolus insulin, excluding mixed
> bolus and infusion products`

The feature assists concept-set authoring. It does not establish clinical
validity, execute a cohort, or automatically save a concept set.

## Architecture and trust boundary

```text
Atlas3 browser
  │ Atlas JWT; same-origin /WebAPI request
  ▼
WebAPI3 Study Agent façade
  │ internal service credential; request validation and audit metadata
  ▼
Study Agent ACP → MCP → vocabulary/model services
```

Atlas3 must not call ACP or MCP directly and must never receive an ACP service
credential or an LLM credential. WebAPI3 enforces Atlas authorization, applies
the product-specific request contract, owns durable session state, and proxies
only the required results.

ACP/MCP are internal-only services in the demonstration network. They are not
host-published ports.

## WebAPI3 API contract

All routes are relative to the normal WebAPI context path and require a valid
Atlas JWT, the `study-agent:concept-set-assist` permission, and the enabled
`slash-ohdsi-concept-search` feature flag.

```text
POST /WebAPI/study-agent/v1/concept-set-sessions
POST /WebAPI/study-agent/v1/concept-set-sessions/{sessionId}/messages
POST /WebAPI/study-agent/v1/concept-set-sessions/{sessionId}/proposals
GET  /WebAPI/study-agent/v1/concept-set-sessions/{sessionId}/review
POST /WebAPI/study-agent/v1/concept-set-sessions/{sessionId}/draft-concept-set
```

`sessionId` is an opaque WebAPI-owned identifier. The browser must not use an
ACP review ID as a session identifier or resume token.

### Create session

```json
POST /study-agent/v1/concept-set-sessions
{
  "command": "/ohdsi",
  "message": "help me find all RxNorm codes for bolus insulin, excluding mixed bolus and infusion products",
  "ui_context": {
    "route": "concepts",
    "tab": "search",
    "vocabulary_filters": ["RxNorm"]
  }
}
```

WebAPI3 removes the command prefix before forwarding a bounded narrative to
ACP's `workflow_context_dialogue` flow. It supplies a fixed product context:

```json
{
  "study_intent": "Create a reviewable Atlas concept set",
  "workflow_type": "concept_set_authoring",
  "current_step": "strategy",
  "current_role": "concept_set_author",
  "current_context": {"source": "atlas3", "route": "concepts", "tab": "search"}
}
```

The response is a WebAPI-owned session view, not a raw ACP response:

```json
{
  "session_id": "csas_opaque_id",
  "state": "needs_clarification",
  "assistant_message": "Do you intend to include ingredient concepts, clinical drugs, or both?",
  "questions": [
    {
      "id": "concept_level",
      "prompt": "Which concept levels should be included?",
      "options": ["ingredient", "clinical_drug", "both"]
    }
  ],
  "allowed_actions": ["reply", "cancel"],
  "expires_at": "2026-09-19T18:00:00Z"
}
```

### Continue dialogue

```json
POST /study-agent/v1/concept-set-sessions/{sessionId}/messages
{
  "message": "Both, and exclude products containing protamine.",
  "answers": {"concept_level": "both"}
}
```

WebAPI3 validates that the session belongs to the caller and that the state
allows a reply. It returns the same session-view shape. The assistant may offer
`request_proposal`, but may not create or modify an Atlas concept set.

### Request a proposal

```json
POST /study-agent/v1/concept-set-sessions/{sessionId}/proposals
{
  "action": "request_proposal",
  "proposal_scope": {
    "vocabulary_ids": ["RxNorm"],
    "concept_levels": ["ingredient", "clinical_drug"],
    "exclude_terms": ["protamine", "mixed", "infusion"]
  }
}
```

This is an explicit user transition from discussion to a bounded proposal. The
first implementation uses a dedicated `concept_set_authoring` ACP flow. It
reuses vocabulary retrieval, structured candidate review, approval, and
technical-validation patterns from `phenotype_make_computable`, but does not
generate a cohort definition, Capr source, or Circe cohort JSON. The existing
`phenotype_make_computable` flow is cohort-definition oriented and its Capr
guidance assumes concept sets are already built.

The concept-set flow preserves these safeguards:

- retrieve candidates; do not treat a retrieval limit as completeness;
- state proposed descendant, mapped-code, and exclusion policies explicitly;
- return candidates and rationale for review;
- do not emit a Capr/Circe artifact or persist an Atlas concept set yet.

The response state becomes `review_ready` only when candidate data and the
proposed policy are present. A compact response can include an opaque
`review_handle`; WebAPI3 pages review rows from its own endpoint and persists
the ACP review manifest/approved-policy provenance necessary for resume.

### Retrieve review

```text
GET /study-agent/v1/concept-set-sessions/{sessionId}/review?offset=0&limit=100
```

The result contains the proposed policy, candidate concepts, inclusion or
exclusion rationale, retrieval diagnostics, and pagination metadata. It must
make clear that the material is a technical proposal, not clinical validation.

### Create draft concept set

```json
POST /study-agent/v1/concept-set-sessions/{sessionId}/draft-concept-set
{
  "action": "create_draft",
  "approved_review_revision": 3,
  "approved_expression_checksum": "sha256:...",
  "concept_set_name": "Bolus insulin (draft)"
}
```

The submitted revision and checksum identify the complete, server-held reviewed
expression. They avoid reducing a concept set to flat inclusion/exclusion IDs
and prevent a browser from silently changing item policy while requesting
persistence. WebAPI3 rejects an expired, missing, or changed review rather than
silently filling in policy fields. On success, it creates a normal Atlas draft
concept set under the current user and returns its standard identifier and editor
URL.

### Canonical reviewed artifact: `ConceptSetExpression`

The review and persistence artifact is Atlas/WebAPI's concept-set expression,
not a Circe cohort definition. Every item retains its own inclusion/exclusion,
descendant, and mapped-code choices:

```json
{
  "items": [
    {
      "concept": {"CONCEPT_ID": 21600714, "VOCABULARY_ID": "ATC"},
      "isExcluded": false,
      "includeDescendants": true,
      "includeMapped": false
    },
    {
      "concept": {"CONCEPT_ID": 21600728, "VOCABULARY_ID": "ATC"},
      "isExcluded": true,
      "includeDescendants": true,
      "includeMapped": false
    }
  ]
}
```

The ACP proposal produces structured candidate and policy data. WebAPI3
hydrates each approved concept from its configured vocabulary source and stores
the normal concept-set item rows through existing WebAPI concept-set services.

#### Mandatory cross-runtime technical validation

ACP performs producer-side technical validation of the canonical expression
using its configured R/CirceR runtime before returning a proposal. This follows
the existing R-validation pattern, but uses a fixed JSON-validation script
rather than LLM-generated R or a cohort-only Capr definition.

Before creating a draft, WebAPI3 independently and mandatorily validates that
same canonical expression using its deployed Java/Circe-based concept-set
implementation and configured vocabulary source. This is the authoritative
acceptance gate for persistence. It includes existing concept-set SQL and
included/mapped-concept lookup behavior where applicable to the selected source.

```text
ACP / R / CirceR validation
  → Can the Study Agent service technically construct and interpret it?

WebAPI3 / Java / Circe validation
  → Will this exact expression work in the deployed Atlas/WebAPI environment?
```

Both checks must pass. ACP validation is technical producer-side evidence; it
does not establish WebAPI compatibility, clinical validity, or permission to
persist a draft without explicit review approval. If the runtimes disagree,
WebAPI3 returns `concept_expression_incompatible` and does not transform, save,
or silently accept the expression.

Persisted review provenance includes the canonical-expression checksum, ACP
validation status and R/CirceR/Capr versions, WebAPI build/version, WebAPI
validation result, and selected vocabulary-source metadata. The demonstration
includes cross-runtime golden expressions and compares resolved
inclusion/mapping behavior across ACP and WebAPI before an interactive release.

Capr/Circe cohort artifacts remain downstream optional outputs when a user later
uses the saved reviewed concept set in a cohort-definition workflow. They are
derived from the concept-set expression and never replace it as the source of
truth for concept-set authoring.

## Atlas3 state model

The command is recognized only when the trimmed search input starts with
`/ohdsi` followed by whitespace. All other values retain the ordinary concept
search behavior.

```text
idle
  → interpreting
  → needs_clarification ↔ interpreting
  → strategy_ready
  → proposing
  → review_ready
  → creating_draft
  → complete

Any non-terminal state → cancelled
Any request state → error
```

Suggested client state fields:

```ts
type ConceptSetAssistantState =
  | 'idle'
  | 'interpreting'
  | 'needs_clarification'
  | 'strategy_ready'
  | 'proposing'
  | 'review_ready'
  | 'creating_draft'
  | 'complete'
  | 'cancelled'
  | 'error'

interface ConceptSetAssistantSession {
  sessionId: string
  state: ConceptSetAssistantState
  assistantMessage?: string
  questions: AssistantQuestion[]
  allowedActions: AssistantAction[]
  review?: ConceptSetReview
  error?: { code: string; message: string; retryable: boolean }
}
```

### Atlas3 interaction surfaces

Atlas3 already renders `ConceptSetEditor` as a page-level, right-side drawer
over both Concepts tabs. The assistant uses that editor rather than adding a
third top-level Concepts tab or replacing the normal vocabulary search table.

```text
Concepts → Search landing page
  /ohdsi narrative
        ↓
Open unsaved ConceptSetEditor drawer + attach a new assistant session
        ↓
Dialogue and proposal review inside the drawer
        ↓
Apply reviewed definition items to Selected
        ↓
Included resolves the vocabulary expansion of Selected
        ↓
Create/Save performs final WebAPI validation and persistence
```

The assistant has two modes:

- **New mode:** a landing-page `/ohdsi` command opens an empty, unsaved
  concept-set drawer. The user names and reviews the draft there before the
  normal Create action persists it.
- **Extension mode:** a `/ohdsi` command in the Search tab of an existing
  concept-set drawer creates a proposed diff against that set's current
  expression and version. It never replaces the existing expression wholesale.

The assistant panel belongs inside the concept-set drawer. It renders
structured clarification questions, strategy cards, rationale, technical
validation status, and an explicit `Apply reviewed items` action; it does not
treat LLM prose as executable instructions.

`Selected` is the exact, editable `ConceptSetExpression`: concept rows and
their individual exclusion, descendant, and mapped flags. This is the
authoritative policy-review surface. `Included` is the non-authoritative,
server-resolved expansion of `Selected`, including descendants where selected.
It is evidence of the definition's effect, not a surface for editing its rules.

An assistant proposal enters a review queue and changes `Selected` only after
the user applies reviewed items. A manual edit to `Selected` after proposal
approval invalidates the approved-expression checksum. The user may continue
editing freely, but Create/Save must obtain a new valid reviewed expression and
complete the mandatory WebAPI validation before persistence.

The clarification option `classification` is intentionally generic rather than
an ATC-only label. `all` means every concept level offered in that response; the
response schema must state the concrete option set so its meaning cannot drift.

## Security, privacy, and failure behavior

- Send narrative authoring intent and vocabulary metadata only. Do not send
  patient rows, person identifiers, clinical notes, or raw EUNOMIA records to
  ACP/LLM services.
- Do not log narrative text, credentials, bearer tokens, or model prompts at
  normal application log level. Audit event metadata and immutable artifact
  identifiers instead.
- WebAPI3 maps ACP timeout/unavailable responses to a retryable, user-safe
  `study_agent_unavailable` response; it never exposes stack traces or service
  credentials.
- A missing or expired ACP review record is not an approval. WebAPI3 either
  resumes from preserved review artifacts or returns `review_expired` and asks
  the user to request a new proposal.
- Permission denial is `403`; an unknown or other-user session is `404` to
  avoid session enumeration.
- The feature is disabled by default and is controlled by the explicit WebAPI3
  feature flag `slash-ohdsi-concept-search` plus the dedicated permission.

## Demonstration requirements

The future `compose.yaml` consumes digest-pinned images for Atlas3, WebAPI3,
Study Agent ACP/MCP, PostgreSQL, and any vocabulary fixture. It should expose
only the reverse proxy/Atlas/WebAPI entry points.

The demo needs a versioned vocabulary subset containing the exact concepts and
relationships used by several specified concept-set scenarios. Include the
relevant OMOP vocabulary tables and relationship/ancestor records; a
`concept`-only subset is not sufficient for meaningful inclusion and exclusion
review. The fixture is built from an agreed vocabulary query when the scenarios
and expected concept IDs are final.

EUNOMIA GiBleed 5.3 is a candidate synthetic data fixture for secondary
case-count demonstration. Such counts must be clearly labelled as
fixture-backed and must not be used to claim clinical validation of the
proposal. Counts in Concepts → Sets are out of scope; the existing Concepts →
Search count columns remain independent of this feature.

Provide two profiles:

- `fixture`: deterministic ACP/model responses suitable for CI and reviewer
  reproduction, with no external model key;
- `live`: actual configured ACP/MCP/model services, with credentials injected
  outside version control and model/version/inference settings recorded in the
  demo manifest.

## Acceptance scenario

1. An authorized user enters the example `/ohdsi` narrative in the Concepts →
   Search landing field.
2. Atlas3 opens an empty, unsaved `ConceptSetEditor` drawer and attaches the
   new Study Agent session to that draft.
3. The drawer assistant panel displays a strategy or structured clarification
   question. The user selects a concept level, including `classification` or
   `all` only when intended, and explicitly requests a proposal.
4. The panel displays candidate rationale and item-level policy. After explicit
   approval, the user applies the reviewed items; they appear in `Selected` with
   their exact exclusion/descendant/mapped flags.
5. The user opens `Included` and observes the resolved extension of the selected
   definition. They can refine the draft using the drawer's normal Search tab.
6. If the user makes a manual Selected-item change, Atlas3 requires renewed
   review and validation before Create/Save.
7. Create/Save succeeds only after the approved expression passes both ACP and
   WebAPI validation. WebAPI3 creates the normal concept set and Atlas3 retains
   it in the same editor drawer.
8. A separate smoke case starts `/ohdsi` from an existing editor Search tab and
   verifies that the proposal is a reviewable diff rather than a replacement.
9. The smoke test verifies that no browser request targets ACP/MCP directly and
   that no secret appears in the browser bundle or committed demo configuration.

## Decisions still required before implementation

- The persistence schema and retention period for assistant sessions, review
  manifests, and audit metadata.
- The fixture vocabulary source/licensing/distribution approach and expected
  concept IDs, after the additional concept-set scenarios are specified.
- The initial generic vocabulary-filter contract: Atlas3 supplies the user's
  active concept-record filters as constraints, and the proposal must surface
  any cross-vocabulary strategy for explicit confirmation. For example, a
  request for RxNorm codes must not silently substitute ATC classification
  concepts merely because they offer convenient hierarchy.
