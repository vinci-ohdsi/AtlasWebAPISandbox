# Study Agent concept-search assistant: integration design

**Status:** proposed contract; no runtime endpoint is implemented by this document.

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
Atlas JWT plus the proposed `study-agent:concept-set-assist` permission.

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
WebAPI adapter may use `phenotype_make_computable` or a dedicated
concept-set-authoring adapter, but it must preserve the same safeguards:

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
  "approved_policy": {
    "include_concept_ids": [1, 2, 3],
    "excluded_concept_ids": [4],
    "include_descendants": false,
    "include_mapped": false
  },
  "concept_set_name": "Bolus insulin (draft)"
}
```

The explicit submitted policy is compared with the reviewed proposal and the
session state. WebAPI3 rejects missing or changed policy fields rather than
silently filling them in. On success, it creates a normal Atlas draft concept
set under the current user and returns its standard identifier and editor URL.

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

The UI should use a side panel or drawer separate from the normal search-result
table. It should render structured questions, strategy cards, policy fields,
and candidate review rows rather than treating LLM prose as executable
instructions. `Create draft concept set` remains disabled until an explicit,
complete policy review has occurred.

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
- The feature is disabled by default and is controlled by an explicit WebAPI
  configuration switch plus the dedicated permission.

## Demonstration requirements

The future `compose.yaml` consumes digest-pinned images for Atlas3, WebAPI3,
Study Agent ACP/MCP, PostgreSQL, and any vocabulary fixture. It should expose
only the reverse proxy/Atlas/WebAPI entry points.

The demo needs a versioned vocabulary subset containing the exact concepts and
relationships used by the bolus-insulin scenario. Include the relevant OMOP
vocabulary tables and relationship/ancestor records; a `concept`-only subset is
not sufficient for meaningful inclusion and exclusion review.

EUNOMIA case counts, if included, are secondary. They must be clearly labelled
as fixture-backed demonstration counts and must not be used to claim clinical
validation of the proposal.

Provide two profiles:

- `fixture`: deterministic ACP/model responses suitable for CI and reviewer
  reproduction, with no external model key;
- `live`: actual configured ACP/MCP/model services, with credentials injected
  outside version control and model/version/inference settings recorded in the
  demo manifest.

## Acceptance scenario

1. An authorized user enters the example `/ohdsi` narrative in Concepts →
   Search.
2. Atlas3 opens the assistant panel and displays a strategy or structured
   clarification question.
3. The user answers and explicitly requests a proposal.
4. Atlas3 displays reviewable candidate concepts, explicit policies, exclusions,
   and rationale; no concept set has been saved.
5. The user approves the displayed policy and explicitly creates a draft.
6. WebAPI3 creates a normal draft concept set and Atlas3 opens it in the usual
   editor.
7. The smoke test verifies that no browser request targets ACP/MCP directly and
   that no secret appears in the browser bundle or committed demo configuration.

## Decisions still required before implementation

- Whether the first proposal adapter is a constrained mode of
  `phenotype_make_computable` or a new concept-set-specific ACP flow.
- The precise role/permission migration and feature-flag name in WebAPI3.
- The persistence schema and retention period for assistant sessions, review
  manifests, and audit metadata.
- The fixture vocabulary source, licensing/distribution approach, and the
  expected concept IDs for deterministic smoke tests.
- Whether the first release supports only a named RxNorm subset or generic
  vocabulary selection.
