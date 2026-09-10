---
name: add-layout-component
description: Add a new form-rendering layout component, implemented in src/common/ts/form-component and wired into src/App/frontend. Use when asked to add, scaffold, or create a new layout/form component for Altinn apps.
---

# Add a layout component

A layout component has two halves: the rendering-only implementation in `form-component`, and the
app-frontend glue that connects it to layout config, data bindings, and validation.

## 1. Shared implementation (`src/common/ts/form-component/src/layout-components/<Name>/`)

Look at an existing component of similar shape before writing new code — `Date` or `Text` for a
label + static/display content, `TextArea` for one with a value + `onChange`.

Files (all required, mirroring existing components):

- `<Name>.tsx` — the component. Reuse `common/LabelComponent` (label with `htmlFor` an input) or
  `common/LabelAsSpan` (label with no interactive input) and `common/ComponentStructure` (grid +
  validation wrapper) rather than building label/wrapper markup by hand.
- `index.ts` — `export { X } from './X'; export type { XProps } from './X';`
- `<Name>.stories.tsx` — declare `<NAME>_PROP_CATEGORIES` (`text` | `data` | `content` | `runtime`,
  see `layout-components/common/README.md`), `satisfies PropCategories<XProps>`, and list it in the
  story meta's `excludeStories`.
- `<Name>.mdx` — three lines wiring `LayoutComponentDocs` to the categories map (copy from any
  sibling component, e.g. `Date/Date.mdx`).
- `<Name>.test.tsx` — render via `renderWithTranslations`.

Register the export in `layout-components/index.ts` (alphabetical `export * from './<Name>';`).

## 2. App-frontend wiring (`src/App/frontend/src/layout/<Name>/`)

- `config.ts` — `new CG.component({ category, availability: 'configurable', metadata, capabilities,
  functionality })`. Pick `category`:
  - `CompCategory.Presentation` — no data binding, ever (e.g. `Paragraph`).
  - `CompCategory.Form` — data-capable. Call `.addDataModelBinding(...)` only once the component
    actually binds data — a `Form` component with no bindings yet is a normal, supported state (see
    `Subform`), and it automatically gets `required`/`readOnly`/etc. via `FormComponentProps`.
  - `.addSummaryOverrides()` and `.extendTextResources(CG.common('TRBLabel'))` +
    `.extends(CG.common('LabeledComponentProps'))` give it the standard title/description/help label.
- `<Name>Component.tsx` — bridges node data to the form-component's props. Use `useLabelData` for
  title/help/description, `useComponentStructureData` for `componentId`/grid sizing. Only add
  `useDataModelBindings` once the component has bindings.
- `index.tsx` — `export class X extends XDef` (from the not-yet-existing `config.def.generated.ts`).
  Must implement `render` (forwardRef wrapping `<XComponent {...props} />`). `Form`/`Container`
  categories must also implement `renderSummary` — return `null` if there's nothing to summarize yet
  (see `PersonLookup`). `renderSummary2` is optional.
- `<Name>Component.test.tsx` — use `renderGenericComponentTest`.

**Do not hand-write `config.def.generated.ts` / `config.runtime.generated.ts`** — the next step
creates them.

## 3. Generate

From `src/App/frontend/`:

```bash
yarn gen
```

This scans `src/layout/*` by directory (no manual registry to update), generates the `.generated.ts`
files for the new component, and regenerates `src/common/ts/layout-contract` (schemas, docs,
`component-catalog.generated.ts`). Expect unrelated-looking diffs in `Summary2`'s generated docs if
the new component is summarizable — that's real: `Summary2` enumerates every summarizable component
type. Commit all generated output alongside your source changes.

## 4. Verify

```bash
# from src/App/frontend/
yarn tsc && yarn lint && yarn test -- <Name> && yarn gen:check

# from src/common/ts/form-component/
yarn typecheck && yarn lint && yarn vitest run <Name>

# from repo root
yarn spell:quick
```

If `spell:quick` fails with `spawnSync typos ENOENT`, the `typos` binary isn't installed in this
environment — fetch the matching release from `crate-ci/typos` on GitHub and put it on `PATH`.
