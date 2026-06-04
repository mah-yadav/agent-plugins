# Node Testing: runners, commands, coverage, mocking

Load this for "how are tests set up", "how do I run one test", coverage, and
mocking approaches.

## Test runner detection

| Runner | Indicator |
|---|---|
| **Jest** | `jest` in devDeps, `jest.config.{js,ts}`, `__tests__/` folders or `*.test.*` files |
| **Vitest** | `vitest`, `vitest.config.{js,ts}` — Vite-powered, ESM-native |
| **Mocha** | `mocha`, `.mocharc.{js,json,yml}` |
| **Node test runner** | `node --test` in scripts, `import { test } from 'node:test'` |
| **AVA** | `ava` in devDeps, `ava` key in package.json |
| **Tap / node-tap** | `tap` |
| **Playwright** | `@playwright/test` — E2E browser testing |
| **Cypress** | `cypress` — E2E browser testing |
| **Supertest** | `supertest` — HTTP integration tests (paired with Jest/Vitest) |

## Test commands

| What | Common commands |
|---|---|
| Unit tests | `npm test` / `pnpm test` / `yarn test` — find in `package.json` `scripts.test` |
| Watch mode | `jest --watch`, `vitest`, `node --test --watch` |
| Coverage | `jest --coverage`, `vitest --coverage`, `c8 node --test` |
| Single test file | `jest path/to/test`, `vitest run path/to/test` |
| Single test by name | `jest -t "test name"`, `vitest -t "test name"` |
| E2E | `npx playwright test`, `npx cypress run`, custom scripts |

## Coverage

- **Istanbul / nyc**: legacy coverage tooling.
- **c8**: native Node V8 coverage, no instrumentation.
- **Jest --coverage** / **Vitest --coverage**: built-in via c8.
- Thresholds: `coverageThreshold` in Jest config, `coverage.thresholds` in Vitest config.

## Mocking

| Approach | How |
|---|---|
| Jest mocks | `jest.mock('path')`, `jest.fn()`, `jest.spyOn(...)` |
| Vitest mocks | `vi.mock('path')`, `vi.fn()`, `vi.spyOn(...)` |
| `sinon` | Cross-runner stubs, spies, fakes |
| MSW (Mock Service Worker) | HTTP-level mocking via fetch interception |
| `nock` | HTTP mocking via `http.request` interception |
| `testcontainers` | Real services in Docker for integration tests |

## Common test types

- **Unit**: pure-function tests, single-module tests with mocks.
- **Integration**: tests touching DB/HTTP/queue — usually with Testcontainers, MSW, or test-specific docker-compose.
- **Contract**: Pact (`@pact-foundation/pact`) for consumer-driven contracts.
- **Snapshot**: `expect(x).toMatchSnapshot()` — verify shape stability.
