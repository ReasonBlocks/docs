---
name: reasonblocks-setup
description: "Connect a Python agent to ReasonBlocks and guide its improvement cycle: assess data, recommend training, review training progress, test candidate models and recommend adoption or further collection. Use when integrating ReasonBlocks or working on a ReasonBlocks model's lifecycle."
metadata:
  compatibility: Client setup requires Python 3.10 or newer and rbtrace 1.1.0. Lifecycle reviews can use existing dashboard reports without the setup CLI.
---

# ReasonBlocks setup

Guide the customer through the next useful step, using evidence from their project
and ReasonBlocks. The coding agent coordinates integration, recommendations and
tests; the customer can run or approve training and releases in the dashboard.
This skill covers an active working session. It does not schedule background
checks or imply that the agent continues monitoring after the session ends.

Read the [setup guide](https://docs.reasonblocks.com/agent-setup.md) when installing
or migrating the client. Its CLI commands are a release preview requiring
`rbtrace==1.1.0`; public PyPI currently has `1.0.0`, which lacks those commands.
Use a preview build only when the user supplies its trusted source or artifact.
Otherwise explain the installation prerequisite and continue independent work.
Lifecycle reviews can use existing dashboard reports without the new CLI. Do not
claim unavailable commands are installed.

ReasonBlocks provides a CLI, a small Python integration helper and this skill.
The existing OpenAI or Anthropic Python client library—the provider SDK—continues
making model calls. Preserve the application's framework and client settings.

## Inspect and obtain the connection

Inspect the dependency manager, real client constructors, model-call paths,
logical task boundaries, launch command and deployment layout. A Python wrapper
may launch JavaScript or native code that actually makes the requests; a Python
helper does not instrument that child process. Integrate at the real boundary
and report unsupported paths.

Use the exact HTTPS source URL and key issued in **Data**. Current support is:

- OpenAI Chat Completions: `/capture/SOURCE_ID/openai/v1`, provider `openai`.
- Anthropic Messages: `/capture/SOURCE_ID/anthropic`, provider `anthropic`.

Do not invent a source URL, treat `rb_live_` telemetry keys as capture keys or
silently switch provider/API to fit this list. Responses, Gemini, Bedrock,
Fireworks and arbitrary upstreams are outside this dashboard setup.

Account sign-in and source/key creation currently happen in the dashboard.
Capture keys expire after seven days; rotation invalidates the previous key.
Connecting and ordinary capture require no sandbox or environment snapshot.

## Install and initialize

Use the project's package manager and record `rbtrace==1.1.0` in its dependency
manifest when available. Install it into the application environment; a global
or `uvx` tool environment alone is insufficient.

```bash
python -m rbtrace init --capture-url "$REASONBLOCKS_CAPTURE_URL" --provider anthropic --path . --json
```

Use `--provider openai` for an OpenAI source. `--dry-run` previews generated
files. The `rbtrace` entrypoint accepts the same arguments. Inspect existing
files before resolving conflicts; do not overwrite a customized helper.

Choose an importable helper location. `--path .` fits flat projects. An installed
`src/my_agent` package can use `--path src/my_agent` and
`from my_agent import reasonblocks_setup` or a package-relative import. A directly
launched script may see a helper beside it but not one at the repository root.
Do not add a `sys.path` workaround. Package the helper and its adjacent
`.reasonblocks/config.json` into the actual image or wheel package data; hidden
directories may otherwise be omitted. `.reasonblocks/SETUP.md` is optional at runtime.

Read the generated helper. Supply the capture key at runtime through
`REASONBLOCKS_CAPTURE_KEY`; the existing `RB_CAPTURE_KEY` alias is accepted, and
both must agree when set. Keep the provider credential in its existing secret
environment. Do not write secrets into CLI arguments, generated files or reports.

## Wire the application

Construct the existing `OpenAI`, `AsyncOpenAI`, `Anthropic` or `AsyncAnthropic`
client with `**reasonblocks_setup.client_kwargs()`. Merge other settings and
headers deliberately. These kwargs supply the exact source URL and capture auth.

Importing `reasonblocks_setup` performs `rbtrace.client.install()` itself, so every
model call the provider SDK makes carries `x-rb-run` and `x-rb-seq`. Do not add a
manual `install()` call. The dashboard groups a task's calls by `x-rb-run`, and
one `x-rb-run` is one training task.

Decide by process shape:

- One task per process: nothing more to do; the helper labels every call with one
  run ID.
- Any long-lived or multi-task process (queue worker, web server, batch loop,
  thread pool): call `reasonblocks_setup.run_headers()` once at each logical task
  boundary and pass the result as `extra_headers=headers` on every model call in
  that task. This is required; without it every task collapses into one
  process-wide `x-rb-run`.

Keep headers in each concurrent job's own state. Use `run_id=existing_task_id`
when a suitable task ID already exists. Never create a new ID per model step or
share one across unrelated tasks. Retain full messages, policy, tool definitions
and actual tool results; the application still executes tools.

Snapshots are optional for basic capture. If the application already has an
actual repeatable starting snapshot, use
`run_headers(snapshot_id=starting_snapshot_id)` and retain it throughout the task.
Do not invent a snapshot or label arbitrary state as resettable.

## Migrate an older generated connection

Obtain a real dashboard source URL and key first, then:

```bash
python -m rbtrace migrate --capture-url "$REASONBLOCKS_CAPTURE_URL" --provider anthropic --path . --json
```

Use the old helper directory for `--path`, `openai` for an OpenAI Chat Completions
source, and `--dry-run` to preview. Migration backs up recognized generated files
and refuses customized helpers. Inspect and merge those customizations instead
of discarding them. It does not edit arbitrary application code.

Update application code to construct the client with `client_kwargs()`. The
generated helper performs `install()` on import, so remove any manual
`rbtrace.client.install()` call (a second call is a harmless no-op). An existing
`shim.run()` / `shim.new_run()` scope can stay or become `run_headers()`; both
name a task, and a process that runs many tasks needs one of them at every task
boundary. Remove the old gateway base-URL setting from deployment configuration, then
restart/reconstruct affected clients and processes. A config-file rewrite alone
does not change an already-constructed client or its capture authentication.
Do not turn unsupported providers or Responses calls into Anthropic/Chat
Completions as an incidental migration. Report the compatibility gap.

## Verify and report

Run `python -m rbtrace doctor --path . --json` in the actual application
environment, using the same helper path as initialization. For uv, use
`uv run python -m rbtrace doctor ...`. It checks local configuration, SDK
availability and capture-key presence; it makes no network calls and does not
verify provider credentials, remote key validity or delivered captures.

Inspect the application diff, run relevant existing local tests, and test the
actual launch from outside the repository directory to verify imports and
config packaging. Live model calls send application content and incur provider
charges; run one only within the user's authorized scope. Confirm source records
in **Data** before claiming end-to-end success.

Report changed files, selected source/provider, task-boundary placement, local
checks and any missing inputs. After setup, recommend the next step from the
workflow below. A request to install is not itself authorization to spend on
training or change production traffic.

## Guide the improvement cycle

Read the [agent workflow](https://docs.reasonblocks.com/agent-workflow.md) when
assessing readiness, following training, testing a candidate or deciding what
to do next. Reuse the customer's current project, source and training run rather
than creating a new run every time they ask for progress.

### Get current evidence

Use the customer's authenticated dashboard or documented control API when
available. Identify the source, pipeline/run, model release and report version
being assessed. The capture key only authenticates capture; it does not grant
permission to read dashboard data or control training. Do not extract browser
session tokens or substitute provider/capture keys for control-plane access.

If current status is unavailable, ask for the relevant dashboard status or report
and continue independent project work such as preparing tests. Distinguish a
recommendation based on inspected data from one awaiting evidence. Do not infer
success from a submitted command, an elapsed time or a raw request count.

### Collect and prepare

- After a verified connection, help the customer collect representative tasks,
  including expected tool paths and known failure cases. Prefer independent task
  groups to many requests from one long conversation.
- Inspect data coverage, capture gaps and available outcome checks. Set aside
  held-out task groups before training; repeated or related attempts must remain
  in the same split. Do not tune on the final test set.
- Recommend **Prepare training** when there is a plausible representative sample
  and replayable tasks. The current recipe's preparation checks decide whether
  there are enough usable training groups after separate calibration and
  evaluation groups are reserved. Do not invent a universal training threshold.
- For full-agent training, check that the customer's resettable tool environment
  is connected. If it is missing, propose the concrete adapter/test-data work
  needed next. Ordinary capture can continue while that work is done.

### Recommend training and follow the run

Use the prepared checks and estimate to explain what is ready, what remains
missing, what training should improve, and the proposed spending cap. Recommend
training when that evidence supports it. The user can approve and start the run
in **Training**, or authorize the coding agent through supported authenticated
tools. Reuse existing authorization and its limits; if approval or a budget is
missing, present the concrete plan before starting paid work.

Read the actual run state on a follow-up. Distinguish preparation, waiting for
review, training/evaluation, budget pause, failure and a completed report. Do not
restart a paused/failed run or submit another paid job just to check progress.
Explain a blocker with the reported reason and the next recoverable action.

### Test the candidate and recommend the next step

When a candidate report is available, inspect its model/release identity,
evaluation split, outcome criteria and missing measurements. A finished training
job is not evidence that the candidate is better. Review the platform's baseline
and candidate task results first.

Help run the customer's held-out integration tests in their own environment
using a supported candidate access path. Keep task inputs, starting state, tool
behavior and success criteria comparable to the current model. Preserve the
distinction between live application tests and the platform's existing evaluation
report. If the current deployment provides no candidate test endpoint, report
that gap; do not enable production traffic or use a different API as a shortcut.

For the current `full_agent_opd` mode, the built-in sandbox evaluation is available
in the results report. `/stage` and the pipeline `/chat/completions` endpoint do
not provide private testing for this mode. Additional live tests require an
approved canary through the capture source, isolated to test traffic. A canary on
a production-fed source can affect production tasks. Check the documented serving
contract before running it, and verify the response release/routing metadata so
original-provider forwarding is not mistaken for a candidate test. Keep the
application's original model identifier as the serving contract requires.

Compare whole-task success, cost and latency against the current model and the
customer's acceptance criteria. Retain unknown/incomplete results as unknown.
Recommend adoption only when the evidence meets those criteria; otherwise point
to specific regressions, missing task coverage or additional evaluation needed.
Do not recommend retraining automatically after every failure.

Before adoption, describe the exact reviewed release, traffic change and rollback
option. Use the customer's existing release authorization or obtain it for that
concrete change. In later active sessions, use observed failures and distribution
changes to recommend further collection, testing or retraining.

End lifecycle updates with one next recommendation, its supporting evidence,
and any action the customer needs to take. Do not claim to have run training,
tests or deployment when only recommending them.

## Training context, only when relevant

Full-agent training requires a sandbox: a test copy of real tools and data that
resets per task. This can be an existing test environment and does not necessarily
mean a customer-hosted server. An adapter ties together snapshot/reset, tool
execution and an independent outcome evaluator. Help implement it when requested,
using the customer's domain contracts, test access and success criteria.

Capture does not create database snapshots. Earlier captures without a restorable
starting state are not automatically trainable; collect tasks with real snapshots
or curate replayable starting tasks for the environment. Link to
[full-agent training](https://docs.reasonblocks.com/full-agent-training.md) for those
requirements. The agent can guide these steps while the environment and its
business tools stay under the customer's control. An API key alone does not
implement the environment adapter or change the backend's execution contract.
