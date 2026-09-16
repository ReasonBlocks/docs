---
name: reasonblocks-setup
description: "Connect or migrate a Python agent to ReasonBlocks dashboard capture, preserve task boundaries, verify the local setup, and guide the improvement cycle: assess data, recommend training, review training progress, test candidate models and recommend adoption or further collection. Use when integrating ReasonBlocks or working on a ReasonBlocks model's lifecycle."
metadata:
  compatibility: Client setup requires Python 3.10 or newer, the published rbtrace 1.1.1 and an existing OpenAI or Anthropic Python SDK. Lifecycle reviews can use existing dashboard reports without the setup CLI.
---

# ReasonBlocks setup

Guide the customer through the next useful step, using evidence from their project
and ReasonBlocks. Connect the customer's existing Python application to dashboard
capture: inspect its client construction and task boundaries, make the integration,
and verify what works locally. The coding agent coordinates integration,
recommendations and tests; the customer can run or approve training and releases
in the dashboard. This skill covers an active working session. It does not schedule
background checks or imply that the agent continues monitoring after the session ends.

Read the [setup guide](https://docs.reasonblocks.com/agent-setup.md) when installing
or migrating the client. Install the published `rbtrace==1.1.1` in the application's
environment and verify its commands there. Do not claim a command is available
until it runs in that environment.

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

Do not invent a source URL, treat `rb_live_` organization/API keys as capture keys or
silently switch provider/API to fit this list. Responses, Gemini, Bedrock,
Fireworks and arbitrary upstreams are outside this dashboard setup.

Account sign-in and source/key creation currently happen in the dashboard.
Capture keys expire after seven days; rotation invalidates the previous key.
Connecting and ordinary capture require no sandbox or environment snapshot.

## Install and initialize

Use the project's package manager and record `rbtrace==1.1.1` in its dependency
manifest. Install it into the application environment; a global
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
See [helper placement](https://docs.reasonblocks.com/agent-setup.md#choose-an-importable-helper-location).

Read the generated helper. Supply the capture key at runtime through
`REASONBLOCKS_CAPTURE_KEY`; the existing `RB_CAPTURE_KEY` alias is accepted, and
both must agree when set. Keep the provider credential in its existing secret
environment. Do not write secrets into CLI arguments, generated files or reports.

## Wire the application

Construct the existing `OpenAI`, `AsyncOpenAI`, `Anthropic` or `AsyncAnthropic`
client with `**reasonblocks_setup.client_kwargs()`. Merge other settings and
headers deliberately. These kwargs supply the exact source URL, capture auth
and `max_retries=0`. A timeout can leave paid work with an unknown outcome;
reconcile it before retrying. Preserve an intentional application retry policy
by explicitly overriding the returned kwargs, rather than adding a retry loop.

Importing the generated helper installs labelling on supported HTTP transports
and adds the configured capture host to the allowlist. `RBTRACE_HOSTS` entries
without a leading dot match exact hosts; `.example.com` matches subdomains.
Every model call the provider SDK makes then carries `x-rb-run` and `x-rb-seq`.
Do not add a manual `rbtrace.client.install()` call. The dashboard groups a task's
calls by `x-rb-run`, and one `x-rb-run` is one training task; it does not read
`x-rb-seq`, which self-hosted gateway reconstruction uses. The labels help group
and order requests. They do not establish complete capture or a replayable
environment. `RBTRACE_DISABLE=1` disables labelling at startup; `doctor` reports
that condition.

Decide the task boundary by process shape, as described in
[Integrating your agent](https://docs.reasonblocks.com/client-integration.md#run-identity-and-sequence-numbers):

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
and actual tool results; the application still executes tools. An existing explicit
`rbtrace.client.run()` scope around each complete task may be retained instead of
`run_headers()`. Without headers or scopes, unscoped calls share one process-wide run.

Snapshots are optional for basic capture. If the application already has an
actual repeatable starting snapshot, use
`run_headers(snapshot_id=starting_snapshot_id)` and retain it throughout the task.
Do not invent a snapshot or label arbitrary state as resettable.

## Migrate an older generated connection

For a generated 1.1.0 connection, preserve its existing source URL and key: pass
the same URL to `--capture-url` and leave the capture key variable unchanged.
For an older gateway setup, obtain a dashboard source URL and key first. Then:

```bash
python -m rbtrace migrate --capture-url "$REASONBLOCKS_CAPTURE_URL" --provider anthropic --path . --json
```

Use the old helper directory for `--path`, `openai` for an OpenAI Chat Completions
source, and `--dry-run` to preview. Migration backs up recognized generated files
under `.reasonblocks/backups/` and refuses customized helpers. Inspect and merge
those customizations instead of discarding them. It does not edit arbitrary
application code. Exact generated 1.1.0 files are recognized and backed up before
replacement; repeating a completed migration changes nothing.

Use current client construction and remove obsolete
`reasonblocks_setup.install()` calls; the generated helper installs labelling on
import, and a separate `rbtrace.client.install()` call is no longer needed (a
second call is a harmless no-op). Keep explicit `rbtrace.client.run()` task scopes
or pass fresh per-task `run_headers()` on every call; a process that runs many
tasks needs one of them at every task boundary.
Remove the old gateway base-URL setting from deployment configuration, then
restart/reconstruct affected clients and processes. A config-file rewrite alone
does not change an already-constructed client or its capture authentication.
Do not turn unsupported providers or Responses calls into Anthropic/Chat
Completions as an incidental migration. Report the compatibility gap. Details:
[Migrate an older generated connection](https://docs.reasonblocks.com/agent-setup.md#migrate-an-older-generated-connection).

## Verify and report

Run `python -m rbtrace doctor --path . --json` in the actual application
environment, using the same helper path as initialization. For uv, use
`uv run python -m rbtrace doctor ...`. It checks local configuration, the Python
version, SDK availability, whether the selected SDK transport can be labelled,
capture-key presence and the task-header contract. It makes no network calls and
does not verify provider credentials, remote key validity or delivered captures.

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
