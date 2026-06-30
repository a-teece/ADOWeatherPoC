# Target a cross-platform-capable desktop stack — Windows first, Mac as a planned second target

**Context.** Weather PoC is a **desktop** application. The immediate target is Windows PC; macOS
is a stated future target, not a maybe. The obvious shortest path to a Windows desktop app is a
Windows-native UI stack (e.g. WPF / WinForms / WinUI), but that path cannot be carried to macOS
and would force a from-scratch rewrite when Mac arrives.

**Decision.** Choose a **cross-platform-capable desktop** approach from the outset, so the Windows
build and the future Mac build share one codebase. Windows is delivered first; Mac support is
designed for, not retrofitted. This ADR fixes the constraint that the stack must not be
Windows-only; the *specific* framework chosen to satisfy it — **.NET MAUI on .NET 10** — is
recorded in `Technical-Context.MD`.

**Consequences.** Platform-specific capabilities the app relies on — OS location service, system
light/dark theme signal, local persistence — must be reached through abstractions with a
per-OS implementation, never via a Windows-only API baked through the app. This rules out
Windows-native-only UI frameworks at the `/init-tech-context` stage. It trades a little
Windows-day-one convenience for avoiding a Mac rewrite later.
