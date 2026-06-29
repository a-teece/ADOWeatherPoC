# Technical Context

Engineering-practice contract for code-writing agents in this project. Every implementation must respect what's written here. If a constraint conflicts with the user's request, flag the conflict — don't silently bypass it.

## Overriding Principles
1. **Secrets never live in the repo.** No credentials or API keys in code or in a committed `.env`; secrets come from a secret store / OS credential manager only. The CI gitleaks scan enforces the detection side of this on every PR.
2. **No breaking changes without an ADR.** Any change that overrides a documented architectural decision requires a new ADR in `docs/adr/` before merge — code never silently overrides the decision log.
3. **Every seam gets a real-IO test on at least one side.** No mock-on-both-sides; defects live at integration seams. A PR that violates this is rejected even if it works. (Expanded in *Testing & the ratchet*.)
4. **Supply-chain dependencies are pinned, never floating.** Third-party CI actions are pinned by commit SHA (not `@vX` tags); NuGet packages are pinned to exact versions. Already practised in `.github/workflows/ci.yml`.

## Frameworks
- **.NET 10 (LTS)** — the runtime and SDK baseline for the whole solution.
- **.NET MAUI (on .NET 10)** — the application framework. Cross-platform UI; the PoC targets **Windows first**, with possible macOS/Linux/mobile use later, so platform-specific code stays behind abstractions rather than baked into the UI layer.
- **MVVM** — the app's structural pattern, via `CommunityToolkit.Mvvm` (source-generated `ObservableObject` / `RelayCommand`).

## 3rd-party tech
- **Open-Meteo** — the weather data source. Free, no API key required; called over HTTPS for forecast/current-conditions data.

## Packages in use
**HTTP:** built-in `HttpClient` via `IHttpClientFactory` (the registered, pooled client — do not `new` up `HttpClient` per call)
**JSON:** `System.Text.Json` (built-in; no Newtonsoft unless a concrete gap forces it)
**Dependency injection:** `Microsoft.Extensions.DependencyInjection` (from the MAUI host builder)
**Logging:** `Microsoft.Extensions.Logging` (from the MAUI host builder)
**MVVM:** `CommunityToolkit.Mvvm`
**UI helpers:** `CommunityToolkit.Maui`
**Testing:** `xUnit` (framework), `FluentAssertions` (assertions), `NSubstitute` (mocking)

Reach for these before introducing a new dependency. A new package is a deliberate decision, not a convenience — prefer the built-in/listed option, and pin any new package to an exact version (see Overriding Principles).

## Instrumentation
Logging is via `Microsoft.Extensions.Logging` only — no Serilog or other structured-logging layer for the PoC. Logs are written to a **local rolling log file** under the app's `AppDataDirectory`, plus debug/console output during development. No remote log sink (Datadog is **not** used in the PoC).

Always logged, at minimum:
- **Every Open-Meteo API call** — the request and its outcome/status.
- **All unhandled exceptions.**

## User Feedback Approach
The app speaks to users **in-app** (no email/Slack/digest channels — it's a desktop client).

- **Error surfacing:** transient failures (e.g. a failed Open-Meteo call or no network) surface as an **inline status message** on the weather view plus a **non-blocking toast/snackbar** (`CommunityToolkit.Maui`). A blocking **modal dialog** is reserved for hard failures only.
- **Tone:** friendly, plain language. Agent-generated user-facing copy never shows error codes, stack traces, or raw exception text to the user. Default phrasing example: *"Couldn't reach the weather service — check your connection and try again."*

## Branching and Environments
- **Model:** trunk-based on GitHub. `main` only ever moves through a pull request that passes the `ci` check; direct commits to `main` are blocked.
- **Branches:**
  - **HITL work** (planning docs, fixes, skill changes) — a feature branch → pull request → `main`.
  - **AFK work** — one `story/<issue#>-<slug>` branch per Story; the orchestrator owns the branch lifecycle and the agent performs no branch operations. Squash-merged into `main` on `Approved`. One Story in flight at a time.
  - The `feature/<id>-<slug>` integration-buffer model is **not yet adopted** — do not pre-adopt it (see `CONTRIBUTING.md`).
- **Environments:** none beyond the local developer desktop yet — this is a Windows desktop app, not a deployed service. No staging/prod server tiers exist; revisit this section if/when a distribution channel is added.
- **Protected:** `main`.

## Git Guardrails
- Never commit to `main` directly — all changes land via PR (see *Branching and Environments*).
- Never force-push, hard-reset, or delete a shared branch without explicit confirmation.
- Never commit secrets; the CI **gitleaks** scan runs on every PR and full history, and a hit fails the build.
- GitHub Actions are **pinned by commit SHA** (version trailed in a comment), never floating tags.
- The CI `GITHUB_TOKEN` starts least-privilege (`contents: read`); widen scopes deliberately and per-job, never inherit write access silently.

## Testing & the ratchet
> Seeded from the Factory standard — **opt-out, not opt-in**. Trim a line only with a deliberate, recorded decision; don't start from nothing. Full standard: <https://github.com/kitcox-dev/enate-claude-skills/blob/main/docs/testing-and-the-ratchet-standard.md> · the *why*: <https://github.com/kitcox-dev/enate-claude-skills/blob/main/docs/agentic-e2e-testing-principles.md>.

- **Test the contract, not the LLM.** Assert on the deterministic envelope (output shape, state transition, side-effect ordering), never on generated text or chain-of-thought.
- **Every seam gets a real-IO test on at least one side.** Mock-on-both-sides only proves the mocks agree; defects live at integration seams, not inside modules. (For this app the headline seam is the Open-Meteo HTTP boundary — exercise it with real I/O on at least one side.)
- **Three tiers, each catching a class the others can't.** Tier 1 — recorded-replay (real local I/O, fakes at the seams, replayed external fixtures; every commit); Tier 2 — live-dependency, disposable (real subprocess against a per-run throwaway substrate; bounded cost, scheduled); Tier 3 — live (real production substrate and subprocess; continuous).
- **Platform matrix is non-negotiable for OS-touching code.** Filesystem/subprocess code runs Tier 1 + Tier 2 on every OS it supports, not just the dev box's. MAUI is Windows-first today but cross-platform-bound — when a target is added, its tiers come with it.
- **Plan test coverage at the Feature level, not the Story level.** Each Feature declares at kickoff which tiers it needs, which fixtures/substrates must exist, and a Tier-2 cost ceiling.
- **The ratchet — every Tier-3 defect pins a cheap-tier regression.** A live defect's fix lands with a Tier-1 (or Tier-2) regression that would have caught it, encoding the **class** of defect **bounded by the seam** where it surfaced — not the exact reproducer, and not a project-wide sweep. A Story that surfaced from a live defect carries that ratcheting fixture as an acceptance criterion.
- **Disposable substrate, not shared.** Tier-2 substrate is created per run and torn down; reproducibility comes from a seed (tarball/manifest/script), never persistent state.
- **Assert on observable behaviour, not internal state.** For Tier 2/3 the target is what an operator could verify from external artefacts — UI state, log events, file outputs.
