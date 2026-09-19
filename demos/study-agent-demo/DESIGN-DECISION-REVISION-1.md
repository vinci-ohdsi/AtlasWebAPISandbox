# Design decision revision 1: cross-runtime concept-set validation

**Status:** agreed design decision.

This revision supersedes the technical-validation paragraph in section 2 of
[`DESIGN-DECISIONS.md`](DESIGN-DECISIONS.md).

## Two mandatory technical validation lanes

ACP performs producer-side technical validation of the canonical Atlas
`ConceptSetExpression` using its configured R/CirceR runtime before returning a
proposal. This follows the existing R-validation pattern, but uses a fixed
JSON-validation script rather than LLM-generated R or a cohort-only Capr
definition.

Before creating a draft, WebAPI3 independently and mandatorily validates the
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
persist a draft without explicit review approval.

## Incompatibility and provenance

If the runtimes disagree, WebAPI3 returns
`concept_expression_incompatible` and does not transform, save, or silently
accept the expression. The user may revise the reviewed proposal and request a
new validation.

Persisted review provenance includes:

- canonical-expression checksum;
- ACP validation status and R/CirceR/Capr versions;
- WebAPI build/version and WebAPI validation result;
- selected vocabulary-source metadata.

The demonstration includes cross-runtime golden expressions and compares their
resolved inclusion/mapping behavior across ACP and WebAPI before an interactive
release.
