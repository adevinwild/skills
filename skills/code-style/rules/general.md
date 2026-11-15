# General rules

These rules apply to all code.

## Naming

### G1. Use kebab-case for file names

- ✅ `user-profile.ts`
- ❌ `UserProfile.ts`, `userProfile.ts`, `user_profile.ts`

### G2. Use descriptive names

A name tells the reader what the value is. Single letters are only for loop indexes, generic type parameters (`T`), and names a framework fixes (Hono's `c`).

```ts
// ✅
const defaultWidth = 100
const activeUsers = users.filter((user) => user.isActive)

// ❌
const d = 100
const au = users.filter((u) => u.isActive)
```

### G3. Write abbreviations in full capitals

```ts
// ✅
const page = "PDP"
function PDPImages() {}
const userID = "42"

// ❌
const page = "Pdp"
function PdpImages() {}
const userId = "42"
```

### G4. Write user-facing strings in sentence case

Capitalize only the first word and proper nouns.

```ts
// ✅
const title = "Product cards"
const message = "Acme: a web design and development agency"

// ❌
const title = "Product Cards"
const message = "Acme: a Web Design and Development Agency"
```

### G18. Name a boolean with `is`, `has` or `should`, then its subject

```ts
// ✅
const isModalOpen = true
const hasUnsavedChanges = false

// ❌
const open = true
const modalOpen = true
const isOpen = true
```

## Values

### G5. Name every number, and every string used in more than one place

A number in the logic hides its meaning. Name it once. Calculate dependent values from that name, so one change updates all of them. A string that appears once and reads clearly stays inline (`"Escape"`, `"free"`, status codes like `201`). Name it when a second place uses it.

```ts
// ✅
const MAX_RETRIES = 3
const RETRY_DELAY_MS = 500
const MAX_WAIT_MS = MAX_RETRIES * RETRY_DELAY_MS

if (attempt > MAX_RETRIES) throw new Error("Too many retries")

// ❌
if (attempt > 3) throw new Error("Too many retries")

const maxWait = 1500
```

Obvious values do not need a name: `0`, `1`, `-1`, `""`, `index + 1`, `length / 2`.

### G6. Name each async result before you use it

Do not pass an `await` into another call. Put each step on its own line with a name that says what it holds.

```ts
// ✅
const user = await getUser(userID)
const invoices = await getInvoices(user.accountID)

// ❌
const invoices = await getInvoices((await getUser(userID)).accountID)
```

The same applies to deep nested calls that are not async. If a reader must read from the inside out, split it.

### G13. Run independent async steps in parallel

When two async steps do not need each other's result, start them together with `Promise.all`. Await in sequence when a step uses the result of the one before, or when order matters: writes inside a transaction, ordered writes, or calls to a rate-limited API.

```ts
// ✅
const [user, settings] = await Promise.all([
  getUser(userID),
  getSettings(userID),
])

// ❌
const user = await getUser(userID)
const settings = await getSettings(userID)
```

### G14. Prefer new values over mutating the ones you received

Build a new array or object with spread, inline. Do not wrap a one-line spread in a helper.

```ts
// ✅
const nextItems = [...items, item]
const nextUser = { ...user, name }

// ❌
items.push(item)

// ❌
function addItem(items: Item[], item: Item) {
  return [...items, item]
}
```

State that a long-lived object owns (a cache inside a client, [TS16](typescript.md#ts16-use-a-class-for-a-long-lived-object-that-holds-state)) is not a received value. It can change in place.

### G22. Return `null` when a lookup finds nothing

`null` means "looked, found nothing". `undefined` means "not set" ([TS13](typescript.md#ts13-use--for-a-field-that-can-be-missing)).

```ts
// ✅
function findUser(id: string): User | null {
  return users.find((user) => user.id === id) ?? null
}

// ❌
function findUser(id: string): User | undefined {
  return users.find((user) => user.id === id)
}
```

## Control flow

### G15. Prefer one ternary for a two-way return

When a function returns one of two values, write a single ternary. Never chain ternaries: use a lookup object ([G16](#g16-map-keys-to-values-with-a-lookup-object)).

```ts
// ✅
function getPlan(user?: User) {
  return user?.isActive ? user.plan : "free"
}

// ❌
function getPlan(user?: User) {
  if (user) {
    if (user.isActive) {
      return user.plan
    }
  }

  return "free"
}
```

### G16. Map keys to values with a lookup object

Use a lookup object, not a `switch` or chained ternaries. Type it as `Record<Key, Value>` so a missing key fails type checking.

```ts
// ✅
const STATUS_LABELS: Record<Status, string> = {
  draft: "Draft",
  live: "Live",
  archived: "Archived",
}

const label = STATUS_LABELS[status]

// ❌
switch (status) {
  case "draft":
    return "Draft"
  case "live":
    return "Live"
  case "archived":
    return "Archived"
}

// ❌
const label = status === "draft" ? "Draft" : status === "live" ? "Live" : "Archived"
```

### G17. Use array methods, not a loop with an accumulator

Use `filter`, `map` and `reduce`. Do not write a `for` or `forEach` loop that adds to an outer variable.

```ts
// ✅
const total = orders
  .filter((order) => order.isPaid)
  .reduce((sum, order) => sum + order.price, 0)

// ❌
let total = 0

for (const order of orders) {
  if (order.isPaid) {
    total += order.price
  }
}
```

Use `for...of` only when each step must `await` the one before, or the loop must `break`.

### G21. Write a guard clause on one line

When an `if` only exits early with one `return` or `throw`, write it on one line without braces. Use braces when the body has more than one statement.

```ts
// ✅
if (!user) return null
if (isPending) return <OrdersSkeleton />

// ❌
if (!user) {
  return null
}
```

## Layout

### G7. Order statements by data flow

Put each statement after the values it uses. Keep related statements together. Separate each group with a blank line.

```ts
// ✅
function checkout(cart: Cart, user: User) {
  const subtotal = sum(cart.items.map((item) => item.price))
  const discount = getDiscount(user, subtotal)
  const total = subtotal - discount

  const address = user.addresses.find((entry) => entry.isDefault)
  const shipping = getShippingCost(address, total)

  return { total: total + shipping, address }
}

// ❌
function checkout(cart: Cart, user: User) {
  const address = user.addresses.find((entry) => entry.isDefault)
  const subtotal = sum(cart.items.map((item) => item.price))
  const discount = getDiscount(user, subtotal)
  const shipping = getShippingCost(address, subtotal - discount)
  const total = subtotal - discount
  return { total: total + shipping, address }
}
```

### G8. Label a group only when it is a separate flow

A label names a step that a reader can skip as one unit. Do not label each group.

```ts
// ✅
async function syncOrders() {
  // Pull
  const [remoteOrders, localOrders] = await Promise.all([
    api.listOrders(),
    db.orders.findMany(),
  ])

  // Reconcile
  const localIDs = new Set(localOrders.map((order) => order.id))
  const missingOrders = remoteOrders.filter((order) => !localIDs.has(order.id))
  await db.orders.createMany(missingOrders)
}

// ❌
function getTotal(cart: Cart) {
  // Get prices
  const prices = cart.items.map((item) => item.price)

  // Sum prices
  const total = sum(prices)

  return total
}
```

### G10. Separate declarations from the `return`

```ts
// ✅
function getFullName(user: User) {
  const fullName = `${user.firstName} ${user.lastName}`

  return fullName.trim()
}

// ❌
function getFullName(user: User) {
  const fullName = `${user.firstName} ${user.lastName}`
  return fullName.trim()
}
```

### G19. Split file sections with blank lines, not heading comments

When constants are shared, move them to a `constants.ts`: one per feature folder when the modules in that feature share them, or one at the root when the whole app does.

```ts
// ✅
const MAX_RETRIES = 3
const RETRY_DELAY_MS = 500

let attempts = 0

// ❌
//
// Constants
//

const MAX_RETRIES = 3
```

### G20. Put helpers above the main export

File order: imports, types, constants, private helpers, then the main export last.

```ts
// ✅
type OrderListProps = { orders: Order[] }

const MAX_ROWS = 10

function toRows(orders: Order[]): Row[] {
  return orders.slice(0, MAX_ROWS).map((order) => ({ id: order.id, label: order.name }))
}

export default function OrderList({ orders }: OrderListProps): JSX.Element {
  const rows = toRows(orders)

  return <Table rows={rows} />
}

// ❌ main export first, helpers below
export default function OrderList({ orders }: OrderListProps): JSX.Element { /* ... */ }

function toRows(orders: Order[]): Row[] { /* ... */ }
```

### G24. Group files by feature

Frontend: one folder per feature, shared UI in a root folder. Backend: one folder per domain or module. When the repo already has a layout, follow it.

```
src/
  features/
    orders/
      order-list.tsx
      use-orders.ts
      constants.ts
      __tests__/
  ui/
    button.tsx
```

## Comments and tests

### G11. Keep only comments that give a reason

Delete a comment that says what the code already says. Keep a comment that tells why, when the reason is not clear from the code.

```ts
// ✅
// Stripe sends amounts in cents.
const amount = event.amount / CENTS_PER_UNIT

// ❌
// Divide amount by 100
const amount = event.amount / 100
```

### G12. Do not write tests that check nothing a human would check

A test must check behavior that can break. Do not write tests that only repeat the implementation or test the language. Remove existing ones only when the task touches them.

```ts
// ✅
it("should apply the member discount to the subtotal", () => {
  expect(getTotal(cart, member)).toBe(90)
})

// ❌
it("is a function", () => {
  expect(typeof getTotal).toBe("function")
})

it("returns the mocked value", () => {
  vi.mocked(getDiscount).mockReturnValue(10)
  expect(getDiscount()).toBe(10)
})
```

### G23. Put tests in `__tests__/`, name cases with `should`

Tests live in a `__tests__/` folder next to the code. Group by subject with `describe`. Each case reads as `it("should ...")`.

```ts
// ✅ src/orders/__tests__/get-total.test.ts
describe("getTotal", () => {
  it("should apply the member discount", () => {
    // ...
  })
})

// ❌ src/orders/get-total.test.ts
test("discount works", () => {
  // ...
})
```
