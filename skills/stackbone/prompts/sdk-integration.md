# `stackbone.prompts` — SDK integration

A versioned prompt catalog that lives in the agent's **own** Postgres (`stackbone_platform.prompts` / `prompt_versions`, installed by the agent's migrations) and compiles templates with the bundled `@stackbone/prompt-compiler` — a zero-dependency Mustache subset (`{{var}}` only). There is **no control-plane round-trip**: the agent owns this data, exactly like `stackbone.secrets` and `stackbone.config`, reaching it over the shared `stackbone.database` pool against `STACKBONE_POSTGRES_URL`.

Use it to keep prompt copy out of the source: editing a prompt becomes an `update()` call (a new immutable version) instead of a redeploy, and every version is retained so you can pin or roll back.

Like every surface except `stackbone.database`, `stackbone.prompts` returns the `{ data, error }` envelope — destructure and handle both branches.

## Connection

`import { stackbone } from '@stackbone/sdk'`. Reach `stackbone.prompts` from any tool's `execute()` or any workflow step — no `createClient()`, no wiring; it reads the agent's own versioned prompt catalog over the injected database lazily.

## Setup

Nothing to construct — `stackbone.prompts` is built on first access from the injected env. The backing tables ship in the agent's migrations; apply them with `stackbone db migrate up` (see the **stackbone-cli** skill). If a call hits a missing table it returns `prompts_not_configured` with a "run `stackbone db migrate up`" hint, not a raw Postgres error.

```ts
// Inside a tool's execute() (or a workflow step):
const { data, error } = await stackbone.prompts.compile('triage-system', {
  severity: input.severity,
});
if (error) throw new Error(error.message);

const completion = await stackbone.ai.chat.completions.create({
  model: 'anthropic/claude-sonnet-4.5',
  messages: [{ role: 'system', content: data.output }],
});
```

## Usage

### Create a prompt (version 1)

```ts
const { data, error } = await stackbone.prompts.create({
  key: 'triage-system', // workspace-unique handle
  name: 'Triage system prompt',
  template: 'You are a {{severity}}-severity triage assistant.',
  description: 'System prompt for the triage flow', // optional
});
if (error) {
  if (error.code === 'prompts_already_exists') {
    /* the key is taken — update() it instead */
  }
  throw new Error(error.message);
}
// data is the Prompt: { key, name, template, version: 1, variables: ['severity'], ... }
```

### Read the raw template, optionally pinned to a version

```ts
const { data, error } = await stackbone.prompts.get('triage-system'); // current version
if (error) throw new Error(error.message);
// data.template is the raw `{{var}}` string; data.version is the live version number

const pinned = await stackbone.prompts.get('triage-system', { version: 2 }); // pin an older version
```

### Compile (render) with variables

```ts
// compile() = get() + render. Returns { output, version }, NOT the raw template.
const { data, error } = await stackbone.prompts.compile('triage-system', { severity: 'high' });
if (error) {
  // prompts_missing_var: a {{var}} in the template had no value in the second argument
  throw new Error(error.message);
}
const systemPrompt = data.output; // the rendered string
```

Pass `{}` as the second argument when the template has no variables — it is required positionally.

### Update (append a new version) and list

```ts
// update() appends a new immutable version and moves the live pointer forward.
const { error } = await stackbone.prompts.update('triage-system', {
  template: 'You are a calm, {{severity}}-severity triage assistant.',
});
if (error) throw new Error(error.message);

const { data: list, error: listError } = await stackbone.prompts.list({ limit: 50 });
if (listError) throw new Error(listError.message);
// list.items is the current version of each non-deleted prompt
```

### Delete (soft delete)

```ts
const { data, error } = await stackbone.prompts.delete('triage-system');
if (error) throw new Error(error.message);
// data = { key, deleted: 1 }. Rows are kept (deleted_at stamped); later get() returns prompts_not_found.
```

## Best practices

1. **Always destructure `{ data, error }`.** A missing prompt or an uncompiled `{{var}}` comes back as an error, never a throw — handle it.
2. **Compile, don't string-concatenate.** Store the template in the catalog and call `compile(key, vars)`; keep prompt copy out of `src/`.
3. **Treat versions as immutable.** `update()` appends a new version; it never edits the old one. Pin with `get(key, { version })` when you need reproducibility.
4. **Keep keys stable.** The `key` is the handle your code calls; renaming it is a breaking change for every caller.
5. **Only `{{var}}` interpolation exists.** The compiler is a Mustache subset — no sections, no partials, no logic. Branch in TypeScript, not in the template.

## Common mistakes

| Mistake                               | Fix                                                                                                   |
| ------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| Reading `data.text` from `compile()`  | The field is `data.output` (and `data.version`). `get()` returns the raw template as `data.template`. |
| Passing variables as the third arg    | `compile(key, vars, { version })` — variables are the **second** positional arg.                      |
| Calling `create()` on an existing key | Returns `prompts_already_exists`; use `update(key, …)` to add a version.                              |
| Expecting `{{#if}}` / loops to work   | Mustache subset is `{{var}}` only — do the logic in TypeScript and pass the result in.                |
| Ignoring `prompts_not_configured`     | The migration hasn't run — `stackbone db migrate up` (see the **stackbone-cli** skill).               |

## Common error codes

| `error.code`              | Cause                                                          | Action                                                           |
| ------------------------- | -------------------------------------------------------------- | ---------------------------------------------------------------- |
| `prompts_not_found`       | No prompt for that key (or that pinned version / soft-deleted) | Check the `key`; `list()` shows the live ones.                   |
| `prompts_already_exists`  | `create()` on a key that's already taken                       | `update(key, …)` instead.                                        |
| `prompts_missing_var`     | A `{{var}}` in the template had no value passed to `compile()` | `error.meta.name` names the missing variable; add it.            |
| `prompts_invalid_request` | Bad key/shape, or the template failed to compile               | `error.message` describes it; fix the call args or the template. |
| `prompts_not_configured`  | The prompts tables aren't migrated yet (`42P01`)               | Run `stackbone db migrate up`.                                   |

See the main [SKILL.md](../SKILL.md) for cross-module patterns (e.g. compile a prompt then feed it to `stackbone.ai`).
