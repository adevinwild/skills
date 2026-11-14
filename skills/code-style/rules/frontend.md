# Frontend rules

These rules apply to React components, pages, and client state.

## Layout

### FE1. Group component declarations by kind

Order: hooks and data, state, derived values, handlers, effects ([FE8](#fe8-effects-go-last-named-by-intent)). Separate the groups with a blank line. See [G9](general.md#g9-group-declarations-by-kind).

```tsx
// ✅
export default function SearchPage(): JSX.Element {
  const router = useRouter()
  const { data: products } = useProducts()

  const [query, setQuery] = useState("")
  const [isMenuOpen, setIsMenuOpen] = useState(false)

  const results = products.filter((product) => product.name.includes(query))
  const hasResults = results.length > 0

  function handleSelect(product: Product) {
    setIsMenuOpen(false)
    router.push(`/products/${product.slug}`)
  }

  return (/* ... */)
}

// ❌
export default function SearchPage(): JSX.Element {
  const [query, setQuery] = useState("")
  const router = useRouter()
  const results = products.filter((product) => product.name.includes(query))
  const [isMenuOpen, setIsMenuOpen] = useState(false)
  const { data: products } = useProducts()
  // ...
}
```

## Errors

### FE2. Put error boundaries around parts that can fail

Wrap each independent part of the page that loads data or can throw. Then one failure does not blank the full page. Show a fallback with a retry action.

Query error states (FE3) do not replace a boundary: they miss render errors. No boundary in the repo and no `react-error-boundary`: write a small class boundary (TS16). Do not skip the rule to avoid a dependency.

```tsx
// ✅
<ErrorBoundary fallback={<SectionError onRetry={refetch} />}>
  <RecommendedProducts />
</ErrorBoundary>

// ❌
<RecommendedProducts />
```

### FE3. Handle loading, empty, and error states

Each view that loads data shows all three states. Do not render only the success state.

```tsx
// ✅
if (isPending) return <OrdersSkeleton />
if (error) return <OrdersError onRetry={refetch} />
if (orders.length === 0) return <EmptyOrders />

return <OrderList orders={orders} />
```

### FE8. Effects go last, named by intent

Put every `useEffect` and `useLayoutEffect` after the handlers, just before the `return`. Give each effect a name that says what it is for: a named function expression, or a one-line comment above an anonymous one.

```tsx
// ✅
function handleSelect(product: Product) { /* ... */ }

useEffect(function closeOnEscape() {
  function handleKeyDown(event: KeyboardEvent) {
    if (event.key === "Escape") setIsMenuOpen(false)
  }

  window.addEventListener("keydown", handleKeyDown)

  return () => window.removeEventListener("keydown", handleKeyDown)
}, [])

// Focus follows the moved card after it remounts.
useLayoutEffect(() => {
  focusRef.current?.focus()
})

return (/* ... */)

// ❌
useEffect(() => { /* ... */ }, [query])   // before the derived values, no name, no comment
const results = products.filter(/* ... */)
```

## Performance

### FE4. Do not derive state with an effect

Calculate derived values during render. An effect that sets state from props or other state causes an extra render.

```tsx
// ✅
const fullName = `${user.firstName} ${user.lastName}`

// ❌
const [fullName, setFullName] = useState("")

useEffect(() => {
  setFullName(`${user.firstName} ${user.lastName}`)
}, [user])
```

### FE5. Load only what the screen needs

- Fetch data in parallel, not one request after another, when the requests do not depend on each other.
- Lazy load heavy components that are not visible on first paint.
- Keep state near the component that uses it, so a change does not render the full tree.

Measure before you add `useMemo`, `useCallback`, or `memo`. If the repo uses the React Compiler, do not add them.

## Composition

### FE6. Compose with children, not with flags

When a component grows a boolean prop for each variant, split it into small parts and pass them as children.

```tsx
// ✅
<Card>
  <CardHeader title="Orders" />
  <OrderList orders={orders} />
  <CardFooter>
    <Button>View all</Button>
  </CardFooter>
</Card>

// ❌
<Card title="Orders" showHeader showFooter footerButtonLabel="View all" isList items={orders} />
```

### FE7. Keep data and display apart

A display component gets data through props and does not fetch. A container component or the page fetches and passes data down. Then you can reuse and test the display component alone.

### FE9. Name handlers `handleX`, callback props `onX`

```tsx
// ✅
type RowProps = { row: Row; onSelect: (id: string) => void }

function Row({ row, onSelect }: RowProps): JSX.Element {
  function handleClick() {
    onSelect(row.id)
  }

  return <button onClick={handleClick}>{row.name}</button>
}

// ❌
function Row({ row, selectRow }: RowProps): JSX.Element {
  function onClick() {
    selectRow(row.id)
  }
  // ...
}
```

### FE10. Destructure props in the signature

```tsx
// ✅
function OrderRow({ order, onSelect }: OrderRowProps): JSX.Element {
  // ...
}

// ❌
function OrderRow(props: OrderRowProps): JSX.Element {
  // props.order, props.onSelect
}
```
