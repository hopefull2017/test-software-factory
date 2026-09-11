# Claris Commerce — Factory Technical Architecture

Companion to the process/flow diagram — this one answers "what actually runs, where, and who's calling whom," not "what happens and who decides." Two representative patterns cover nearly every AI-driven box in the process diagram; rather than re-diagramming all 18 boxes' infrastructure individually, this traces one worked example of each and states where the rest map to.

## The systems involved

```mermaid
flowchart LR
    subgraph state["Ticket / code state"]
        Jira[("Jira<br/>ticket status + comments")]
        Repo[("Target repo<br/>branches, PRs, commits")]
        PG[("Postgres<br/>via docker-compose")]
    end

    subgraph orch["Orchestration"]
        GHA["GitHub Actions<br/>scheduled + PR-triggered workflows"]
    end

    subgraph compute["Compute"]
        Hosted["GitHub-hosted runner<br/>(ubuntu-latest) — default"]
        SelfHosted["Self-hosted runner<br/>(inside Apple network)<br/>— only when a ticket needs<br/>internal reachability"]
    end

    subgraph ai["AI"]
        API["Anthropic Messages API<br/>— simple judgment calls"]
        CCode["Headless Claude Code<br/>— needs real repo access"]
    end

    Jira <-->|poll / status updates| GHA
    GHA -->|dispatches jobs to| Hosted
    GHA -.->|only when needed| SelfHosted
    Hosted -->|simple calls| API
    Hosted -->|repo-access calls| CCode
    SelfHosted -.->|repo-access calls| CCode
    CCode <--> Repo
    CCode <--> PG
    Hosted <-->|opens/updates| Repo

    classDef state fill:#f4c95d,stroke:#a8791f,color:#1a1a1a,stroke-width:2px;
    classDef orch fill:#b5838d,stroke:#6d4c56,color:#fff,stroke-width:2px;
    classDef compute fill:#8ecae6,stroke:#1b6ca8,color:#1a1a1a,stroke-width:2px;
    classDef ai fill:#a8dadc,stroke:#1d3557,color:#1a1a1a,stroke-width:2px;

    class Jira,Repo,PG state;
    class GHA orch;
    class Hosted,SelfHosted compute;
    class API,CCode ai;
```

**Nothing here is a long-running server.** Every box in "Compute" is ephemeral — spun up per job, torn down after. The only things that persist between runs are Jira's status field and the git repo/PR itself. That's deliberate: no orchestration server to build or maintain, no in-memory state to lose on a crash.

---

## Pattern 1: simple judgment call — traced through Requirements Gate (boxes 2-5)

No repo access needed, just: read text, return a structured verdict, write the result back to Jira.

```mermaid
sequenceDiagram
    actor PM
    participant Jira
    participant GHA as GitHub Actions<br/>(scheduled trigger)
    participant Runner as Runner (ubuntu-latest)
    participant Claude as Anthropic API
    actor RO as Requirements Owner

    PM->>Jira: set status = Ready for Factory

    loop every N minutes
        GHA->>Runner: cron fires, spin up fresh VM
        Runner->>Jira: JQL query — status = "Ready for Factory"
        Jira-->>Runner: matching ticket(s)
    end

    Runner->>Claude: Messages API call<br/>(Requirements Evaluator prompt + ticket text,<br/>forced structured output)
    Claude-->>Runner: PASS / FAIL + findings

    alt FAIL or human rejects
        Runner->>Jira: comment (findings)<br/>status -> Needs Clarification
        Note over Jira: waits for PM to fix + resubmit
    else PASS
        Runner->>Jira: comment (checklist)<br/>status -> Pending Requirements Approval
        Note over Runner: job ends here — nothing is "waiting" in memory
        RO->>Jira: reviews comment, transitions<br/>status -> Requirements Approved
        Note over Jira: next scheduled poll picks this up —<br/>creates the branch + commits task.yaml
    end
```

Same shape covers **box 16, Release Gate** — a policy check that doesn't need repo access either, just evaluates metadata (approved PR, CI evidence) and decides auto vs. needs-human-approval.

---

## Pattern 2: real repo access — traced through PR Creation / Agent Execution (box 9)

Needs an actual checkout, a database, and a full agentic loop — this is where headless Claude Code (not a single API call) does the work.

```mermaid
sequenceDiagram
    participant GHA as GitHub Actions<br/>(triggered on Design Review<br/>Gate approval)
    participant Runner as Runner<br/>(ubuntu-latest, or self-hosted<br/>if this ticket needs<br/>Apple-internal reachability)
    participant Docker as Docker<br/>(Postgres container)
    participant CC as Claude Code<br/>(headless, -p mode)
    participant Repo as Git worktree<br/>(this ticket's branch)
    participant GH as GitHub API

    GHA->>Runner: spin up fresh VM/runner
    Runner->>Repo: checkout ticket's branch<br/>(already has design.yaml + plan.yaml)
    Runner->>Docker: docker compose up -d
    Docker-->>Runner: Postgres ready
    Runner->>Runner: nvm / corepack / pnpm install,<br/>copy env templates, run migrations

    Runner->>CC: claude -p "implement the approved<br/>design + plan" (cwd = worktree)
    activate CC
    CC->>Repo: read existing code + patterns,<br/>write new files
    CC->>Docker: run app against Postgres,<br/>run pnpm build / lint / test
    Docker-->>CC: results
    CC->>Repo: commit changes to the branch
    deactivate CC

    Runner->>GH: open / update the PR<br/>(diff + design/plan linked)
    GH-->>Runner: PR updated
    Note over Runner: job ends — the PR now carries<br/>all further state, nothing else is "waiting"
```

Same shape covers **box 6 (Design Plan)**, **box 7 (Implementation Plan)**, and **box 11 (Implementation Verification)** — all need to actually read the codebase, not just judge text. Design Plan and Implementation Plan likely run as one continuous headless Claude Code invocation, since they're sequential AI-only steps with nothing in between that needs a fresh trigger.

---

## Which pattern each process-diagram box uses

| Box | Step | Pattern |
|---|---|---|
| 3 | Requirements Gate | Simple (API call) |
| 6-7 | Design Plan / Implementation Plan | Heavy (headless Claude Code) |
| 9 | PR Creation | Heavy (headless Claude Code) |
| 10 | CI/CD | Neither — plain GitHub Actions build/test, no AI call at all |
| 11 | Implementation Verification | Heavy (needs to inspect the actual diff) |
| 16 | Release Gate | Simple (policy check on metadata) |
| 5, 8, 12, 17 | Human gates | No compute — a human acts directly in Jira or on the PR |

## Two things worth calling out explicitly in the discussion

- **V0 vs. V1**: right now, this entire sequence runs on a developer's own laptop, triggered by hand — none of this is wired to GitHub Actions yet. The diagrams above describe the V1 target, once the factory CLI is built and the team is ready to automate the trigger.
- **Self-hosted runner is conditional, not default**: only the specific job for a ticket that actually needs to reach something inside Apple's network uses it. Everything else — which is most of this pipeline — stays on cheap, standard GitHub-hosted runners.
