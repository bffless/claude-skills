# Block catalog

Every block is a React component exported from `@components`. Import only what you use:

```mdx
import { Meta, Callout, DataModel, Steps } from '@components';
```

All components are styled by the `--wf-*` tokens in `src/styles/tokens.css`. Props below are the
public surface — keep to them so plans stay consistent.

---

### `<Meta>` — document header (use first)

```mdx
<Meta title="Add settings" status="proposed" date="2026-06-17"
  objective="One line on the outcome." tags={['backend']} />
```

`title` (req), `status` (`proposed | approved | done`, default `proposed`), `objective`, `date`,
`tags: string[]`.

### `<Callout>` — decisions, risks, notes

```mdx
<Callout kind="decision">Store settings as a jsonb column.</Callout>
```

`kind`: `decision | warn | ok | note` (default `note`). Optional `title`. Use `decision` to make a
hard-to-reverse choice the headline; `warn` for risks.

### `<Steps>` — ordered implementation steps

```mdx
<Steps items={[
  'Plain step',
  { title: 'Add column', detail: 'jsonb default {}', files: ['db/schema/workspace.ts'] },
]} />
```

Items are strings or `{ title, detail?, files?: string[] }`. Name real files per step.

### `<Checklist>` — verification / acceptance

```mdx
<Checklist items={[ { label: 'PATCH merges, not replaces', done: false } ]} />
```

Items are strings or `{ label, done?: boolean }`.

### `<Columns>` — side-by-side comparison

```mdx
<Columns columns={[
  { title: 'Before', tone: 'before', body: '...' },
  { title: 'After',  tone: 'after',  body: '...' },
]} />
```

`tone`: `neutral | before | after`. Use `body` for data, or one child element per column.

### `<DataModel>` — table / type shape with change flags

```mdx
<DataModel table="workspace" caption="1 column added" fields={[
  { name: 'id', type: 'uuid' },
  { name: 'settings', type: 'jsonb', change: 'added', note: 'default {}' },
]} />
```

`fields[]`: `{ name, type, note?, change?: 'added'|'removed'|'changed' }`. Flag touched fields.

### `<ApiEndpoint>` — HTTP contract

```mdx
<ApiEndpoint method="PATCH" path="/api/workspaces/:id" change="added"
  summary="merge partial settings"
  request={`{ "theme": "dark" }`}
  response={`{ "settings": { "theme": "dark" } }`} />
```

`method`: `GET|POST|PUT|PATCH|DELETE`. `request`/`response` are example body strings.

### `<FileTree>` — touched files with stats

```mdx
<FileTree files={[
  { path: 'src/app.ts', change: 'changed', add: 18, del: 2 },
  { path: 'src/new.ts', change: 'added', add: 40 },
]} />
```

`files[]`: `{ path, change?: 'added'|'removed'|'changed', add?, del?, note? }`. Indent with leading
spaces in `path` to suggest hierarchy.

### `<Diff>` — a specific code change

```mdx
<Diff file="db/schema.ts" summary="add settings column" lines={[
  " export const workspace = pgTable('workspace', {",
  "+  settings: jsonb('settings').notNull().default({}),",
  " });",
]} />
```

`lines[]`: each prefixed with `+`, `-`, or ` ` (context). Long hunks: pass `collapsed` to fold
behind a native `<details>` toggle (zero JS). Keep summary one line.

### `<Diagram>` — architecture / data flow

```mdx
<Diagram caption="Request flow" nodes={[
  { id: 'ui', label: 'Console UI' },
  { id: 'api', label: 'API', sub: 'PATCH /workspaces/:id' },
  { id: 'db', label: 'Postgres' },
]} />
```

`nodes[]`: `{ id, label, sub? }`, `direction`: `row | col`. For freeform layouts pass children and
arrange with flex/grid instead of `nodes`.

### `<Wireframe>` — UI mockup chrome

```mdx
<Wireframe surface="browser" url="app.example.com/settings" caption="Settings tab">
  <div style={{ display: 'grid', gap: 10 }}>
    <div className="wf-eyebrow">Workspace settings</div>
    <button className="wf-btn primary">Save</button>
  </div>
</Wireframe>
```

`surface`: `browser | desktop | mobile | popover | panel` (use `mobile` only for phone-specific
work). Build the screen body from bare flex/grid + the `.wf-*` helper classes and `<Icon>`. Never
hard-code colors/fonts inside.

### `<Icon>` — inline SVG markers for wireframes

```mdx
<Icon name="search" />
```

Names: `mail lock search plus x check chevronDown chevronRight more user settings calendar bell send
edit arrowRight`. Inherits `currentColor`; unknown names render a neutral box.

### `<QuestionForm>` — open questions (BOTTOM only)

```mdx
<QuestionForm questions={[
  { q: 'Allow-list keys or free-form?', options: ['Allow-list', 'Free-form'], recommend: 'Allow-list' },
]} />
```

Items are strings or `{ q, options?: string[], recommend? }`. Only ever at the end of a document.

---

## Helper classes (for wireframe/diagram children)

From `src/styles/app.css`: `.wf-card`, `.wf-box`, `.wf-pill` (`.accent`), `.wf-chip`, `.wf-muted`,
`.wf-eyebrow`, `.wf-btn` (`.primary`), `.wf-icon`. Use these instead of inline color styles.
