The exact-`0.7.0` gate is hard to defend: Dify itself never rejects older DSL, and the version string is a weak proxy for file content. Recommendation below: a range gate with a floor of `0.1.5` plus content sniffing, not exact equality. Nothing under `E:/github` was written; a partial clone in a temp dir outside the workspace was used and deleted.

## (a) DSL version timeline

Verified from the constant's history, the patch of each bump commit, and node entity diffs between tags.

| DSL   | Dify release (date)                                     | Bump commit (PR)                                       | Change                                                       | What a normalizer must handle in files of that version       |
| ----- | ------------------------------------------------------- | ------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 0.1.0 | 0.6.15 (2024-07-23); in-code comment maps it to "0.6.0" | `46a5294d94` (#6287)                                   | First envelope: `kind`/`version`/`app` + `workflow` or `model_config` | Legacy `variable-assigner` aggregator with `output_type: array`; if-else with flat `conditions` + `logical_operator`; no node `version`/`retry_config`/`error_strategy`; pre-plugin provider ids; no `dependencies` |
| 0.1.1 | 0.7.0 (2024-08-14)                                      | `5b77ef01d4` (#7235)                                   | `conversation_variables`; new `assigner` node (v1)           | Assigner v1: `assigned_variable_selector`, `write_mode`, `input_variable_selector` |
| 0.1.2 | 0.8.0 (2024-09-10)                                      | `dabfd74622` (#8192)                                   | Parallel execution                                           | Without a bump: `iteration-start`, `document-extractor`, `list-operator`, `file`/`file-list` start variables, HTTP body as a list (0.10.0) |
| 0.1.3 | 0.11.0 (2024-11-05)                                     | `7f583ec1ac` (#10278)                                  | Release chore                                                | Iteration parallel fields (`is_parallel`, `parallel_nums`, `error_handle_mode`) |
| 0.1.4 | 0.13.0 (2024-12-03)                                     | `7a962b9f03` (#11284)                                  | Variable assigner v2                                         | Assigner v2: `version: "2"`, `items[]`; every node gains `version: "1"` |
| 0.1.5 | 0.14.2 (2024-12-23) to 1.2.0                            | `e88ea71aef` (#12017)                                  | Retry and node error handling                                | Straddles the 1.0.0 plugin break, so the version alone is ambiguous; `agent` node; `dataset_ids` may be AES-encrypted |
| 0.2.0 | 1.3.0 (2025-04-23)                                      | `e9aedc701c` (#18550)                                  | LLM structured output                                        | Default missing `structured_output_enabled` / `structured_output` |
| 0.3.0 | 1.4.0 (2025-05-15)                                      | `937d151187` (#19731)                                  | Assigner gains `remove-first` / `remove-last`                | Default filling only                                         |
| 0.3.1 | 1.7.0 (2025-07-23)                                      | `7ec94eb83c` (#22830)                                  | Tool `credential_id`, `tool_node_version`                    | Default filling only                                         |
| 0.4.0 | 1.8.0 (2025-08-27)                                      | `f04844435f` (#24539)                                  | Start `checkbox`; parameter types renamed; code output types widened; LLM `reasoning_format` | Map parameter extractor `bool` to `boolean` and `select` to `string` |
| 0.5.0 | 1.10.0 (2025-11-13)                                     | `b76e17b25d` (#27644)                                  | Trigger nodes                                                | Reject or ignore `trigger-webhook`, `trigger-schedule`, `trigger-plugin` |
| 0.6.0 | 1.13.0 (2026-02-11)                                     | `03e3acfc71` (#31646, reverted), `a1fc280102` (#32060) | Human-input node                                             | Reject or implement `human-input`                            |
| 0.7.0 | 1.16.0 (2026-07-17), still current at 1.20.0 and `main` | `f86bfb2a31` (#38849)                                  | Agent package DSL (`agent`, `agent_packages`, app mode `agent`) | Tolerant parsing: see below                                  |

- **Unreleased value:** `0.2.0` briefly appeared in Nov 2024 (`5172f0bf39`, downgraded same day by `556de444e8`) and never shipped.
- **The version string misses real changes.** Node schemas changed in patch versions or with no bump at all (HTTP body, file variables, the plugin break, parameter type renames). After 0.7.0, with no bump, came `site` (#42863), LLM `model_selector` (#39125), human-input v2, `knowledge-retrieval-v2`, `agent-v2`, and a default `.ifpkg` export (#42552, 2026-09-20).
- **`.ifpkg` is a zip** with a `format: dify.app` manifest plus the DSL and resources; explicit YAML export still exists.
- **Legacy app modes** (`chat`, `completion`, `agent-chat`) carry `model_config` at every DSL version. Dify imports them as-is; conversion is a separate action that creates a new app (`api/services/workflow/workflow_converter.py`).

## (b) Recommended minimum

- **Gate:** accept a range, `0.1.5 <= v <= 0.7.0`, and treat anything newer as "unsupported" (mirroring Dify's confirm step) rather than rejecting everything but one string.
- **Why `0.1.5` as the floor:** it covers Dify 0.14.2 through 1.2.0, and the only real ambiguity is plugin era. Detect that from content: presence of `dependencies`, or `provider` ids shaped like `langgenius/openai/openai`.
- **Why `0.2.0` is the clean baseline:** from there, everything is additive (new optional fields, enum values, node types, sections), so the normalizer is default-filling plus alias mapping.
- **Below `0.1.5`:** reject unless there is demand. It adds assigner v1 conversion, the old HTTP body shape, and the legacy aggregator, and all of it is pre-plugin.
- **Regardless of version:** validate node `data.type` against what the engine supports. Version says little about which nodes a file contains.

## (3) How Dify's importer treats older DSL today

`api/services/dsl_version.py` on `main` (`check_version_compatibility`):

- **Invalid version string or non-string:** `FAILED` (`version: 1` fails; YAML float).
- **Newer than current:** `PENDING` (user confirms in `confirm_import`).
- **Older major:** `PENDING`.
- **Older minor:** `COMPLETED_WITH_WARNINGS`, no confirmation.
- **Nothing older is rejected.** There is no minimum-version check.

Normalisation it performs in `api/services/app_dsl_service.py`:

- Defaults missing `version` to `"0.1.0"` and `kind` to `"app"`.
- Strips `graph.viewport`; decrypts or passes through `dataset_ids`.
- For `imported_version <= "0.1.5"` (the one hard-coded comparison), derives plugin `dependencies` from the graph or `model_config`.
- Does no node migration at import; the graph is stored as JSON.
- At graph initialisation, `api/core/workflow/human_input_adapter.py` rewrites `variable-assigner` to `variable-aggregator`, normalises human-input recipients, and moves legacy tool `tool_configurations` into `tool_parameters`.
- Entity validators accept `structured_output_enabled` alongside `structured_output_switch_on`, and the old parameter type names.
- Export always stamps the current version, so "upgrade" is really re-export.

Evolution of the check:

- 0.6.x to 0.10.x and 0.11.0 to 0.14.x: warn only.
- 1.0.0 to 1.3.0: any major or minor difference gave `PENDING`.
- 1.3.1 (#18798) onward: the logic above.

## (4) End/Output node type

- **String is `end` in both.** No `output` node type exists in either enum.
- **graphon `main`** (`94109ce8`, 2026-11-25), `src/graphon/enums.py`: `NodeType = str`, and `BuiltinNodeTypes` has 25 members. It has not changed since graphon's first commit.
  - `start`, `end`, `answer`, `llm`, `knowledge-retrieval`, `if-else`, `code`, `template-transform`, `question-classifier`, `http-request`, `tool`, `datasource`
  - `variable-aggregator`, `variable-assigner` (legacy), `loop`, `loop-start`, `loop-end`, `iteration`, `iteration-start`, `parameter-extractor`, `assigner`, `document-extractor`, `list-operator`, `agent`, `human-input`
- **Dify 1.9.0** (`2e2c87c5a1`), `api/core/workflow/enums.py`: `NodeType(StrEnum)`, also 25 members.
- **Differences:** 1.9.0 has `knowledge-index` and lacks `human-input`; graphon is the reverse. No shared string changed.
- **Elsewhere:** trigger types live in `api/constants/trigger.py`, not in graphon.

## (5) Human-input assignee

There is no assignee or approver model; whoever holds the form token can respond.

- **graphon:** `HumanInputNodeData` has only `type`. The node delegates to a host callback returning `PauseRequested`, `Completed(selected_handle, inputs, outputs)` or `Expired`.
- **Dify 1.13.0 and `main`:** node data is `form_content`, `inputs`, `user_actions` (free-form buttons), `timeout`, and `delivery_methods`.
- **Recipients are delivery targets only.** `POST /api/form/human_input/<form_token>` submits with no login and IP rate limiting only; `submit_form_by_token` checks just that the form is active.

## (6) LLM node memory

- **Config** (`MemoryConfig`, identical at 1.9.0 and graphon `main`): `role_prefix: {user, assistant} | null`, `window: {enabled: bool, size: int | null}`, `query_prompt_template: str | null`; the whole `memory` may be null.
- **Runtime source:** there is no message service, just direct queries.
  1. Read `sys.conversation_id` from the variable pool; absent (workflow apps) means no memory.
  2. Load the `conversations` row by app and id.
  3. Query `messages` by `conversation_id`, newest first: `query` becomes the user turn, `answer` the assistant turn. Files come from `message_files`.
- **Limits:** `window.size` applies only when `window.enabled`, capped at 500 messages and 2000 tokens.
- **Where it lives:** `api/core/workflow/nodes/llm/llm_utils.py` (`fetch_memory`) and `api/core/memory/token_buffer_memory.py` at 1.9.0; `api/core/workflow/node_factory.py` on `main`.

## UNVERIFIED

- **Bump reasons for 0.1.1, 0.1.3 and 0.1.5:** chore PRs with boilerplate bodies; associations are from timing and release notes.
- **Legacy `output_type: array`:** whether current Dify still handles it (no special case found in 1.9.0's aggregator node).
- **Legacy if-else edges:** the exact `sourceHandle` to `case_id` mapping.
- **Node-type validation on `main`:** confirmed absent at import only for 1.9.0 (`sync_draft_workflow` validates `features` only).
- **"Output" as a UI label** for the End node.
- **Export format before 0.6.15**, and the distribution of DSL versions in the wild (GitHub code search gave unusable counts).