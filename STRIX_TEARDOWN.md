# Strix — Reverse-Engineering Teardown & Reuse Assessment

> Internal engineering analysis for a pentesting-toolkit port. Names are accurate throughout — provenance is the deliverable. Licensing facts are flagged so **you** decide; no licensing decision is made here.

## 0. Identity & License (ground truth)

- **Project:** Strix — `strix-agent` v1.4.1. Publisher "usestrix" (`hi@usestrix.com`), site strix.ai / docs.strix.ai, source github.com/usestrix/strix.
- **License: Apache-2.0** (`LICENSE`, `pyproject.toml`). Permits reuse, modification, redistribution, and **commercial/internal/proprietary** use. Conditions: retain copyright + license notices, retain `NOTICE` contents if present, and **state significant changes** in modified files. No copyleft — your derived toolkit does not have to be open-sourced.
- **Build/dist:** hatchling PyPI wheel + PyInstaller one-file binaries (`strix.spec`), CI matrix of 5 OS/arch targets, curl|sh installer (`scripts/install.sh`).

## 1. What Strix is, in one paragraph

A host-side Python orchestrator that runs **teams of LLM agents** which perform application penetration testing inside a **Docker sandbox** (Kali-based image) preloaded with real offensive tooling. It is **not a custom agent loop** — it is a thin, opinionated layer over the **OpenAI Agents SDK** (`openai-agents[litellm]==0.14.6`, imported as `agents`), using **LiteLLM** as the multi-provider model backend. Agents are given a Jinja-composed system prompt assembled from a large markdown **"skills" knowledge base**, a fixed set of host-side function tools, and SDK sandbox shell/filesystem tools. Findings are validated with working PoCs, scored with CVSS, deduped (deterministic + LLM-judge), and emitted as Markdown/CSV/JSON/SARIF/encrypted-PDF, viewable in a local web dashboard.

## 2. Architecture map

```
CLI (argparse) ─ strix/interface/main.py
  ├── TUI (Textual/Rich)  │  Viewer (stdlib HTTP server + prebuilt React/Vite SPA)
  └── runner.py ── run_strix_scan
        ├── config/  pydantic-settings; StrixProvider(MultiProvider) over LiteLLM
        ├── agents/  factory.build_strix_agent → SandboxAgent (+ Jinja system prompt + skills)
        ├── core/    execution loop (Runner.run_streamed), AgentCoordinator (graph/mailbox/snapshot)
        ├── llm/     context_budget + compaction (summarize-and-truncate)
        ├── tools/   host-side @function_tool families + SDK sandbox tools
        ├── runtime/ docker-py backend; one container per scan; Caido sidecar
        └── report/  finding model, dedupe, SARIF, CVSS, writers
```

### External products/frameworks it depends on (named)
| Layer | Dependency |
|---|---|
| Agent loop / sandbox | **OpenAI Agents SDK** `openai-agents==0.14.6` (+ `agents.sandbox`) |
| Model routing | **LiteLLM** (any provider it supports) + `openai` SDK (Codex/Responses path) |
| Settings / prompt | **pydantic-settings**, **Jinja2** |
| Sandbox runtime | **docker-py**, image `ghcr.io/usestrix/strix-sandbox:1.1.0` on **kalilinux/kali-rolling** |
| HTTP proxy | **Caido** (`caido-cli` 0.56.0) via **`caido-sdk-client`** GraphQL SDK |
| Browser | **agent-browser@0.26.0** npm CLI driving **Chromium over CDP** (not Playwright/Selenium) |
| Web research | **Perplexity API** (`sonar-reasoning-pro`) |
| Reporting | **`cvss`** (CVSS3), **reportlab** + **pypdf**, SARIF 2.1.0 (hand-built) |
| UI | **Textual** + **Rich** (TUI), **React/Vite/TS** (viewer SPA) |
| Telemetry | **PostHog** + **Scarf** |

## 3. Subsystem detail

### 3.1 Orchestration core (`core/`, `agents/`, `llm/`)
- **Loop:** `Runner.run_streamed(...)` per agent turn; streamed events forwarded to TUI. Turn ends only when a **lifecycle tool** returns success (`finish_scan` root / `agent_finish` child); non-interactive mode force-injects "call a lifecycle tool" and retries to `max_turns`.
- **Per-agent state:** SDK `SQLiteSession` (`agents.db`). Inter-agent messages = appended session items.
- **Multi-agent:** `AgentCoordinator` (`core/agents.py`) owns the graph (statuses/parents/mailboxes/runtimes), snapshots atomically to `agents.json` (powers **resume**). Children are detached `asyncio` tasks in the same process/sandbox. Primitives: `create_agent`, `view_agent_graph`, `send_message_to_agent` (interrupts target stream), `wait_for_message` (parks on an asyncio Event), `stop_agent` (graceful cancel, leaf-first cascade), `agent_finish`.
- **Resilience (the transferable gold):** proactive + reactive **compaction**, context-overflow→retry, image-budget enforcement + image-strip retry on 400/422, transient-model backoff (4–5 retries, exp 2→90s), provider-refusal detection, Codex guardrail parking.
- **LLM layer:** `StrixProvider(MultiProvider)` — only `openai`/`litellm`/`any-llm` go SDK-native; everything else passed to LiteLLM with prefix preserved (`anthropic/…`, `vertex_ai/…`, `deepseek/…`, `ollama_chat/…`). Token accounting via `litellm.get_model_info`/`token_counter`. Compaction (`llm/compaction.py`) summarizes oldest history into a `<conversation-checkpoint>` message, snapping the split so **no tool call is left open** — security-tuned summary prompt preserves findings/creds/payloads verbatim.

### 3.2 Tooling (`tools/`)
Two kinds: **host-side function tools** (`@function_tool`, schema auto-derived from signature + docstring — docstrings ARE the LLM tool descriptions) wired by explicit import list + `_BASE_TOOLS` tuple in `agents/factory.py` (no decorator registry — every family `__init__.py` is empty); and **SDK sandbox tools** (`exec_command`, `write_stdin`, `apply_patch`, `view_image`) emitted per-run by the SDK Shell/Filesystem capabilities.

- **proxy/ (centerpiece):** `caido_api.py` (589 LOC) + `tools.py`. Async wrapper over Caido GraphQL/SDK. 6 tools: `list_requests` (HTTPQL filter language), `view_request` (regex reflection hunting), `repeat_request` (**Repeater**-equivalent with framing-header sanitization against smuggling desync), `list_sitemap`/`view_sitemap_entry` (attack-surface tree), `scope_rules`. Serialized behind an asyncio lock (transport not concurrency-safe). **No active scanner through Caido** — scanning is sandbox CLIs.
- **agent_browser/:** shells out to `agent-browser` CLI; always-loaded skill drives it; XSS validation via real JS execution, screenshots → `view_image`; traffic Caido-captured.
- **shell/:** SDK ShellTool, runs in the Kali container, non-interactive by default; `write_stdin` needs `tty=true`. Strix wraps to force bash, clamp output, humanize errors. All children inherit proxy env → Caido.
- **agents_graph/, reporting/, notes/ (shared scratchpad), todo/ (per-agent), thinking/ (no-op CoT channel), finish/, load_skill/, web_search/ (Perplexity), output_store.py** (head/tail+spill result bounding).

### 3.3 Runtime / sandbox (`runtime/`, `containers/`)
- **One Docker container per scan** via docker-py, driven entirely through the SDK session (`docker exec`); no in-container daemon. `StrixDockerSandboxClient` copies SDK v0.14.6 `_create_container` **verbatim** and adds `NET_ADMIN`/`NET_RAW` caps, `host.docker.internal`. **Permissive posture by design:** caps only added never dropped, no seccomp/userns, user `pentester` has passwordless sudo — appropriate only for a deliberately isolated sandbox.
- **Baked-in toolchain (Dockerfile):** ProjectDiscovery `httpx`/`katana`/`nuclei`/`subfinder`/`naabu` + `nmap` (setcap), `sqlmap`, `ffuf`, `wapiti`, `gospider`, `interactsh-client`, `cvemap/vulnx`; SAST/SCA `semgrep`, `bandit`, `retire.js`, `ast-grep`, `tree-sitter`, `trufflehog`, `gitleaks`, `trivy`; `jwt_tool`; Chromium; python venv with `caido-sdk-client` + `caido_api.py`.
- **Caido** runs as in-container sidecar on `127.0.0.1:48080`; self-signed "Testing Root CA" MITM trusted system-wide; all agent HTTP forced through it via proxy env. Host bootstraps via GraphQL guest login + temp project.
- **Backend registry** (`register_backend`) anticipates remote/cloud backends; **only local Docker ships.**

### 3.4 Knowledge base (`skills/`) — the standout asset
- **58 markdown files, ~10,184 lines.** Trivial YAML frontmatter (`name`/`description`, stripped at load). Loader (~230 LOC) resolves bare/`category/name`, supports external override dirs (`register_skill_dir`), caps 5 skills/agent.
- **vulnerabilities/ (25 files, ~4,926 lines):** sql_injection, xss, ssrf, ssti, rce, idor, xxe, csrf, open_redirect, path_traversal_lfi_rfi, insecure_deserialization, insecure_file_uploads, prototype_pollution, nosql_injection, race_conditions, http_request_smuggling, header_injection, mass_assignment, broken_function_level_authorization, business_logic, authentication_jwt, weak_password_detection, information_disclosure, subdomain_takeover, llm_prompt_injection.
- **frameworks/** (django, fastapi, nestjs, nextjs), **technologies/** (supabase, firebase_firestore, auth0, active_directory, grafana_prometheus), **cloud/** (aws, gcp, kubernetes), **protocols/** (graphql, oauth), **tooling/** (11 CLI command playbooks), **reconnaissance/**, **scan_modes/** + **coordination/** (internal, Strix-orchestration-specific).

### 3.5 Reporting (`report/`)
- **Finding model** (`state.py`): rich dict (id/title/severity/CVSS+8-metric breakdown/CWE/CVE/code_locations with fix_before/after/poc_script/remediation/…).
- **Dedupe:** deterministic `(CVE, ecosystem, package)` + **LLM-as-judge** for the rest (fails open).
- **SARIF 2.1.0** (`sarif.py`): **fully self-contained, stdlib-only**, GitHub code-scanning compatible, CWE→STRIDE table, stable `partialFingerprints`, SARIF fixes, strips PoC bodies. High quality.
- **CVSS:** thin wrapper over `cvss` lib. **PDF:** reportlab+pypdf, AES-256 encrypted locally (password never leaves machine). `safe_fence()` prevents Markdown code-block breakout from LLM-authored PoC.

### 3.6 Interface
- CLI (argparse): `--target/-t`, `--scan-mode {quick,standard,deep}`, `--scope-mode`, `--max-budget`, `--resume`, etc. Requires Docker + `STRIX_LLM`.
- **TUI:** Textual/Rich. **Viewer:** stdlib `ThreadingHTTPServer` on `127.0.0.1` random port + prebuilt React/Vite SPA; reads run files off disk; two-gate auth (bootstrap token → session cookie; history/report-send gated behind **email OTP via the strix.ai relay**).

## 4. Phone-home / coupling (privacy review for a port)
- **Scanning itself is fully offline** with your own `STRIX_LLM` key + local Docker. No login required.
- **Telemetry (PostHog + Scarf):** scan start/end, finding severity/CWE/CVE-bool, skill-load (non-builtin reported as literal `"custom"`), aggregate token/cost, error_type. Explicitly **no targets/URLs/code/prompts**. Random per-process id. **Kill both with `STRIX_TELEMETRY=0`.** Endpoints: `us.i.posthog.com`, `strix.gateway.scarf.sh`.
- **Viewer relay** (`STRIX_APP_URL=https://app.strix.ai`): email OTP, encrypted-report email delivery, feedback — **optional viewer features only**; can demand a work email to unlock history (lead-gen gate).
- **Update check:** GitHub/PyPI, ≤1/day, `STRIX_NO_UPDATE_CHECK` to disable.

## 5. Reuse / port assessment + attribution matrix

Tiers: **LIFT** = portable near-standalone · **ADAPT** = portable logic, cut couplings · **REWRITE/REPLACE** = SDK/infra-bound or domain-specific.

| Component | Tier | Notes / coupling | Attribution if reused |
|---|---|---|---|
| **skills/ markdown corpus** (vuln/framework/tech/cloud/protocol/tooling) | **LIFT** | Vendor-neutral offensive knowledge; trivial schema; loader has 2 easily-stubbed deps. The single highest-value asset. | Apache-2.0: these are copyrighted source files — keep notices / NOTICE, state edits. (Prose, but still licensed.) |
| **report/sarif.py** | **LIFT** | stdlib-only `dict→SARIF 2.1.0`; CWE→STRIDE, fingerprints, fixes. Drop-in for any scanner matching the finding shape. | Apache-2.0 notice in file header. |
| **CVSS logic** (`tools/reporting` 8-metric→vector→score) | **LIFT** | Thin over `cvss` PyPI lib. | Trivial; keep `cvss` lib's own BSD license. |
| **report_pdf.py** (reportlab+pypdf, AES-256) | **LIFT** | Only couples to disk-read helpers. | Apache-2.0 header. |
| **writer.py** `safe_fence`/atomic writes | **LIFT** | One `core.paths` import. | Apache-2.0 header. |
| **skills loader** (`skills/__init__.py`) | **ADAPT** | Clean resolver + override dir; stub telemetry + resource-path. | Apache-2.0. |
| **llm/compaction.py + context_budget.py** | **ADAPT** | Most cleanly factored engine pieces; depend on LiteLLM + a Session iface + StrixProvider for the summary call. Security summary prompt is domain-tuned. | Apache-2.0; note it's a derivative. |
| **proxy/caido_api.py** | **ADAPT** | Correct, deep Caido wrapper (HTTPQL, framing sanitization, sitemap GraphQL, replay hygiene). Needs `caido-sdk-client` + a running Caido. | Apache-2.0; also pulls Caido's own license at runtime. |
| **AgentCoordinator** (`core/agents.py`) | **ADAPT** | Generic asyncio graph/mailbox/snapshot; only couples to a Session type + a write lock. | Apache-2.0. |
| **config/codex.py** (ChatGPT OAuth+PKCE client) | **ADAPT** | Self-contained; `requests`/`httpx`/`openai` only. | Apache-2.0. |
| **config/settings.py + loader.py** | **ADAPT** | Standalone pydantic-settings; env>JSON>default; note global memoized singleton. | Apache-2.0. |
| **output_store.py** (result bounding/spill) | **ADAPT** | stdlib apart from injected spill writer. | Apache-2.0. |
| **agents_graph/ tools** | **ADAPT** | Good primitives but need the coordinator + runner `spawn_child_agent` closure. | Apache-2.0. |
| **runtime/local_dir_staging.py** | **LIFT** | stdlib symlink-safe tree staging; directly liftable. | Apache-2.0. |
| **containers/Dockerfile** (Kali tool image) | **ADAPT** | Generic pentest image; strip Caido/agent-browser/`caido_api` conventions if unwanted. | Apache-2.0 for the Dockerfile; each baked tool keeps its own license (nmap=NPSL/custom, sqlmap/nuclei/ffuf etc. vary — **audit per-tool for your distribution**). |
| **core/execution.py loop** | **REWRITE-around-SDK** | Logic reusable but hard-wired to `Runner.run_streamed` + SDK/openai/docker exception taxonomy. Keep the SDK or rewrite. | Apache-2.0. |
| **agents/factory.py** | **REWRITE-around-SDK** | Tightly bound to `agents.sandbox` Filesystem/Shell + `tool_use_behavior`. ChatCompletions↔Responses tool shim is useful but SDK-specific. | Apache-2.0. |
| **runtime/docker_client.py** | **REPLACE/CAUTION** | **Verbatim copy of SDK 0.14.6 internals**; pinned, brittle on SDK bumps. Reuse inherits exact-version coupling. | Apache-2.0 — but note it mirrors SDK code; honor the SDK's (also Apache-2.0) notices. |
| **system_prompt.jinja + persona/methodology** | **REWRITE** | Mechanism (Jinja) portable; content is pentest-specific. Persona names a fictional "OmniSecure Labs" (prompt framing). | Apache-2.0 if you keep text. |
| **telemetry/, viewer relay auth** | **DROP** | Strix keys/endpoints + strix.ai lead-gen gate. Remove for an internal tool. | n/a (remove). |
| shell/apply_patch/view_image | **NOT STRIX CODE** | SDK-provided capabilities; value is the Kali image + Caido-proxied env, not Strix code. | These come under the OpenAI Agents SDK license, not Strix. |

### Load-bearing couplings to budget for
1. **OpenAI Agents SDK** is pervasive (loop, factory, sessions, provider, hooks, compaction, sandbox) — the deepest dependency; its `agents.sandbox` is comparatively new/private-helper-reliant.
2. **Process-global `ReportState` singleton** carries cost/usage + budget hooks + the LiteLLM cost callback.
3. **Docker sandbox assumption** throughout (spill-to-workspace, Filesystem/Shell).
4. **LiteLLM global monkey-patching** at configure time (not embedding-friendly if host also uses LiteLLM).
5. **Lifecycle-tool "done" contract** baked into both the loop and the prompt.

## 6. Recommended port strategy (highest value / lowest coupling first)
1. **Take the `skills/` corpus + loader** → your own prompt system. (Biggest win, cleanest lift.)
2. **Take `sarif.py` + CVSS + writer + report_pdf** → a standalone findings/report module for any scanner.
3. **Take `caido_api.py`** if you standardize on Caido; otherwise mine its HTTPQL/framing/sitemap patterns.
4. **Take `compaction.py` + `context_budget.py` + the retry/guardrail scaffolding** as a resilience layer — rewrite the thin SDK-session seam to your loop.
5. **Take `AgentCoordinator`** as your multi-agent substrate.
6. **Fork the Dockerfile** as your sandbox image; per-tool license audit before any external distribution.
7. **Drop** telemetry + viewer relay; decide your own sandbox hardening (Strix's permissive posture is intentional but you may want cap_drop/seccomp).

---

### Attribution bottom line (informational — your call to make)
Everything above is Apache-2.0. For an internal-only toolkit, the practical obligations are light: keep the license text + any `NOTICE`, preserve copyright headers in files you copy, and note your modifications. If you ever **distribute** the derived toolkit, the same conditions apply plus the per-tool licenses of anything baked into the container image. The one thing that would actually create exposure is removing/obscuring the provenance — which is exactly why this report keeps names intact.
