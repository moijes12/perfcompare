# AGENTS.md

Guidance for coding agents working in [mozilla/perfcompare](https://github.com/mozilla/perfcompare).

PerfCompare is a React SPA that compares Firefox performance results between revisions (and over time). It talks to [Treeherder](https://treeherder.mozilla.org) APIs, and can retrigger jobs via Taskcluster after OAuth. User-facing docs: [Firefox Source Docs — PerfCompare](https://firefox-source-docs.mozilla.org/testing/perfdocs/perfcompare.html).

Issues live on **Bugzilla** (`Testing :: PerfCompare`, whiteboard `[pcf]`), not GitHub Issues.

## Stack

- **Node 22** (see `package.json` `engines`; CI uses `cimg/node:22.14`)
- **npm** (lockfile: `package-lock.json`)
- React 19 + TypeScript (strict) + webpack 5 (`webpack/`)
- React Router 8 data APIs (`createBrowserRouter`, route loaders)
- MUI v7 + Emotion, plus some Typestyle in older views
- Redux Toolkit (`src/common/store.ts`, `src/reducers/`)
- Jest 30 + Testing Library + `@fetch-mock/jest`
- ECharts for graphs; `fast-kde` / `src/utils/kde.js` for distribution/mode analysis

Dev server: `npm run dev` → `http://localhost:3000` (`historyApiFallback` is on). Production-style static serve: `npm start` (Express, `server.js`).

## Commands

```bash
npm install
npm run dev              # webpack-dev-server
npm test                 # jest
npm run test:watch
npm run test:coverage
npm run test:update      # refresh Jest snapshots after you verify the UI
npm run lint
npm run lint:fix
npm run format           # prettier --write
npm run format:check
npm run tsc
npm run test-all         # test + format:check + lint + tsc (run before a PR)
npm run fix-all          # snapshot update + format + lint:fix
```

CI (CircleCI: `.circleci/config.yml`) runs coverage tests, ESLint, Prettier, and `tsc` in parallel. PRs also get a Netlify preview deploy.

## Layout

```
src/
  components/            # UI by area: Search, CompareResults, Shared, TaskclusterAuth
  components/App.tsx     # route table (canonical list of URLs)
  components/**/loader.ts  # React Router loaders (fetch + validate query params)
  common/                # store, constants, test-version registry
  common/testVersions/   # Student-T vs Mann-Whitney-U strategies
  logic/                 # Treeherder, Taskcluster, Lando, credential storage
  reducers/              # Redux slices (theme, selected revisions, comparison, column prefs)
  hooks/                 # typed Redux hooks + table/filter/URL helpers
  resources/Strings.tsx  # user-visible copy
  types/                 # CompareResultsItem, MannWhitneyResultsItem, Framework, etc.
  theme/                 # MUI protocol theme (light/dark)
  styles/                # Typestyle leftovers
  utils/
  mockData/              # recorded Treeherder payloads used by tests
  __tests__/             # Jest tests mirroring component areas
    utils/               # render helpers, fixtures, jest setup (not test files)
webpack/
server.js
```

Entry: `src/index.tsx` → Redux `Provider` → `App` → `RouterProvider`.

## Architecture notes

### Routes

Defined in `src/components/App.tsx`. Loaders parse/validate search params and fetch compare data. Views then `useLoaderData()`. Important paths:

| Path | Loader | View |
| --- | --- | --- |
| `/` | `Search/loader.ts` | search / compare form |
| `/compare-results` | `CompareResults/loader.ts` | compare two (or more) revisions |
| `/compare-over-time-results` | `overTimeLoader.ts` | compare over a time range |
| `/subtests-compare-results` | `subtestsLoader.ts` | subtests for one parent signature |
| `/subtests-compare-over-time-results` | `subtestsOverTimeLoader.tsx` | subtests over time |
| `/compare-hash-results`, `/compare-lando-results` | hash/lando loaders | resolve try/Lando ids then reuse results view |
| `/taskcluster-auth` | auth loader | Taskcluster OAuth callback |

Do not import `router` from application code; it is exported for tests only.

### Test versions (comparison statistics)

Compare UI is **strategy-based**, not a pile of `if (testVersion === …)` in every column.

- Union type: `TestVersion` in `src/types/types.ts`
- Registry: `src/common/testVersions/index.ts` (`getStrategy`, `getTestVersionOptions`)
- Implementations: `studentT.tsx`, `mannWhitney.tsx`

To add a test version: new strategy file implementing `TestVersionStrategy`, register it, extend the `TestVersion` union. Do not hard-code dropdown options.

Student-T and Mann-Whitney-U return different result shapes (`CompareResultsItem` vs `MannWhitneyResultsItem`, combined as `CombinedResultsItemType`). Column renderers, expanded rows, and “is this a regression?” all go through the strategy.

### Data fetching

Treeherder lives in `src/logic/treeherder.ts` (`https://treeherder.mozilla.org/api/...`). Compare results use `/api/perfcompare/results/` with `test_version`, `replicates`, and `no_subtests` as relevant. Taskcluster retrigger is `src/logic/taskcluster.ts` plus `Retrigger/` UI.

### State

Keep URL as the source of truth for comparisons (revs, repos, framework, test version, search). Redux is for UI prefs and the search-form selection:

- `theme` — light/dark
- `selectedRevisions` — search form
- `comparison` — in-progress compare form state
- `columnPrefs` — simplified vs advanced columns, warning dismissal, cookies

Use `useAppDispatch` / `useAppSelector` from `src/hooks/app.ts`, not untyped `useDispatch`/`useSelector`.

Copy belongs in `src/resources/Strings.tsx`, not inline (especially user-facing errors, tooltips, modal text).

## Style

- Prettier: `singleQuote`, `jsxSingleQuote`, `trailingComma: all`, `tabWidth: 2`, `endOfLine: lf` (`.prettierrc.js` + `package.json`)
- ESLint flat config: `eslint.config.mjs` (typescript-eslint type-checked, import order, React, Jest/Testing Library on tests)
- Imports: `import/order` with `react` first, then builtin / external / internal, blank lines between groups, alphabetized
- TypeScript `strict` + `strictNullChecks`. Unused args may be prefixed with `_`
- Prefer existing MUI components and patterns (e.g. `Grid` size props, `sx`) over new CSS systems
- Match neighboring file conventions (named exports are common for components)

## Tests

- Location: `src/__tests__/<Area>/<Name>.test.tsx` (and `__snapshots__/`)
- Setup: `src/__tests__/utils/setupTests.ts` — fake timers pinned to `Wed, 09 Oct 2024 12:45:17 GMT`, global `fetchMock.mockGlobal()`, echarts/KDE/Taskcluster Hooks mocked, cookies cleared
- Render: `render` / `renderWithRouter` from `src/__tests__/utils/test-utils.tsx` (Redux, MUI theme, snackbars, Virtuoso mock). Re-exports `screen`, `within`, `fireEvent`, `waitFor`, `act`
- Fixtures: `getTestData()` in `src/__tests__/utils/fixtures.ts`; larger payloads in `src/mockData/`
- Network: mock Treeherder with `fetchMock.get('begin:https://treeherder.mozilla.org/...', …)` (see existing compare/search tests). Unmocked requests 404 via `fetchMock.catch(404)`
- Fake timers: `userEvent.setup({ advanceTimers: jest.advanceTimersByTime })` when using user-event
- Advanced columns (Cliff’s Delta, CLES, significance) are **off** by default (simplified view). Call `enableAdvancedColumns()` when asserting on them
- UI changes usually need snapshot updates (`npm run test:update`) **after** you have verified the UI. Commit snapshot files with the PR
- Tests in `src/__tests__/utils/` are helpers, not suites (`testPathIgnorePatterns`)

Prefer Testing Library queries (`getByRole`, `findByRole`, `getByLabelText`) over test ids unless the existing test already uses them.

## Pull requests

Target **`main`**. Production is a separate branch (`production` → https://perf.compare/); see `Deployment.md`.

Title and body should cite the Bugzilla id:

```
Bug-XXXXXXX: Short description

What changed and why.

Fixes [Bug-XXXXXXX](https://bugzilla.mozilla.org/show_bug.cgi?id=XXXXXXX)
```

After opening the PR, attach the PR URL on the Bugzilla bug.

Keep forks current with rebase onto `upstream/main`, not merge.

Dependency bumps to `package.json` / `package-lock.json` auto-request review from CODEOWNERS.

## Gotchas

- **Retriggering** requires Taskcluster credentials in storage; with none, `RetriggerButton` opens the sign-in modal. Retrigger is skipped when `base_retriggerable_job_ids` / `new_retriggerable_job_ids` are empty.
- **Replicates** vs runs: results have `base_runs` / `new_runs` and `*_replicates`. Student-T has a replicates toggle; Mann-Whitney-U does not use that control the same way.
- Do not call `router` from feature code. In tests, prefer `renderWithRouter` + `window.history.replaceState`.
- `npm run test:a11y` is referenced in CircleCI with `|| true` and is not a required script.
- webpack polyfills Node builtins (`buffer`, `crypto-browserify`, etc.) for Taskcluster client code in the browser.
