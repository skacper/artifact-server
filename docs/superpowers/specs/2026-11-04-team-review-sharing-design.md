# Team review sharing on Artifact Server — design

Status: approved in brainstorming 2026-11-04 (owner: Kacper Skorupski); awaiting written-spec review
Branch: `kks/team-review-sharing-spec` of the fork `skacper/artifact-server` (planning branch, never sent upstream)
Decision record: Plannotator Inbox thread "Team collaboration on Plannotator" (`msg_01M6K3QT1ZBK862F29NPHD6GPW`, project mehos)

## 1. Problem

Agents produce specs, page designs, prototypes and plans as Markdown and HTML. The owner needs to hand a
specific person a link on purpose and get comments back to the agent:

- **Krzysztof**, a long-term collaborator who owns areas of shared projects (e.g. DreamGo). He comments,
  and his own agents publish too.
- **Occasionally one outside person** (a client), short-term, one at a time, with no repository access and
  no company account.

Plannotator cannot do this today:

- The Plannotator Inbox is one person's single-machine window. It binds `127.0.0.1`, its MCP tools answer
  only on that machine, and it refuses reverse proxies. It is not a shared or server inbox.
- Open source link sharing (`plannotator.adcookie.org`, portal + paste) is deprecated upstream. It is
  asynchronous: comments come back by hand, there are no accounts, and there is no approval status.
- Plannotator Workspaces, the author's team product, is closed source and hosted. It is in private beta
  with a waitlist and has no date. The open attempt (`feat/collab`, Live Rooms V1, PR #591) was closed.

`plannotator/artifact-server` (MIT OR Apache-2.0, same author) is the open, self-hosted team system. It has
versioned artifacts, comments per exact version (ADR 0019), OIDC login, members, attribution, and MCP and
CLI publishing for agents. It has three gaps that matter here:

1. Membership is installation-wide: every member manages every project (ADR 0015). An outside person
   cannot be admitted without seeing everything.
2. Comments never go through a public link (ADR 0019). A commenter must log in and be admitted by hand.
3. Upstream is stalled: nothing has been merged since v0.1.2 (2026-09-24), and issue #118 asks for
   co-maintainers.

## 2. Goals and non-goals

Goals:

- G1. The owner and Krzysztof publish (by agent over MCP, or by CLI) and comment with their own identities
  on one team installation.
- G2. Comments reach the publishing agent: it polls with MCP, and Pi and Claude Code Channels get live
  delivery where upstream supports it.
- G3. One outside person at a time can comment on their own material only, logging in with a one-time
  email code.
- G4. Everything general is built so it can go upstream. Only a thin deployment layer is specific to
  adcookie.
- G5. The data can be recovered: D1 Time Travel plus a weekly export of D1 and R2 to GCS.

Non-goals: live co-editing, notifications (upstream excludes them; SCP-003), changes to Plannotator itself,
changes to mehos, replacing Workspaces if it ships.

## 3. Decisions (with rationale)

| # | Decision | Why |
| --- | --- | --- |
| D1 | Use artifact-server, not a fork of Plannotator core | Plannotator moves daily (187 of the last 200 merged PRs are the author's); a fork would drift within weeks. artifact-server is the open team system and its maintainers ask for help. |
| D2 | Run it on Cloudflare (Worker + D1 + R2) | It is the author's primary hosted target; a second (guest) installation is a second stage at near-zero cost; nothing to patch on vps-main. |
| D3 | Cloudflare Access is the OIDC provider (Google for the team, one-time email PIN for guests) | No new identity service; outsiders need no account; Access is already managed in Terraform. Agents use artifact-server API keys (generic OIDC on Cloudflare has no MCP OAuth). |
| D4 | Deploy the stack with upstream Alchemy; only Access and identity providers in Terraform | Upstream migrations and the D1 schema run through their deploy; rewriting it in Terraform means owning a deploy for an app that keeps changing. Recorded as a narrow, documented exception to "Terraform = cloud, DNS, Access" in `adcookie-infra`. |
| D5 | The outside-person (guest) flow is the first priority, delivered right after the spike: a separate guest installation now, and guest reviewer access scoped to projects as the first product contribution | The owner rated guest reviews as more important now than team work. The separate installation gives strong isolation with no fork patch. It is retired once project-scoped guest access (C2) is released, upstream or in our fork. |
| D6 | Backups: D1 Time Travel + weekly export of D1 and R2 to GCS | Files live only in R2 and R2 has no snapshots; GCS is the house object store (WIF, no static keys). |
| D7 | Contribute upstream from the fork `skacper/artifact-server`; our fork's `main` carries patches only while they wait for a merge | Upstream contributions from a personal fork are the norm; a fork `main` with our patches is the fallback if upstream stays stalled. |

## 4. Architecture

### 4.1 General part (goes upstream, in upstream conventions)

Each contribution follows the repo's `AGENTS.md`. It gets an ADR in `project/spec/decisions/` where the
product changes, requirement IDs in `project/spec/conformance.yml`, normal and hostile tests named by
requirement ID, evidence in `project/evidence/`, and a green `pnpm verify:iteration`.

| ID | Contribution | Upstream form | Notes |
| --- | --- | --- | --- |
| C1 | Live qualification of Cloudflare Access as the OIDC provider on the Cloudflare package | Evidence record in `project/evidence/` (no secrets), ledger entry, fixes to `deploy/cloudflare/README.md` and the public guide | The README states that a live Access round trip "is not attached to the release ledger". We need it anyway; it is the cheapest first PR. |
| C2 | Guest reviewers scoped to projects: a `guest` role that sees and comments only on the projects it is granted, cannot publish, manage or list other projects; full per-project membership for regular members is a later step on the same model | New ADR partly superseding 0015; ledger requirements; implementation in the application core and every persistence adapter (SQLite, Postgres, D1); review UI and MCP listing scoped by grant | The guest role comes first because it is the narrowest change that serves outsiders. Discuss it in an upstream issue before coding (CONTRIBUTING asks for discussion before larger features). It lets one installation serve outsiders, which removes the guest installation (D5). |
| C3 | Markdown preview in Review (#78) | PR building on, or co-authored with, the #78 prototype; Preview/Source views; comments on rendered Markdown | Coordinate with the #78 author first, to avoid a duplicate PR. |
| C4 | Scheduled export of a running installation's D1 + R2 to any S3-compatible target or a local directory | Command in `deploy/cloudflare/scripts`, documented next to `RECOVERY.md`; restore compatible with the existing recovery command | Must not require stopping the installation (unlike the recovery command); consistency rules documented. |

Order: C1 → C2 issue (discussion opened early, because guests are the priority) → C4 → C3 → C2 implementation. The small, evidence-only work goes first, to build trust with the maintainers.
Our deployment depends on C1 being proved (by us) and on C4 existing, at least in our fork.

### 4.2 Our deployment (adcookie, thin)

Lives in `adcookie-infra` (third-party author, so the deployment stays with us):

- `deployments/artifact-server/`:
  - the installation config JSON per stage, no secrets, with secret references only
    (`cloudflare-secrets-store://…`, values from `op://`);
  - the pinned upstream tag or fork commit;
  - Make targets wrapping `alchemy plan|deploy`;
  - a runbook, `catalog-info.yaml`, and the backup job (runs C4 from CI with WIF to GCS).
- `stacks/cloudflare/`: Access SaaS OIDC applications for both installations, the identity providers
  (Google, one-time PIN), and the policies. The team policy admits the owner and Krzysztof; the guest
  policy admits a list variable of outside emails.
- The documented Alchemy exception (D4) in `stacks/cloudflare/CLAUDE.md` / `docs/architecture.md`.

| Stage | Application domain | Content domain (wildcard, separate zone) | Members |
| --- | --- | --- | --- |
| `probe` (spike, deleted afterwards) | `artifacts-probe.adcookie.org` | `*.artifacts-probe-content.adcookie.ovh` | owner |
| `team` | `artifacts.adcookie.org` | `*.artifact-content.adcookie.ovh` | owner, Krzysztof; their agents through API keys |
| `guest` | `review.adcookie.org` | `*.review-content.adcookie.ovh` | owner + one outside person at a time |

Agent connection:
- `artifactserver connect` plus the upstream skill (`npx skills add plannotator/artifact-server`) on the
  owner's machines and Krzysztof's;
- one API key per person and machine, stored in 1Password and never committed;
- a global skill note in `kks-os` saying when to publish to `team` and when to `guest`.

### 4.3 Data flow

1. An agent publishes a file or directory over MCP and gets back the full-screen review link.
2. The owner sends the link to Krzysztof or to the guest.
3. The reviewer logs in through Access (Google or an email PIN). artifact-server admits them only if an
   administrator pre-admitted them.
4. The reviewer comments on the exact version.
5. The agent reads the comment threads over MCP, answers or publishes a new version, and resolves the
   threads.

Guest lifecycle:
1. Admit the person.
2. Publish only their material.
3. After the review, deactivate them and delete the project.
4. Remove the email from the Access list variable.

The runbook covers this as one checklist.

## 5. Phases

- **P0 Spike (throwaway `probe` stage).**
  - Deploy upstream v0.1.2 with Access OIDC.
  - Prove: browser login (Google and PIN), pre-admission refusal, MCP publish with an API key from
    Claude Code and from Pi, a comment by a second identity, and the agent reading it.
  - Record the evidence for C1.
  - Destroy the stage.
  - Exit criteria: every step above works, or a blocking upstream defect is filed with a reproduction.
- **P1 Guest installation (priority).**
  - Terraform Access with the one-time PIN identity provider and the guest email list variable.
  - The `guest` stage, the guest lifecycle checklist in the runbook, and the first real outside review.
  - Guest material is short-lived; D1 Time Travel covers it until P3.
  - Open the C2 issue upstream now.
- **P2 Team installation.** The `team` stage, members (owner, Krzysztof), API keys, the card and the skill
  note. Krzysztof connects his agents.
- **P3 Backups.** C4 in the fork, the weekly CI job to GCS for both stages, and one restore test into
  empty targets (per upstream `RECOVERY.md`).
- **P4 Upstream contributions:** C1 and C4, then C3, then the C2 implementation. When C2 is released,
  retire the guest installation and move guests into `team` with the guest role.

## 6. Testing and verification

- Upstream work: `pnpm verify:iteration` green; hostile tests for every security boundary C2 touches.
  These include a cross-project read, a list, a comment and an MCP call by a non-member of the project,
  and installation membership without project membership.
- Deployment: `alchemy plan` reviewed before every deploy. After a deploy, a scripted probe checks the
  application health endpoint, the content host isolation (application routes unavailable on the content
  domain), and an MCP `list` with an API key.
- Backups: a restore test into empty D1/R2 before P3 counts as done, repeated after storage changes.

## 7. Risks

| Risk | Mitigation |
| --- | --- |
| The Cloudflare package is not release-qualified; the Access round trip has never been proved live | P0 spike on a throwaway stage before anything else; the findings go upstream as C1. |
| Alchemy 2.0 is beta and a second infrastructure tool | One stack only, pinned versions (as upstream pins them), plan before every apply, exception documented. |
| Upstream stays stalled; our PRs wait | The fork's `main` carries the patches; the deployment pins a fork commit; we offer co-maintenance in #118. |
| Membership is installation-wide until C2 | The guest installation (D5); never admit an outsider to `team`. |
| Guest data before backups exist (P1 before P3) | Guest material is short-lived and also kept by the publishing agent's repo; D1 Time Travel covers records. |
| Workspaces ships and makes this redundant | Agents use a skill and MCP, so switching is cheap; the owner is on the Workspaces waitlist. |

## 8. Repositories and working rules

- Fork `skacper/artifact-server`: `upstream` = `plannotator/artifact-server`. Upstream PR branches are cut
  from `upstream/main` and never contain `docs/superpowers/`. This planning branch holds only the spec and
  plans.
- `adcookie-infra`: the deployment and Access (P0 needs the Access app too; the `probe` stage config may
  live there from the start).
- Execution runs in a new agent session in a worktree of the fork, started from this spec (handoff).
  Plans: `docs/superpowers/plans/` on this planning branch.
