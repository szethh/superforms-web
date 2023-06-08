# Migration guide

<svelte:head><title>Migration guide from 0.x to 1.0</title></svelte:head>

Lists the breaking changes that you need to address to upgrade from v0.x to v1.0.

## Updating

To update, change your `package.json` entry for `sveltekit-superforms` to `^1.0.0`:

```json
{
  "devDependencies": {
    "sveltekit-superforms": "^1.0.0",
  }
}
```

The guide is written with the affected methods in the headlines, so you can scan through this page and apply the changes if you're using them in your code.

### superForm

For type safety reasons, you cannot pass `null`, `undefined` or arbitrary data to `superForm` anymore. Instead you should be using `superValidate` to get the initial form object that should be sent to `superForm`, which is what's been done if you're returning `{ form }` from a load function or form action.

So if the form data is coming from the server this is not a problem, but on the client, especially in SPA settings, you must use `superValidate` or `superValidateSync` before calling `superForm`. Read more on the [SPA page](/concepts/spa) about this.

### valid, empty, firstError

The `$valid`, `$empty` and `$firstError` stores are removed from the client, they weren't that useful. `$allErrors` can be used instead, together with the new `$posted` store (which shows if the form has been previously posted or not).

`empty` is removed from the object returned from `superValidate`, which type was previously called `Validation` but is now called `SuperValidated`.

### setError

The `setError` function doesn't handle form-level errors anymore, because it conflicts with client-side validation. Use refine/superRefine on the schema, or the `message` helper instead.

```ts
const schema = z
  .object({
    password: z.string().min(8),
    confirmPassword: z.string()
  })
  .refine((data) => password == confirmPassword, `Passwords doesn't match.`);
```

The above error set in `refine` will be available on the client as `$errors._errors` as before, and will be automatically removed (or added) during client-side validation.

If you'd like the error to persist, using `message` will persist it until the next form submission.

```ts
const form = await superValidate(request, schema);

if (!form.valid) return fail(400, { form });

if (form.data.password != form.data.confirmPassword) {
  // Stays until form is posted again, regardless of client-side validation
  return message(form, `Passwords doesn't match`, { status: 400 });
}
```

Finally, the status option for `setError` (and `message`) must be in the `400-599` range.

### setError (again), validate, proxy functions

`FieldPath` is gone - the above methods are now using a string accessor like `tags[2].id` instead of the previous array syntax `['tags', 2, 'id']`.

```diff
const { form, enhance, validate } = superForm(data.form)

- validate(['tags', i, 'name'], { update: false });
validate(`tags[${i}].name`, { update: false });
```

This also applies to generic components. The types have been simplified, so you should change them to this, also described on the [componentization page](https://superforms.vercel.app/components):

```svelte
<script lang="ts">
  import type { z, AnyZodObject } from 'zod';
  import type { ZodValidation, FormPathLeaves } from '$lib';
  import { formFieldProxy, type SuperForm } from '$lib/client';

  type T = $$Generic<AnyZodObject>;

  export let form: SuperForm<ZodValidation<T>, unknown>;
  export let field: FormPathLeaves<z.infer<T>>;

  const { value, errors, constraints } = formFieldProxy(form, field);
</script>
```

Also note that **arrays and objects cannot be used in formFieldProxy**. So if your schema is defined as:

```ts
import { formFieldProxy } from 'sveltekit-superforms/client';

const schema = z.object({
  tags: z
    .object({
      id: z.number(),
      name: z.string().min(1)
    })
    .array()
});

const formData = superForm(data.form);

// This won't work properly!
const tags = formFieldProxy(formData, 'tags');

// Not this either
const tag = formFieldProxy(formData, 'tags[0]');

// But this will work
const tagName = formFieldProxy(formData, 'tags[0].name');
```

> The `FormPathLeaves` type is there to prevent using a schema field that isn't at the end of the schema (the "leaves" of the schema tree)

This only applies to `formFieldProxy`, since it maps to errors and constraints as well as the form. If you want to proxy just a form value, the `fieldProxy` will work with any part of the schema.

```ts
import { fieldProxy } from 'sveltekit-superforms/client';

const { form } = superForm(data.form);
const tags = fieldProxy(form, 'tags');
```

### allErrors, firstError

The signature for `allErrors` and `firstError` has changed, to make it easier to group related messages:

```diff
- { path: string[]; message: string[] }
{ path: string; messages: string[] }
```

The path follows the same format as the above described string accessor path. If you want to display all messages grouped:

```svelte
{#if $allErrors.length}
  <ul>
    {#each $allErrors as error}
      <li>
        <b>{error.path}:</b>
        {error.messages.join('. ')}.
      </li>
    {/each}
  </ul>
{/if}
```

Or as before, separate for each error:

```svelte
{#if $allErrors.length}
  <ul>
    {#each $allErrors as error}
      {#each error.messages as message}
        <li>
          <b>{error.path}:</b> {message}.
        </li>
      {/each}
    {/each}
  </ul>
{/if}
```

### defaultData

The undocumented `defaultData` function is now called `defaultValues`. You can use this to get the default values for a schema.

```diff
- import { defaultData } from 'sveltekit-superforms/server`
import { defaultValues } from 'sveltekit-superforms/server`
```

See [the API](/api#defaultvaluesschema) for documentation.

### message, setMessage

The `valid` option is removed from `message/setMessage`, any status >= 400 will return a fail. As with `setError`, the status code must be in the `400-599` range.

### meta

The virtually unused `meta` store, which containted some basic metadata about the schema, has been removed. Use the Zod schema directly instead for reflection.

## Client options

The following `superForm` options have changed:

### resetForm

Resetting the form now works without `use:enhance` and without JavaScript! Just set the `resetForm` option to `true`.

If you have used the function version of `resetForm`, `() => boolean`, it is now synchronous.

### errorSelector

The default `errorSelector` is now `[aria-invalid="true"],[data-invalid]`, so if you want to be more accessibility-friendly:

```diff
 <input
   name="name"
   bind:value={$form.name}
-  data-invalid={$errors.name}
   aria-invalid={$errors.name ? 'true' : undefined}
 />
```

## Server options

The following `superValidate` options have changed:

### noErrors

`noErrors` is removed from the options. Use `errors` instead to determine if errors should be added or not to the validation.

```ts
// Add errors to an empty form
const form = await superValidate(schema, { errors: true });
```

The [changelog](https://github.com/ciscoheat/sveltekit-superforms/blob/main/CHANGELOG.md) has a full list of changes for 1.0, and as usual, let me know on Github or Discord if something is unclear or not working.