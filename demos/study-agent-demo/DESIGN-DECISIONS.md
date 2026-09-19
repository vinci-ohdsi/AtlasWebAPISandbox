# Study Agent concept-search design decisions

**Status:** agreed design direction, pending implementation.

This addendum supersedes the matching proposed sections of
[`DESIGN.md`](DESIGN.md).

## 1. Proposal flow: concept-set-specific orchestration

The first implementation will use a dedicated `concept_set_authoring` ACP flow.
It may reuse vocabulary retrieval, structured candidate review, approval, and
technical-validation primitives from `phenotype_make_computable`, but it must
not generate a cohort definition as part of normal concept-set authoring.

This is intentional: `phenotype_make_computable` is a cohort-definition flow.
Its Capr/Circe output is valuable when assembling a cohort, but its Capr
guidance assumes the concept sets already exist. A concept-set proposal needs a
smaller artifact with a different completion condition: a reviewable Atlas
concept-set expression.

R/Capr/CirceR can remain a downstream optional integration when a user later
uses the approved concept set in a cohort definition. They are not the source
of truth for the concept-set draft.

## 2. Canonical draft artifact: Atlas `ConceptSetExpression`

The draft must retain the policy on every item; it cannot be represented as
flat `include_concept_ids` and `excluded_concept_ids` lists. The canonical
reviewed artifact is the normal Atlas/WebAPI expression shape:

```json
{
  "items": [
    {
      "concept": {
        "CONCEPT_ID": 21600714,
        "VOCABULARY_ID": "ATC"
      },
      "isExcluded": false,
      "includeDescendants": true,
      "includeMapped": false
    },
    {
      "concept": {
        "CONCEPT_ID": 21600728,
        "VOCABULARY_ID": "ATC"
      },
      "isExcluded": true,
      "includeDescendants": true,
      "includeMapped": false
    }
  ]
}
```

The actual expression retains the full concept records, as in normal Atlas
concept-set JSON. WebAPI3 hydrates reviewed concept IDs against its configured
vocabulary source before persisting concept-set item rows through existing
concept-set services.

WebAPI3 may run an optional technical check using existing concept-set SQL or
included/mapped-concept lookup functionality for a selected source. That check
does not imply clinical validity.

## 3. Explicit reviewed-expression persistence

The draft-creation request identifies the reviewed server-side expression by
revision and checksum rather than allowing a browser to submit altered policy
fields while asking WebAPI to persist them:

```json
POST /WebAPI/study-agent/v1/concept-set-sessions/{sessionId}/draft-concept-set
{
  "action": "create_draft",
  "approved_review_revision": 3,
  "approved_expression_checksum": "sha256:...",
  "concept_set_name": "Bolus insulin (draft)"
}
```

If the reviewer edits an item, its inclusion/exclusion/descendant/mapped policy
becomes a new server-side review revision and must be re-confirmed. WebAPI3
rejects an expired or changed review instead of silently filling in policy.

## 4. Feature access

The WebAPI3 feature flag is:

```text
slash-ohdsi-concept-search
```

It is disabled by default and is used together with the proposed
`study-agent:concept-set-assist` permission.

## 5. Generic vocabulary filtering

The feature is designed for generic vocabulary selection. Atlas3 supplies the
user's active Concepts → Search record filters as bounded constraints to the
WebAPI/ACP request. The UI must render any proposal that crosses vocabulary
boundaries as an explicit strategy requiring confirmation.

This matters for the insulin example: a request specifically for RxNorm codes
cannot silently become an ATC-classification expression simply because ATC
offers a convenient hierarchy. The assistant can propose that alternative, but
the user must explicitly approve it.

## 6. Demonstration fixtures and counts

The static vocabulary fixture will be selected after several concept-set use
cases and expected concept IDs are specified. It must include the vocabulary
tables and relationship/ancestor records necessary to review each scenario;
`concept` alone is insufficient.

EUNOMIA GiBleed 5.3 is a candidate synthetic fixture for secondary case-count
demonstration. Fixture-backed counts do not constitute clinical validation.
Adding counts to Concepts → Sets is out of scope; the existing Concepts →
Search count columns remain independent of this feature.

## 7. Remaining interactive decision

The persistence schema and retention period for assistant sessions, reviewed
expressions/manifests, and audit metadata remain to be designed interactively.
