---
name: security-review
description: Security audit of a codebase or change. Maps the codebase first, then checks package vulnerabilities, leaked secrets, code vulnerabilities, and config/CI, and reports each proven issue with evidence, severity, and the smallest fix. Use when the user asks for a security review or audit, to find vulnerabilities, to check dependencies/packages for CVEs, or whether some code is safe.
---

# Security review

Find bugs that let a less-trusted actor cross a trust boundary. Report only what the source proves. **Read-only:** describe fixes, never edit code, never install packages, never probe live or deployed systems.

## Pick a scope

If the request already names the scope (a diff, PR, file, feature, or "the whole repo"), use it and skip the question. Otherwise ask once, before reading any code:

> What should I audit?
> 1. **Whole project** (default)
> 2. **A particular feature** (tell me which, e.g. "checkout", "auth", `src/api/upload`)
> 3. **My uncommitted changes**
>
> No reply within 10 seconds → I'll audit the whole project.

Use a multiple-choice question tool if your agent has one. Wait 10 seconds for an answer if your agent can time out a question; if it cannot, wait for the reply. In a non-interactive run (CI, headless, `-p`), don't ask: audit the whole project.

Then pick the mode:

- **Focused** — a feature, file, diff, or "is this safe?". Find the feature's code first (routes, handlers, components, DB tables it touches), then do phases 1–4 yourself, limited to that code and what it calls. No sub-agents.
- **Full audit** — whole project. Run every phase. If your agent can launch sub-agents, run each prompt below as its own sub-agent (phase 2 in parallel); otherwise run them yourself, one at a time, in order.

Each prompt below is self-contained: pass it with `{TARGET}` (absolute repo path) and `{MAP}` (phase 1 output) filled in. Sub-agents return findings as text; only you write the final report.

## Phase 1 — Understand the codebase

```
You are mapping {TARGET} for a security audit. Read only; do not run project code.

Read the README, package manifests and lockfiles, framework config, route/handler
directories, auth code, DB schema/migrations, Dockerfiles, CI workflows, and .env.example.

Return, in under 60 lines:
1. STACK: languages, frameworks, package managers + lockfiles, DB, hosting/BaaS.
2. ENTRY POINTS: every place outside input arrives — HTTP routes, server actions,
   API/RPC/webhook handlers, CLI args, file/upload parsers, queue consumers,
   LLM tool calls. Give file paths.
3. PRINCIPALS: anonymous, user, other tenant, admin, internal service — which exist.
4. ASSETS: secrets, PII, other users' data, payments, admin actions, the host.
5. CONTROLS: where authn, authz, input validation, output escaping, rate limiting,
   and tenant scoping actually happen (file:line). Note entry points with none.
6. TRUST BOUNDARIES: the 5–10 places a lower-trust principal reaches a higher-trust
   asset, ranked by risk.
7. OUT OF REPO: controls that live elsewhere (proxy, WAF, cloud IAM, DB policies
   not in migrations) that later findings may depend on.
```

## Phase 2 — Audit (run in parallel)

### 2a. Package vulnerabilities

```
You are auditing third-party dependencies of {TARGET}. Codebase map:
{MAP}

1. For each lockfile, run the matching scanner ONLY if it is already installed
   (never install anything; never run install scripts):
   npm audit --json | pnpm audit --json | yarn npm audit --json | bun audit |
   pip-audit | cargo audit | govulncheck ./... | bundle-audit | osv-scanner -r .
   If none runs, say so and fall back to reading the lockfile for versions you
   know are vulnerable.
2. For every high/critical advisory, check REACHABILITY: grep for imports of the
   package and calls to the vulnerable function/feature. Classify each as
   reachable (cite file:line), not reachable, or unknown. Dev-only deps that never
   ship to production are low priority.
3. Also flag: packages with postinstall/preinstall scripts, git/URL/tarball deps,
   no lockfile committed, wildcard or `latest` version ranges, names that look
   like typosquats of popular packages, and packages that look abandoned.
Return per issue: package@version, advisory ID, severity, reachability with
evidence, and the smallest safe upgrade (exact version).
```

### 2b. Secrets and sensitive data

```
You are hunting leaked secrets in {TARGET}. Codebase map:
{MAP}

1. If gitleaks or trufflehog is installed, run it on the working tree and git history.
2. Otherwise grep tracked files AND `git log -p --all` for: private keys
   (BEGIN ... PRIVATE KEY), AWS (AKIA...), GitHub (ghp_, github_pat_), Stripe (sk_live_),
   Slack (xox), Google (AIza), OpenAI/Anthropic (sk-), JWT secrets, DB URLs with
   passwords, and `password=`/`secret=`/`token=` assignments with literal values.
3. Check: committed .env files, .gitignore missing .env, secrets in client-exposed
   env vars (NEXT_PUBLIC_*, VITE_*, EXPO_PUBLIC_*, REACT_APP_*), service-role/admin
   keys imported into client code, secrets written to logs or error responses.
Never print a full secret: show the first 4 characters, then ****.
Return per issue: file:line (or commit hash), secret type, still in HEAD or only
in history, whether it reaches the client, and the fix (rotate + remove).
```

### 2c. Code vulnerabilities

Split by trust boundary from the map: one sub-agent per 2–3 boundaries for large repos, one total for small ones.

```
You are hunting code vulnerabilities in {TARGET}, focused on: {BOUNDARIES}.
Codebase map:
{MAP}

For each entry point in your focus, follow untrusted input to where it is used.
Check:
- Access control: missing/wrong authz, IDOR (object ID from request with no
  ownership check), tenant ID from request instead of session, admin routes
  reachable by users, mass assignment of role/isAdmin/ownerId.
- Auth & sessions: JWT signature/alg not verified, non-constant-time token compare,
  guessable or reusable reset/magic-link tokens, missing CSRF on cookie-auth writes,
  session not invalidated on logout/password change.
- Injection: string-built SQL/NoSQL, shell commands with input, template injection,
  path traversal and zip-slip, unsafe deserialization, eval/new Function.
- Web/client: XSS (dangerouslySetInnerHTML, innerHTML, v-html, unescaped templates),
  open redirects, postMessage without origin check, SSRF (server fetches a
  user-supplied URL), CORS reflecting any origin with credentials.
- Data exposure: responses returning whole DB rows (hashes, tokens, other users'
  fields), stack traces to clients.
- BaaS (Supabase/Firebase/InsForge): RLS disabled or `using (true)`, policies not
  scoped to auth.uid(), public storage buckets holding private files.
- Resource abuse: unbounded upload/body size, no pagination cap, ReDoS on user
  input, paid APIs (LLM, SMS, email) callable anonymously or without limits.
- LLM features: prompt injection reaching a tool with side effects, model output
  rendered as HTML or executed, tenant A's data in tenant B's context.
- Business logic: negative quantities/prices, race conditions on balances or
  one-time actions, skipping steps in multi-step flows.

Return only candidates where you can name all five: ATTACKER, INPUT (entry
file:line), BROKEN CONTROL (file:line), VICTIM, RESULT. Include the source lines
as evidence. Return nothing else; no best-practice advice.
```

### 2d. Configuration, infrastructure, CI

```
You are auditing configuration of {TARGET}. Codebase map:
{MAP}

Check:
- Security headers / framework config: CSP, HSTS, X-Frame-Options or frame-ancestors,
  cookie flags (HttpOnly, Secure, SameSite) on session cookies, debug mode or
  source maps enabled in production config.
- Docker: runs as root, secrets in ENV/ARG or copied layers, `latest` base images,
  .dockerignore missing .env/.git.
- CI (.github/workflows etc.): pull_request_target checking out PR code with
  secrets, unpinned third-party actions (not pinned to a SHA), secrets echoed,
  overly broad `permissions:`, `curl | sh` installs.
- Infra-as-code / platform config (vercel.json, wrangler.toml, firebase.json,
  terraform): public buckets, wildcard IAM, exposed admin ports, open CORS.
Return per issue: file:line, what is wrong, what an attacker gains, the fix. Mark
anything that depends on settings outside the repo as needs-validation.
```

## Phase 3 — Verify (fresh agent per finding)

Deduplicate phase 2 results by root cause. Give each remaining candidate to a **new** agent that did not find it:

```
Try to DISPROVE this security finding in {TARGET}. Read the cited code and
everything on the path: middleware, wrappers, framework defaults, DB policies,
validation layers.
{FINDING}

Answer with exactly one verdict:
- CONFIRMED: the attack works; restate attacker → input → broken control →
  victim → result with file:line evidence you checked yourself.
- NEEDS VALIDATION: it depends on a fact outside the repo; name that exact fact
  and how the owner can check it safely.
- HARDENING: another layer already blocks it (cite it), or it is a best practice
  with no reachable attack.
- REJECTED: wrong; explain why in one line.
```

Drop rejected ones. Never upgrade a verdict yourself without new evidence.

## Phase 4 — Severity and report

Severity applies to **confirmed** findings only:

- **Critical:** unauthenticated code execution, full database access, arbitrary account takeover, or a live production secret exposed.
- **High:** fully defeats a security control with real consequences — auth bypass, cross-tenant read/write, stored XSS hitting other users, authenticated RCE, reachable critical CVE.
- **Medium:** real boundary violation with limited reach or uncommon preconditions.
- **Low:** leaks non-secret internals, or needs heavy effort for little gain.

Tiebreaker between high and medium: does it fully defeat the control, or only weaken it? If you can't state the concrete damage, it's lower than it feels. Unreachable CVEs are at most low.

Report, most severe first:

```
## Summary
<counts by severity> · scanners run: <list, or "none available"> · mode: focused|full

### [HIGH] <one-line title>
**Category:** package | secret | code | config
**Where:** path/to/file.ts:42
**Attack:** <attacker> sends <input> → <broken control> → <result for victim>
**Evidence:** <the few lines that prove it>
**Fix:** <smallest change at the last trusted decision point, plus a regression test>
```

Then **Needs validation** (unknown fact + how to check), **Hardening notes** (one line each), and **Not covered** (what was skipped and why). If nothing was found, say so and list what was checked; never pad with weak findings.
