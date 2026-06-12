# Hello Cribl App

A reference app for developers learning the Cribl App Platform.

This intentionally small app demonstrates the core concepts you need to build real Cribl Apps:
- How an app runs inside Cribl (sandboxed iframe, platform globals)
- How to build interactive UI with React
- How to persist state with the platform KV store
- How to call a read-only Cribl REST API gracefully

---

## Where to start reading

| File | What it shows |
|---|---|
| `src/App.tsx` | The full app — four sections, each teaching one concept |
| `src/kv.ts` | KV store helpers: `kvGet`, `kvSet`, `kvDelete` |
| `src/App.css` | Card-based layout and minimal styles |
| `AGENTS.md` | Platform documentation: globals, KV API, fetch proxy, routing |

---

## How KV persistence works

The Cribl platform provides a scoped key-value store for each app. Reads and writes go through `fetch()` against `window.CRIBL_API_URL` — the platform proxy intercepts these calls and injects auth headers automatically. Your app never handles tokens.

```
// Read a value
GET  ${CRIBL_API_URL}/kvstore/<key>

// Write a value
PUT  ${CRIBL_API_URL}/kvstore/<key>    body: JSON

// Delete a value
DELETE ${CRIBL_API_URL}/kvstore/<key>
```

This app stores two keys:

| Key | Type | Used by |
|---|---|---|
| `counter` | `number` | Interactive counter (Section 2) |
| `settings` | `{ greeting, developerName, favoriteProduct }` | App settings (Section 3) |

KV calls in `kv.ts` wrap each operation in a try/catch. If the call fails (e.g. in local dev mode where `CRIBL_API_URL` is not set), the error is swallowed and the app continues working with in-memory state.

---

## Optional platform API example

Section 4 calls `GET ${CRIBL_API_URL}/master/groups` to list config group IDs. This is a read-only, zero-input call that demonstrates how to use the Cribl REST API from an app.

**This call is gated carefully:**
- It checks that `window.CRIBL_API_URL` is defined before attempting the fetch
- If the response is not `ok` (e.g. 403 Forbidden), it falls through to a friendly amber note
- The note tells the user the rest of the app still works — because it does
- There are no retries

---

## Designing for permission variance

Not all Cribl users have the same role. An app that hard-fails when a low-privilege user opens it is not a good app. Follow these patterns:

1. **Gate optional API calls** — check `CRIBL_API_URL` is set, then check `res.ok` before using the response
2. **Separate required from optional** — KV reads/writes for core app state are low-privilege; surfacing admin data is optional
3. **Show a human note, not an error state** — amber info box instead of a red error
4. **Never block the whole page** — if one section can't load, the others must keep working

---

## Running locally

```bash
npm install
npm run dev
```

In local dev mode, `window.CRIBL_API_URL` is not set, so KV calls silently no-op and all state is in-memory only. The UI is still fully interactive.

To test KV persistence and the platform API section, install the app into a running Cribl instance:

```bash
npm run package          # increments patch version, builds, creates .tgz
npm run package -- --minor   # increment minor version instead
```

Then upload the `.tgz` to your Cribl instance under **Apps**.

---

## Releasing

Releases are driven by Git tags. The workflow at `.github/workflows/release.yml` triggers on any pushed tag matching `v*` — it runs `npm ci`, lints, packages the app at the tag's version (the leading `v` is stripped, so `v1.0.0` → `1.0.0`), and creates a GitHub Release with the built `.tgz` attached and auto-generated notes.

You don't need to bump the version in `package.json` in a commit — the workflow stamps it from the tag name at build time.

To cut a release from a clean `main`:

```bash
git tag v1.0.0
git push origin v1.0.0
```

Then watch the run or open the release page:

```bash
gh run watch
gh release view v1.0.0 --web
```

A few notes if you're adopting this flow in your own app:

- The tag **must** start with `v` (e.g. `v1.0.0`), but the published artifact is versioned without the `v`.
- The release step uses `fail_on_unmatched_files: true`, so the release fails if `npm run package` doesn't produce a `build/*.tgz`. To sanity-check before tagging:

  ```bash
  npm ci && npm run lint && npm run package -- --version 1.0.0
  ls build/*.tgz
  ```

  Discard any local `package.json` version bump from the dry run — only the tagged commit matters to CI.
- If you need to retag, delete the bad tag locally and on the remote first:

  ```bash
  git tag -d v1.0.0
  git push origin :refs/tags/v1.0.0
  ```

---

## Installing from Git

Cribl can install an app directly from source in a Git repository — no need to download the `.tgz` from the GitHub Release.

In the Cribl UI, go to **Apps → Install App** and choose **Import From Git**, then fill in:

| Field | Value |
|---|---|
| **URL** | The HTTPS Git URL of the repo (e.g. `https://github.com/criblapps/cribl-hello-apps.git`) |
| **Branch or tag** | The release tag, e.g. `v1.0.0` |

**The "Branch or tag" field is required.** If you leave it blank, the import will fail. Use the tag you pushed in the [Releasing](#releasing) step (`v1.0.0`, `v1.1.0`, etc.) so installs are pinned to a specific, reproducible version rather than tracking a moving branch.

---

## Project layout

```
src/
  App.tsx       — single-screen app (start here)
  App.css       — styles
  kv.ts         — KV store helpers
  main.tsx      — React entry point (unchanged from scaffold)
config/
  proxies.yml   — external domain allowlist (empty — this app has no external calls)
AGENTS.md       — platform developer documentation
```
