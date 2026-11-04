# P0 Probe Spike + P1 Guest Installation — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Prove Cloudflare Access as the OIDC provider for upstream Artifact Server v0.1.2 on a throwaway `probe` stage (evidence for C1), then run a separate `guest` installation where one outside person at a time reviews their own material with a one-time email code.

**Architecture:** The application stack (Worker + D1 + R2 + domains) is deployed by upstream's pinned Alchemy package from a pinned upstream checkout, wrapped by thin Make targets in `adcookie-infra/deployments/artifact-server/`. Access SaaS OIDC applications, identity providers and policies live in Terraform in `adcookie-infra/stacks/cloudflare/`. Secrets live in 1Password and reach processes only as environment variables through `op run`.

**Tech Stack:** Cloudflare Workers/D1/R2/Access, Alchemy `2.0.0-beta.79`, Wrangler `4.123.0`, Terraform Cloudflare provider v5, 1Password CLI (`op`), Node `>=24.12.0`, pnpm `10.34.3`, bash.

**Spec:** [../specs/2026-11-04-team-review-sharing-design.md](../specs/2026-11-04-team-review-sharing-design.md) (read it fully first). Handoff: [../handoff-2026-11-04.md](../handoff-2026-11-04.md).

## Global Constraints

- Upstream source pinned to tag `v0.1.2` = commit `76d2015d8ca95afeaa83322d3036fd46d67643d8` (repo `plannotator/artifact-server`).
- Tool pins come from upstream `deploy/cloudflare/package.json`; never upgrade them here.
- Stages and domains (spec §4.2):
  - `probe` → `artifacts-probe.adcookie.org`, content `artifacts-probe-content.adcookie.ovh`; deleted after P0.
  - `guest` → `review.adcookie.org`, content `review-content.adcookie.ovh`.
- Zones: application `adcookie.org`, content `adcookie.ovh` (`af3a6050b4f347f52f8a5827e3f45255`); IDs from `stacks/cloudflare/environments/*.tfvars*` (`var.zone_ids`).
- OIDC callback: `https://<applicationDomain>/auth/callback`; scopes `openid email profile`; PKCE on.
- Issuer = the exact `issuer` field of the Access app's discovery document (shape `https://<team>.cloudflareaccess.com/cdn-cgi/access/sso/oidc/<client_id>`).
- Secrets: only `op://` and `cloudflare-secrets-store://` references in files; values never printed, committed or pasted into issues/evidence.
- Every Cloudflare deploy, DNS change, Terraform apply and Secrets Store write is outward-facing: show `alchemy plan` / `terraform plan` output in the Inbox thread "artifact-server rollout" and wait for the owner's approval before the **first** apply of each stage (probe, guest) and each Terraform apply.
- Alchemy telemetry off: `ALCHEMY_TELEMETRY_DISABLED=1`, `DO_NOT_TRACK=1`.
- Upstream-bound branches are cut from `upstream/main`, never contain `docs/superpowers/`, and are opened as PRs/issues only after the owner approves the text in the Inbox.
- Work in Herdr worktrees (`herdr worktree create --cwd <repo> --branch <kebab> --base <default> --no-focus`), one per repo/branch: `adcookie-infra` → branch `artifact-server-probe` (P0) and `artifact-server-guest` (P1); fork → `c1-cloudflare-access-live` from `upstream/main`.
- Access variables controlling who gets in must exist in **both** `environments/dev.tfvars` and `environments/prod.tfvars.tpl` (stack gotcha: a missing entry silently deletes a `count`-gated policy).
- Never admit an outside person to anything except the `guest` installation.

## Review Focus

1. **R2 not enabled on the account** — `stacks/cloudflare/CLAUDE.md` records a 403 "Please enable R2". The stack needs R2. Expected: preflight detects it before any write and stops with an Inbox question (enabling R2 is a dashboard/billing step only the owner can do). → Task 1.
2. **Content host serves application routes** — a deploy where `*.content` routes reach `/api`, `/mcp` or `/auth` breaks origin isolation. Expected: smoke fails. → Task 3 (`smoke.sh` content-isolation check).
3. **Access allows, Artifact Server not admitted** — a person on the Access list who was never admitted must be refused without a member or session being created. Expected: refusal page, no member row. → Task 5 matrix row `preAdmissionRefusal`; Task 10 guest checklist.
4. **Guest list variable emptied or missing in one tfvars** — the guest policy disappears and either nobody or (with an empty include) a misconfigured policy applies. Expected: `terraform plan` shows the guest app always keeps the owner rule; the list adds emails only. → Task 8 (policy always includes the owner; validation on the variable).
5. **Guest sees another guest's material** — two guests overlapping in time on the installation-wide membership model. Expected: the runbook forbids a second active guest; checklist step verifies the member list has exactly owner + one guest before publishing. → Task 10.

---

## P0 — Probe spike

### Task 1: Preflight (read-only)

**Files:** none committed. Results go into the Inbox thread "artifact-server rollout" (create it with `project_path` = this fork; the skill `plannotator` covers `send_message`).

- [ ] **Step 1: Check R2 is enabled on the account.**
  Run with the stack's API token (owner-only vault, via `op-cto exec --`):
  `curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" https://api.cloudflare.com/client/v4/accounts/$ACCOUNT_ID/r2/buckets | jq '.success, .errors'`
  Expected: `true`, `[]`. A 403/10042 means R2 is off → send an Inbox `:::question` ("Enable R2 in the dashboard? It is needed for every stage") and stop P0 until answered.
- [ ] **Step 2: Find or define the Alchemy credential.** Look in 1Password for a Cloudflare API token usable by Alchemy (Workers, D1, R2, DNS, Workers Routes, Secrets Store, Account Settings read). If none, create a scoped token through the Cloudflare API from the stack's token, store it as `op://<owner vault>/artifact-server-alchemy/credential`. Never use `alchemy login`/`wrangler login` interactively.
- [ ] **Step 3: Check whether the Alchemy state store exists** (`alchemy cloudflare bootstrap` is account-level and writes Secrets Store). Run `pnpm exec alchemy plan` dry against an empty stage only after Task 2; if bootstrap is needed, include it in the first Inbox approval request.
- [ ] **Step 4: Read IdP IDs.** `GET /accounts/$ACCOUNT_ID/access/identity_providers` → record the IDs and types of Google and `onetimepin`, and the Zero Trust team domain. Expected: both exist (onetimepin is already enabled per `access_policies.tf` comment).
- [ ] **Step 5: Post the preflight summary to the Inbox thread** (no secrets): R2 status, token item path, state store status, IdP IDs, team domain.

### Task 2: Terraform — probe Access OIDC app (adcookie-infra)

**Files:**
- Create: `stacks/cloudflare/artifact_server.tf`
- Modify: `stacks/cloudflare/variables.tf`, `stacks/cloudflare/environments/dev.tfvars`, `stacks/cloudflare/environments/prod.tfvars.tpl`, `stacks/cloudflare/outputs.tf`, `stacks/cloudflare/CLAUDE.md` (file table + D4 exception)
- Create: `deployments/artifact-server/scripts/store-oidc-secret.sh`

**Interfaces:**
- Produces: Terraform outputs `artifact_server_oidc` = map stage → `{ client_id, issuer }` (non-sensitive) and `artifact_server_oidc_client_secrets` (sensitive, map stage → secret); 1Password item `op://<owner vault>/artifact-server-<stage>-oidc/credential`.
- Produces: variable `artifact_server_probe_second_identity` (string email; the owner's non-adcookie address used as the second commenter via PIN).

- [ ] **Step 1: Write `artifact_server.tf`** with `cloudflare_zero_trust_access_application.artifact_server_probe`: `type = "saas"`, `name = "Artifact Server — probe"`, `saas_app = { auth_type = "oidc", redirect_uris = ["https://artifacts-probe.adcookie.org/auth/callback"], grant_types = ["authorization_code_with_pkce"], scopes = ["openid","email","profile"] }`, `allowed_idps` = Google + onetimepin IDs from Task 1 (managed here, not ignored — this app is new), `auto_redirect_to_identity = false`, policies = existing `tylko kacper` + new account policy `artifact_server_probe_second_identity` (`email` include). Header comment: why SaaS/OIDC and the D4 exception (Alchemy deploys the app stack; Terraform owns only Access).
- [ ] **Step 2: Add the variable to both tfvars files and outputs.** Issuer output is built as `"https://${var.access_team_domain}/cdn-cgi/access/sso/oidc/${client_id}"`; Task 4 verifies it against discovery.
- [ ] **Step 3: Validate.** Run `make validate STACK=cloudflare` → `Success! The configuration is valid.`
- [ ] **Step 4: Plan.** `op-cto exec -- ./scripts/tf.sh cloudflare plan -target=<the new resources> -out=/tmp/cf-artifact-probe.tfplan`. Expected: only creates (1 app, 1 policy), 0 changes, 0 destroys.
- [ ] **Step 5: Ask in the Inbox** with the plan text; wait for approval. Then `apply /tmp/cf-artifact-probe.tfplan`.
- [ ] **Step 6: Write `store-oidc-secret.sh <stage>`**: reads `terraform output -json artifact_server_oidc_client_secrets` inside `tf.sh`, pipes the one value into `op item create`/`op item edit` (category API Credential, title `artifact-server-<stage>-oidc`) via stdin; prints only the item reference. Run it for `probe`. Expected: `op read op://…/artifact-server-probe-oidc/credential | wc -c` > 20.
- [ ] **Step 7: Update `CLAUDE.md`** file table with `artifact_server.tf`, and the D4 exception paragraph (spec D4 rationale, one paragraph). Commit, push, open the PR in `adcookie-infra`; merge after CI green.

### Task 3: Deployment wrapper + smoke (adcookie-infra)

**Files:**
- Create: `deployments/artifact-server/upstream.lock` (`repo=plannotator/artifact-server`, `ref=v0.1.2`, `commit=76d2015d8ca95afeaa83322d3036fd46d67643d8`)
- Create: `deployments/artifact-server/stages/probe.json` (copy of upstream `examples/cloudflare-access.config.json` with real non-secret values: stage `probe`, `installationName` `artifacts-probe`, domains above, `dnsZoneIds`, account ID, `bootstrapAdministratorEmail` = owner, `oidcClientId`, `oidcIssuer`, `oidcClientSecretRef` = `cloudflare-secrets-store://artifact-server-probe-oidc`, `environment` `development`, `deletionProtection` false)
- Create: `deployments/artifact-server/.env.tpl` (`CLOUDFLARE_API_TOKEN`, `ARTIFACT_SERVER_API_TOKEN`, `ARTIFACT_SERVER_OIDC_CLIENT_SECRET` as `op://` refs per stage, `ALCHEMY_TELEMETRY_DISABLED=1`, `DO_NOT_TRACK=1`)
- Create: `deployments/artifact-server/Makefile`, `deployments/artifact-server/scripts/checkout-upstream.sh`, `deployments/artifact-server/scripts/smoke.sh`, `deployments/artifact-server/tests/smoke.bats` (or the repo's existing shell-test style — check `make test-stack-autoupdate`)
- Modify: root `Makefile` (`test:` gains `test-artifact-server`)

**Interfaces:**
- Produces: `make -C deployments/artifact-server {validate,plan,deploy,destroy,smoke} STAGE=<probe|guest>`.
- Produces: `scripts/smoke.sh <applicationDomain> <contentDomain>` with env `ARTIFACT_SERVER_SMOKE_API_KEY`; exit 0 = all checks pass; prints one `ok|fail <check>` line per check.
- Produces: per-stage ARTIFACT_SERVER_API_TOKEN item `op://<owner vault>/artifact-server-<stage>-api-token/credential` (generated once, `openssl rand -base64 32` piped into `op`).

- [ ] **Step 1: `checkout-upstream.sh`** clones/fetches into `${XDG_CACHE_HOME:-~/.cache}/artifact-server/<commit>`, checks out the lock commit, verifies `git rev-parse HEAD` equals the lock, runs `pnpm install --frozen-lockfile` at root and in `deploy/cloudflare`. Fails if the commit differs.
- [ ] **Step 2: `make validate STAGE=…`** runs the upstream parser on `stages/$STAGE.json`: `ARTIFACT_SERVER_CLOUDFLARE_CONFIG="$(cat stages/$STAGE.json)"` and a one-off `node --import tsx -e` that runs `parseCloudflareDeploymentInput` from `deploy/cloudflare/src/deployment-input.ts` and prints `valid`. Expected for probe: `valid`. Also fail if any `stages/*.json` contains a string that is neither a reference nor non-secret (grep for `op://` absence is fine; grep for `-----BEGIN`, `secret":"` values not starting with `cloudflare-secrets-store://`).
- [ ] **Step 3: `make plan|deploy|destroy`** = `op run --env-file=.env.tpl -- pnpm --dir <checkout>/deploy/cloudflare exec alchemy <verb> --stage $STAGE --profile default alchemy.run.ts` (+ `--yes` for deploy/destroy), with `ARTIFACT_SERVER_CLOUDFLARE_CONFIG` and `ALCHEMY_STAGE` exported. `.env.tpl` selects per-stage refs through `$STAGE` (one template, `op run` resolves `op://…/artifact-server-${STAGE}-…`).
- [ ] **Step 4: Write `smoke.sh` tests first** against a local stub (python `http.server` handlers or `nc`; follow the repo's existing shell-test pattern). Cases, each asserting the `ok|fail` line:
  - `health`: `GET https://$APP/health` → 200.
  - `ready`: `GET https://$APP/ready` → 200.
  - `content-isolation`: `GET https://smoke-probe.$CONTENT/api/projects`, `/mcp`, `/auth/login` each → not 2xx and no JSON with `projects` key.
  - `mcp-list`: `POST https://$APP/mcp` with `Authorization: Bearer $ARTIFACT_SERVER_SMOKE_API_KEY`, JSON-RPC `initialize` then `tools/list` → result contains `artifact_capabilities`.
  - `mcp-unauthenticated`: same `tools/list` without the header → 401.
  Run tests → FAIL (script missing). Implement `smoke.sh`. Run → PASS.
- [ ] **Step 5: Card and docs.** `catalog-info.yaml` (Component, `spec.type: service`, owner, links to both domains, `kks/upstream: plannotator/artifact-server`), short `AGENTS.md` (what lives here, the make targets, approval rule). Run `make catalog-validate` → pass. Run `make test` → pass.
- [ ] **Step 6: Commit and PR** in `adcookie-infra` (same branch as Task 2 or follow-up PR); merge after CI green.

### Task 4: Deploy the probe stage

- [ ] **Step 1:** Generate the probe API token item (Task 3 interface). Create the Secrets Store value `artifact-server-probe-oidc` only if upstream's deploy requires it to exist beforehand (README: the deploy process resolves the ref from env; confirm in `src/stack.ts` and note which). 
- [ ] **Step 2:** `make validate STAGE=probe` → `valid`. Fetch `https://<team>.cloudflareaccess.com/cdn-cgi/access/sso/oidc/<client_id>/.well-known/openid-configuration`; its `issuer` must equal `oidcIssuer` byte for byte; fix `stages/probe.json` otherwise.
- [ ] **Step 3:** `make plan STAGE=probe` (plus `alchemy cloudflare bootstrap` if Task 1 found no state store). Post the plan to the Inbox and wait for approval.
- [ ] **Step 4:** `make deploy STAGE=probe`. Then `make plan STAGE=probe` again → only `noop` actions.
- [ ] **Step 5:** `make smoke STAGE=probe` with a placeholder key → all checks `ok` except `mcp-list` (no key yet; run it again in Task 5).

### Task 5: Live verification matrix (C1 evidence)

The owner's browser is involved for logins. Drive it with `claude-in-chrome` where the owner has authorised it; read PIN codes through the Gmail tools only if the owner says so. Record each row as `pass|fail` plus a one-line note; never record codes, tokens, cookies or keys.

**Files:**
- Create (fork, branch `c1-cloudflare-access-live` from `upstream/main`): `project/evidence/cloudflare-access-oidc-2026-11.json` (shape follows `project/evidence/cloudflare-staging-e2e-2026-08-27.json`: `schemaVersion`, `qualification`, `sourceCommit` = `76d2015`, `result`, `environments.probe` with `identityProvider: "cloudflare-access-oidc"`, `checks`, `findings`).

- [ ] **Step 1: Run each check, filling `checks`:**
  - `accessGoogleLogin` — owner signs in with Google → lands on `/auth/callback` then the signed-in review app; becomes administrator (bootstrap).
  - `accessPinLogin` — second identity signs in with the email PIN.
  - `accessPolicyDenial` — an email outside both policies cannot finish Access login.
  - `preAdmissionRefusal` — second identity, before admission: Access passes, Artifact Server refuses; Settings → Members shows no new member.
  - `admission` — owner admits the second identity; its next login gets a session.
  - `apiKeyIssue` — owner issues a key (capabilities incl. `comment:write`, expiry 14 days), stored as `op://…/artifact-server-probe-agent-key/credential`.
  - `mcpPublishClaudeCode` — `claude mcp add --transport http artifacts-probe https://artifacts-probe.adcookie.org/mcp --header "Authorization: Bearer <key>"` (key injected from `op`, not typed into history); publish a small HTML directory; record the review link.
  - `mcpPublishPi` — same from Pi's MCP configuration with the same header.
  - `secondIdentityComment` — second identity comments on the exact version.
  - `agentReadsComment` — the agent lists threads over MCP and sees that comment; replies and resolves it.
  - `smoke` — `make smoke STAGE=probe` all `ok`.
- [ ] **Step 2: Defects.** For each failure: reproduce minimally, write a finding (`id: ACC-00n`, `status`, `summary`). A blocking defect gets an upstream issue draft in the Inbox (approval before `gh issue create`). P0 exit = every row passes, or each blocking failure has a filed upstream issue with reproduction.
- [ ] **Step 3: Fill the evidence file**, then check it holds no secret: `grep -Ei 'bearer|secret|token\"\s*:\s*\"[A-Za-z0-9+/=]{20,}|@gmail' project/evidence/cloudflare-access-oidc-2026-11.json` → no match (emails of people other than placeholder roles are replaced by `owner`/`second-identity`).
- [ ] **Step 4: Report** the matrix to the Inbox thread.

### Task 6: C1 upstream PR (prepare; open after approval)

**Files (fork, branch `c1-cloudflare-access-live`):**
- Modify: `project/spec/conformance.yml` — add a `deployment: cloudflare` evidence entry to `AUTH-019` (`tests: [AUTH-019-B, AUTH-019-F]`, `run: project/evidence/cloudflare-access-oidc-2026-11.json`, `recorded_at` = real timestamp). Only for rows that passed.
- Modify: `deploy/cloudflare/README.md` (replace "a live Cloudflare Access round trip is not attached to the release ledger" with a pointer to the evidence), `apps/site/src/content/docs/docs/deploy/cloudflare-access.mdx` (same sentence, plus any step that proved wrong in Task 5, e.g. the exact issuer, the IdP selection, PKCE grant name).

- [ ] **Step 1:** Make the edits.
- [ ] **Step 2:** `pnpm install && pnpm verify:iteration` → green. Paste the tail of the output into the PR body draft.
- [ ] **Step 3:** `git diff upstream/main --stat` contains no `docs/superpowers/`.
- [ ] **Step 4:** Push the branch to `origin` (fork). Send the PR title/body draft to the Inbox; after approval `gh pr create --repo plannotator/artifact-server --head skacper:c1-cloudflare-access-live`.

### Task 7: Destroy the probe stage

- [ ] **Step 1:** Ask in the Inbox (destroy is irreversible for the probe data; data is throwaway). After approval `make destroy STAGE=probe`.
- [ ] **Step 2:** Delete the retained D1 database by UUID and R2 bucket by exact name, taken from the deploy output / Alchemy state — never by wildcard (README "Run the real-account probe" cleanup rule). Empty the bucket first with `wrangler r2 object delete` per listed key.
- [ ] **Step 3:** Verify: `GET /accounts/$ACCOUNT_ID/d1/database?name=<exact>` → empty; R2 bucket list has no probe bucket; `dig artifacts-probe.adcookie.org` → no record.
- [ ] **Step 4:** Terraform: remove `artifact_server_probe` app and its policy (keep the file for guest), plan → only those destroys → Inbox → apply. Delete the probe 1Password items and the probe MCP entries (`claude mcp remove artifacts-probe`, Pi config). Remove `stages/probe.json`. PR, merge.

---

## P1 — Guest installation

### Task 8: Terraform — identity providers and guest Access app

**Files:**
- Modify: `stacks/cloudflare/artifact_server.tf`, `variables.tf`, both tfvars files, `outputs.tf`, `CLAUDE.md`
- Create: `stacks/cloudflare/identity_providers.tf`

**Interfaces:**
- Produces: variable `artifact_server_guest_emails` (`list(string)`, default `[]`, validation: each matches `^[^@\s]+@[^@\s]+$` and none ends with `@adcookie.pl`); output `artifact_server_oidc["guest"]`; 1Password item `artifact-server-guest-oidc`.

- [ ] **Step 1: Import the existing IdPs** (Google, onetimepin) as `cloudflare_zero_trust_access_identity_provider` resources with `import {}` blocks; write their config to match. Plan must show `2 to import, 0 to add, 0 to change, 0 to destroy` — any change means the config does not match yet; fix the HCL, never apply a change to a shared IdP.
- [ ] **Step 2: Guest app** `artifact_server_guest` (same shape as probe; redirect `https://review.adcookie.org/auth/callback`; `allowed_idps` = both IdP resource IDs). Policies: existing `tylko kacper` + `artifact_server_guests` (include = emails from the variable; `count = length(var.artifact_server_guest_emails) > 0 ? 1 : 0`; `session_duration = "24h"`). Add the variable with `[]` to **both** tfvars files.
- [ ] **Step 3:** `make validate STACK=cloudflare`; plan with targets → imports + guest app + nothing else. Inbox → apply. Run `store-oidc-secret.sh guest`. Update `CLAUDE.md` file table. PR, merge.

### Task 9: Deploy the guest stage

**Files:** Create `deployments/artifact-server/stages/guest.json` (`stage` `guest`, `installationName` `artifacts-review`, `review.adcookie.org` / `review-content.adcookie.ovh`, `environment` `production`, `deletionProtection` true, guest OIDC values, `oidcClientSecretRef` `cloudflare-secrets-store://artifact-server-guest-oidc`). If the parser rejects this combination, use the combination it requires and note why in the PR.

- [ ] **Step 1:** Create the guest API token item. `make validate STAGE=guest` → `valid`; issuer equals discovery `issuer`.
- [ ] **Step 2:** `make plan STAGE=guest` → Inbox approval → `make deploy STAGE=guest` → second plan only `noop`.
- [ ] **Step 3:** Owner logs in with Google (becomes administrator), issues the owner-agent key (`op://…/artifact-server-guest-agent-key/credential`, 90-day expiry), registers MCP `artifacts-review` in Claude Code and Pi with the header. `make smoke STAGE=guest` → all `ok`.
- [ ] **Step 4:** Commit `stages/guest.json`, PR, merge.

### Task 10: Guest runbook and lifecycle

**Files:** Create `deployments/artifact-server/RUNBOOK.md`; modify `deployments/artifact-server/AGENTS.md` (link).

- [ ] **Step 1: Write the checklist** (one section, in order, each step with its command):
  1. Precondition: Settings → Members lists only the owner (no active guest). If not, finish the previous guest first.
  2. Add the email to `artifact_server_guest_emails` in both tfvars → plan → apply (PR).
  3. Admit the person in Settings → Members.
  4. Create a project for them; publish only their material into it (`artifactserver publish … --profile guest` or MCP).
  5. Send the review link; the person logs in with the email PIN.
  6. The agent reads threads over MCP, answers, republishes, resolves.
  7. After the review: deactivate the member, delete the project, remove the email from the variable → plan → apply.
  8. Verify: the person's next login is refused by Access.
  Plus: deploy/upgrade (`upstream.lock` bump → validate → plan → Inbox → deploy → smoke), D1 Time Travel restore pointer (no backups until P3), and the rule "never admit an outsider to `team`".
- [ ] **Step 2:** Dry-run the checklist with the owner's second identity as the "guest" end to end; fix any step that did not match reality. Record the dry run in the Inbox thread.

### Task 11: Open the C2 discussion upstream

**Files:** none in repos; the issue text is drafted in the Inbox.

- [ ] **Step 1:** Draft an issue for `plannotator/artifact-server`: problem (outsiders need per-project review without installation-wide membership, ADR 0015), proposal (a `guest` role scoped to granted projects: see/comment only; cannot publish, manage or list other projects; per-project membership for regular members later on the same model), affected surfaces (core, SQLite/Postgres/D1 adapters, review UI, MCP listing), hostile tests listed in spec §6, offer to implement and to help in #118. Reference #89 (auto-admission) and ADR 0019.
- [ ] **Step 2:** Send the draft to the Inbox; after approval `gh issue create --repo plannotator/artifact-server`. Post the link to the thread.

### Task 12: First real outside review (owner-triggered)

- [ ] **Step 1:** When the owner names the person and the material, run the Task 10 checklist for real. Report completion (and anything the runbook got wrong, fixed in a PR) to the Inbox thread.

---

## Exit criteria

- **P0:** every Task 5 row passes, or each blocking failure has an upstream issue with a reproduction; evidence file holds no secrets; C1 PR drafted (opened once approved); probe resources verified gone (Task 7 Step 3).
- **P1:** guest stage live with a no-op second plan and green smoke; runbook dry-run done; C2 issue opened; Task 12 waits only on the owner.
