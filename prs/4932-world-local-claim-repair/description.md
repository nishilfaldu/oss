<!-- Title: [world-local] Finish step and wait writes that a crash cut short -->
<!-- Replace each UPLOAD src with the GitHub attachment URL after you drag the PNG into the editor. -->

### Description

The Local World records a step or wait change in three writes: a claim file in `.locks/` (only one writer can create it), then the step or wait file, then the event. Replay reads only the events. If the process dies between the claim and the event, every later write finds the claim and refuses, so nothing writes the event and the run stays `running` forever.

<img width="2176" height="784" alt="world-local-claim-gap" src="https://github.com/user-attachments/assets/569cc236-bb24-4773-b920-284dddc63d9a" />

After a restart:

- **`step_created` missing** (lazy `step_started`): every try fails with `Step "…" already created`, and the run replays 140–450 times a second.
- **`step_completed` missing**: every try fails with `Cannot modify step in terminal state`, and the run goes quiet. If the step file was still `running`, the step body runs again on every restart and its result is thrown away.
- **`wait_completed` missing**: every try fails with `Wait "…" already completed`, and the run replays once a second.

This PR lets the next write finish a change whose writer is gone.

<img width="2176" height="516" alt="world-local-claim-gap-rule" src="https://github.com/user-attachments/assets/447ddf95-be64-4da7-bb42-d290713c3be7" />

**Changes**

When a write finds the claim already taken, it now asks one question: did the writer die before it wrote the event?

- **The claim says who wrote it.** Today the claim file is empty. Now it holds the writer's process id and host name, plus a random id for the process.
- **A dead writer's claim is taken over.** The next write takes the claim when the event is not in the log and the writer is dead: its process no longer runs, or the claim is more than 30 s old. The 30 s rule is for writers that can't be checked, such as one on another machine. This World already treats its lock files as stale after 30 s.
- **The new owner finishes the job.** It writes the step or wait file and the missing event. If two writes try at the same time, only one wins.
- **A finished step keeps its result.** If the step file already says `completed` or `failed`, the missing event is built from that file.

`step_created` and `wait_created` have the same gap and get the same fix. The logic is in the new `src/storage/claim-owner.ts`.

**Results** from a crash test that SIGKILLs a `nitro dev` server at each of these writes. It uses `workflow` 5.2.2 with this branch's `world-local` built in:

| Kill point | Runs | `main` | This PR |
| --- | ---: | --- | --- |
| After the `.terminal` claim, before `step_completed` | 6 | 6 stuck, also after a 2nd restart | 6 complete in 28.6–29.8 s* |
| After the `.completed` claim, before `wait_completed` | 6 | 6 stuck, also after a 2nd restart | 6 complete in 0.5–1.0 s |
| After the event (control) | 6 | 6 complete in 0.5–0.7 s | same |
| Saved data of 3 runs stuck after `step_created` was lost (13 of 124 random kills got stuck) | 3 | 3 stuck | 3 complete in 30–33 s** |

\* The test sets a 30 s inline ownership lease, and the step is redelivered when it ends. The takeover itself is immediate.
\** Empty claims from 5.1.1, which wait for the 30 s age rule.

**Related:** #3081 (same bug on Postgres), #3575, #4882 and #4883 (the Postgres fixes), #4922 and #4911 (`run_created` on the Local World), #4908 (names the in-band lock holder the same way), #4906 (World start at boot, needed for Express and Nitro to recover runs at all).

### How did you test your changes?

New `src/storage/claim-repair.test.ts`, 21 tests, 17 of which fail on `main`. Each crash is a real SIGKILL: `claim-repair.child.mjs` opens its own storage on the test's directory and kills itself right after the claim or right before the event.

- Lazy `step_started`, `step_created`, `wait_created` and `wait_completed`, killed at both points: the next write finishes it, with exactly one event. A later duplicate is still refused.
- `step_completed` killed after the claim: the next one lands its own result. `step_completed` and `step_failed` killed after the step file: a redelivered `step_started` is refused, and the terminal event now holds the saved result or error.
- Two storage instances finding the same orphaned claim: one event.
- A claim naming a live process, another host, or no one: refused until 30 s old. A claim whose event exists: refused, even with a dead writer.

### PR Checklist - Required to merge

- [x] 📦 `pnpm changeset` was run to create a changelog for this PR (`patch` for `@workflow/world-local`)
- [x] 🔒 DCO sign-off passes (run `git commit --signoff` on your commits)
- [ ] 📝 Ping `@vercel/workflow` in a comment once the PR is ready, and the above checklist is complete


