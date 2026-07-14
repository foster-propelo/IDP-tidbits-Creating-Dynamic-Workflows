# IDP Tidbit: Use and Create Dynamic Pickers in an IDP Workflow

Harness University Tidbit repo. This walks through building an IDP Workflow input that
populates its own dropdown at runtime, using two different sources:

1. **Built-in Harness pickers** — zero-config pickers backed by your Harness account/catalog.
2. **A REST API-backed picker** — a dropdown fed by a live call to an external/internal API.

## Why Dynamic Pickers?

A static `enum` dropdown in a Workflow form goes stale the moment the underlying data
changes. Dynamic Pickers solve that by fetching valid options at runtime, so users always
pick from current, valid values — fewer failed workflow runs from bad/expired input.

Reference docs:
- [Dynamic Workflow Picker based on API Response](https://developer.harness.io/3k-docs/internal-developer-portal/flows/workflows-tutorials/dynamic-picker/) — the API picker mechanics, backend proxy setup, and response-parsing options (`valueSelector`, `labelSelector`, `arraySelector`).
- [Configuring Workflow Inputs — Harness-specific UI Pickers](https://developer.harness.io/3k-docs/internal-developer-portal/flows/create-workflow/flows-input#harness-specific-ui-pickers) — the full list of built-in pickers (`HarnessOrgPicker`, `HarnessProjectPicker`, `HarnessAutoOrgPicker`, `HarnessUserGroupPicker`, `HarnessOwnerPicker`, plus the catalog-driven `EntityFieldPicker`/`EntityPicker`).


## Prerequisites

- A Harness account with the IDP module enabled.
- For the API picker: an HTTP endpoint you can query (your own service, or any public
  test API) that returns either a plain array or an array of objects. Our examples use a public API, however we encourage you to replace this. 

## Step 1 — Set up the Backend Proxy (API picker only)

The API picker needs a Backend Proxy so the Workflow frontend can call a third-party API
without exposing credentials in the browser.

1. In IDP, go to **Configure → Plugins → Plugin Marketplace → Configure Backend Proxies**.
2. Paste in `proxy-config/backend-proxy-config.yaml`, swapping `target` for your API's base
   URL and setting the `PROXY_IDP_TIDBIT_TOKEN` secret if your API needs auth.
3. Save, then sanity-check it by hitting
   `https://idp.harness.io/<ACCOUNT_ID>/idp/api/proxy/idp-tidbit-api/<some-path>`.

> Built-in Harness pickers (Step 2) don't need any of this — they authenticate using the
> logged-in user's own session.

## Step 2 — Import the Workflows

Add each YAML in `workflows/` as a Workflow in IDP (**Create → Workflow**, paste the YAML,
or push it through your Git provider if using Git Experience):

- **`api-picker-workflow.yaml`** — a dropdown (`api_choice`) sourced entirely from your
  proxied API. Includes a conditional request: the API path filters on an earlier
  `category` input.
- **`builtin-picker-workflow.yaml`** — `HarnessProjectPicker`, `HarnessAutoOrgPicker`,
  `HarnessUserGroupPicker`, and `HarnessOwnerPicker`, all populated straight from your
  Harness account with no proxy required.
- **`combo-workflow.yaml`** — both approaches in one form, useful for contrasting them
  directly.

Each workflow's `steps.trigger` calls a Harness Pipeline and forwards the selected values
as `inputset`. Replace `<YOUR_PIPELINE_URL>` with the pipeline you import in Step 3.

## Step 3 — Import the Pipeline

Import `pipelines/echo-variables-pipeline.yaml` into your project. It's intentionally
minimal: a Custom stage with one Shell Script step that echoes every variable the
Workflow passed in, so you can confirm end-to-end that the picker's selected value made
it through. Point each workflow's `steps.trigger.input.url` at this pipeline once
imported.

## Step 4 — Run It

1. Open the Workflow in IDP's self-service catalog.
2. Fill out the form — for the API picker, confirm the dropdown options match what your
   API returns; for the built-in pickers, confirm they reflect your actual Harness
   account data.
3. Submit, then check the pipeline execution logs for the `echo` output confirming the
   values arrived correctly.

## Notes for Contributors / Swapping In Your Own Setup

- Replace `<your-api-host>` in `backend-proxy-config.yaml` with your real API.
- Replace `<YOUR_PIPELINE_URL>`, `<YOUR_PROJECT_ID>`, and `<YOUR_ORG_ID>` placeholders
  with values from your own Harness account.
- No PATs/tokens are committed anywhere in this repo — use Harness secrets for the proxy
  token and your own PAT when testing the `HarnessAuthToken` field.
