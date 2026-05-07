# Lab: Caching Dependencies — Speed Up Jobs with actions/cache

## Goal

Build a three-job GitHub Actions workflow for the **Harmonic Haven** React app.
You'll learn how to cache `node_modules` between jobs so that downstream jobs skip `npm ci` entirely.

- The `install` job runs `npm ci` once and saves `node_modules` to the cache.
- The `build` job restores the cache and runs `npm run build` — no install needed.
- The `test` job restores the cache and runs `npm test` — no install needed.

---

## Getting Started

```bash
npm install
npm run build      # compiles to dist/
npm test           # runs vitest with coverage
```

---

## Core concepts

### Why cache `node_modules`?

`npm ci` on every job takes 30–60 seconds. A cache hit takes under 5 seconds.
On a real project with hundreds of dependencies this multiplies across every push and PR.

### How `actions/cache` works

The action runs in two phases automatically:

1. **Restore phase** (start of job): looks up `key` in the cache store, extracts files to `path`
2. **Save phase** (end of job): if the key was new, saves `path` back to the store

Only the job that writes the cache needs to run `npm ci`. Downstream jobs just restore and use.

### Cache keys and `hashFiles()`

The key must be deterministic. `hashFiles('**/package-lock.json')` returns a SHA-256 of the lock file.
When the lock file changes, the key changes, the cache misses, and `npm ci` runs again to write a fresh entry.

### `restore-keys` fallback

When the exact key misses, `restore-keys` are tried as prefix matches.
You get a partially valid `node_modules` rather than nothing — better than a cold start.

### Skipping `npm ci` on a cache hit

The `actions/cache` step exposes an output: `cache-hit`. Use it to skip install:

```yaml
- name: Install dependencies
  if: steps.cache.outputs.cache-hit != 'true'
  run: npm ci
```

Give the cache step an `id:` so you can reference its output.

---

## Workflow shape

```
install  →  (cache node_modules)
              ↓               ↓
           build            test
        (restore cache)  (restore cache)
        npm run build      npm test
```

---

## Acceptance criteria

The workflow you write must:

- [ ] Trigger on `push` to any branch and on `pull_request`
- [ ] Have an `install` job that caches `node_modules` using `actions/cache@v4`, with key `${{ runner.os }}-node_modules-${{ hashFiles('**/package-lock.json') }}` and a `restore-keys` fallback, and runs `npm ci` only on a cache miss
- [ ] Have a `build` job that declares `needs: install`, restores the cache with the same key pattern, and runs `npm run build` — no `npm ci`
- [ ] Have a `test` job that declares `needs: install`, restores the cache with the same key pattern, and runs `npm test` — no `npm ci`
- [ ] On a second push with no `package-lock.json` changes, all three jobs show a cache hit in their logs

---

## Hints

- Give the cache step in `install` an `id:` (e.g., `id: cache-node-modules`) so you can reference `steps.cache-node-modules.outputs.cache-hit`
- Every job starts on a fresh runner — always add `actions/checkout@v4` before the cache restore step
- `build` and `test` do not need the `if:` condition — they're readers, not writers; a partial restore is fine
- Watch the "Cache" step logs: you'll see `"Cache restored successfully"` on a hit and `"Cache not found for key..."` on a miss
- Do **not** use `setup-node`'s built-in `cache:` input — it uses `~/.npm`, not `node_modules`, and still runs `npm ci`

---

## Where the workflow goes

Create your workflow at `.github/workflows/ci.yml`.
