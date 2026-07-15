# IDP Tidbit: Use and Create Dynamic Pickers in an IDP Workflow

Harness University Tidbit repo. This walks through building an IDP Workflow input that
populates its own dropdown at runtime, using two different sources:

1. **Built-in Harness pickers** — zero-config pickers backed by your Harness account.
2. **A REST API-backed picker** — dropdowns fed by a live call to an external/internal API.

## Why Dynamic Pickers?

A static `enum` dropdown in a Workflow form goes stale the moment the underlying data
changes. Dynamic Pickers solve that by fetching valid options at runtime, so users always
pick from current, valid values — fewer failed workflow runs from bad/expired input.

Reference docs:
- [Dynamic Workflow Picker based on API Response](https://developer.harness.io/3k-docs/internal-developer-portal/flows/workflows-tutorials/dynamic-picker/) — the API picker mechanics, backend proxy setup, and response-parsing options (`valueSelector`, `labelSelector`, `arraySelector`).
- [Configuring Workflow Inputs — Harness-specific UI Pickers](https://developer.harness.io/3k-docs/internal-developer-portal/flows/create-workflow/flows-input#harness-specific-ui-pickers) — the full list of built-in pickers (`HarnessOrgPicker`, `HarnessProjectPicker`, `HarnessAutoOrgPicker`, `HarnessUserGroupPicker`, `HarnessOwnerPicker`, plus the catalog-driven `EntityFieldPicker`/`EntityPicker`).

## Repo Structure

```
IDP-tidbits-Creating-Dynamic-Workflows/
├── README.md
├── proxy-config/
│   └── backend-proxy-config.yaml     # Backend Proxy setup - powers both API pickers below
├── workflows/
│   ├── api-picker-workflow.yaml      # "API Dropdown Selection" - two API-backed pickers
│   └── built-in-picker.yaml          # "Built In Harness Pickers" - zero-config pickers
└── pipelines/
    └── echo-variables-pipeline.yaml  # Echoes whatever the picker(s) passed in
```

## Prerequisites

- A Harness account with the IDP module enabled.
- For the API picker: our example points at a public mock API
  (`fake-json-api.mock.beeceptor.com`) so you can run it with no setup — swap it for your
  own API whenever you're ready.

## Step 1 — Set up the Backend Proxy (API picker only)

The API picker needs a Backend Proxy so the Workflow frontend can call an API without
exposing credentials in the browser. `proxy-config/backend-proxy-config.yaml` defines
**two** proxy endpoints used by `api-picker-workflow.yaml`:

- **`/test`** — points at a public mock API (`fake-json-api.mock.beeceptor.com`), no auth
  needed. This is what `Generic_API_Picker` uses.
- **`/harness`** — points at `app.harness.io` itself, so you can populate a picker from
  your own Harness entities via API. This is what `Harness_API_Picker` uses, and it
  **requires two placeholders to be filled in**:
  - `${Your_Harness_Secret}` → create a secret with a Harness API key and reference it here.
  - `Harness-Account: YOUR-ACCOUNT-ID` → your Harness account ID.

To set it up:
1. In IDP, go to **Configure → Plugins → Plugin Marketplace → Configure Backend Proxies**.
2. Paste in `proxy-config/backend-proxy-config.yaml`, filling in the two placeholders above.
3. Save

> Built-in Harness pickers (`built-in-picker.yaml`) don't need any of this — they
> authenticate using the logged-in user's own session.

## Step 2 — Import the Pipeline

Import `pipelines/echo-variables-pipeline.yaml` first — both workflows trigger it, and it
just echoes back whatever variables it receives so you can confirm things work end to end.
Fill in before importing:

- `projectIdentifier: YOUR-PROJECT-ID`
- `orgIdentifier: YOUR-ORG-ID`

Once imported, copy its Pipeline Studio URL — you'll need it in Step 3.

## Step 3 — Import the Workflows

Add each YAML in `workflows/` as a Workflow in IDP (**Create → Workflow**, paste the YAML,
or push it through your Git provider if using Git Experience):

- **`api-picker-workflow.yaml`** ("API Dropdown Selection") — two `SelectFieldFromApi`
  pickers: `Generic_API_Picker` (public mock data) and `Harness_API_Picker` (your own
  Harness entities via the `/harness` proxy). Before importing, replace:
  - `url: YOUR-HARNESS-PIPELINE-URL` → the pipeline URL from Step 2.

- **`built-in-picker.yaml`** ("Built In Harness Pickers") — `HarnessProjectPicker`,
  `HarnessAutoOrgPicker`, and `HarnessUserGroupPicker`, all populated straight from your
  Harness account with no proxy required. Before importing, replace:
  - `url: YOUR-PIPELINE-URL-HERE` → the pipeline URL from Step 2.
  - ⚠️ **Watch the `orgId` casing.** The parameter is defined as `orgId`, but the
    `inputset` currently sends `orgID: ${{ parameters.orgID }}` — that property doesn't
    exist, so it resolves to null. It must read:
    ```yaml
    inputset:
      projectId: ${{ parameters.projectId }}
      orgId: ${{ parameters.orgId }}
      userGroup: ${{ parameters.userGroup }}
    ```
    This has to match the pipeline's variable name (`orgId`) too, or the value won't
    make it through even once the workflow side is fixed.

## Step 4 — Run It

1. Open either Workflow in IDP's self-service catalog.
2. Fill out the form — for the API workflow, confirm both dropdowns populate; for the
   built-in workflow, confirm the values reflect your actual Harness account data.
3. Submit, then check the pipeline execution logs for the `echo` output confirming the
   picker values arrived correctly. Note the pipeline's variable names must match what
   each workflow sends — e.g. this pipeline expects `orgId` (not `orgID`); a mismatch here
   is the most common reason a value shows up as `null`.

## Placeholders to Replace (checklist)

| File | Placeholder | Replace with |
|---|---|---|
| `pipelines/echo-variables-pipeline.yaml` | `YOUR-PROJECT-ID` | Your Harness project identifier |
| `pipelines/echo-variables-pipeline.yaml` | `YOUR-ORG-ID` | Your Harness org identifier |
| `proxy-config/backend-proxy-config.yaml` | `${Your_Harness_Secret}` | A Harness secret holding an API key |
| `proxy-config/backend-proxy-config.yaml` | `YOUR-ACCOUNT-ID` | Your Harness account ID |
| `workflows/api-picker-workflow.yaml` | `YOUR-HARNESS-PIPELINE-URL` | Pipeline Studio URL from Step 2 |
| `workflows/built-in-picker.yaml` | `YOUR-PIPELINE-URL-HERE` | Pipeline Studio URL from Step 2 |

No PATs/tokens are committed anywhere in this repo — use a Harness secret for the proxy
key and your own PAT when testing the `HarnessAuthToken` field.
