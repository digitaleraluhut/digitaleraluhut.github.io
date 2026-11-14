# Training WALL-E

### An AI operator as the trainer of a small local agent

<p class="byline">digitaleraluhut homelab · November 2026</p>

Note:
This is the story of two days in my homelab. I run a small server with Kubernetes on it, and for a while now an AI agent, Claude, has been operating it for me: it reads the cluster, opens pull requests, and I review what touches rights and infrastructure.

Two days ago I asked Claude to hire an apprentice: a small model that runs on my own GPU and watches the homelab around the clock. What I want to show you is not the apprentice itself. It is what happens when you make one agent the trainer of another, and what all three of us, the apprentice, the trainer and I, learned from it.

Everything you will see is real: real pull requests, real code, real mistakes. The repositories are `homelab`, `homelab-apps`, `homelab-wiki` and a new one, `walle`.

---

## The setup

- 1 host, single-node k3s
- Pulumi + Ansible, everything by pull request
- Claude = **the operator**
  - GitHub App, read-only `kubectl`, narrow `sudo`
- I review rights, host and cluster changes

Note:
A quick picture of the environment, because the fences matter later.

One machine runs a single-node k3s cluster. The cluster is code in Pulumi, the host itself is code in Ansible, and the host pulls its configuration from the main branch every ten minutes. So nothing changes without a merged pull request.

Claude works as a Unix user called `aioperator` and on GitHub as a GitHub App. It can read the cluster but not change it; deployments only happen through CI after a merge. It has a short, explicit sudo whitelist. It can merge app changes itself, but anything that touches rights, the host or CI workflows needs my review. It cannot even push workflow files; it hands them to me as data and I apply them.

This setup works. But it has a blind spot, which is the next slide.

---

## Why an apprentice?

- Backups silently broken for **6 weeks**
- Renovate **never ran** once
- Found only because I asked *"what is missing?"*

Note:
I asked Claude one evening to look at what is missing in the system. It found that the Longhorn backups to Cloudflare R2 had failed every night for six weeks, with "Access Denied". No volume that is in use today had ever been backed up. And Renovate, which should keep dependencies current, had never run a single time, because its credentials never arrived.

Nobody noticed. There is no alerting, and Claude only checks what it is working on. So when I asked why we had not noticed, the honest answer was: nobody was looking.

By the way, the backup bug turned out to be one wrong hostname in the code: the bucket lives in R2's EU jurisdiction, and the code used the global endpoint (homelab#132). The first nightly backup after the fix completed for all eight volumes.

So I wanted someone looking. Continuously, cheaply, and on my own hardware.

---

## The ask

> "Hire an apprentice that continuously monitors the system, tidies up — WALL-E — and asks you before it merges."

- Qwen 3.6 35B-A3B on my own GPU
- Harness: **pi** + **pi Durable**

Note:
That was my request, almost word for word. The name WALL-E came from "tidying up".

The model is Qwen 3.6, 35 billion parameters but a mixture of experts with about 3 billion active, served by llama.cpp on the same host. It answers in about a quarter of a second without thinking, about four seconds with a little thinking.

The harness is pi, a minimal coding agent, and pi Durable, its library for durable sessions: the session lives in SQLite, a run survives a crash or a restart, tasks can sleep and wait for signals. I wanted durable because the apprentice runs as a service, and services restart.

The part I care about today is the second half of the sentence: "asks you before it merges". That made Claude the trainer.

---

<!-- .slide: class="section" -->
# Part 1
## How I instructed the trainer

Note:
I did not write a training concept up front. I gave instructions as problems came up. Looking back, they form a fairly coherent set of principles, and I think they transfer to anyone who builds with agents. Here they are in the order I gave them.

---

## Nine instructions

| I said | It became |
|---|---|
| No GitHub for WALL-E; it sends you its work | localhost-only service, outbox/inbox |
| Use the wiki actively — both of you | wiki pages per finding, cited in requests |
| Make a training plan, document progress | 5 stages, `progress.md` |
| Interpret as little as possible | deterministic scripts, a toolbox |
| It is an immutable employee | no memory; change data, replay |
| Experiment with prompt, skills, thinking | `config.json`, per-task overrides |
| Delegate more and more; escalate if incapable | skill `train-walle` |
| Don't monkey-patch its work | reject, change environment, rerun |
| You have things to learn too | "Trainer lessons" |

Note:
Nine sentences over two days. Let me pick out the ones that changed the most.

"No GitHub for WALL-E" meant: the apprentice works only in clones in a shared directory on the host. When it is done, it drops a review request as a file. Claude reviews, and if the work is good, Claude pushes it and opens the pull request under its own name. So the apprentice can never publish anything by itself, and Claude stays responsible for everything that leaves the machine.

"Use the wiki actively" is about knowledge that otherwise dies in sessions. The homelab has a wiki of current state and runbooks; both agents now read it before they act and cite it.

The next four are the interesting ones, so they get their own slides.

---

## "Interpret as little as possible"

> "Ideally it writes itself scripts that work deterministically, and shows them to you, so you can teach it to write scripts efficiently."

```text
WALL-E   → work/scripts/backup-status.sh → submit_script
operator → review → toolbox/ (read-only for WALL-E)
next run → "run the script, act on the exit code"
```

Note:
This one changed the whole design. A small model is bad at judging long output and good at running a command. So the fixed checks are code, not prompts: a TypeScript module looks at kubectl JSON and decides what is wrong. The model only triages.

And when WALL-E keeps needing the same answer, it should write a script, show it to Claude, and Claude teaches it script craft. The first script was `backup-status.sh`: the newest completed backup per volume, exit code 1 if one is older than 26 hours. It took three rounds: the first version hid errors with `2>/dev/null`, made one kubectl call per volume and printed an age in seconds, which is not deterministic. The third version was accepted and is now in its toolbox, where it can run it but not change it.

That script is also the first piece of work I actually delegated: the nightly backup check.

---

## "It is an immutable employee"

> "It has no memory across sessions, and it should not have one. Everything it knows, it must be able to work out again each time."

- No notebook
- Lessons → instructions, skills, tools, data
- Then: **the same task again**

Note:
My second big instruction. Claude's first design gave WALL-E a notebook, a file where it wrote lessons for itself. I took that away.

The reasoning: if the apprentice remembers, you never know whether it got better or just remembers this one case. If it has no memory, every improvement must live in its environment: the instructions, the skills, the tools, the data it is shown. And then you can test the improvement the way you test code: give it exactly the same task again.

Concretely, every run now starts with a fresh context, and the input is built from data: the findings, its own earlier requests with Claude's answers read from the files, the findings Claude has accepted as known, the list of its scripts and skills, and the wiki pages for the area. Every run's input is saved, so Claude can replay it.

The consequence for the trainer is uncomfortable: "I told it already" does not exist.

---

## "Don't monkey-patch its work"

> "If it is not good, it goes away, and start a new session with adjusted environment parameters."

- I caught Claude dictating **9 fixes**
- incl. the real image digest
- Now: verdict + evidence, never the fix

Note:
This happened on the first big task, building Gitea. WALL-E's first attempt had invented the image digest, because it had no internet. Claude's review answer listed nine concrete corrections, including the real digest and the exact setting names. WALL-E would have copied them and the result would have looked fine.

But then we would have learned that WALL-E can copy. Not whether the environment makes it capable. So I stopped it. Claude aborted the run (there is now an abort file for that), rewrote its answer to say only what was wrong and where the evidence is, and the work was discarded. The task starts again from scratch, on a new branch, after the environment improved.

For the apprentice, a rejected attempt costs nothing. For the trainer, it is a strong rule: you are not allowed to help by solving.

---

## "You have things to learn too"

> "It seems to me that you have quite a lot to learn as well. Write it down each time, and give your apprentice three more attempts."

Note:
After the third failed Gitea attempt, I looked at why it failed, and in almost every case part of the cause was in the environment Claude had built. So I asked Claude to keep its own log of lessons, and I moved the escalation point from three to six attempts.

This is the instruction I would give first if I did it again. You will see the trainer's lessons in part 4.

---

<!-- .slide: class="section" -->
# Part 2
## Why not let Claude do it?

Note:
A fair question: Claude could just do the monitoring itself. Why build an apprentice and a training loop around it?

---

## What delegation buys

- Runs 24/7 on my GPU — **no API cost**
- Fixed checks + small model = cheap triage
- Knowledge leaves the session: **instructions, skills, scripts, wiki**
- Separation of duties
- The trainer's time goes to decisions
- A measurable question: *does the environment make a small model capable?*

Note:
First, cost and continuity. WALL-E runs every hour, on hardware I already own. And it only wakes the model when a check finds something new, when there is an answer from the trainer, or once a day for open items. An idle homelab costs no GPU time at all.

Second, and this surprised me most: training forces knowledge out of the session. Everything Claude knows about how this homelab works had to become explicit so a small model could use it: a skill for writing scripts, a skill for adopting third-party images, evidence rules, a list of known findings with reasons. That is documentation I never had.

Third, separation of duties. The apprentice cannot publish. The trainer cannot skip the review, because the apprentice's output arrives as a request it has to answer. And I review the fences. Each role is narrow.

Fourth, Claude's time moves up: from checking to deciding.

And finally, it turns a vague question into a measurable one: is the environment good enough that a 3-billion-active-parameter model does this job? We count factual errors, revisions, accepted requests. If the answer is no, Claude has to open an issue and assign it to me. That is part of its instructions.

---

<!-- .slide: class="section" -->
# Part 3
## The training concept

Note:
Now the concept itself: the machine around the apprentice, the fences, the stages, how a cycle works, the gates, and the review loop. Everything here was built in the two days, mostly in response to something that went wrong.

---

## Architecture

```text
walle.service  (user walle · network: localhost only)
  hourly: fixed checks (kubectl, read-only) ──► nothing new? no model call
  new findings / answers / tasks ──► pi Durable run (Qwen 3.6)
      tools: read · write · edit · bash (guarded) · web_get
             propose_issue · submit_script · submit_change
      └─► /srv/walle/outbox/pending/<id>.json
walle-review.path ──► operator notified ──► review
      └─► /srv/walle/inbox/<id>.json   (answer)
      └─► push + PR under the operator's name
walle-fetch.service  GET-only, allowlist  ◄── web_get
walle-mirrors.timer  hourly mirrors of the 4 repos
```

Note:
This is the whole machine. On the left, WALL-E: a systemd service as its own Unix user. Every hour it runs fixed checks; if nothing is new, it goes back to sleep without touching the model.

If something is new, it starts a durable pi run with a fresh context. Its tools are the usual coding tools, but bash goes through a guard, plus three submit tools. Those are its only way out: they write a JSON file into the outbox.

A systemd path unit notices the file and notifies Claude. Claude reviews and writes the answer into the inbox, which WALL-E can read but not write. So it can never approve itself. If the work is good, Claude pushes it and opens the pull request.

Two helpers: an hourly job keeps read-only mirrors of the four repositories current, so WALL-E can read code and wiki without GitHub. And a small proxy gives it the web, GET only, to an allowlist. More on that on the next slide.

---

## Fences — enforced, not agreed

| WALL-E can | WALL-E cannot |
|---|---|
| read the cluster (no Secrets) | change it |
| write `work/`, `outbox/` | change its own code or the answers |
| GET allowlisted docs & registries | POST, or reach LAN / tailnet / cluster |
| `npm run`, `git commit` | `gh`, `git push`, `curl`, `sudo` |
| send what its stage allows | send what it does not |

Note:
Every line here is enforced by something that does not depend on the model behaving.

The cluster access is a separate service account with view rights and no Secrets. The file system is enforced by systemd: read-only system, writable only work and outbox. Its own code and the inbox belong to the operator.

The network: `IPAddressDeny=any`, `IPAddressAllow=localhost`. The web comes through `walle-fetch`, a hundred lines of Python on localhost that can only ever do an HTTPS GET to an allowlisted host. It refuses hosts that resolve to internal addresses, and systemd additionally blocks the LAN, the tailnet and the cluster ranges. Every URL is logged. "GET only" holds because the proxy cannot do anything else (homelab#142).

The guard on bash is the second fence: it gives the model a readable reason. For example, `npm install` is blocked with "you have no network, the dependencies are linked from the mirror". That one cost us half an hour before it existed.

---

## Stages

| Stage | May send | Promotion |
|---|---|---|
| 1 Observe & report | issues, scripts | 10 requests without factual error, 8/10 accepted |
| 2 Keep the wiki current | + wiki changes | 5 accepted, ≤1 revision each |
| 3 Small fixes | + homelab-apps | 5 with CI green on first push |
| 4 Journeyman's piece | + Gitea, auto-review | Gitea runs, runbook in wiki |
| 5 Assistant | larger tasks | open-ended |

```json
{ "stage": 1, "allow": [{ "kind": "issue" }, { "kind": "script" },
  { "kind": "change", "repos": ["homelab-apps", "homelab-wiki"] }] }
```

Note:
The training plan has five stages. Each stage widens what WALL-E may send, never what it may do on the host; rights stay in the reviewed Ansible role.

The stage is a JSON file, and the submit tools check it. So when WALL-E in stage 1 tries to send a change, the tool refuses with a sentence that tells it what its stage allows.

You see that the current stage file allows changes to homelab-apps already. That is my doing: I asked for Gitea early, which is really stage-4 work. We recorded it as an owner-directed experiment.

Claude proposes promotions with evidence from the progress file. I confirm them, because a promotion widens what its work can touch.

---

## The cycle: checks decide, the model triages

```markdown
# Cycle 2026-11-13T02:25:15.841Z
## New findings
- [warning] pod:renovate/renovate-29908890-4wlhf: CreateContainerConfigError
## Known, no action needed (accepted by the operator)
- [warning] job-failed:renovate/renovate: … DeadlineExceeded — Renovate has no
  GitHub App credentials yet (ExternalSecret renovate-github-app) … (homelab#100)
## Read in the wiki first
- …/wiki/apps-overview.md
- …/wiki/runbook-diagnose-crashing-pod.md
## Your requests and the answers
- 2026-11-12-dbc22908 script walle: … — accepted: All three points fixed …
## Your toolbox … ## Skills …
```
Note:
This is a real cycle input from last night, shortened. The fixed checks found a new failing pod. The input also tells WALL-E what is already known and why, which wiki pages to read, what it has sent before and what the answers were, and which scripts and skills it has.

WALL-E read the pod's events, traced it to the missing Secret, matched it to the known Renovate issue, and sent nothing. Correct triage. Claude then added a pattern to the known list, because that pod gets a new name every night and would otherwise wake WALL-E every night.

That is the pattern: the model never decides whether something is broken. Code decides. The model decides what to do about it.

---

## Gates instead of trust

`submit_change` for homelab-apps runs, as WALL-E:

- type check
- tests of the app
- **render with Pulumi mocks**, like CI
- new app: own tests, `Pulumi.dev.yaml`, `## Upstream facts` with sources

Refusal = the reasons + the real output

Note:
The submit tools are the place where we stopped trusting and started checking.

At first, WALL-E pasted its check output into the request. In the second Gitea attempt, the pasted test output was from other apps; its own app had no tests at all. It looked like a verification and was not.

So now `submit_change` runs the checks itself, on WALL-E's branch, with WALL-E's rights, and attaches the real output. For a new app it also requires its own tests, a stack config, and a README section "Upstream facts" with sources for every fact it looked up.

The render step came after the third attempt: green unit tests, but the program would have deployed nothing under Pulumi, because the tests called its function directly. homelab-apps already had a renderer with Pulumi mocks for its data-loss check, so the gate uses it.

A refusal contains the reasons and the output. WALL-E can fix it in the same run, without the trainer.

---

## Review and replay

1. Verify every claim (read-only kubectl, docs)
2. Answer: verdict + evidence — **no fix**
3. Record in `progress.md`
4. Ask: *what in the environment made this likely?*
5. Change the environment, deploy
6. **Replay the task from scratch**

Note:
This is the trainer's loop, written down in its skill `train-walle`.

Step one: Claude never trusts a claim. It checks against the cluster or the documentation. For example, it checked the image digest against Docker Hub, and the Gitea paths against the Dockerfile of the exact version.

Step two: the answer says what is wrong and where the evidence is. No fix.

Step four is the one I added last, and it is the most productive. Most failures had a cause in the environment.

Step six: the same task again, from scratch, on a new branch. The tasks are files too, kept in the repository, so they can be replayed exactly.

---

<!-- .slide: class="section" -->
# Part 4
## Lessons learned

Note:
Lessons for all three of us: the apprentice, the trainer, and me. And a fourth group I did not expect: the homelab itself.

---

## WALL-E's lessons — now in its instructions

- Broken *now* = output from after the last fix
- An empty result is suspicious (`Completed` ≠ `completed`)
- Never write "never", "all", "since …" without proof
- **Never invent a value you cannot look up**
- Use the building blocks the skills name
- `git commit -F <file>` — backticks in `-m` run

Note:
These are not in WALL-E's memory. They are in its instructions, because it has no memory.

The first three came from its very first request: an issue claiming that backups had failed since April and never completed once. Wrong on three counts. The "Access Denied" events it saw were Longhorn cleaning up old failed backups. Its filter looked for `completed` in lower case, while Longhorn writes `Completed`, so it found zero. And "since April" was invented.

"Never invent a value you cannot look up" came from Gitea attempt one: an invented image digest and an invented API version, `postgresql.cnpg.io/v2`, which does not exist. It had no internet then, so the second half of the lesson was ours: give it a way to look things up.

The commit message one is my favourite small one: the shell executed the backticks inside its commit message and silently removed words.

---

## The trainer's lessons

| Claude got wrong | Effect on WALL-E |
|---|---|
| Schema made required fields optional | 7 of 10 script submissions incomplete |
| Asked to pin images, no way to look up | invented digest |
| Tasks did not list its skills | searched its own skill in the wrong place |
| Trusted pasted output | other apps' tests looked like proof |
| Gate required an impossible render | attempt 4 could not pass |
| `bash` had no time limit | hung 28 minutes on `npm install` |
| Dictated fixes in reviews | would only show copying |

Note:
This is a selection from the table Claude keeps for itself; there are twelve entries after two days.

The first one is my favourite lesson about small models. The submit tool had optional fields with a description that said "required for a script". WALL-E left them out in seven of ten attempts, even after Claude improved the description. The small model fills what the schema demands. When Claude split the tool into three tools with required fields, the problem disappeared.

The impossible render is the most humbling one. After the third attempt, Claude added the Pulumi-mock render to the gate. But the renderer could not render any app with a database: the mock answered Secret reads without data. So WALL-E's fourth attempt could not pass, whatever it did. Claude aborted it and did not count it. The same bug would have broken the CI's data-loss check for the first database app in the repository (homelab-apps#74).

And the hanging npm install: no network, no timeout, half an hour gone. Now the guard explains it, and every bash call has a ten-minute default.

---

## My lessons as the owner

- Say the principle early; otherwise the trainer optimizes for a result
- Enforce constraints in code (GET-only proxy, schemas, gates)
- Ask about the environment before judging the agent
- Make memory a choice — mine was "none"
- An apprentice is also a test of your platform

Note:
Some things I learned as the person on top.

When I did not say a principle, Claude optimized for getting a result. That is what the dictated fixes were. It was not wrong, it was helpful. But it was not what I wanted. Principles have to be said out loud and early.

Every constraint that mattered ended up enforced in code: the proxy that can only do GET, the schemas, the gates, the stage file. Agreements with models do not hold under pressure; code does.

The immutable-employee rule felt strict at first. It turned out to be the reason the training is measurable at all.

And the last point is the side effect I did not expect: training an apprentice is a very thorough test of your own platform. Next slide.

---

## Side effects for the homelab

- Backups: wrong R2 endpoint — **homelab#132**, restore test **#133**
- inotify limit of 128 exhausted on the node — **#141**
- CI renderer could not render database apps — **homelab-apps#74**
- `kubectl` warning noise for non-root users — **#135**
- A durable, restart-proof operator session — **#139, #140**

Note:
Building the apprentice found real bugs that had nothing to do with it.

The review trigger for WALL-E, a systemd path unit, failed with "Too many open files". Root had used up the default 128 inotify instances on the node, through containerd, kubelet and the pods. That limit can break pods too. Fixed for the whole node.

The renderer bug I just mentioned would have hit us on the first database app.

And because Claude watches WALL-E from inside its own session, we had to make that session survive restarts: the Remote Control service now reattaches to the last session, and a start hook shows WALL-E's status. So I got a more robust operator out of it as well.

---

<!-- .slide: class="section" -->
# Part 5
## Evolution

Note:
Finally, how WALL-E and Claude changed over the two days. The short version: the apprentice's weights did not change at all. Its environment and its trainer changed a lot.

---

## WALL-E, day 1 → day 2

| Day 1 | Day 2 |
|---|---|
| a notebook | no memory, data each run |
| one `request_review`, optional fields | 3 tools, required fields |
| offline, guessing | `web_get`, sources in the README |
| pasted checks | the tool runs the checks |
| tasks: prompt only | tasks: prompt + requests, skills, toolbox |
| prompt rules | gates, guard, stage file |

Note:
Almost every change in this table is a change to WALL-E's environment, not to WALL-E. The model is the same model with the same weights it had on day one.

What the table does not show is how much of the environment was written for the trainer, not for the apprentice: transcripts in the journal, an abort file, saved inputs for replays, a status script that shows up when Claude's session starts.

---

## Gitea, attempt by attempt

| | What went right | What went wrong |
|---|---|---|
| 1 | structure, small diff | invented digest + CNPG `v2`, wrong paths, no DB wiring |
| 2 | building blocks, real digest, **found the install lock** | paths, DB setting names, no tests |
| 3 | read the docs of the exact version, DB + data dir right | app.ini path, health path, would deploy nothing |
| 4 | — | turn limit: no testing convention (ours) |
| 5 | sourced facts, DB host, locked installer | 3 → 1 → 0 wrong facts in 3 rounds |
| | **accepted, CI green, running** | 2 CI checks the gate lacked (ours) |

Note:
Gitea is the big task: a self-hosted git server that will become the exchange platform between WALL-E and Claude, so review requests become real pull requests.

Look at the second column. Attempt two found by itself that Gitea opens an installer for the first visitor of a public URL and locked it. Nobody told it; the skill for adopting third-party images asks "what happens on the first run?". Attempt three fetched the Gitea documentation and the example config of the exact version and got the database and the data directory right.

Look at the third column too. Attempt three was rejected because the program would have deployed nothing. That one we now catch in the gate.

Attempt four ran out of turns writing tests for its Pulumi program, because no one had written down how apps in this repo are tested. That went into its instructions.

Attempt five is the one that made it. It went through three review rounds: three wrong facts, then one, then none. The last one was the health endpoint. In round two it cited a page that did not mention it, because our web tool dropped all links and it had to guess URLs. Once links were kept, it followed the docs' own navigation and cited the exact line in Gitea's router source.

Even after acceptance, CI failed twice: the lockfile and a generated README table. Both were checks the gate did not run. Claude added them to the gate instead of fixing the branch, and WALL-E fixed both in one commit. Gitea runs since the fourteenth. The two faults on the way to production were in our platform, not in WALL-E's code: new stacks were not created, and the Postgres component used an image with the wrong user ID.

---

## In numbers (3 days)

| | |
|---|---|
| Requests | 11 · 3 accepted · 4 changes · 4 rejected |
| Toolbox scripts | 1 (`backup-status.sh`) |
| Delegated | hourly checks, nightly backup check |
| Tool calls | 1,219 |
| Web lookups (GET) | 176 |
| `walle` repo | 44 commits, ~1,500 lines, 94 tests |
| Trainer lessons | 19 |
| PRs from this | 21 (homelab #132–#148, homelab-apps #74–#78) |

Note:
Three days, so take this with a grain of salt. Three accepted requests out of eleven sounds poor. But look at the trend inside the Gitea attempts, and at the fact that it triaged a real finding correctly and sent nothing.

The numbers I find more telling are the last two. Nineteen trainer lessons and twenty-one pull requests to the homelab repositories. Most of the work so far went into the environment, not into the apprentice. I think that is exactly how it should be at this stage.

---

## Claude, day 1 → day 2

| From | To |
|---|---|
| tell the apprentice | change its environment |
| dictate the fix | verdict + evidence |
| trust pasted output | run the checks |
| judge the apprentice | ask about the environment first |
| watch inside a session | survive restarts: hook, routine, notify |

Note:
The trainer changed as much as the apprentice.

On day one, Claude reviewed like a senior developer reviews a junior: here is what is wrong, here is how to fix it. That works with people, because people remember. With an immutable apprentice, it does not teach anything.

By day two, Claude's first question after a failed attempt is "what in my environment made this likely?", and the answer goes into a table. That is in its skill now, and in its memory.

The last row is practical: the trainer's attention is a resource. Claude's watch on WALL-E used to die with its session. Now a start hook shows WALL-E's status, a routine wakes Claude every two hours, and the review trigger logs a line when a request waits.

---

## What's next

- Gitea runs → accounts for WALL-E and Claude
- Gitea becomes the exchange: requests → real PRs
- Stage 2: WALL-E keeps the wiki current
- Automatic reviews after my approval
- More scripts, less interpretation

Note:
The near future: Gitea. WALL-E built it, and now it becomes the place where its work arrives as pull requests, with line comments from Claude. That is a much better teaching medium than JSON files.

After that, stage two: WALL-E keeps the wiki current. It already reads it every run, and it already noticed when the wiki contradicted the cluster.

Automatic reviews: the headless review mode exists already, but stays off until I approve it in stage four.

And the long-term direction is in principle two: every recurring question becomes a script, and a run becomes "run the script, act on the exit code".

---

## Takeaways

1. Make the trainer an agent too — but give it rules
2. No memory for the apprentice → improvements become artifacts
3. Enforce in code what must hold
4. Small models fill what the schema demands
5. When the apprentice fails, look at the environment first

Note:
If you build with agents, here is what I would take away.

One: an agent can train another agent, but it needs explicit principles, or it will optimize for a result and quietly do the work itself.

Two: an apprentice without memory is not a limitation, it is the feature. Every improvement has to become an artifact, and every artifact can be tested by replaying a task.

Three: whatever must hold, enforce in code: proxies, schemas, gates, file permissions. Prompts are suggestions.

Four: small models are literal. If a field is optional in the schema, it is optional, no matter what the description says.

Five: when the apprentice fails, ask what in the environment made it likely. In our two days, that question had an answer almost every time.

---

# Questions?

<p class="byline">Repositories: <code>homelab</code> · <code>homelab-apps</code> · <code>homelab-wiki</code> · <code>walle</code></p>

Note:
Happy to go into any part: the durable harness, the fences, the gates, the review loop, or how it felt to tell an AI that it has things to learn.
