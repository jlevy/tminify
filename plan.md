# tminify Redesign Plan

**Status:** Draft. Design proposal, not yet approved or implemented.

**Branch:** `claude/code-review-ecosystem-YHjD7`

---

## Summary

Reframe `tminify` from "HTML + Tailwind v4 minifier" to **"the smallest Python
dependency that gives you a working JS toolchain end-to-end, without rediscovering
the install/cache/version/concurrency tricks every time."**

Tailwind compilation and HTML minification become the *first two backends* rather
than the package's reason for existing. New backends (esbuild, swc, oxc-minify,
lightningcss, …) follow the same shape. The current `tminify(src, dest)` function
survives as a *recipe* layered on top of two backends.

The single most important architectural shift: **prefer distributing native
single-file binaries** (Tailwind standalone, esbuild, swc, oxc-minify, lightningcss
all ship them) over `npm install`. This drops the Node.js prerequisite entirely for
most backends, eliminates the lockfile / supply-chain problem, and gives users a
`pip install` experience that "just works."

---

## 1. Background

### 1.1 What `tminify` is today

A small Python package (~350 lines) that wraps two Node-based tools:

- `@tailwindcss/cli` v4 — compiles utility classes used in an HTML page into a
  minimal CSS bundle.
- `html-minifier-terser` — minifies HTML, including inline CSS and inline JS via
  Terser.

The CLI accepts an HTML file, optionally detects a Tailwind v4 Play CDN script,
compiles + inlines the CSS, then minifies the whole thing. JS dependencies are
installed lazily via `npm install` into
`site-packages/tminify/javascript/node_modules` on first run.

Key files:

- `src/tminify/main.py` — orchestration, regex HTML surgery, subprocess wiring.
- `src/tminify/cli.py` — argparse front end.
- `src/tminify/javascript/package.json` — JS dependency manifest (no lockfile).
- `tests/test_cli.py` — single 480-line test function with 12 sub-cases.

### 1.2 Original value proposition

> Drop a Tailwind v4 Play CDN script into your HTML, prototype freely, then run one
> Python command to get a self-contained, production-minified file.

This is real and uniquely served by tminify — most Tailwind tooling assumes you have
adopted a full JS project structure.

### 1.3 What's worked

- The CDN-detect → compile → inline workflow is a useful UX, not duplicated
  elsewhere.
- The Python tooling story is solid: uv, hatchling, ruff, basedpyright, dynamic
  versioning, trusted-publisher PyPI release, multi-version Python CI.
- Package boundary is small enough that maintenance burden has been low.

### 1.4 What's degraded since release

- **Tailwind v4 has moved to CSS-first config.** The current `tailwind.config.js`
  generation is now a legacy compatibility bridge.
- **`corePlugins.preflight` is no longer supported in v4** — the `--preflight` flag
  silently does nothing (`main.py:31-41`).
- **`html-minifier-terser` is effectively unmaintained** (no release since late
  2023). Active fork `html-minifier-next` exists.
- **Tailwind ships a standalone single-file binary** (`@tailwindcss/standalone`)
  that needs no Node.js. This obsoletes half of tminify's npm-install logic.
- **CSS minification is now native to Tailwind v4** via Lightning CSS — the
  `--minify-css` switch in the HTML minifier is partially redundant when Tailwind
  is in play.

### 1.5 Open code-review issues (full list in the prior review)

**Correctness:**

- `--preflight` non-functional (`main.py:31-41`).
- No `package-lock.json`; non-reproducible installs (intentionally excluded by
  `src/tminify/javascript/.gitignore`).
- No subprocess timeouts on any `subprocess.run`.
- Concurrency race in `get_js_dir()` (`main.py:78-103`).
- Pre-flight tool check (`shutil.which("npx")`) only runs on the Tailwind path
  (`main.py:163`).

**Design fragility:**

- Regex-based HTML surgery: multiple patterns, edge cases for multi-CDN scripts,
  attribute quoting, `<style type="...">`, conditional comments.
- `node_modules` lives in `site-packages` (read-only installs break; stale modules
  across upgrades).
- Logging levels inverted (info-level messages emitted at WARNING).
- CLI flag style (`--no_minify`).

**Tests:**

- One giant `test_cli_comprehensive` function, all subprocess-based; slow and hard
  to triage failures.

---

## 2. Ecosystem context (Jan 2025 → May 2026)

### 2.1 Tailwind

- v4 launched January 2025 with the **Oxide** Rust engine, 5–100× faster rebuilds,
  and **CSS-first configuration** as the default.
- v4.1 / v4.2 (early 2026) added a webpack plugin, new color palettes, and
  logical-property utilities.
- **`@tailwindcss/standalone`** ships per-platform single-file binaries that bundle
  Lightning CSS and Parcel Watcher. No Node required.
- `corePlugins`, `safelist`, `separator` from v3's JS config are unsupported in v4.
- `tailwind.config.js` works only via the `@config` CSS directive — explicitly
  documented as a temporary bridge.
- Content detection is automatic; `@source` is the explicit-add directive.

### 2.2 JS minifier landscape

| Tool | Status | Distribution | Speed | Compression |
|---|---|---|---|---|
| `terser` | Maintained | npm | Slow | Best |
| `html-minifier-terser` | **Unmaintained since 2023** | npm | Slow | Good (uses Terser inside) |
| `html-minifier-next` | Active fork (j9t) | npm | Slow | Good (drop-in replacement) |
| `@minify-html/node` | Active, Rust core | npm + native | Very fast | Aggressive (occasional JS edge cases) |
| `oxc-minify` | Vite default since 2026 | npm + native | 30–90× faster than Terser | ~1% larger than Terser |
| `@swc/html` / `swc` | Active | npm + native | Fast | Good |
| `esbuild` | Active | native binary | Fast | Good |
| `lightningcss` | Active | native binary + npm | Fast | CSS-only |

### 2.3 Native binaries are now the norm

For every modern JS build tool that matters in 2026, the maintainers publish
per-platform single-file binaries on GitHub releases:

- Tailwind: `tailwindcss-{linux,macos,windows}-{x64,arm64}`
- esbuild: `esbuild-{platform}`
- swc: `swc-{platform}`
- oxc: `oxc-{platform}`
- lightningcss: per-platform via N-API (also npm-distributed)

A Python harness that downloads, caches, and shells out to these binaries needs
**no Node.js at all** for the majority of cases.

---

## 3. Reframe: from "Tailwind minifier" to "JS toolchain harness"

### 3.1 New value proposition

> The smallest Python dependency that gives you a working modern JS build tool,
> end-to-end, with `pip install`.

A Python developer who wants to run Tailwind / esbuild / swc / oxc-minify /
lightningcss against their files should be able to:

```python
from <package> import tailwind, html_minify

css = tailwind.compile(css_input, sources=["page.html"])
minified = html_minify.run(html_input)
```

…without installing Node, writing a `package.json`, learning Tailwind v4's `@source`
directive, debugging npm cache permissions, or pinning a lockfile. The harness
handles install, version pinning, caching, subprocess plumbing, error translation,
atomic IO, and concurrency.

### 3.2 What this is *not*

- Not a replacement for a real JS project setup (Vite, Next.js, etc.).
- Not a wrapper for *any* JS tool — explicitly curated.
- Not a Node version manager (use `nodeenv` if a real Node install is needed).
- Not a bundler with opinions; composing tools is the user's job.

### 3.3 Naming

`tminify` is the wrong name once Tailwind isn't the centerpiece. Open question —
flagged for later. Decision deferred until the architecture stabilizes.

---

## 4. Architecture

Three layers, in increasing specificity:

```
┌────────────────────────────────────────────┐
│  Recipes  (composed workflows, optional)   │
│  e.g. tminify_html(src, dest)              │
├────────────────────────────────────────────┤
│  Backends  (declarative per-tool adapters) │
│  Tailwind, html-minifier-next, esbuild, …  │
├────────────────────────────────────────────┤
│  Core  (install / cache / run / lock)      │
│  ~300–500 lines, stable                    │
└────────────────────────────────────────────┘
```

### 4.1 Core (Layer 1)

Solves once, used everywhere:

- **Install strategies.** Pluggable: `NativeBinary(github_release, version,
  asset_pattern, checksum)` and `NpmPackage(name, version, lockfile)`.
- **Cache location.** `platformdirs.user_cache_dir("<package>") / <tool>-<version>`.
  Read-only-site-packages-safe.
- **Version pinning.** Each tool pinned to a specific version. Upgrade by bumping
  the constant; no floating ranges.
- **Subprocess primitives.** Timeouts, stdin/stdout, structured errors, working
  directory, env scrubbing.
- **Concurrency.** File-lock around install to prevent races when multiple
  processes start cold.
- **Atomic IO.** Inline a 10-line `atomic_output_file` rather than depending on
  `strif`.
- **Checksum verification.** For native binaries, verify against published SHA256
  from the GitHub release (or pinned in the package).

### 4.2 Backends (Layer 2)

Each backend is mostly metadata:

```python
# Pseudo-code, illustrative

tailwind = NativeBinary(
    name="tailwindcss",
    version="4.1.8",
    github="tailwindlabs/tailwindcss",
    asset=lambda plat, arch: f"tailwindcss-{plat}-{arch}",
    checksums={...},
)

html_minifier = NpmPackage(
    name="html-minifier-next",
    version="...",
    bin="html-minifier-next",
)

esbuild = NativeBinary(...)
swc = NativeBinary(...)
oxc_minify = NativeBinary(...)
lightningcss = NativeBinary(...)
```

A backend exposes a thin Python API tailored to that tool's idioms
(`tailwind.compile(...)`, `html_minifier.run(...)`, `esbuild.bundle(...)`). The
wiring beyond install/run is deliberately small.

**Hard discipline rule:** if a backend exceeds ~80 lines, that's a smell. Break it
up or push complexity into a recipe.

### 4.3 Recipes (Layer 3)

Optional, opinionated, composed workflows. The current `tminify()` function —
detect Tailwind CDN, extract inline CSS, compile, inline, minify the result —
becomes a recipe of ~50 lines built on top of the `tailwind` and `html_minifier`
backends.

Recipes can ship in the package (one or two reference recipes) or live in user
code. They benefit from the lower layers but are not architecturally privileged.

---

## 5. Backend roadmap

Phased introduction. Order = priority:

| # | Backend | Install | Why |
|---|---|---|---|
| 1 | Tailwind | Native binary (`@tailwindcss/standalone`) | Original use case; kills Node dep on the Tailwind path |
| 2 | `html-minifier-next` | npm | Replaces unmaintained html-minifier-terser; only remaining Node dep |
| 3 | esbuild | Native binary | Most useful JS bundler/minifier |
| 4 | lightningcss | Native binary | Standalone CSS minifier; complements Tailwind |
| 5 | oxc-minify | Native binary | Fastest JS minifier; Vite's default |
| 6 | swc | Native binary | TS / JSX transpilation; broader use cases |

Backends 1 and 2 are sufficient to preserve current functionality. 3 onward are
extensions. After backend 1, the Tailwind path no longer requires Node at all.

---

## 6. Major adjustments and simplifications from current code

### 6.1 Correctness fixes (do these regardless of redesign)

- **Drop `tailwind.config.js` generation entirely.** Use `@source` in CSS only.
- **Fix `--preflight`** by switching to selective `@import "tailwindcss/theme.css"`
  + `@import "tailwindcss/utilities.css"` instead of `@import "tailwindcss"`.
- **Commit `package-lock.json`** for any remaining npm-installed tools; switch
  `npm install` → `npm ci`.
- **Add subprocess timeouts** on every `subprocess.run`.
- **File-lock the install path** to fix the `get_js_dir` race.
- **Up-front tool availability check** for both Tailwind and minifier paths.

### 6.2 Structural simplifications

- **Move install location** from `site-packages/.../node_modules` to
  `user_cache_dir("<pkg>") / <tool>-<version>`.
- **Replace regex HTML surgery** in the Tailwind recipe with a real parser
  (`selectolax` preferred for speed; `BeautifulSoup` if a pure-Python dep is
  required).
- **Replace `html-minifier-terser`** with `html-minifier-next` as the default HTML
  minifier backend.
- **Use `--ignore-custom-comments`** for the generated-by tag rather than
  post-minification rewriting.
- **Inline `atomic_output_file`** (drop `strif` dep); inline format helpers (drop
  `prettyfmt` dep). Both are tiny and reduce dep surface.

### 6.3 Surface polish

- Rename `--no_minify` → `--no-minify` (keep old form as alias for one release).
- Rename `--tailwind` → `--force-tailwind` (clearer semantics).
- Fix log levels: routine progress at INFO, real warnings at WARNING.
- Drop the `<4.0` upper bound on `requires-python`.

### 6.4 Tests

- Split `test_cli_comprehensive` into ~12 named tests using `pytest`'s `tmp_path`
  fixture.
- Convert most tests to in-process calls to `tminify(...)`; keep one or two
  subprocess tests for the CLI surface.
- Matrix CI across Linux + macOS + Windows; pin Node version.

### 6.5 What we drop

- The `corePlugins` config dict (broken in v4 anyway).
- The legacy JS-config bridge (use `@source` directly).
- The `delete=False` named tempfile pattern (use `with` blocks).
- Implicit dependency on whatever Node version the runner has.

---

## 7. Migration path

Phased to keep the package usable at every step:

### Phase 0: Fix correctness in place

Branch off, ship a `0.2.0` that fixes preflight, lockfile, timeouts, race, and the
v4-config legacy. No public API change. Drops users out of the silent-bug zone
immediately.

### Phase 1: Extract core

Internal refactor: pull install/cache/run primitives into a `core/` module. Convert
the existing Tailwind + html-minifier-terser logic into thin internal users of
core. No user-visible change.

### Phase 2: Native-binary Tailwind backend

Ship a `tailwind` backend that uses `@tailwindcss/standalone`. Default to it. The
npm path remains as fallback for one release. After this phase, users without Node
can use the Tailwind path.

### Phase 3: Switch HTML minifier

Replace `html-minifier-terser` with `html-minifier-next` in the default minifier
backend. One-line dep change at the JS layer; no user-visible difference.

### Phase 4: Public API

Expose `tailwind`, `html_minify` as documented backends. Move the existing
`tminify(src, dest)` function to `recipes.tminify_html(src, dest)`. Keep a
top-level alias for compatibility.

### Phase 5: Rename (if doing it)

Rename the package. Old `tminify` becomes a thin shim that re-exports from the new
package and prints a deprecation notice on import.

### Phase 6: Add backends

esbuild, lightningcss, oxc-minify, swc — one per release, each ~a day of work given
the core abstraction.

---

## 8. Open design questions

These need decisions before Phase 1 lands.

1. **Naming.** What's the package called? Affects PyPI registration, docs, and
   rename timing.
2. **Single package vs split.** Ship `<core> + <all backends>` as one package, or
   split (`<core>`, `<core-tailwind>`, `<core-esbuild>`, …)? Single is simpler to
   start; splitting later is straightforward.
3. **Backend API style.** Function-style (`tailwind.compile(css, sources=...)`) or
   class-style (`Tailwind(version=...).compile(...)`)? Function-style is friendlier
   for the common case; class-style allows version overrides.
4. **Configuration.** Per-call kwargs only, or also a project-level config
   (`pyproject.toml [tool.<pkg>]`)?
5. **Binary trust.** SHA256 from the GitHub release manifest is the realistic
   floor. Sigstore / cosign verification is the upper bound — probably overkill for
   this scope, but worth deciding once.
6. **Recipe surface.** How many recipes ship in-package? Just `tminify_html`? Or
   also a few obvious ones (`bundle_js`, `minify_css`)?
7. **Watch mode.** Tailwind, esbuild, swc all have native watch. Does the harness
   expose a unified watch API, or is that a recipe concern?
8. **Windows binaries.** All target tools ship Windows binaries, but path/quoting
   bugs are the usual landmine. Test matrix priority.

---

## 9. References

### tminify (this repo)

- Source: `src/tminify/main.py`, `src/tminify/cli.py`
- JS deps: `src/tminify/javascript/package.json`
- Tests: `tests/test_cli.py`
- Prior code review: in conversation history on this branch.

### Tailwind CSS

- [Tailwind v4.0 release post (Jan 2025)](https://tailwindcss.com/blog/tailwindcss-v4)
- [Tailwind CLI docs](https://tailwindcss.com/docs/installation/tailwind-cli)
- [Standalone CLI announcement (no Node.js)](https://tailwindcss.com/blog/standalone-cli)
- [`tailwind.config.js` removal in v4 (Discussion #17168)](https://github.com/tailwindlabs/tailwindcss/discussions/17168)
- [Disabling Preflight on Tailwind v4 (Discussion #17481)](https://github.com/tailwindlabs/tailwindcss/discussions/17481)
- [Cannot disable preflight styles (Issue #15723)](https://github.com/tailwindlabs/tailwindcss/issues/15723)
- [Standalone CLI tutorial (Discussion #15855)](https://github.com/tailwindlabs/tailwindcss/discussions/15855)
- [Tailwind v4 deep dive: Oxide engine (DEV, 2026)](https://dev.to/dataformathub/tailwind-css-v4-deep-dive-why-the-oxide-engine-changes-everything-in-2026-2595)
- [Tailwind v4 migration guide (2026)](https://dev.to/pockit_tools/tailwind-css-v4-migration-guide-everything-that-changed-and-how-to-upgrade-2026-5d4)
- [Tailwind release notes — April 2026](https://releasebot.io/updates/tailwind)

### JavaScript minifier ecosystem

- [terser/html-minifier-terser](https://github.com/terser/html-minifier-terser) — original tool
- [j9t/html-minifier-next](https://github.com/j9t/html-minifier-next) — active fork
- [wilsonzlin/minify-html](https://github.com/wilsonzlin/minify-html) — Rust HTML minifier
- [JS minification benchmarks (privatenumber)](https://github.com/privatenumber/minification-benchmarks)
- [Oxc Minifier alpha announcement](https://oxc.rs/blog/2025-03-13-minifier-alpha)
- [html-minifier-terser Snyk page](https://security.snyk.io/package/npm/html-minifier-terser)
- [Top HTML minifiers (2025 survey)](https://listofaitools.com/tool/html-minifier/)

### Build tooling

- [esbuild](https://esbuild.github.io/) — single-binary distribution
- [swc](https://swc.rs/) — Rust JS compiler
- [oxc](https://oxc.rs/) — JS oxidation compiler
- [Lightning CSS](https://lightningcss.dev/) — Rust CSS bundler/minifier
- [Vite build options](https://vite.dev/config/build-options) — current default minifiers reference

### Python ecosystem patterns

- [platformdirs](https://github.com/platformdirs/platformdirs) — cross-platform cache location
- [pre-commit](https://pre-commit.com/) — Python tool that manages multi-language toolchains
- [nodeenv](https://github.com/ekalinin/nodeenv) — Node-via-Python install
- [uv](https://docs.astral.sh/uv/) — packaging baseline

---

*This plan is a living document. Edit freely; commit decisions back as they are
made. The "Open design questions" section is the working agenda.*
