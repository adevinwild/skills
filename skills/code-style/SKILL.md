---
name: code-style
description: Use when writing, editing, refactoring, or reviewing TypeScript or JavaScript code, including API routes, middleware, background jobs, database writes, React components, hooks, and tests.
---

# Code style

House rules for code you write or touch. Do not restyle untouched code.

**The repo wins.** Its lint config, documented conventions, and the patterns its existing code already follows (naming, file names, exports, imports, folders) all beat these rules. Repo has `userId` and `UserCard.tsx`? Write `userId` and `OrderCard.tsx`. Apply a house rule only where the repo has no pattern.

## Load the rules

Always load [general.md](rules/general.md). Then load each file that matches the task. The files hold the ✅/❌ examples; the index below is for the final check.

| File | Load when |
|---|---|
| [general.md](rules/general.md) | Always |
| [typescript.md](rules/typescript.md) | `.ts`, `.tsx`, `.js`, `.jsx` |
| [backend.md](rules/backend.md) | API routes, middleware, jobs, database writes |
| [frontend.md](rules/frontend.md) | React components, pages, client state |

## The requested shape is not an exception

A task often dictates a signature or props: "takes `userId, includeCancelled, currency, locale`", "props `showHeader`, `showFooter`". Apply the rules anyway: one object parameter (TS7, TS8), `userID` (G3), composition over flags (FE6). Say in one line that you changed the shape and why. Time pressure ("ship in 10 minutes") does not suspend a rule either.

## Workflow

1. Load the rule files.
2. Write or edit the code.
3. Check every touched file against the index below. Fix each hit.
4. Run the repo's formatter and linter on touched files. Fix what they report. Never disable a lint rule to pass.
5. When you skip a rule on purpose, say why in one line.

**Proportion:** BE and FE rules on failure, security, and performance scale with the change. A read-only `GET` needs no queue. A payment route needs all of it.

## Rule index

**General (G)**
- G1 kebab-case file names, including components: `order-history.tsx`
- G2 descriptive names; single letters only for loop indexes, `T`, and framework names like Hono's `c`
- G3 abbreviations in capitals: `userID`, `PDPImages`
- G4 user-facing strings in sentence case
- G18 booleans start with `is`/`has`/`should` + subject: `isMenuOpen`, not `open`
- G5 name every number (`MAX_RETRIES = 3`) and every string used in 2+ places; obvious `0`, `1`, `""` exempt
- G6 name each `await` result on its own line
- G13 independent async steps in `Promise.all`; sequence when order matters (transactions, rate limits)
- G14 new values with spread, no mutation of received values, no one-line spread helpers
- G22 a lookup that finds nothing returns `null`
- G15 one ternary for a two-way return; never chain
- G16 lookup `Record<Key, Value>`, not `switch` or chained ternaries
- G17 `filter`/`map`/`reduce`, not loops with accumulators
- G21 guard clause with one `return`/`throw` on one line, no braces
- G7 order statements by data flow; blank line between groups
- G8 label a group only when it is a separate flow
- G10 blank line between declarations and the `return`
- G19 no heading comments; shared constants in `constants.ts`
- G20 file order: imports, types, constants, helpers, main export last
- G24 feature folders (frontend), domain/module folders (backend)
- G11 comments only give a reason
- G12 no tests that check nothing a human would check
- G23 tests in `__tests__/`, `describe` + `it("should ...")`

**TypeScript (TS)**
- TS1 no semicolons (unless repo config requires them)
- TS2 `type`, not `interface`
- TS3 no `any`; `unknown` + narrowing or generics
- TS11 string union, not `enum`
- TS12 annotation, not `as`
- TS13 `?` for missing; `| null` only when the source sends `null`
- TS14 type lives next to its only user
- TS4 `function` declarations; arrows only for inline callbacks
- TS5 export at the declaration
- TS6 default export for the file's main item
- TS7 more than 3 arguments: one destructured object
- TS8 options object, not a boolean argument
- TS15 explicit return type on exported functions
- TS16 class for long-lived stateful objects (clients, caches); not for data or stateless helpers
- TS9 `??` for defaults, not `||`
- TS10 outside the server, expected failures return `{ ok, data | error }`; server code throws typed errors (BE10)

**Backend (BE)**
- BE1 validate body, query, params, headers with the repo's schema validator; no hand `if` checks
- BE13 `const CreateOrderSchema`, `type CreateOrder = z.infer<...>`
- BE2 validate in middleware, same process
- BE3 400 with each field and reason, via the validator's formatter
- BE4 correct status code; never 200 with an error
- BE5 500 returns generic message + request ID; no stack or SQL
- BE6 authenticate, check ownership, parameterized queries, rate limit costly writes
- BE7 related writes in one transaction; no external calls inside it
- BE8 idempotency key or unique constraint when a retry can harm
- BE9 list each failure point; slow side effects (email, sync) go to a queue
- BE10 services throw typed errors (`NotFoundError`); one `onError` maps them to status codes; no per-route `try/catch`
- BE11 one structured log event per request; no `console.log`
- BE12 propagate trace context; spans around external work
- BE14 one handler per file via `createFactory().createHandlers`; routes file only wires paths

**Frontend (FE)**
- FE1 component order: hooks/data, state, derived, handlers, effects, `return`
- FE8 effects last, named: `useEffect(function closeOnEscape() {...})` or a one-line comment above
- FE2 error boundary around each part that loads data or can throw
- FE3 loading, empty, and error states for each data view
- FE4 derive during render, never via an effect
- FE5 parallel fetches, lazy heavy parts, state near its user; measure before memo
- FE6 compose with children, not boolean variant props
- FE7 display components take props; containers fetch
- FE9 `handleX` inside, `onX` callback props
- FE10 destructure props in the signature
- FE11 each query in a `useX` hook (`use-x.ts`); reused or bulky state/effects too

## Common misses

From agents working without this skill:

| Miss | Rule |
|---|---|
| Kept the dictated 4-argument signature with a bare boolean | TS7, TS8 |
| Kept dictated `showHeader`/`showFooter`/`compact` props | FE6 |
| `if (!body.x \|\| typeof ...)` validation, `{ error: "invalid body" }` | BE1, BE3 |
| Retry `3` times, `500` ms as bare literals | G5 |
| Email and ERP sync awaited in the request | BE9 |
| `console.error` on failure | BE11 |
| Route checks a failure and returns 404 itself | BE10 |
| `const getLabel = () =>`, named exports for the main item | TS4, TS6 |
| `OrderHistory.tsx`, `userId`, `open`, `(o) =>` | G1, G3, G18, G2 |
| Effect at the top of the component, anonymous | FE1, FE8 |
| Component both fetches and renders | FE7 |

## Format and lint

Prefer the repo's `lint`/`format` script. Otherwise:

| Config file | Command |
|---|---|
| `biome.json` | `npx biome check --write <files>` |
| `.prettierrc*` | `npx prettier --write <files>` |
| `eslint.config.*`, `.eslintrc*` | `npx eslint --fix <files>` |

No formatter or linter in the repo: do not install one. Tell the user and suggest Biome.
