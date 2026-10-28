# Bruno Automation Webinar

Companion workspace for the **Automation Deep Dive** segment of the Bruno
community webinar. One collection, run three ways:

| Where | How |
|---|---|
| In the app | open the workspace, pick the `staging` environment, hit Send or use the Collection Runner |
| In the terminal | `bru run --env staging --exclude-tags data-driven` from the collection folder |
| In CI | [`.github/workflows/api-tests.yml`](.github/workflows/api-tests.yml) runs it on every pull request and push to `main` with the official `usebruno/bruno-cli-action` |

Same files, no export, no translation step. The script you write in the
request editor is the script CI runs.

## Get started

```bash
git clone https://github.com/bruno-collections/bruno-automation-webinar.git
cd bruno-automation-webinar
```

**App:** Bruno → *Open Workspace* → choose this folder. Select the `staging`
environment in the top-right dropdown and send `auth/login`.

**CLI** (Node.js 18+, no desktop app needed):

```bash
npm install -g @usebruno/cli
cd collections/orders-api
bru run --env staging --exclude-tags data-driven
```

The collection targets [DummyJSON](https://dummyjson.com), a free public fake
API with a real login endpoint and a protected `/auth/me`. No signup, no key.
The `staging` environment holds its published demo credential.

## What's inside

```
workspace.yml                     the Bruno workspace
collections/orders-api/
  opencollection.yml              hoisted auth + shared pre/post scripts
  environments/staging.yml        non-secret config + the public demo password
  environments/ci.yml             same config, password left empty on purpose
  auth/      login, me            pre-request checks, assert + extract, bru.setVar
  orders/    create, list, get    chaining through runtime vars
             create-from-file     data-driven run over data/order-lines.csv
  flows/     Check stock ...      setNextRequest branching, skipRequest, the `slow` tag
data/order-lines.csv              rows for the data-driven request
.github/workflows/api-tests.yml   smoke on every push, regression on main, nightly
WALKTHROUGH.md                    the webinar demo, step by step
```

Every request and folder has a **Docs** tab in the app explaining what it
shows and the command to run it.

## CLI cheat sheet

Run these from `collections/orders-api/`.

| Need | Command |
|---|---|
| Everything | `bru run --env staging --exclude-tags data-driven` |
| Fast gate, stop on first failure | `bru run --env staging --tags smoke --bail` |
| Full regression without slow requests | `bru run --env staging --tags regression --exclude-tags slow` |
| One folder or file | `bru run auth --env staging` / `bru run orders/list.yml --env staging` |
| Override one variable | `bru run flows --env staging --env-var productId=117` |
| Inject a secret the env file leaves empty | `bru run --env ci --env-var password=emilyspass --tags smoke` |
| Data-driven | `bru run orders/create-from-file.yml --env staging --csv-file-path ../../data/order-lines.csv` |
| Only requests with tests | `bru run --env staging --tests-only --exclude-tags data-driven` |
| Reports | see below |

Multiple tags go in one comma-separated value (`--exclude-tags slow,data-driven`),
not repeated flags.

### Reports and exit codes

```bash
bru run --env staging --tags regression --exclude-tags slow \
  --reporter-junit ../../reports/results.xml \
  --reporter-html  ../../reports/results.html \
  --reporter-json  ../../reports/results.json \
  --reporter-skip-headers Authorization Cookie Set-Cookie \
  --reporter-skip-response-body

echo $?   # 0 = all passed, 1 = a request, test or assertion failed
grep -c eyJ ../../reports/results.json   # 0 = no token in the report
```

JUnit for the CI system, HTML for people, JSON for dashboards, all from one
run. The exit code is what fails the build; CI parses nothing.

Skipping `Authorization` alone is not enough. The token also travels in the
login response body, its `Set-Cookie` headers, and the `Cookie` header sent
back on later calls. The flags above drop all of them, at the cost of response
bodies in the report. The `grep` looks for `eyJ`, the start of every JWT, so
it checks for the token itself rather than a header name.

## CI

[`api-tests.yml`](.github/workflows/api-tests.yml) has three jobs:

- **smoke** on every pull request and push to `main`: `--tags smoke --bail`.
- **regression** on pushes to `main` and manual runs: full suite, three
  reporters, reports uploaded as a build artifact. `workflow_dispatch` accepts
  a `base-url` input so you can point a run at a preview deployment.
- **nightly** on a schedule (or manual run): adds the `slow` requests and the
  data-driven run.

Every job pins the CLI with `bru-version`, so CI runs the same version your
team runs locally. Bump it deliberately, not by surprise.

Secrets never touch the repo. The `ci` environment commits the non-secret
config and leaves `password` empty; the pipeline injects it with
`--env-var password=${{ secrets.API_PASSWORD }}`. Without it, `auth/login`'s
pre-request script throws before the call is sent, so a missing secret is a
loud failure rather than a mysterious 400. Because this demo backend uses a
public credential, the workflow falls back to it so forks stay green. Remove
that fallback for a real API.

GitLab CI and Jenkins use the same CLI flags; only the YAML around them changes.

## Links

- Bruno CLI: https://docs.usebruno.com/bru-cli/overview
- Scripting reference: https://docs.usebruno.com/testing/script/getting-started
- GitHub Action: https://github.com/usebruno/bruno-cli-action
- Bruno: https://www.usebruno.com
