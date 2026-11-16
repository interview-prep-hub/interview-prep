# Forms and Validation in React

> **Tier:** T1 (know cold) · **Related:** [Hooks](02-hooks.md), [Data fetching](04-data-fetching-tanstack-query.md), [Patterns](07-patterns-and-architecture.md), [React 19 forms](09-react-19-and-server-components.md), [Testing](10-testing.md), [TypeScript in practice](../03-languages/typescript/04-ts-in-practice.md), [Rails views and forms](../05-ruby-on-rails/07-views-erb-forms-assets.md), [DRF](../10-python-django/04-django-rest-framework.md)
> **On your resume:** Forms are where your React work meets your APIs: Rails validations behind the Colochousing web app and REST API, forms in the PureLogics LMS backed by DRF serializers, forms in the Coloc resident app (React Native), and the TypeScript platform (oRPC, TanStack Start), where client and server can share one schema.

## Quick revision
- Controlled input: `value` + `onChange`, React state is the source of truth; easy live validation and formatting, one render per keystroke. Uncontrolled: the DOM holds the value; read it with `FormData` or a ref on submit (`defaultValue` sets the start value).
- Never switch an input between the two: initialize state with `''`, not `undefined`. `<input type="file">` is always uncontrolled.
- Validate on submit first, then on blur or on change once a field is touched; the server validates again and has the final word.
- React Hook Form (RHF) is uncontrolled by default, so typing does not re-render the form. Core API: `register`, `handleSubmit`, `formState.errors`, `Controller`, `watch`/`useWatch`, `reset`, `setError`, `useFieldArray`.
- Zod: one schema gives runtime validation and the TypeScript type (`z.infer`); `zodResolver(schema)` plugs it into RHF; `.refine` with `path` handles cross-field rules.
- Share schemas between the client and a TypeScript backend through a shared package; with Rails or DRF, mirror the key rules and map server errors.
- Map server field errors with `setError(field, { message })`; put form-level errors under `root`.
- Accessibility: `<label htmlFor>`, `aria-invalid`, `aria-describedby` pointing at the error text, `role="alert"`, focus on the first invalid field.
- Prevent double submits: disable while submitting (`isSubmitting`, `isPending`) and make the server idempotent.
- Buttons inside a form default to `type="submit"`; give other buttons `type="button"`.
- Debounce search inputs and abort stale requests.
- React 19: `<form action={fn}>`, `useActionState`, `useFormStatus`.

## 1. Controlled vs uncontrolled inputs

```tsx
import { useState, type FormEvent } from 'react';

// Controlled: React state drives the input
function NameField() {
  const [name, setName] = useState('');
  return <input value={name} onChange={(e) => setName(e.target.value.trimStart())} />;
}

// Uncontrolled: the DOM keeps the value; read it on submit
function NewsletterForm({ onSubscribe }: { onSubscribe: (email: string) => void }) {
  function handleSubmit(e: FormEvent<HTMLFormElement>) {
    e.preventDefault();
    const data = new FormData(e.currentTarget);
    onSubscribe(String(data.get('email')));
  }
  return (
    <form onSubmit={handleSubmit}>
      <input name="email" type="email" required />
      <button type="submit">Subscribe</button>
    </form>
  );
}
```

| | Controlled | Uncontrolled |
|---|---|---|
| Source of truth | React state | the DOM |
| Reading the value | any time, from state | on submit (`FormData`) or through a ref |
| Re-renders while typing | one per keystroke | none |
| Live validation, formatting, dependent fields | easy | harder |
| File inputs | impossible | the only option |
| Typical use | small forms, inputs that drive other UI | simple forms, React Hook Form, React 19 form actions |

- `value` without `onChange` makes the field read-only (React warns); use `defaultValue` or `readOnly`.
- "A component is changing an uncontrolled input to be controlled" means `value` went from `undefined` to a string. Initialize with `''`.
- Checkboxes use `checked` (or `defaultChecked`); `<select>` and `<textarea>` take `value` like inputs.
- `e.target.value` is always a string, even for `type="number"`: convert it (`valueAsNumber`, `Number()`).

## 2. Multi-field forms with plain state

```tsx
import { useState, type ChangeEvent, type FormEvent } from 'react';

type SignupValues = { email: string; password: string; agree: boolean };
type Errors = Partial<Record<keyof SignupValues, string>>;

function validate(v: SignupValues): Errors {
  const errors: Errors = {};
  if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(v.email)) errors.email = 'Enter a valid email';
  if (v.password.length < 8) errors.password = 'Use at least 8 characters';
  if (!v.agree) errors.agree = 'You must accept the terms';
  return errors;
}

function SignupForm({ onSignup }: { onSignup: (v: SignupValues) => void }) {
  const [values, setValues] = useState<SignupValues>({ email: '', password: '', agree: false });
  const [submitted, setSubmitted] = useState(false);
  const errors = validate(values); // derived on every render, never stored

  function handleChange(e: ChangeEvent<HTMLInputElement>) {
    const { name, value, type, checked } = e.target;
    setValues((v) => ({ ...v, [name]: type === 'checkbox' ? checked : value })); // one handler, keyed by name
  }

  function handleSubmit(e: FormEvent<HTMLFormElement>) {
    e.preventDefault();
    setSubmitted(true); // from now on, show errors
    if (Object.keys(errors).length === 0) onSignup(values);
  }

  return (
    <form onSubmit={handleSubmit} noValidate>
      <input name="email" type="email" value={values.email} onChange={handleChange} />
      {submitted && errors.email && <p role="alert">{errors.email}</p>}
      {/* password and agree fields follow the same pattern */}
      <button type="submit">Create account</button>
    </form>
  );
}
```

This is fine for three fields. Once you add async validation, field arrays, touched tracking per field, server errors and dozens of inputs, a form library pays for itself.

## 3. Validation strategies

| Strategy | When it runs | Pros | Cons |
|---|---|---|---|
| HTML constraints (`required`, `type="email"`, `minLength`, `pattern`) | browser, on submit | free, accessible | limited rules, browser-styled messages |
| On submit | when the user submits | no nagging while typing | late feedback on long forms |
| On blur | when a field loses focus | feedback per field | needs touched tracking |
| On change after touched or after first submit | live | fast correction | more work per keystroke |
| Async (email already registered) | debounced, on change or blur | catches server rules early | needs cancellation and race handling |
| Server | on every request | the only trustworthy check | needs error mapping |

A good default: validate on submit, then re-validate on change so errors disappear as the user fixes them. That is RHF's default (`mode: 'onSubmit'`, `reValidateMode: 'onChange'`); `mode: 'onTouched'` validates a field after its first blur. Add `noValidate` to the `<form>` when you render your own messages instead of the browser's bubbles.

## 4. React Hook Form with Zod

```tsx
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

const signupSchema = z
  .object({
    name: z.string().min(2, 'Name is too short'),
    email: z.string().email('Enter a valid email'),
    password: z.string().min(8, 'Use at least 8 characters'),
    confirmPassword: z.string(),
    age: z.number().int().min(18, 'You must be 18 or older'), // an empty field gives NaN and fails here
  })
  .refine((v) => v.password === v.confirmPassword, {
    message: 'Passwords do not match',
    path: ['confirmPassword'], // attach the error to this field
  });

type SignupValues = z.infer<typeof signupSchema>; // the type comes from the schema

export function SignupForm() {
  const {
    register,
    handleSubmit,
    setError,
    reset,
    formState: { errors, isSubmitting },
  } = useForm<SignupValues>({
    resolver: zodResolver(signupSchema),
    defaultValues: { name: '', email: '', password: '', confirmPassword: '' },
    mode: 'onTouched',
  });

  async function onSubmit(values: SignupValues) {
    // Only called when the schema passes; isSubmitting stays true until this promise settles
    const res = await fetch('/api/signup', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(values),
    });
    if (res.status === 422) {
      const body = (await res.json()) as { errors: Record<string, string[]> };
      applyServerErrors(body.errors, setError, ['name', 'email', 'password', 'confirmPassword', 'age']);
      return;
    }
    reset(); // back to defaultValues
  }

  return (
    <form onSubmit={handleSubmit(onSubmit)} noValidate>
      <label htmlFor="email">Email</label>
      <input
        id="email"
        type="email"
        {...register('email')}
        aria-invalid={errors.email ? true : undefined}
        aria-describedby={errors.email ? 'email-error' : undefined}
      />
      {errors.email && <p id="email-error" role="alert">{errors.email.message}</p>}

      <label htmlFor="age">Age</label>
      <input id="age" type="number" {...register('age', { valueAsNumber: true })} />
      {errors.age && <p role="alert">{errors.age.message}</p>}

      {/* name, password and confirmPassword follow the same pattern */}
      {errors.root?.server && <p role="alert">{errors.root.server.message}</p>}
      <button type="submit" disabled={isSubmitting}>{isSubmitting ? 'Creating...' : 'Create account'}</button>
    </form>
  );
}
```

How it works:
- `register('email')` returns `{ name, onChange, onBlur, ref }`. RHF reads values from the DOM through the ref, so typing does not re-render the component (uncontrolled by default).
- `handleSubmit(onSubmit)` prevents the default submit, runs validation, focuses the first invalid field, and calls `onSubmit` with typed values only when valid.
- `formState` is a proxy: RHF re-renders only for the properties you read, so destructure just what you use (`errors`, `isSubmitting`, `isDirty`, `isValid`).
- `valueAsNumber` converts the string; an empty number field becomes `NaN`, which the schema then rejects.
- Zod version note: Zod 4 (2025) adds top-level helpers such as `z.email()` and renames some error options; the chained style above is the long-standing API. Check your installed version when copying examples.

## 5. Controller, watch, reset and useFieldArray

**`Controller`** wraps components that do not expose a DOM input with a ref: UI-library selects, date pickers, rich text editors, and every input in React Native.

```tsx
import { Controller, useForm } from 'react-hook-form';
import { Text, TextInput } from 'react-native';

function EventForm() {
  const { control, handleSubmit } = useForm<{ title: string }>({ defaultValues: { title: '' } });
  return (
    <Controller
      control={control}
      name="title"
      rules={{ required: 'Title is required' }}
      render={({ field: { value, onChange, onBlur }, fieldState: { error } }) => (
        <>
          <TextInput value={value} onChangeText={onChange} onBlur={onBlur} placeholder="Event title" />
          {error && <Text>{error.message}</Text>}
        </>
      )}
    />
  );
}
```

**`watch` and `useWatch`** subscribe to values for dependent UI:

```tsx
const plan = watch('plan'); // re-renders this whole form component whenever "plan" changes
// {plan === 'business' && <input {...register('vatNumber')} />}
// In a child component, useWatch({ control, name: 'plan' }) re-renders only that child.
```

**`reset`** loads or clears values and resets dirty state, for example in an edit form once data arrives:

```tsx
const { data: profile } = useQuery(profileQuery);
useEffect(() => {
  if (profile) reset({ name: profile.name, email: profile.email }); // defaults and isDirty now match the server
}, [profile, reset]);
// Alternative: useForm({ values: profile }) updates the form when the data changes.
```

**`useFieldArray`** manages dynamic lists of fields:

```tsx
type InvoiceValues = { items: { description: string; amount: number }[] };

function InvoiceForm({ onSave }: { onSave: (v: InvoiceValues) => void }) {
  const { control, register, handleSubmit } = useForm<InvoiceValues>({
    defaultValues: { items: [{ description: '', amount: 0 }] },
  });
  const { fields, append, remove } = useFieldArray({ control, name: 'items' });

  return (
    <form onSubmit={handleSubmit(onSave)}>
      {fields.map((field, index) => (
        <div key={field.id}> {/* RHF's generated id, never the index */}
          <input {...register(`items.${index}.description`)} placeholder="Description" />
          <input type="number" {...register(`items.${index}.amount`, { valueAsNumber: true })} />
          <button type="button" onClick={() => remove(index)}>Remove</button>
        </div>
      ))}
      <button type="button" onClick={() => append({ description: '', amount: 0 })}>Add line</button>
      <button type="submit">Save</button>
    </form>
  );
}
```

Other useful APIs: `setValue`, `getValues`, `trigger` (validate on demand) and `formState.isDirty` for "unsaved changes" prompts.

## 6. Sharing schemas with the backend

In a TypeScript monorepo, define the schema once and import it on both sides. The form gets instant feedback, the API gets real validation, and both share the inferred type.

```ts
// packages/shared/src/schemas/bill.ts
import { z } from 'zod';

export const createBillSchema = z.object({
  title: z.string().min(1).max(120),
  amountCents: z.number().int().positive(),
  splitBetween: z.array(z.string()).min(1, 'Pick at least one resident'),
});
export type CreateBillInput = z.infer<typeof createBillSchema>;
```

```ts
// API (oRPC-style sketch; check the oRPC docs for exact imports)
export const createBill = os.input(createBillSchema).handler(async ({ input }) => saveBill(input));

// Hono alternative: app.post('/bills', zValidator('json', createBillSchema), (c) => ...)

// Web form
const form = useForm<CreateBillInput>({ resolver: zodResolver(createBillSchema) });
```

With a Rails or DRF backend you cannot share Zod. The model validations (Rails `validates`) or serializer rules (DRF) stay the source of truth; mirror the obvious rules in Zod for fast feedback, map server errors back onto fields, and optionally generate client types from an OpenAPI spec. Client-side validation is UX, never security: a request can always skip your form.

## 7. Mapping server errors to fields

Rails typically answers `422` with `{ "errors": { "email": ["has already been taken"] } }` (from `render json: { errors: user.errors }`); DRF answers `400` with `{ "email": ["user with this email already exists."], "non_field_errors": ["..."] }`.

```ts
import type { FieldValues, Path, UseFormSetError } from 'react-hook-form';

export function applyServerErrors<T extends FieldValues>(
  errors: Record<string, string[]>,
  setError: UseFormSetError<T>,
  fields: readonly Path<T>[],
) {
  for (const [field, messages] of Object.entries(errors)) {
    const message = messages.join(' ');
    if ((fields as readonly string[]).includes(field)) {
      setError(field as Path<T>, { type: 'server', message }, { shouldFocus: true });
    } else {
      setError('root.server', { type: 'server', message }); // base, non_field_errors, unknown keys
    }
  }
}
```

Render `errors.root?.server?.message` near the submit button. Normalize the error shape in one API helper so every form handles Rails, DRF or oRPC errors the same way.

## 8. Accessibility

- Every input has a visible `<label htmlFor>` (or wraps the input). A placeholder is not a label.
- Mark invalid fields with `aria-invalid` and link the message with `aria-describedby`; render messages as text, not only a red border.
- Announce errors with `role="alert"` (or an `aria-live` region for form-level status).
- Move focus to the first invalid field on submit (RHF does this by default).
- Group radios and checkboxes in `<fieldset>` with a `<legend>`; use real `<button>`s so Enter and Space work.

```tsx
import { useId, type ComponentProps } from 'react';

type TextFieldProps = { label: string; error?: string } & ComponentProps<'input'>;

// React 19: `ref` from register() arrives as a normal prop and is spread onto <input>.
// React 18: wrap this component in forwardRef instead.
export function TextField({ label, error, ...inputProps }: TextFieldProps) {
  const id = useId();
  const errorId = `${id}-error`;
  return (
    <div className="field">
      <label htmlFor={id}>{label}</label>
      <input
        id={id}
        aria-invalid={error ? true : undefined}
        aria-describedby={error ? errorId : undefined}
        {...inputProps}
      />
      {error && <p id={errorId} role="alert">{error}</p>}
    </div>
  );
}
// <TextField label="Email" type="email" error={errors.email?.message} {...register('email')} />
```

## 9. File inputs

A file input's value is read-only, so it is always uncontrolled. Read `e.target.files`, validate type and size on the client (and again on the server), preview with an object URL, and upload with `FormData`.

```tsx
function AvatarUpload() {
  const [file, setFile] = useState<File | null>(null);
  const [preview, setPreview] = useState<string | null>(null);

  useEffect(() => {
    if (!file) return;
    const url = URL.createObjectURL(file);
    setPreview(url);
    return () => URL.revokeObjectURL(url); // free the memory when the file changes or on unmount
  }, [file]);

  function handleChange(e: ChangeEvent<HTMLInputElement>) {
    const selected = e.target.files?.[0] ?? null;
    if (selected && selected.size > 2 * 1024 * 1024) {
      alert('Maximum size is 2 MB');
      e.target.value = ''; // clear it so the same file can be picked again
      return;
    }
    setFile(selected);
  }

  async function upload() {
    if (!file) return;
    const body = new FormData();
    body.append('avatar', file);
    await fetch('/api/avatar', { method: 'POST', body }); // no Content-Type: the browser adds the multipart boundary
  }

  return (
    <>
      <input type="file" accept="image/png,image/jpeg" onChange={handleChange} />
      {file && preview && <img src={preview} alt="Avatar preview" width={96} />}
      <button type="button" onClick={upload} disabled={!file}>Upload</button>
    </>
  );
}
```

With RHF, `register('avatar')` gives you a `FileList`. Large files are often uploaded straight to object storage with a presigned URL (Rails Active Storage direct uploads work this way) instead of through your API server.

## 10. Debounced search inputs

Keep the input controlled and responsive, debounce the value that drives the request, and let the query layer cancel stale requests:

```tsx
function MemberSearch() {
  const [query, setQuery] = useState('');
  const debounced = useDebounce(query.trim(), 300); // custom hook from the hooks file
  const { data = [], isFetching } = useQuery({
    queryKey: ['members', 'search', debounced],
    queryFn: ({ signal }) => fetchJson<Member[]>(`/api/members?q=${encodeURIComponent(debounced)}`, signal),
    enabled: debounced.length >= 2,
  });

  return (
    <div role="search">
      <label htmlFor="member-search">Search members</label>
      <input id="member-search" type="search" value={query} onChange={(e) => setQuery(e.target.value)} />
      {isFetching && <span aria-live="polite">Searching...</span>}
      <ul>{data.map((m) => <li key={m.id}>{m.name}</li>)}</ul>
    </div>
  );
}
```

Without TanStack Query, run the fetch in an effect keyed on the debounced value with an `AbortController` (full version in the [question bank](99-interview-questions.md)). If the list is filtered locally rather than fetched, `useDeferredValue` keeps typing smooth without delaying results artificially ([hooks](02-hooks.md)).

## 11. Preventing double submit

- Disable the submit button while the request runs: RHF's `isSubmitting` stays true until an async `onSubmit` settles; a mutation exposes `isPending`.
- State updates are asynchronous, so two very fast clicks can both start before the re-render disables the button. Guard with a ref when it matters:

```tsx
const inFlight = useRef(false);
async function handlePay() {
  if (inFlight.current) return; // a second click before the re-render is ignored
  inFlight.current = true;
  try {
    await payRent();
  } finally {
    inFlight.current = false;
  }
}
```

- Make the server safe anyway: idempotency keys for payments, unique constraints for sign-ups (see [payments and subscriptions](../13-architecture-and-practices/08-payments-and-subscriptions.md)).

## 12. React 19 form actions (overview)

React 19 lets a `<form>` take a function as its `action`. React calls it with the `FormData`, tracks the pending state, and resets uncontrolled fields after success.

```tsx
import { useActionState } from 'react';

async function subscribe(_prev: string | null, formData: FormData): Promise<string | null> {
  const res = await fetch('/api/subscribe', { method: 'POST', body: formData });
  return res.ok ? null : 'Could not subscribe, try again';
}

function Newsletter() {
  const [error, formAction, isPending] = useActionState(subscribe, null);
  return (
    <form action={formAction}>
      <input name="email" type="email" required />
      <button disabled={isPending}>{isPending ? 'Subscribing...' : 'Subscribe'}</button>
      {error && <p role="alert">{error}</p>}
    </form>
  );
}
```

Actions shine with Server Functions in frameworks (progressive enhancement, no client fetch code). For complex client-side validation, field arrays and instant feedback, React Hook Form is still the practical choice; the two can be combined. Details in [React 19 and Server Components](09-react-19-and-server-components.md).

## Common pitfalls
- Initializing a controlled input with `undefined`, triggering the uncontrolled-to-controlled warning.
- Buttons without `type="button"` inside a form, which submit it.
- Forgetting `e.preventDefault()` in a plain `onSubmit`, causing a full page reload.
- Index keys in dynamic field lists (use `field.id` from `useFieldArray`).
- Treating `e.target.value` from number inputs as a number.
- Setting `Content-Type` manually when sending `FormData`, which breaks the multipart boundary.
- Relying on client validation alone, with no server checks.
- Calling `watch()` with no arguments at the top of a large form, re-rendering everything on each keystroke.
- Showing every error on first render, before the user touched anything.
- Placeholder-only fields with no labels; error messages that are only a color.

## How to talk about this in an interview
- Lead with the trade-off: "Controlled inputs give me live control at the cost of a render per keystroke; for real forms I use React Hook Form, which is uncontrolled by default, with a Zod schema so validation and types come from one place, and the server stays the final authority."
- Connect to your backends: Rails returns validation errors per attribute and DRF per serializer field, so you map them onto form fields with `setError`. That shows you think across the stack.

> **Fill in:** a form you built at Devntech (a Colochousing admin or resident flow, or a page migrated to the TanStack Start platform): which library you used, how validation worked, and how server errors reached the user.

> **Fill in:** the sign-up or enrollment forms in the PureLogics LMS: how you validated on the client, and how DRF serializer errors were displayed.

- In React Native there is no DOM, so React Hook Form's `register` does not apply and inputs go through `Controller` (or plain controlled state); mention it when you talk about forms in the Coloc resident app.
- If asked about accessibility, list concrete attributes (`htmlFor`, `aria-invalid`, `aria-describedby`, `role="alert"`) rather than saying "I make it accessible".

## Interview questions

**Q1. What is the difference between controlled and uncontrolled components?**
In a controlled input, React state holds the value and `onChange` updates it, so you can validate, format and react on every keystroke at the cost of a render each time. In an uncontrolled input, the DOM holds the value and you read it on submit with `FormData` or a ref, using `defaultValue` for the initial value. File inputs are always uncontrolled.
*Follow-up:* Which does React Hook Form use? Uncontrolled by default, through refs, which is why it re-renders so little.

**Q2. What causes "A component is changing an uncontrolled input to be controlled"?**
The input's `value` was `undefined` (so React treated it as uncontrolled) and later became a string. It usually comes from `useState<string>()` or from data that arrives later. Initialize with `''` or use `value ?? ''`.
*Follow-up:* And the reverse warning? A value that becomes `undefined` or `null`, for example after a reset that drops the field.

**Q3. How do you handle a form with many fields in plain React?**
Keep one state object, use a single change handler that reads `e.target.name` and updates that key immutably, derive errors from the values with a `validate` function, and track touched or submitted state to decide when to show them. It works for small forms; beyond that, a library handles touched state, async validation and arrays for you.
*Follow-up:* Why derive errors instead of storing them? They can never drift out of sync with the values.

**Q4. Why use React Hook Form, and how does it minimize re-renders?**
It registers inputs through refs and reads their values from the DOM, so typing does not re-render the form component. `formState` is a proxy that re-renders only for the properties you read, and `useWatch` isolates subscriptions in child components. You also get schema resolvers, field arrays, focus management and server error helpers.
*Follow-up:* When does RHF re-render? When subscribed `formState` properties change (errors, `isSubmitting`) or watched values change.

**Q5. What does register return, and when do you need Controller?**
`register('email')` returns `name`, `onChange`, `onBlur` and `ref`, which you spread onto a native input. `Controller` (or `useController`) is for components that do not expose a native input with a ref: UI-library selects, date pickers, rich text editors, and all React Native inputs. Its `render` prop receives `field` (`value`, `onChange`, `onBlur`, `ref`) and `fieldState`.
*Follow-up:* What about custom input components? Pass the ref through: a normal prop in React 19, `forwardRef` in React 18.

**Q6. How do you integrate Zod with React Hook Form, and why use a schema?**
Define a Zod object schema, infer the form type with `z.infer<typeof schema>`, and pass `resolver: zodResolver(schema)` to `useForm`. Validation rules and TypeScript types then come from one source, cross-field rules live in `.refine`, and the same schema can validate the request on a TypeScript backend.
*Follow-up:* Where does the resolver come from? The `@hookform/resolvers` package.

**Q7. How do you validate that two fields match, such as password confirmation?**
Use `.refine` on the object schema, comparing the two values and setting `path: ['confirmPassword']` so the error attaches to that field. With plain RHF rules, use `validate: (value, formValues) => value === formValues.password || 'Passwords do not match'`. Re-validate the confirm field when the password changes (`trigger`).
*Follow-up:* Should the server also check it? The server checks the password itself; confirmation is a client UX aid.

**Q8. How do you display server-side validation errors from a Rails or DRF API?**
Normalize the response (Rails: `errors` keyed by attribute; DRF: keys per field plus `non_field_errors`), then call `setError(field, { type: 'server', message })` for known fields and `setError('root.server', ...)` for the rest. Render `errors.root?.server` near the submit button and focus the first failing field.
*Follow-up:* What happens when the user edits the field? RHF clears that field's error on the next validation of the field.

**Q9. How do you share validation between frontend and backend?**
With a TypeScript backend, put Zod schemas in a shared package and use them in the form resolver and in the API input validation (oRPC input schemas, Hono `zValidator`), so types and rules match. With Rails or DRF, keep server validations as the source of truth, mirror the obvious rules for UX, and map server errors back onto fields.
*Follow-up:* Is client validation a security measure? No, anyone can call the API directly.

**Q10. When should validation run: on change, on blur or on submit?**
Validate on submit first so users are not nagged while typing, then re-validate on change so errors clear as they fix them; or validate on blur once a field has been touched. Expensive or server checks (email taken) run debounced or on blur. RHF's defaults (`mode: 'onSubmit'`, `reValidateMode: 'onChange'`) match this.
*Follow-up:* Why not validate every keystroke from the start? An email field would be "invalid" after the first character.

**Q11. How do you make forms accessible?**
Visible labels tied to inputs (`htmlFor` and `id`, generated with `useId`), `aria-invalid` on failing fields, `aria-describedby` linking the error text, `role="alert"` or an `aria-live` region for messages, focus moved to the first invalid field, fieldsets with legends for groups, and real buttons. Do not rely on color or placeholders.
*Follow-up:* How would you test it? Query by role and label in Testing Library, and run an automated checker such as axe.

**Q12. How do you handle file uploads in React?**
Use an uncontrolled `<input type="file">` and read `e.target.files`; validate type and size; show a preview with `URL.createObjectURL` and revoke it in a cleanup; upload with `FormData` without setting `Content-Type`. For large files, upload directly to storage with a presigned URL, and always validate again on the server.
*Follow-up:* How do you show upload progress? `fetch` has no simple upload progress event; use `XMLHttpRequest` (`upload.onprogress`) or a library built on it.

**Q13. How do you prevent double submission?**
Disable the button while the request runs (`isSubmitting` from RHF, `isPending` from a mutation), guard the handler with a ref for very fast double clicks, and make the server idempotent (idempotency keys for payments, unique constraints for sign-ups). The UI guard is convenience; the server guard is correctness.
*Follow-up:* Why might the disabled state not be enough? Two clicks can land before React re-renders the disabled button.

**Q14. How would you build a debounced search input?**
Keep the input controlled for responsiveness, derive a debounced copy of the query with `useDebounce` (300 ms is typical), and fetch with the debounced value in the query key and `enabled` for a minimum length. Abort or ignore stale requests: TanStack Query passes a `signal`; in a plain effect use an `AbortController`.
*Follow-up:* Debounce or `useDeferredValue`? Debounce for network requests, `useDeferredValue` for expensive local filtering.

**Q15. What is useFieldArray for, and why use field.id as the key?**
It manages dynamic lists of fields (invoice lines, attendees) with `fields`, `append`, `remove`, `insert` and `move`, and registers them as `items.${index}.name`. `field.id` is a stable generated id; index keys would attach input state to the wrong rows after a removal or reorder.
*Follow-up:* How do you validate the array? With a Zod array schema (`z.array(...).min(1)`) through the resolver.

**Q16. What are React 19 form actions, and how do they differ from onSubmit?**
You pass a function to `<form action={fn}>`; React calls it with the `FormData` inside a transition, exposes pending state (`useActionState`, `useFormStatus`), and resets uncontrolled fields after success. With Server Functions the form can even work before JavaScript loads. `onSubmit` is a plain event handler where you manage `preventDefault`, pending state and resets yourself.
*Follow-up:* Does this replace React Hook Form? Not for complex client-side validation; they solve different problems and can be combined.

## Further reading
- [Reacting to input with state](https://react.dev/learn/reacting-to-input-with-state)
- [React Hook Form docs](https://react-hook-form.com/)
- [Zod docs](https://zod.dev/)
