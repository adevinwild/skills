# TypeScript rules

These rules apply to `.ts`, `.tsx`, `.js`, and `.jsx` files.

## Formatting

### TS1. Do not use semicolons

Omit semicolons at the end of statements. If the repo's formatter or lint config requires semicolons, follow the config.

```ts
// ✅
const total = subtotal - discount

// ❌
const total = subtotal - discount;
```

## Types

### TS2. Use `type`, not `interface`

```ts
// ✅
type User = {
  id: number
  name: string
}

// ❌
interface User {
  id: number
  name: string
}
```

### TS3. Do not use `any`

`any` turns off type checking. Use a precise type. When the type is not known, use a generic or `unknown`, then narrow it. For external input, use a schema validator. See [BE1](backend.md#be1-validate-every-input-with-a-schema-validator).

```ts
// ✅
function first<T>(items: T[]): T | undefined {
  return items[0]
}

function getMessage(error: unknown) {
  return error instanceof Error ? error.message : String(error)
}

// ❌
function first(items: any[]): any {
  return items[0]
}

function getMessage(error: any) {
  return error.message
}
```

### TS11. Prefer a string union over an `enum`

```ts
// ✅
type Status = "draft" | "live" | "archived"

// ❌
enum Status {
  Draft = "draft",
  Live = "live",
  Archived = "archived",
}
```

### TS12. Prefer a type annotation over an `as` cast

`as` tells TypeScript to trust you, so a wrong value passes. An annotation checks the value.

```ts
// ✅
const config: Config = {
  retries: MAX_RETRIES,
  mode: "strict",
}

// ❌
const config = {
  retries: MAX_RETRIES,
  mode: "strict",
} as Config
```

### TS13. Use `?` for a field that can be missing

Add `| null` only when the source really sends `null`, for example a database column or an API response.

```ts
// ✅
type User = {
  nickname?: string
}

type UserRow = {
  nickname?: string | null
}

// ❌
type User = {
  nickname: string | undefined
}
```

### TS14. Keep a type next to its only user

A type used by one file goes at the top of that file. Move it to a shared file only when a second file needs it.

```ts
// ✅ user-card.tsx
type UserCardProps = {
  user: User
}

// ❌ types/user-card.ts, imported only by user-card.tsx
export type UserCardProps = {
  user: User
}
```

## Functions and exports

### TS4. Use `function` declarations for named functions

Function declarations are hoisted and show `function` at the start of the line. Use arrow functions only for inline callbacks.

```ts
// ✅
function getUserData(id: number) {
  // ...
}

const names = users.map((user) => user.name)

// ❌
const getUserData = (id: number) => {
  // ...
}
```

### TS5. Export at the declaration

The reader sees that the item is public where it is defined.

```ts
// ✅
export default function fetchData(): Promise<Data> {
  // ...
}

// ❌
function fetchData(): Promise<Data> {
  // ...
}

// many lines later
export default fetchData
```

### TS6. Use a default export for the main item of a file

```ts
// ✅
export default function UserProfile(): JSX.Element {
  // ...
}

// ❌
export function UserProfile() {
  // ...
}

// ❌
export const UserProfile = () => {
  // ...
}
```

### TS7. Use one object parameter when a function takes more than 3 arguments

Named fields read clearly at the call site and do not depend on order. Destructure the object in the signature.

```ts
// ✅
type CreateUserInput = {
  name: string
  email: string
  role: Role
  isActive: boolean
}

function createUser({ name, email, role, isActive }: CreateUserInput) {
  // ...
}

createUser({ name: "Ada", email: "ada@example.com", role: "admin", isActive: true })

// ❌
function createUser(name: string, email: string, role: Role, isActive: boolean) {
  // ...
}

createUser("Ada", "ada@example.com", "admin", true)
```

### TS8. Prefer an options object over a boolean argument

A bare `true` at the call site does not say what it turns on. Pass it as a named field.

```ts
// ✅
formatDate(date, { shouldIncludeTime: true })

// ❌
formatDate(date, true)
```

### TS15. Prefer an explicit return type on exported functions

```ts
// ✅
export default async function getUser(id: string): Promise<User> {
  // ...
}

// ❌
export default async function getUser(id: string) {
  // ...
}
```

### TS16. Use a class for a long-lived object that holds state

An API client, a cache, or a rate limiter keeps state across calls. Write it as a class, not a closure factory. Do not use classes for data or stateless helpers. Typed errors and React error boundaries are classes too.

```ts
// ✅
class StripeClient {
  private cache = new Map<string, Customer>()

  constructor(private token: string) {}

  async getCustomer(id: string): Promise<Customer> {
    const cached = this.cache.get(id)
    if (cached) return cached

    const customer = await request({ token: this.token, path: `/customers/${id}` })
    this.cache.set(id, customer)

    return customer
  }
}

// ❌
function createStripeClient(token: string) {
  const cache = new Map<string, Customer>()

  async function getCustomer(id: string): Promise<Customer> {
    // ...
  }

  return { getCustomer }
}
```

## Values and errors

### TS9. Prefer `??` for defaults

`||` also replaces `0`, `""` and `false`, which are real values. `??` replaces only `null` and `undefined`. For a condition, use the repo's nullish helper when one exists (for example `isUndefinedOrNull`).

```ts
// ✅
const limit = options.limit ?? DEFAULT_LIMIT

if (isUndefinedOrNull(options.limit)) {
  // ...
}

// ❌
const limit = options.limit || DEFAULT_LIMIT
```

### TS10. Outside the server, return a result object for expected failures

In client code, scripts, and libraries, a function that can fail in a known way returns `{ ok: true, data }` or `{ ok: false, error }`. The caller must check `ok` before it can read `data`, so no failure goes unhandled. Use the repo's `Result` type when one exists.

On the server, services throw typed errors and the app error handler maps them to status codes ([BE10](backend.md#be10-handle-errors-in-one-place)).

```ts
// ✅ client
type Result<T, E = string> = { ok: true; data: T } | { ok: false; error: E }

async function getUser(id: string): Promise<Result<User>> {
  const response = await fetch(`/users/${id}`)
  if (!response.ok) return { ok: false, error: "not-found" }

  const body = await response.json()
  const user = UserSchema.parse(body)

  return { ok: true, data: user }
}

const result = await getUser(userID)
if (!result.ok) return showError(result.error)

// ❌ client
async function getUser(id: string) {
  const response = await fetch(`/users/${id}`)
  if (!response.ok) throw new Error("User not found")

  return response.json()
}
```
