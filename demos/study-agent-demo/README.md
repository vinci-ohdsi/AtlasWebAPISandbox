# Study Agent (/ohdsi) Atlas 3/WebAPI  demo

## Video of the intended demo in action

This demo uses forked Atlas 3 and WebAPI to  test integration of AI support using a forked version of [Study Agent](https://github.com/OHDSI/StudyAgent)  for cohort and concept set definition creation. This short [video on panopto](https://pitt.hosted.panopto.com/Panopto/Pages/Viewer.aspx?id=1696bc80-59e3-40e6-abc6-b4d40129d912&start=352) illustrates initial functioning work on the concept.


## Quick overview

To run this demo a developer would:

1. Clone the AtlasWebAPISandbox.
2. Clone/create sibling worktrees for the three forks at the commits/tags recorded in demo-manifest.local.json.
3. Create their ignored local StudyAgent/WebAPI configuration, including the phenotype index and credentials.
4. Start MCP → ACP → WebAPI → Atlas as documented.
5. Run the scripts to verify the checkpoint pins and, after browser use, the HTTP archive (HAR) network boundary.

No GitHub workflow is involved in running this local demo. The Sandbox has a generic workflow that validates committed demo-manifest.json files for digest-pinned OCI demos, but this local source-built profile uses demo-manifest.local.json, so it is intentionally outside that workflow.

The final source record for the successful OHDSI 2026 Global Symposium
demonstration is [OHDSI Global Symposium 2026](releases/ohdsi-global-symposium-2026.json).
It records the exact Study Agent, Atlas3, WebAPI3, R-package, and conference
client commits. It is provenance for the demonstrated stack, not a portable
production deployment profile.

## Detailed instructions

This is the profile for a demo using the completed 2026-09-29 Atlas
concept-set and cohort-assistance checkpoint. It is a **source-built local
integration demonstration**, not a portable or production deployment. In
particular, it does not provide digest-pinned component images, Docker Compose,
or a redistributable Athena vocabulary fixture.

The source boundary is recorded in
[`demo-manifest.local.json`](demo-manifest.local.json). Before running the demo, all
three component worktrees must be at the listed commits—not merely on a branch
with the same name.

## What this demo proves

- Atlas calls only its normal `/WebAPI` endpoint. WebAPI calls ACP server-side;
  ACP calls MCP server-side. ACP/MCP credentials never reach the browser.
- Concept-set assets, cohort criterion bindings, and cohort logic are separate
  reviewed artifacts.
- No concept policy, Circe draft, or cohort logic decision is implicit. The
  user must review and confirm it before a draft is emitted.
- The two implemented multi-component projections are shown as narrow,
  testable templates: Condition index + overlapping Visit, and Drug index +
  pre-index supporting Condition.

It does **not** prove clinical validity, a complete vocabulary retrieval
universe, portable deployment, or production readiness.

## Prerequisites

- The sibling `StudyAgent`, `Atlas3`, and `WebAPI3` worktrees shown in the
  manifest, with their normal build dependencies installed.
- Java 21, Node/npm, `uv`, and an R runtime/library configured for the local
  Study Agent checkout.
- A local, non-production PostgreSQL WebAPI database with a compatible local
  vocabulary source and an Atlas administrator. Licensed vocabulary and model
  credentials remain operator-provided inputs.
- A service or pair of services to handle OpenAI API compliant chat completion and text  embedding requests. A useful project that has been tested for this role is https://github.com/dbmi-pitt/llm-shim.git  but Ollama and Open-WebUI have also been tested and found to work. 
- ACP and MCP listening only on local/internal addresses. Do not expose their  ports to the browser network. The ACP service configured to 1) use the chat completion/text embedding services and 2) use an indexed phenotype library set up under the `data/phenotype_index` folder (configured through configured through StudyAgent’s paths.phenotype_index in its config.yaml). The README.md for Study Agent explains how to create an index using demo cohorts provided in the repository. 

Copy `.env.example` to ignored `.env`, replace every `replace-with-*` value,
and export it before starting WebAPI. Copy
`config/webapi-study-agent-demo.yaml.example` to an ignored local WebAPI YAML
file and merge it with the standard WebAPI configuration. The template names
only the Study Agent flags, the server-side ACP endpoint, database settings,
and the Atlas origin; it is not a complete WebAPI configuration.

```sh
cd sandbox/AtlasWebAPISandbox/demos/study-agent-demo
cp .env.example .env
cp config/webapi-study-agent-demo.yaml.example config/webapi-study-agent-demo.local.yaml
set -a; . ./.env; set +a
python3 scripts/verify_local_profile.py
```

Do not put model keys, database passwords, JWT secrets, or ACP/MCP credentials
in Git, Atlas environment variables, or browser build variables.

## Ordered startup and verification

Run these from the repository root unless a command says otherwise. Keep each
service in its own terminal and capture its log for the meeting record.

1. Confirm the source boundary.

   ```sh
   python3 sandbox/AtlasWebAPISandbox/demos/study-agent-demo/scripts/verify_local_profile.py
   ```

   It must report all three component tags and `HEAD` values as `PASS`. If a
   worktree has advanced, reset neither it nor its unrelated work: use a clean
   worktree at the recorded commit instead.

2. Start MCP, then ACP, using the configured StudyAgent local profile. The
   local `config.yaml` and ignored `secrets.env` remain the source of model,
   vocabulary, and internal endpoint settings.

   ```sh
   cd sandbox/StudyAgent
   uv run study-agent-mcp --config ./config.yaml --profile native
   # separate terminal, after MCP is ready
   uv run study-agent-acp --config ./config.yaml --profile native
   ```

   Confirm ACP is listening at `STUDY_AGENT_ACP_BASE_URL` and has its configured
   server-side MCP client. A configured live model must have its model
   identifier/version, serving stack, and inference settings recorded in the
   meeting notes; `latest` is not reproducible provenance.

3. Build and start WebAPI with the local merged YAML. WebAPI Flyway startup is
   the migration gate. Use the normal WebAPI launch method for the local
   checkout, passing the copied configuration as an additional configuration
   location; for example:

   ```sh
   JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64 \
   PATH=/usr/lib/jvm/java-21-openjdk-amd64/bin:$PATH \
   mvn -f sandbox/WebAPI3/pom.xml -DskipTests compile

   JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64 \
   PATH=/usr/lib/jvm/java-21-openjdk-amd64/bin:$PATH \
   mvn -f sandbox/WebAPI3/pom.xml spring-boot:run \
     -Dspring-boot.run.arguments="--spring.config.additional-location=file:$(pwd)/sandbox/AtlasWebAPISandbox/demos/study-agent-demo/config/webapi-study-agent-demo.local.yaml"
   ```

   Verify Flyway completed and the WebAPI log reports both Study Agent surfaces
   enabled. In PostgreSQL, confirm these migration-created tables exist in the
   configured OHDSI schema: `study_agent_concept_set_session`,
   `study_agent_concept_set_review`, `study_agent_cohort_definition_session`,
   `study_agent_cohort_definition_review`, and
   `study_agent_cohort_definition_specification`.

4. Log in as a local administrator. Confirm the `Study Agent` role (or the two
   permissions `study-agent:concept-set-assist` and
   `study-agent:cohort-definition-assist`, plus normal create permissions) is
   assigned to the meeting user. The feature is unavailable by design without
   both permission and the enabled WebAPI properties.

5. Start Atlas in development mode, pointing its proxy only at WebAPI.

   ```sh
   WEBAPI_URL=http://localhost:8080 npm --prefix sandbox/Atlas3 run dev
   ```

   Open `http://localhost:5173`. In the browser Network panel, filter for
   `/WebAPI`; the requests should use the Atlas origin/proxy. There must be no
   request to the ACP or MCP origin.

## Demo walkthrough

Use an authorized demo account and non-PHI narrative. Keep each draft unsaved
until its explicit review step says otherwise.

1. **Concept-set review — Selected versus Included.** From Concepts search,
   begin `/ohdsi` concept-set authoring. Clarify scope, explicitly request the
   bounded proposal, inspect the candidate rationale and policy, and choose
   **Apply reviewed items**. Show the exact item flags in **Selected**, then
   open **Included** to show its derived expansion. State that the retrieval
   slice is not a claim of completeness and that an edit invalidates review
   provenance until renewed review.

2. **Library/recommendation to an unsaved draft.** From Cohort Definitions,
   use either AI phenotype search or library browse. Open a candidate marked
   Circe-available into an **unsaved** Atlas draft; for a conversion-required
   source, show it as reference evidence rather than an executable cohort.
   Do not save or run generation as part of this story.

3. **Esketamine + MDD multi-component plan.** Start a new computable cohort
   plan for esketamine exposure with major depressive disorder supporting
   evidence. Review each concept-set policy, bind the Drug index and Condition
   evidence roles, explicitly set the pre-index evidence window and exit
   strategy, and confirm cohort logic. Only then generate and show the
   **unsaved** cohort draft. Point out the supported template boundary; this is
   not a general Boolean/recurrence cohort builder.

If a step is incomplete, stop at the review surface. Do not infer policy,
criteria, timing, or exit choices to keep the narrative moving.

## Browser boundary evidence

Export a HTTP Archive (HAR) after exercising at least one `/ohdsi` story, then run:

```sh
python3 sandbox/AtlasWebAPISandbox/demos/study-agent-demo/scripts/verify_browser_har.py \
  meeting.har --atlas-origin http://localhost:5173 --webapi-origin http://localhost:8080
```

The check fails if the browser called the local ACP/MCP origins or if a WebAPI
request used an unexpected origin. Retain the HAR with demo evidence only if
it contains no sensitive user data.

## Refresh rules

- After Java changes, rebuild and restart WebAPI.
- After Atlas changes, refresh/restart its dev server.
- After MCP tools, prompts, retrieval, or configuration changes, restart MCP.
- After ACP contracts/server changes, restart ACP after MCP.

Source edits or a running old process are not evidence that the meeting is
using the checkpoint.

## Lightweight automated checks

```sh
python3 sandbox/AtlasWebAPISandbox/demos/study-agent-demo/scripts/verify_local_profile.py
python3 -m py_compile sandbox/AtlasWebAPISandbox/demos/study-agent-demo/scripts/*.py
npm --prefix sandbox/Atlas3 run type-check
JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64 PATH=/usr/lib/jvm/java-21-openjdk-amd64/bin:$PATH \
  mvn -q -f sandbox/WebAPI3/pom.xml -DskipTests compile
```

The first check validates the actual source pins and required local template
fields. The latter two validate the pinned frontend/backend source. They do not
substitute for the live migration, permission, connectivity, and browser-HAR
gates above.
