# ai-api-testing

Automated API regression for the public [Swagger Petstore](https://petstore3.swagger.io/) API
(`https://petstore3.swagger.io/api/v3`), written as a Postman collection and run with Newman.

## What is tested

One CRUD scenario for `/pet`, run in this order:

| # | Request | Endpoint | Checks |
|---|---------|----------|--------|
| 1 | Create pet | `POST /pet` | status 200, JSON, Pet schema, response equals sent data |
| 2 | Get pet | `GET /pet/{petId}` | status 200, JSON, Pet schema, data equals created pet |
| 3 | Update pet | `PUT /pet` | status 200, JSON, Pet schema, new `name` and `status` returned |
| 4 | Get pet after update | `GET /pet/{petId}` | status 200, JSON, Pet schema, update was saved |
| 5 | Delete pet | `DELETE /pet/{petId}` | status 200 |
| 6 | Verify deletion | `GET /pet/{petId}` | status 404 |

Expected status codes and the Pet schema are taken from the Petstore OpenAPI
(`https://petstore3.swagger.io/api/v3/openapi.json`).

How the collection is built:

- `baseUrl` is a collection variable (`https://petstore3.swagger.io/api/v3`).
- `petId` is generated unique on every run in the pre-request script of *Create pet* — no fixed test data.
- The request body and the expected result are built from the same object (`expectedPet`),
  so every later step compares the API response with what was actually sent.
- The collection cleans up after itself: step 5 deletes the created pet, step 6 confirms it is gone
  and clears the run-time variables.

The requests depend on each other, so run the whole collection in order, not single requests.

## Project structure

```
collections/petstore-collection.json   Postman collection (v2.1)
results/                               machine-readable run results (JSON), not committed
reports/                               human-readable reports (HTML), not committed
```

## Run in GitHub Actions

Workflow: `.github/workflows/newman.yml` — **API regression (Newman)**.

| Trigger | Jira check | Regression |
|---|---|---|
| push to `main` (`collections/**` or the workflow changed) | no | always |
| manual run, `jira_gate` unchecked | no | always |
| manual run, `jira_gate` checked | yes | only if Jira has matching issues |
| weekly, Monday 07:00 UTC (`schedule`) | yes | only if Jira has matching issues |

Manual run: **Actions → API regression (Newman) → Run workflow**.

Jobs:

1. `jira-gate` — only for the weekly run and a manual run with `jira_gate`:
   - fails if the secrets `JIRA_EMAIL` / `JIRA_API_TOKEN` are missing (the run is not skipped silently);
   - checks the login with `GET /rest/api/3/myself` and fails unless it is a real active Atlassian user —
     Jira can answer anonymously (HTTP 200 and 0 issues) when credentials are wrong;
   - searches with `POST /rest/api/3/search/jql`:
     `project = DS AND labels = "api-testing" AND status = "IN QA"`
     and writes the login and the found issues (with links) to the run summary.
2. `newman` — runs the collection with reporters `cli` (summary table in the log), `json` and `htmlextra`.
   Skipped when the gate found no issues; never runs after a failed gate. A failed assertion makes the run red.
   Artifacts (30 days): `newman-json-report`, `newman-html-report`.
3. `publish-report` — publishes the HTML report to GitHub Pages: on `main` only, also when assertions fail,
   and only when `newman` really ran and produced the report.

### Jira secrets

Repository secrets (never in code). With [GitHub CLI](https://cli.github.com/):

```bash
gh secret set JIRA_EMAIL -R Shoolia/ai-api-testing
gh secret set JIRA_API_TOKEN -R Shoolia/ai-api-testing
```

- `JIRA_EMAIL` — the e-mail you sign in to Atlassian (ds-test-rd.atlassian.net) with.
- `JIRA_API_TOKEN` — created at https://id.atlassian.com/manage-profile/security/api-tokens
  with **Create API token** (not "with scopes"). Tokens expire: create a new one and run
  `gh secret set JIRA_API_TOKEN` again before the expiry date.

Note: GitHub disables scheduled workflows in public repositories after 60 days without
repository activity; re-enable it on the Actions tab if that happens.

## HTML report

Latest report from `main`: **https://shoolia.github.io/ai-api-testing/**

The page is replaced on every run on `main`. Reports of older runs stay available as the
`newman-html-report` artifact on each run's page.

One-time setup (repository admin): **Settings → Pages → Build and deployment → Source: GitHub Actions**.

The report shows request and response bodies. That is fine for the public Petstore; for an API with
personal data or tokens add `--reporter-htmlextra-skipSensitiveData` (or `--reporter-htmlextra-skipHeaders "Authorization"`),
because a GitHub Pages site of a public repository is visible to everyone.

## Run locally

Requires [Node.js](https://nodejs.org/) (LTS).

```bash
npm install -g newman
newman run collections/petstore-collection.json
```

Save a JSON result as well:

```bash
newman run collections/petstore-collection.json -r cli,json --reporter-json-export results/newman-report.json
```

With the HTML report (open `reports/index.html` in a browser afterwards):

```bash
npm install -g newman-reporter-htmlextra
newman run collections/petstore-collection.json -r cli,htmlextra --reporter-htmlextra-export reports/index.html
```

Run against another server (the collection variable is overridden for this run only):

```bash
newman run collections/petstore-collection.json --env-var baseUrl=https://your-host/api/v3
```

## Run in Postman

Postman → **Import** → select `collections/petstore-collection.json` → open the collection → **Run**.
Keep the default order of requests.
