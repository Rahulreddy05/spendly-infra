# Spendly Engineering Standards

These rules apply to **every change** in every Spendly repo (`spendly-api`,
`spendly-web`, `spendly-mobile`, `spendly-shared`, `spendly-infra`). They are not optional and are not restated in
task requests — read this file before writing code and check your change
against the Definition of Done at the bottom before calling it finished.

The same file lives in each repo. If you change it, change all five.

---

## 1. Product

Spendly shows a signed-in user **where their money comes from and where it goes**:

- How much came in (income) and how much went out (expenses), per year and month
- What they spend the most on, and where most of their income comes from
- Per-account breakdowns

Data comes from two sources, shown together in the same analytics:

1. **Linked accounts** — pulled from the user's bank through a `BankDataProvider`.
   Plaid (US banks) is the first provider. Indian banks
   (Account Aggregator, e.g. Setu) are planned; adding one must mean adding a new
   provider class and registering it, **not** editing the services that use providers.
2. **Manual accounts** — cash or anything not linkable; entries typed by the user.

## 2. Production readiness

Every feature is built as if it ships to real users today:

- Input is validated at the boundary (zod on the API, form validation on the web).
- Errors go through one central handler; internals (stack traces, ORM errors,
  provider error bodies) never reach the client.
- Structured logging (pino). Never log secrets, tokens, passwords, full account
  numbers, or raw provider payloads.
- Health endpoints: `/health/live` (process) and `/health/ready` (dependencies).
- Graceful shutdown on `SIGTERM`.
- Configuration comes from environment variables, validated at startup
  (`src/config/env.ts`). The app refuses to start with invalid config.
- Idempotency: anything triggered by a webhook or a retry must be safe to run twice.

## 3. Constants — no magic values

- Business constants (category lists, limits, event names, provider ids, error
  codes, route prefixes, query keys, UI labels that repeat) live in
  **`src/constants/<domain>.constants.ts`** and are imported from there.
- Never inline a magic string or number in business logic. `if (status === 'posted')`
  is wrong; `if (status === PROVIDER_TX_STATUS.POSTED)` is right.
- Constants are `as const` objects or tuples so TypeScript derives the union types
  from them — the constant is the single source of truth for the type.
- Environment-specific values (URLs, secrets, TTLs) are **config, not constants**:
  they go in `.env` / `src/config/env.ts`, never in a constants file.

## 4. Design patterns and structure

- **Layered modules**: `routes` (HTTP: parse → call service → serialise) →
  `service` (business rules) → Prisma. Routes hold no business logic.
- **Dependency injection via a composition root** (`src/container.ts` on the API).
  Services are classes that receive their dependencies in the constructor.
  Nothing outside the composition root constructs Prisma, Plaid, or providers.
  This is what makes services unit-testable with fakes.
- **Strategy + Adapter for external data** — `BankDataProvider` is the interface;
  `PlaidProvider` adapts Plaid to it. Services depend on the
  interface only. Provider-specific types never leak past the adapter.
- **Registry (factory)** — `ProviderRegistry` returns the provider for a given id.
- **Strategy for categorisation** — `TransactionCategorizer` interface; rules live
  in constants, not in code branches.
- **Pure functions for calculations** (analytics maths, mappers) so they can be
  unit-tested without a database.
- SOLID, small files, one responsibility each. Prefer composition over inheritance.
- **Shared code lives in `@rahulreddy05/spendly-shared`** (types, constants,
  API client, formatting). Web and mobile import it; never copy it into an app.
  A change there is released as a new semver version, then apps upgrade.
- The web app mirrors this: API calls via the shared client, server state in TanStack Query
  hooks in `src/hooks/`, presentational components in `src/components/`, pages in
  `src/pages/`.

## 5. Testing

- **Every feature ships with tests in the same change.** No test, not done.
- **Unit tests** for pure logic (mappers, categoriser, analytics calculations,
  formatters, hooks, components) — no network, no database.
- **Integration tests** for API routes against the real test database
  (`spendly_test`), using `app.inject`.
- **External services are always faked in tests.** Tests never call Plaid or any
  real provider; inject a fake `BankDataProvider` through the container.
- Test the security rules explicitly: user scoping (user A can never see user B's
  data), validation rejections, webhook signature rejection.
- Coverage thresholds are enforced in `vitest.config.ts`; do not lower them.
- `npm run typecheck && npm run lint && npm test` must pass before any commit.

## 6. Security and money

- **Money is integer minor units** (`amountCents`). Never floats. Format only at the
  presentation edge. Amounts are stored positive; `direction` (`INCOME`/`EXPENSE`)
  says which way it moved.
- **No full card or account numbers**, ever. Last 4 digits only. No CVV, no expiry.
- **Every query is scoped by the authenticated `userId`.**
- Webhooks are verified with the provider's signature before anything is read.
- Secrets live in `.env` locally and Kubernetes `Secret`s in clusters — never in
  git, images, ConfigMaps, or logs.
- Web: access token in memory only; never `localStorage`.
- Plaid keys: sandbox keys for development. Production keys only in the
  production Secret. Bank access tokens are stored encrypted and never sent to clients.

## 7. Docker and Kubernetes

- Every deployable has a multi-stage `Dockerfile`: small runtime image, non-root
  user, no dev dependencies, `HEALTHCHECK`.
- `spendly-infra` holds `docker-compose.yml` (full local stack) and Kubernetes
  manifests using **Kustomize**: `k8s/base` + `k8s/overlays/{local,prod}`.
- Every Deployment has: resource requests **and** limits, liveness/readiness/startup
  probes, `securityContext` (non-root, read-only root fs, no privilege escalation,
  drop all capabilities), rolling updates with `maxUnavailable: 0`.
- Stateless services scale with a `HorizontalPodAutoscaler` and are protected by a
  `PodDisruptionBudget`. Spread replicas with `topologySpreadConstraints`.
- Config in `ConfigMap`, secrets in `Secret`, database migrations run before the
  new API version serves traffic.
- `NetworkPolicy`: only the API may reach the database.
- The web app and API are served on one origin via Ingress (`/` → web,
  `/api` → API), so production needs no CORS.

## 8. Mobile (spendly-mobile)

- React Native with **Expo** (managed workflow, Expo Router). `ios/` and
  `android/` are generated — configure native behaviour in `app.json` and
  config plugins, never by hand. Add packages with `npx expo install`.
- Tokens: access token in memory; refresh token in the Keychain/Keystore via
  `expo-secure-store`. Never AsyncStorage for secrets.
- Every request sends `X-Client-Platform` / `X-Client-Version`. A breaking API
  change must raise `MIN_IOS_APP_VERSION` / `MIN_ANDROID_APP_VERSION` and keep
  older app versions working until then — installed apps are not updated instantly.
- Accessibility: every interactive element has a role and label; touch targets
  are at least 44pt; charts have a text equivalent.
- Tests: Jest (`jest-expo`) + React Native Testing Library, routes tested with
  `expo-router/testing-library`. Never call the real API or native modules.
- Builds and store releases go through EAS; Docker/Kubernetes apply to the
  backend and web only.

## 9. Code style and git

- TypeScript strict mode everywhere. No `any` without a comment saying why.
- ESLint + Prettier; zero warnings.
- Match the surrounding code's naming, comment density, and idioms.
- Conventional commits: `feat:`, `fix:`, `test:`, `chore:`, `docs:`, `refactor:`.
- API response shape changes are made in the API **and** the web types in the same
  piece of work (`spendly-web/src/types/api.ts` mirrors the API).
- Keep `CLAUDE.md` in each repo up to date with the data model and API surface.

---

## Definition of Done

- [ ] Follows sections 2–9 above
- [ ] No magic values — new constants are in `src/constants/`
- [ ] Unit and/or integration tests added and passing; coverage thresholds met
- [ ] `typecheck`, `lint`, `test`, and `build` all pass
- [ ] Docker image builds; Kubernetes manifests updated if config/env changed (backend/web)
- [ ] Mobile: `npx expo-doctor` passes and iOS + Android bundles build (`npx expo export`)
- [ ] `.env.example`, `CLAUDE.md`, and README updated if behaviour or config changed
