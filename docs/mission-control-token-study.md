# Making Mission Control bringup use fewer tokens

*Study run on November 13, 2026 · 11 evaluations · GPT-6 Sol, medium reasoning*

The selected bringup skill kept startup polling inside a tool execution. In the
final comparison, it used **34.1% fewer total tokens** and **26.5% fewer uncached input tokens** than the original.
Both runs finished in about 173 seconds.

All 11 evaluations passed the same readiness checks: reward 1, an HTTP 200 health
response, and all seven required services running. This was an exploratory study
of one task, with two runs each for the original and selected skills. The results
support further testing; they do not establish a general improvement across tasks.

- [Browse the experiment and traces in Helix Studio][experiment] (NVIDIA access required).
- [Read the selected skill change][winning-diff].
- [Use the selected branch][winning-branch]: `token-study/orchestrated-startup`,
  commit `8c31de7d4fec7bd87e2709173c4d263d0ce1fa14`.

## The question

Can we use `nemo evals agent-evaluate` to improve an agent skill through a repeatable
loop: run a task, inspect the trace, change the skill, and measure the result?

The task was deliberately simple:

> Bring up the Isaac cloud services stack.

We reproduced the [reference evaluation][reference] as `base`, then tested eight
skill variants. We finished by repeating the best observed variant as
`optimized-repeat` and running the original skill again as `control`.

Every run checked out [this fork][fork]. Only the skill commit changed between
treatments; the prompt, model, reasoning effort, runtime image, resources, time
budgets, startup scripts, and scorer stayed fixed. Evaluation names described the
idea being tested.

## What worked: let the tool wait

The base trace showed that the agent repeatedly returned to the model while Docker
pulled images and started containers. Each return replayed the growing conversation,
including the skill and previous tool output. Input tokens dominated usage; the
final answer contributed little.

The selected change adds a short section to the existing skill. It tells the agent
to start the runner once, wait for that session inside an awaited loop, and return
the exit status and final output tail when it finishes. It also explains how to
coordinate the outer execution wait with the inner session waits.

```text
Repeated model polling:
  model → start command → model → poll session → model → poll session → ...

Tool-managed waiting:
  model → start command and await session in a loop → model → check readiness
```

Both runs of this variant followed that pattern. Some outer execution waits still
returned to the model, but fewer model turns were needed overall. The normal
preflight, service checks, and health checks remained in place.

The final pair gives the more conservative comparison:

| Metric | Original skill (`control`) | Selected skill (`optimized-repeat`) | Change |
| --- | ---: | ---: | ---: |
| Total input + output tokens | 181,745 | 119,776 | 34.1% fewer |
| Uncached input tokens | 23,821 | 17,509 | 26.5% fewer |
| Model-facing tool calls | 9 | 7 | 2 fewer |
| Agent elapsed time | 172.3 s | 173.4 s | Essentially unchanged |

The first selected-skill run used 105,403 total tokens, compared with 321,374 for
`base`. That larger difference includes extra Dispatch recovery in the base run,
so it is less useful for estimating the skill change's effect. Averaging both runs
of each skill gives a 55.2% total-token reduction, but the fresh comparison above
is the better headline.

## What the other attempts taught us

**A shorter introduction did not guarantee less reading.** The first compact skill
linked to a detailed reference, which the agent immediately opened in full. A later,
self-contained short path avoided that reference read and used 167,146 tokens, but
still needed repeated model-facing startup polls.

**Quiet logs alone did not reduce model turns.** Redirecting startup output led the
agent to inspect log tails and the startup script while waiting. Those extra calls
offset the smaller output. We dropped this as a standalone approach.

**More recovery advice could mean more investigation.** A detailed MQTT startup
recovery recipe prompted listener, DNS, and configuration checks. At 419,217 tokens,
this was the most expensive trial. We dropped the verbose recipe.

**Combining ideas did not reliably combine their benefits.** The compact skill with
waiting guidance used 373,651 tokens and 23 tool calls. Its trace showed 1-second
outer execution waits around 30-second inner waits, followed by repeated 10-second
cell waits. The agent did not follow the intended orchestration pattern. This trial
was fast in elapsed time, but expensive in repeated input context.

These traces helped explain the measurements. A shorter document, a quieter tool,
or a faster run was not enough by itself to identify a token-efficient skill.

## All results

All rows passed readiness checks. Links open the individual evaluations in Helix.
Tool calls here mean calls made by the model, not every underlying shell operation
or session poll inside a tool loop.

| Evaluation | Total tokens | Uncached input | Tool calls | Agent time |
| --- | ---: | ---: | ---: | ---: |
| [base][run-base] | 321,374 | 26,519 | 15 | 229.0 s |
| [shorter-skill-intro][run-shorter] | 290,063 | 25,724 | 13 | 220.3 s |
| [patient-startup-waits][run-patient] | 201,718 | 20,575 | 11 | 199.5 s |
| [quiet-startup-output][run-quiet] | 319,070 | 20,478 | 18 | 192.7 s |
| [mqtt-startup-recovery][run-mqtt] | 419,217 | 31,293 | 17 | 272.0 s |
| [batch-preflight-start][run-batch] | 240,394 | 24,238 | 11 | 206.1 s |
| [orchestrated-startup][run-orchestrated] | 105,403 | 17,297 | 6 | 188.9 s |
| [complete-short-path][run-complete] | 167,146 | 20,834 | 9 | 190.7 s |
| [short-orchestrated-startup][run-combined] | 373,651 | 19,501 | 23 | 170.8 s |
| [optimized-repeat][run-repeat] | 119,776 | 17,509 | 7 | 173.4 s |
| [control][run-control] | 181,745 | 23,821 | 9 | 172.3 s |

## How the study used agent-evaluate

For each variant, we validated the native job spec, checked admission, and submitted
an agent-evaluate job to the `isaac` workspace under the
`mission-control-bringup-minimize-tokens` experiment. Up to two jobs ran concurrently.
The installed CLI exposed evaluation-name prefixes rather than exact names, so the
submissions used the same native agent-evaluate endpoint with an explicit name.

Each hypothesis lived on a separate fork branch. `GIT_REPO_REF_0` pinned the full
commit SHA, making the evaluated skill revision unambiguous. The original skill was
pinned to `73c10389ca964f9346b44e6aae82e4482c669f4f`. Variant branches use the
`token-study/<evaluation-name>` naming convention; `optimized-repeat` reused the
exact `orchestrated-startup` commit, and `control` reused the original commit.

The native jobs retained model traces, tool calls, scores, source revisions, and
container evidence. We reconciled captured token usage with job accounting and
checked that the retained source SHA matched the requested fork commit. Submitted
specs matched after normalizing only that SHA, the evaluation name, and the variant
label. Skill variants changed Markdown only.

For reproducibility, the model was GPT-6 Sol with medium reasoning through the
Codex adapter. The reference runtime was held at
`nvcr.io/nvidian/osmo/evaluation-runtime:ecolter-test-ship-208`, pinned to digest
`sha256:f164342f567dea1afd7bdea28201db8577ce4a8431097fe742ceccf43e706c4f`.
The runtime, verifier, and startup scripts were not tuned during this study.

## How to interpret the numbers

- **Total tokens** are input plus output across model requests. Cached input is
  included. Reasoning tokens are already part of output and are not added again.
- **Uncached input** is input minus cached input. Neither token measure is a dollar
  cost estimate; provider billing and cache pricing were not established.
- **Agent time** runs from the first captured model request to the last response.
  It includes intervening tools and waits, but excludes allocation, bootstrap, and
  result publication. Whole-job duration was longer.
- **Readiness** required HTTP 200 and seven running services: `mission-control`,
  `mission-database`, `mission-dispatch`, `wpg`, `cuopt`, `mosquitto`, and `postgres`.
  It does not prove that a robot mission completed or that services stayed stable.

Most variants had one trial, selected adaptively after earlier traces. Startup
races, cache state, and shared infrastructure varied. Even two trials of the
selected and original skills are too few to establish statistical reliability.
No evaluated output was repaired, and failed optimization ideas remain in the
results rather than being excluded.

## What to try next

Keep the orchestration change as the leading candidate. A stronger follow-up would
randomize several original/modified run pairs and record whether startup recovery
was needed. Compare total tokens, uncached input, elapsed time, and readiness
together. Then test the skill on other requested modes and existing-stack cases.

For this task, the most useful optimization target was the number of times the
model had to resume work during startup. The native trace-to-branch-to-evaluation
loop made that behavior visible and let us test changes without altering the task.

[experiment]: https://helix.osmo.nvidia.com/studio/workspaces/isaac/experiment/mission-control-bringup-minimize-tokens
[reference]: https://helix.osmo.nvidia.com/studio/workspaces/demo/experiment/mission-control-setup-model-sweep-2/mc-sweep2-gpt-sol-me-zg9a4zjx
[fork]: https://github.com/ecolternv/mission-control-cloud-evals
[winning-branch]: https://github.com/ecolternv/mission-control-cloud-evals/tree/token-study/orchestrated-startup
[winning-diff]: https://github.com/ecolternv/mission-control-cloud-evals/compare/73c10389ca964f9346b44e6aae82e4482c669f4f...8c31de7d4fec7bd87e2709173c4d263d0ce1fa14
[run-base]: https://helix.osmo.nvidia.com/studio/workspaces/isaac/experiment/mission-control-bringup-minimize-tokens/base
[run-shorter]: https://helix.osmo.nvidia.com/studio/workspaces/isaac/experiment/mission-control-bringup-minimize-tokens/shorter-skill-intro
[run-patient]: https://helix.osmo.nvidia.com/studio/workspaces/isaac/experiment/mission-control-bringup-minimize-tokens/patient-startup-waits
[run-quiet]: https://helix.osmo.nvidia.com/studio/workspaces/isaac/experiment/mission-control-bringup-minimize-tokens/quiet-startup-output
[run-mqtt]: https://helix.osmo.nvidia.com/studio/workspaces/isaac/experiment/mission-control-bringup-minimize-tokens/mqtt-startup-recovery
[run-batch]: https://helix.osmo.nvidia.com/studio/workspaces/isaac/experiment/mission-control-bringup-minimize-tokens/batch-preflight-start
[run-orchestrated]: https://helix.osmo.nvidia.com/studio/workspaces/isaac/experiment/mission-control-bringup-minimize-tokens/orchestrated-startup
[run-complete]: https://helix.osmo.nvidia.com/studio/workspaces/isaac/experiment/mission-control-bringup-minimize-tokens/complete-short-path
[run-combined]: https://helix.osmo.nvidia.com/studio/workspaces/isaac/experiment/mission-control-bringup-minimize-tokens/short-orchestrated-startup
[run-repeat]: https://helix.osmo.nvidia.com/studio/workspaces/isaac/experiment/mission-control-bringup-minimize-tokens/optimized-repeat
[run-control]: https://helix.osmo.nvidia.com/studio/workspaces/isaac/experiment/mission-control-bringup-minimize-tokens/control
