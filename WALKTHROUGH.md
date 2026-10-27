# Walkthrough

The webinar demo, in order, so you can follow along live or replay it
afterwards. About 20 minutes. Every step names the request to open and the
exact command to run.

Before you start: open the workspace in Bruno, select the **staging**
environment, and have a terminal open in `collections/orders-api/`.

## 1. One collection, three places it runs (2 min)

Open the collection and show the folder tree. Open `opencollection.yml` in an
editor, or the collection **Settings → Script** tab. Point out:

- Auth is set once, at collection level: Bearer `{{accessToken}}`.
- The collection pre-request script stamps `X-Request-Id` and `X-Timestamp`
  on every call. Written once, inherited by every request.

Nothing else needs to be said yet. The rest of the demo proves the claim.

## 2. Pre-request: set up the call before it leaves (3 min)

Open **auth/login**.

- **Script tab, pre-request**: reads `username` and `password` from the
  environment and throws if either is missing. Inputs come from the right
  scope; nothing is pasted in.
- **Body**: `{{username}}` / `{{password}}` interpolated from the environment.

Send it. Then open **orders/create** and show its pre-request script building
an `Idempotency-Key` at send time. Send it, open the **Timeline** tab and
point at the generated headers: `Idempotency-Key`, `X-Request-Id`,
`X-Timestamp`.

## 3. Post-response: assert, extract, chain (4 min)

Back to **auth/login**:

- **Tests tab**: three `test()` blocks. **Assertions tab**: the one-liners.
- **Script tab, post-response**: `bru.setVar("accessToken", ...)` and
  `bru.setVar("userId", ...)`. Runtime only. Never written to disk.

Now **auth/me**: it has no auth of its own (`inherit`), no headers of its own,
and it returns 200 with the user that just logged in. The token flowed through
the collection-level auth.

Then **orders/create → orders/list → orders/get** in order. Each one reads
what the previous one extracted (`{{userId}}`, then `{{existingOrderId}}`).
Three requests in, nothing was typed by hand.

Optional, if there is time: collection **Settings → Auth → No Auth**, re-send
`auth/me`, watch it 401 and the test go red. Switch it back.

## 4. Patterns that scale (3 min)

Three things to point at, no new requests to send:

1. **Hoisting**: `orders/folder.yml` has a folder-level pre-request script.
   If there is no token in scope, it runs `bru.runRequest("auth/login")`.
   That is why `bru run orders` works on its own.
2. **Branching**: open the **flows** folder. `Check stock` calls
   `bru.runner.setNextRequest()` based on the response. `Notify out of stock`
   and `Nightly report` drop themselves with `bru.runner.skipRequest()`.
3. **Tags**: `smoke`, `regression`, `data-driven`, `slow` in each request's
   Settings tab. The runner and the CLI select by them.

Run the flows folder in the **Collection Runner** once with `productId=1`
(default) and once after changing it to `117` in the environment. Different
requests light up.

## 5. The CLI (4 min)

Switch to the terminal, already in `collections/orders-api/`.

```bash
bru run --env staging --exclude-tags data-driven
```

Same scripts, same tests, same console lines, no export. Point at the summary
table and the exit code:

```bash
echo $?
```

Then the flags people actually use, one at a time:

```bash
bru run --env staging --tags smoke --bail
bru run flows --env staging --env-var productId=117
bru run orders/create-from-file.yml --env staging --csv-file-path ../../data/order-lines.csv
```

The data-driven one shows the iteration table: one request file, three rows,
three runs.

## 6. Reports and exit codes (2 min)

```bash
bru run --env staging --tags regression --exclude-tags slow \
  --reporter-junit ../../reports/results.xml \
  --reporter-html  ../../reports/results.html \
  --reporter-json  ../../reports/results.json \
  --reporter-skip-headers Authorization
open ../../reports/results.html
```

Three formats from one run. Open the HTML report. Then show the secret
scrubbing worked:

```bash
grep -ci '"authorization"' ../../reports/results.json   # 0
```

## 7. GitHub Actions (3 min)

Open [`.github/workflows/api-tests.yml`](.github/workflows/api-tests.yml).

- `usebruno/bruno-cli-action@v1`: `command: run --env ci ...`, no leading
  `bru`.
- `--env-var password=${{ secrets.API_PASSWORD }}`: the secret lives in the CI
  vault, the `ci` environment commits only non-secret config.
- Show that the missing secret fails loudly, locally:

  ```bash
  bru run --env ci --tags smoke --bail           # pre-request script throws
  bru run --env ci --tags smoke --bail --env-var password=emilyspass   # green
  ```

- Three jobs: smoke on every push, regression on main with the HTML report as
  an artifact, nightly with the slow and data-driven runs.

If the repo is on GitHub, open the Actions tab and show a green run and the
uploaded report artifact.

## 8. Recap (1 min)

- **Scripts**: compute inputs pre-request, assert and extract post-response,
  hoist shared logic, keep secrets in runtime vars.
- **CLI**: `bru run --env <name>` with tags, data files and reporters.
  Non-zero exit on failure.
- **CI**: the official action plus the pipeline's secret store. Smoke on every
  push, full regression on merge.

One collection on disk, zero translation.
