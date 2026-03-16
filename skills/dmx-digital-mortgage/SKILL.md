---
name: dmx-digital-mortgage
description: Fill out and navigate a live DMX digital mortgage application through the shadow-cljs nREPL. Use this whenever the user asks to fill out a DM, complete a DM, fill out a digital mortgage, walk through a mortgage application, continue a DMX application, inspect the current DMX panel, fill fields or whole panels, move forward/back through the application, or drive a smoke-style DMX walkthrough from browser state. Treat “DM” in the DMX repository as “digital mortgage” and proactively use this skill even if the user does not explicitly mention nREPL, panels, forms, or `resources/smoke/inputs.edn`.
---

# DMX Digital Mortgage Application Driver

Use this skill to drive a live DMX application one panel at a time through the browser-backed shadow-cljs REPL.

If the user asks to fill out an application, continue through the application proactively without pausing for confirmation between panels. When you hit an error panel, a stuck transition, or a recoverable failure such as credit pull failure, use the panel's retry path or navigate back and resubmit, then continue forward automatically. Keep going until the application is complete, genuinely blocked after reasonable retry attempts, the browser session disconnects, or the user explicitly asks you to stop.

## What matters

- The live client state is in `dm-client.store/store`
- Update state through `dm-client.store/dispatch!`
- Navigation helpers live in `dm-client.user-actions`
- Prefer values from `resources/smoke/inputs.edn` whenever possible
- Treat smoke inputs as a lookup source for the active panel, not a batch payload for future panels
- Determine company, environment, and tenant from the running app before choosing config files
- Prefer filling applications with a loan officer selected when that path is available, because DMX is used primarily by retail loan officers
- Always inspect the current panel before filling it
- Fill only fields that are currently visible on the active panel
- When a panel has safe smoke-backed values for all currently visible required fields, fill it promptly and advance
- After navigation, wait for network delay before concluding that the panel did or did not change

## Core workflow

### 1. Connect to the live CLJS REPL

First use the `clojure-eval` skill.

Typical sequence:

```bash
clj-nrepl-eval --discover-ports
clj-nrepl-eval -p <PORT> "(shadow/repl :app)"
```

If the session falls out of CLJS mode, reset and re-enter:

```bash
clj-nrepl-eval -p <PORT> --reset-session "(shadow/repl :app)"
```

If you see `No available JS runtime`, the browser runtime disconnected. Reconnect to the live browser-backed `:app` build before continuing.

### 2. Inspect the current panel before changing anything

Use the live store, selectors, and visible-field logic:

```clojure
(require '[dm-client.store :as store]
         '[dm.selectors :as selectors]
         '[dm.field-values :as field-values]
         :reload)

(let [state @store/store
      form-index (selectors/get-current-form state)]
  {:current-form (:current-form state)
   :form-index form-index
   :visible-fields (vec (field-values/get-visible-fields state))
   :field-values (get-in state [:field-values form-index])})
```

This is the default source of truth.

### 3. Choose values

Prefer `resources/smoke/inputs.edn` first.

Use smoke data narrowly:

- start from the current panel's `visible-fields`
- choose values for the currently visible fields on that panel
- treat smoke inputs as a lookup table for the active panel
- after any field update that changes branching, re-read `visible-fields` and then fill any newly visible fields on that same panel

Read the relevant entries for the current form, usually from:

- `:default`
- `:ef`
- `:purchase`, `:refinance`, or other scenario-specific keys
- borrower/co-borrower fixture keys when identity or credit fields are needed

If the smoke file does not cover the current panel, inspect:

- `flow/fields.cljc` for valid field option values
- `flow/panels.cljc` for required/visible fields and dynamic branches

Read field definitions for enum and select values whenever smoke inputs do not already provide them.

### 4. Fill the current panel

For a whole-panel update, prefer `:merge-field-values`:

```clojure
(require '[dm-client.store :as store]
         '[dm.selectors :as selectors]
         :reload)

(let [form-index (selectors/get-current-form @store/store)]
  (store/dispatch!
   [:merge-field-values {:form-index form-index
                         :field-values {:some-field "value"
                                         :other-field 123}}]))
```

Important guardrails:

- apply the smallest field map that fully covers the current visible panel
- write values into the active form index for the fields the user can currently see
- if only 2 of 8 smoke values are visible right now, fill those 2
- after filling, inspect `visible-fields` again before deciding what else belongs on the current panel

For a single field:

```clojure
(store/dispatch!
 [:set-field-value {:form-index form-index
                    :field-key :some-field
                    :value "value"}])
```

After filling, re-read the current form values from `@store/store` to verify the update landed.

### 5. Advance one panel at a time

Use the app’s own navigation helper:

```clojure
(require '[dm-client.user-actions :as actions] :reload)
(store/dispatch! actions/navigate-next)
```

Then wait for async work to settle before checking the result. In practice, re-check after roughly 3-6 seconds first, then allow more time if the transition is clearly server-backed or still loading.

Re-read state after waiting:

```clojure
(let [state @store/store]
  {:current-form (:current-form state)
   :ui-state (:ui-state state)
   :async-worker-count (:async-worker-count state)})
```

If the panel did not change immediately, give async work time to settle and then confirm the next panel from live state.

### 6. Recover from stuck panels and retryable errors

Treat recoverable failures as part of the normal walkthrough and keep the application moving.

When navigation appears stuck:

- wait for async work to settle and re-check `:current-form`, `:ui-state`, and `:async-worker-count`
- if the panel is unchanged but still looks valid, try advancing again
- if the panel exposes a clearer recovery path, use that panel-native path

When an error panel or failure state appears:

- inspect the rendered panel and code paths for panel-specific retry behavior
- use panel-native retry actions when they exist, such as dispatching the same action the UI button uses
- otherwise navigate back to the most recent actionable panel, verify required values are still present, and re-submit
- after retrying, keep going automatically if the application resumes

Reasonable retry strategy:

- use 1-2 targeted retries for recoverable failures
- after each retry, wait for async transitions and confirm the new panel
- treat the flow as blocked only after retries fail and there is no panel-specific recovery path left

## Important DMX-specific lessons

### Configuration files

When you need configuration-backed values, derive the correct config files from the live app context.

- company: usually comes from the host, such as `gri` in `http://gri.localhost:8080`
- environment: on localhost this is usually `dev` unless you know otherwise
- tenant: combine them as `company-environment`, for example `gri-dev`

Use those values to choose the matching config files.

Example for `http://gri.localhost:8080`:

- `resources/config/branding/gri.edn`
- `resources/config/env/dev.edn`
- `resources/config/tenants/dev/gri.edn`

When looking up tenant-specific data such as test LO IDs, branding options, feature flags, or environment-dependent behavior, read the files for the active company and environment.

### Loan officer selection

Prefer the loan-officer-selected path when possible, because that better matches normal retail DMX usage.

For `:lo-selection`, use the app's normal LO-selection path and wait for the app to load the authoritative loan-officer record from `/api/loan-officer` before advancing.

Use test LOs for LO-related selection panels. Valid test LO IDs (also referred to as `emp-id` and `loId`) can be found in the active tenant config file at key path `[:loan :test-lo-ids]` (and their corresponding names can be found in the comments).

Verify both:

- field value at `[:field-values {:form :lo-selection :index 0} :lo-selection]`
- top-level `:loan-officer`

Wait until the top-level `:loan-officer` is populated with the server-loaded record for the chosen LO before advancing. Re-check after a short delay and confirm it matches the selected LO.

Prefer a configured test LO when available.

### New-account email addresses

When testing account creation, generate a unique email address for the new account.

Example pattern:

```clojure
(str "john.homeowner." (.getTime (js/Date.)) "@example.com")
```

### Property/location panels

Some location panels are intentionally constrained and expect consistent city/county/state/zip values. Use smoke-backed combinations for those panels.

### Dynamic panels

Some panels may expose different required fields depending on the branch chosen. Re-read visible fields after each meaningful update and then fill the branch that is actually active.

### Summary/router panels

Some panels have no visible editable fields and simply act as routers or summaries. When `visible-fields` is empty, advancing is often the correct action.

## Quirks / gotchas

### `:lo-selection` requires a specific sequence

The LO-selection flow is easy to get wrong if you only write a raw value into the field.

For a normal retail LO flow:

1. set `:working-with-lo` to `true` so the LO picker becomes visible
2. set `:lo-selection` to a map like `{:displayName "TestLo" :employeeId 12657}` rather than a bare string or number
3. dispatch `[:merge-query-params {:emp-id 12657}]` so the app performs its normal LO-loading path
4. wait for top-level `:loan-officer` to populate before advancing

Set `:lo-selection` to the full LO map shape rather than a bare string or number.

### Loader panels are not the same as router panels

Treat these panel types differently:

- loader panels such as `:credit-pull` and `:submit-voie-order` usually show a loading component and should generally be left alone while async work completes
- router/summary panels such as `:reo-collection-subflow`, `:add-income-information`, or `:add-asset-information` often have no visible editable fields and are good candidates for `navigate-next`

If a panel is visibly loading and `:ui-state` is still active, prefer waiting over dispatching another next action.

### Some async steps take much longer than ordinary navigation

The common 3-6 second re-check window is often too short for:

- account creation
- credit pull
- VOIE submission
- final application submission from `:application-summary`

For those, expect waits closer to 10-20 seconds, and sometimes longer in slower environments.

### `:application-summary` may sit in a long loading state after submission

After dispatching next from `:application-summary`, the app may remain on that same panel while submission is in progress.

Treat this state as in-progress submission work. Re-check:

- `:ui-state`
- `:async-worker-count`
- loan creation data such as top-level `:loan`

Only treat the flow as stuck after a reasonable longer wait and no observable progress.

### EF detection can be inferred from the active panel sequence

If the live flow moves into panels such as `:user-info-ef` instead of `:user-info`, you are in an EF-style path.

When that happens, prefer smoke values from `:ef` in addition to `:default`, and expect EF-specific panel names such as `:user-residence-ef` and `:credit-check-consent-borrower-ef`.

## Recommended helper shape

When doing a longer walkthrough, it is fine to define tiny CLJS helpers in the live REPL, for example:

```clojure
(defn panel-state [] ...)
(defn fill-current! [m] ...)
(defn advance! [] ...)
```

Keep helpers small and always verify state after each step.

## Definition of done for each panel

Before moving on, confirm:

- you know the current panel
- you know the currently visible fields
- values were written into the current form index
- any panel-specific validity signal is present
- navigation finished and the new panel was confirmed after waiting for async work

## Preferred style

- Drive the real browser session, not a fake local simulation
- Use the application’s own `dispatch!` and navigation functions
- Prefer smoke inputs over invention
- Prefer realistic retail flows with an LO attached when the application supports that path
- Move one panel at a time
- Re-check state constantly
- Retry recoverable failures proactively before declaring the flow blocked
- When the user wants the application filled, keep going panel-to-panel without asking "continue?" or requesting confirmation after each successful step, including after successful retries
