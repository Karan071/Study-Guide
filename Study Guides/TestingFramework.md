# Software Testing, Frontend to Backend: An SDE's Study Guide

Audience: SDEs who write, review, and maintain tests for web applications (browser frontends plus HTTP/API backends, monolith or microservices).
Scope: 26 core concepts grouped into four parts (Foundations, Frontend, Backend, Quality & Delivery). Each concept is broken into **Core Mechanism**, **Engineering Blueprint**, **Production Pitfalls**, **SDE Best Practices**, and **Resources**.
Examples use TypeScript (Vitest/Jest, React Testing Library, Playwright, Supertest) because the ideas transfer directly; equivalents for Python, Java, and Go are listed in [Section 4](#4-test-runners-and-frameworks).
Last reviewed: 2026-10-12. Tool APIs change between major versions; check the linked docs before copying a snippet into a new project.

---

## Table of Contents

0. [System Map: How the 26 Concepts Fit Together](#0-system-map-how-the-26-concepts-fit-together)

**Part A: Foundations**

1. [Why We Test and the Core Vocabulary](#1-why-we-test-and-the-core-vocabulary)
2. [Test Pyramid, Trophy, and Honeycomb](#2-test-pyramid-trophy-and-honeycomb)
3. [Test Doubles: Dummy, Stub, Spy, Mock, Fake](#3-test-doubles-dummy-stub-spy-mock-fake)
4. [Test Runners and Frameworks](#4-test-runners-and-frameworks)
5. [TDD and BDD](#5-tdd-and-bdd)

**Part B: Frontend Testing**

6. [Frontend Unit Testing (Logic, Hooks, State)](#6-frontend-unit-testing-logic-hooks-state)
7. [Component Testing (Testing Library)](#7-component-testing-testing-library)
8. [Mocking the Network in the Frontend (MSW)](#8-mocking-the-network-in-the-frontend-msw)
9. [Snapshot and Visual Regression Testing](#9-snapshot-and-visual-regression-testing)
10. [Accessibility Testing](#10-accessibility-testing)
11. [End-to-End Testing (Playwright, Cypress)](#11-end-to-end-testing-playwright-cypress)
12. [Frontend Performance Testing](#12-frontend-performance-testing)

**Part C: Backend Testing**

13. [Backend Unit Testing (Services and Domain Logic)](#13-backend-unit-testing-services-and-domain-logic)
14. [API and Integration Testing](#14-api-and-integration-testing)
15. [Database Testing (Testcontainers, Migrations, Isolation)](#15-database-testing-testcontainers-migrations-isolation)
16. [Contract Testing (Pact, OpenAPI)](#16-contract-testing-pact-openapi)
17. [Testing Async Code, Queues, and Time](#17-testing-async-code-queues-and-time)
18. [Load and Performance Testing (k6)](#18-load-and-performance-testing-k6)
19. [Security Testing](#19-security-testing)
20. [Resilience and Chaos Testing](#20-resilience-and-chaos-testing)

**Part D: Quality and Delivery**

21. [Test Data Management (Fixtures, Factories, Seeds)](#21-test-data-management-fixtures-factories-seeds)
22. [Code Coverage and Mutation Testing](#22-code-coverage-and-mutation-testing)
23. [Property-Based Testing and Fuzzing](#23-property-based-testing-and-fuzzing)
24. [Flaky Tests](#24-flaky-tests)
25. [Testing in CI/CD](#25-testing-in-cicd)
26. [Testing in Production](#26-testing-in-production)
27. [Production Checklist & Cheat Sheet](#27-production-checklist--cheat-sheet)
28. [Master Resource List](#28-master-resource-list)

---

## 0. System Map: How the 26 Concepts Fit Together

Every test answers the same question: *if I change the code, will I find out quickly and cheaply that I broke something?* The concepts differ in **how much of the system is real** during the test, and therefore in how fast, how stable, and how convincing the test is.

| Question the test answers | Layer | Concepts |
| --- | --- | --- |
| Does this function or class compute the right thing? | Unit | 3, 6, 13, 23 |
| Does this UI piece behave correctly for a user? | Component | 7, 8, 9, 10 |
| Do my code and its real dependencies (DB, HTTP, queue) work together? | Integration | 14, 15, 17 |
| Do two services agree on the shape of their API? | Contract | 16 |
| Does the whole product work for a user, end to end? | End-to-end | 11 |
| Is it fast, safe, and robust enough? | Non-functional | 12, 18, 19, 20 |
| Can we trust the suite itself? | Suite health | 21, 22, 24, 25, 26 |

### What is real and what is faked at each level

```text
                     Browser       Frontend code     Network      Backend code     Database / queue / 3rd party
                    ---------     ---------------   ---------    --------------   ------------------------------
Frontend unit [6]      -            real (1 fn)          -              -                     -
Component     [7]   jsdom (fake)    real (1 tree)    MSW (fake) [8]     -                     -
Frontend E2E  [11]  real            real              real or mocked  real or stub       real or seeded
Backend unit  [13]     -               -                 -           real (1 class)     doubles [3]
API/integr.   [14]     -               -              in-process     real (whole app)   real DB in container [15]
Contract      [16]     -           consumer side      Pact mock     provider side      provider states
Load          [18]  synthetic VUs      -              real           real               real (staging)
Prod checks   [26]  real users      real              real           real               real
```

### End-to-end request, with each concept placed

```text
User clicks "Pay"                                              Where it is tested
   |
   v
<PayButton/> renders, handles click, shows spinner        -> Component test [7], a11y [10], visual [9]
   |  usePayment() hook builds request                     -> Frontend unit [6]
   v
fetch POST /api/payments                                  -> MSW in component tests [8]
   |                                                         Contract test: consumer expectation [16]
   v
Express/Fastify route -> validation -> PaymentService     -> API test with Supertest [14]
   |                                    |                    Backend unit with fake gateway [13][3]
   |                                    v
   |                         PaymentRepo -> Postgres       -> DB test with Testcontainers [15]
   |                                    |
   |                                    v
   |                         publish "payment.created"     -> Async/queue test [17]
   v
201 JSON -> UI shows "Paid"                               -> E2E with Playwright [11]
                                                             Load test p95 < 300 ms [18]
                                                             Authz test: user A can't pay for B [19]
                                                             Gateway down -> graceful error [20]
```

### Decision defaults (use these unless you have a measured reason not to)

| Decision | Default |
| --- | --- |
| Test runner (JS/TS) | Vitest for Vite projects, Jest for existing Jest codebases; one runner per repo. |
| Component tests | React/Vue/Svelte Testing Library + `user-event`; query by role and label, not CSS classes. |
| Network in frontend tests | MSW handlers shared by unit, component, Storybook, and local dev. |
| E2E | Playwright; a small set of critical user journeys, not every edge case. |
| Backend API tests | Run the real app in-process (Supertest/`app.inject`) against a real DB in a container. |
| Database | Testcontainers with the same engine and major version as production; never SQLite as a stand-in for Postgres. |
| Test doubles | Fake or stub only at the edges you don't own (payment gateway, email, clock); use real code inside the boundary. |
| Service-to-service | Consumer-driven contract tests (Pact) or schema validation against the OpenAPI spec. |
| Coverage | Track it, gate on "no drop" for changed lines; use mutation testing on critical modules. |
| Flaky tests | Quarantine within a day, fix or delete within a sprint; never "retry until green" silently. |

---

# Part A: Foundations

## 1. Why We Test and the Core Vocabulary

A test is a small program that runs your code with known inputs and checks that the output is what you expect. Its job is to give **fast, trustworthy feedback**: it should fail when behavior breaks and pass when behavior is fine, even if the internals change.

### Core mechanism

- **System Under Test (SUT):** the thing you are checking: a function, a component, an endpoint, or the whole app.
- **Assertion:** a statement that must be true, e.g. `expect(total).toBe(42)`. A test with no assertion only proves that the code didn't throw.
- **Test case:** one scenario, one reason to fail. **Test suite:** a group of related cases (`describe` block, test file, or whole project).
- **Fixture / setup:** the state the test needs before it runs (objects, DB rows, a rendered component). **Teardown:** cleaning it up afterwards.
- **Arrange, Act, Assert (AAA)**, also written **Given, When, Then:** set up the world, do one thing, check the result. Keeping these three blocks visible is the single biggest readability win.
- **Four qualities of a good test** (Khorikov): protection against regressions, resistance to refactoring (does not break when internals change), fast feedback, maintainability. You can't max all four; unit tests trade realism for speed, E2E tests trade speed for realism.
- **Black-box vs white-box:** black-box tests use only the public interface (inputs and outputs); white-box tests know the internals. Prefer black-box: they survive refactors.
- **Test sizes (Google):** *small* = one process, no I/O, no sleep; *medium* = one machine, may use localhost DB/network; *large* = multiple machines. Size is about resources, which is a more useful label than "unit" vs "integration".

### Engineering blueprint

A well-shaped test:

```ts
import { describe, it, expect } from 'vitest';
import { applyDiscount } from './pricing';

describe('applyDiscount', () => {
  it('takes 10% off orders of 100 or more', () => {
    // Arrange
    const order = { subtotal: 200, coupon: 'SAVE10' };

    // Act
    const total = applyDiscount(order);

    // Assert
    expect(total).toBe(180);
  });

  it('ignores the coupon below the 100 threshold', () => {
    expect(applyDiscount({ subtotal: 99, coupon: 'SAVE10' })).toBe(99);
  });
});
```

Naming pattern: `<unit> <does what> <when/under which condition>`. When the test fails, the name alone should tell a reader what is broken.

| What to test | Why |
| --- | --- |
| Happy path | proves the feature works at all |
| Boundaries (0, 1, max, max+1, empty, null) | most bugs live at edges |
| Error paths (invalid input, dependency fails, timeout) | production spends a lot of time here |
| Regressions (every fixed bug) | stops it from coming back |
| Security rules (who may do what) | the cost of a miss is high |

### Production pitfalls

- Tests that assert on implementation (private methods, call order of internal helpers) and break on every refactor; teams then stop refactoring or stop trusting tests.
- Tests with many assertions about unrelated behavior; one failure hides the rest and the name lies.
- Logic in tests (`if`, loops, computing the expected value with the same algorithm as the code). A bug in the code becomes a bug in the test.
- Shared mutable state between tests (module-level variables, a DB that is never reset): tests pass alone and fail together, or depend on order.
- "Assertion-free" tests written only to raise coverage.

### SDE best practices

- One behavior per test; several `expect` calls are fine if they describe that one behavior.
- Write the expected value as a literal (`180`), not as a formula copied from the code.
- Make every test independent and runnable alone, in any order, in parallel.
- Every bug fix ships with a test that failed before the fix.
- Treat test code as production code: review it, refactor it, delete tests that no longer earn their keep.

### Resources

- [Martin Fowler: Given When Then](https://martinfowler.com/bliki/GivenWhenThen.html)
- [Google Testing Blog: Test Sizes](https://testing.googleblog.com/2010/12/test-sizes.html)
- [Software Engineering at Google, Ch. 11 to 14 (free online)](https://abseil.io/resources/swe-book)
- Vladimir Khorikov, *Unit Testing Principles, Practices, and Patterns* (Manning, 2020)

---

## 2. Test Pyramid, Trophy, and Honeycomb

These are shapes that describe **how many tests of each kind** a healthy suite has. They exist because each level costs differently: unit tests are cheap and fast but prove little about the whole; E2E tests prove a lot but are slow and brittle.

### Core mechanism

```text
  Test Pyramid (Cohn)        Testing Trophy (Dodds)         Honeycomb (Spotify, microservices)
                                                   
        /\   E2E                  ___  E2E                        ___ Integrated (few)
       /  \                      /   \                           /   \
      /----\  Service           | Integ |  <- most value        |  Integration  |  <- most tests
     /      \  (integration)    |ration |                       |  (service +   |
    /--------\                   \_____/                        |   real deps)  |
   /   Unit   \  <- most tests    |Unit|                         \___________/
  /____________\                 [Static]  types, lint              Implementation detail (few)
```

| Model | Main idea | Fits best |
| --- | --- | --- |
| Pyramid | many unit, fewer service, very few UI tests | backend with rich domain logic |
| Trophy | static analysis base; *integration* tests give the best confidence per cost | frontend apps (component tests are the "integration" layer) |
| Honeycomb | microservices have little logic and lots of wiring; test each service through its API with real dependencies | thin CRUD/glue services |

The shared rule behind all three: **push each test to the lowest level that can catch the bug**, and use higher levels only for what lower levels can't see (wiring, configuration, real browsers, real networks).

### Engineering blueprint

A practical split for a typical web product:

| Layer | Rough share | Runtime per test | Runs when |
| --- | --- | --- | --- |
| Static (TypeScript, ESLint, type-checked SQL) | always on | instant | editor + every commit |
| Unit (pure logic, hooks, domain) | 40 to 60% | < 10 ms | every save / commit |
| Component + API integration | 30 to 50% | 10 ms to 1 s | every commit / PR |
| Contract | per service pair | < 1 s | every PR on both sides |
| E2E critical journeys | 5 to 10% (tens, not thousands) | 5 to 60 s | every PR (smoke) + nightly (full) |
| Load, security, chaos | scheduled | minutes to hours | nightly / pre-release |

Shape check with real numbers: if your E2E suite takes longer than 15 minutes on a PR, or your unit tests mostly assert mock calls, the shape is wrong.

### Production pitfalls

- **Ice-cream cone (inverted pyramid):** hundreds of slow, flaky UI tests; few unit tests. CI takes an hour and is red half the time.
- **Hourglass:** many unit + many E2E, nothing in the middle; wiring bugs (wrong SQL, wrong JSON field name) are only found by slow E2E.
- Arguing about labels ("is this unit or integration?") instead of asking "what bug would this catch, and how fast?"
- Unit-testing glue code (controllers that just call a service) with heavy mocks; the test proves the mock was called, not that anything works.

### SDE best practices

- Decide per module: logic-heavy code gets unit tests; wiring-heavy code gets integration tests.
- When an E2E test catches a bug, ask "could a lower-level test have caught this?" and add that one too.
- Keep the E2E suite as a short list of business-critical journeys (sign up, log in, checkout, core CRUD).
- Measure suite runtime and flake rate per layer; the shape should be driven by data, not dogma.

### Resources

- [Martin Fowler: The Practical Test Pyramid](https://martinfowler.com/articles/practical-test-pyramid.html)
- [Kent C. Dodds: The Testing Trophy and Testing Classifications](https://kentcdodds.com/blog/the-testing-trophy-and-testing-classifications)
- [Spotify Engineering: Testing of Microservices (Honeycomb)](https://engineering.atspotify.com/2018/01/testing-of-microservices)
- [Martin Fowler: Test Pyramid (bliki)](https://martinfowler.com/bliki/TestPyramid.html)

---

## 3. Test Doubles: Dummy, Stub, Spy, Mock, Fake

A **test double** is a stand-in for a real dependency, like a stunt double in a film. You use one when the real thing is slow (network), non-deterministic (clock, random), costly (payment gateway), or hard to put into a specific state (a server returning 503).

### Core mechanism

Gerard Meszaros' taxonomy (often all loosely called "mocks"):

| Double | What it does | Verifies | Example |
| --- | --- | --- | --- |
| Dummy | passed in to fill a parameter, never used | nothing | `new Service(null as any)` for an unused logger |
| Stub | returns canned answers | nothing (you assert on SUT output) | `getRate: () => 1.1` |
| Spy | records how it was called (and may call through) | after the fact | `vi.spyOn(console, 'error')` |
| Mock | pre-programmed with expectations; fails if calls don't match | interactions | "expect `send` called once with X" |
| Fake | a working, lightweight implementation | nothing directly | in-memory repository, fake SMTP server |

Two schools:

- **Classical (Detroit):** use real objects wherever possible; double only shared, out-of-process dependencies (DB, network, clock). Assert on **state/output**.
- **Mockist (London):** double every collaborator of the class under test. Assert on **interactions**.

Classical tests survive refactors better. Interaction checks are right for **outgoing side effects you own the contract for** (an email was sent, an event was published), not for internal helper calls.

### Engineering blueprint

Dependency injection makes doubles easy; a hard-coded `import` of a singleton makes them hard.

```ts
// production code: depends on an interface, not a concrete SDK
export interface PaymentGateway { charge(cents: number, token: string): Promise<{ id: string }>; }
export interface Clock { now(): Date; }

export class CheckoutService {
  constructor(private gw: PaymentGateway, private orders: OrderRepo, private clock: Clock) {}
  async checkout(cartId: string, token: string) {
    const cart = await this.orders.getCart(cartId);
    const { id } = await this.gw.charge(cart.totalCents, token);
    return this.orders.markPaid(cartId, id, this.clock.now());
  }
}
```

```ts
// test: fake repo (real behavior, in memory), stub gateway, fixed clock
it('marks the cart paid with the gateway charge id', async () => {
  const repo = new InMemoryOrderRepo([{ id: 'c1', totalCents: 5000 }]);       // fake
  const gw = { charge: vi.fn().mockResolvedValue({ id: 'ch_123' }) };          // stub + spy
  const clock = { now: () => new Date('2026-01-01T00:00:00Z') };               // stub

  await new CheckoutService(gw, repo, clock).checkout('c1', 'tok_visa');

  expect(repo.get('c1')).toMatchObject({ status: 'paid', chargeId: 'ch_123' }); // state
  expect(gw.charge).toHaveBeenCalledWith(5000, 'tok_visa');                      // outgoing contract
});
```

Module mocking (when you can't inject):

```ts
vi.mock('./emailClient', () => ({ sendEmail: vi.fn().mockResolvedValue(undefined) }));
// Jest: jest.mock('./emailClient', () => ({ ... }))
```

### Production pitfalls

- **Over-mocking:** mocking the DB layer, the service layer, and the mapper means the test checks only that functions call each other. Real SQL bugs ship.
- **Mock drift:** the stub returns `{ userId }` but the real API now returns `{ user_id }`; tests stay green, prod breaks. Fix with contract tests [16] or typed doubles generated from the real interface.
- Mocking what you don't own directly (an SDK's internal classes); SDK upgrades break tests for no reason. Wrap the SDK in your own small adapter and double the adapter.
- Forgetting to reset mocks between tests (`mockReturnValueOnce` leaking), producing order-dependent failures.
- Using `vi.mock`/`jest.mock` hoisting in ways that confuse readers; the mock silently applies to the whole file.

### SDE best practices

- Prefer, in order: real object > fake > stub > mock. Mock only at architectural boundaries.
- Type your doubles (`satisfies PaymentGateway`, `vi.mocked(fn)`) so the compiler catches drift.
- Configure `restoreMocks: true` / `clearMocks: true` in the runner config so every test starts clean.
- Keep fakes in a shared `test/fakes` folder and test the fakes themselves against the same contract tests as the real implementation.
- Inject time, randomness, and IDs (`Clock`, `IdGenerator`) instead of calling `Date.now()` or `crypto.randomUUID()` deep inside logic.

### Resources

- [Martin Fowler: Mocks Aren't Stubs](https://martinfowler.com/articles/mocksArentStubs.html)
- [xUnit Patterns: Test Double](http://xunitpatterns.com/Test%20Double.html)
- [Vitest: Mocking guide](https://vitest.dev/guide/mocking)
- [Jest: Mock Functions](https://jestjs.io/docs/mock-functions)
- Steve Freeman & Nat Pryce, *Growing Object-Oriented Software, Guided by Tests* (the mockist view, done well)

---

## 4. Test Runners and Frameworks

A **test runner** finds your test files, runs them (often in parallel), and reports results. An **assertion library** provides `expect(...)`. A **framework** bundles both plus mocking, coverage, watch mode, and reporting.

### Core mechanism

- **Discovery:** files matching a pattern (`*.test.ts`, `test_*.py`, `*_test.go`).
- **Isolation:** each test file usually runs in its own worker/module registry, so globals and module state don't leak between files.
- **Lifecycle hooks:** `beforeAll`, `beforeEach`, `afterEach`, `afterAll` (pytest uses fixtures with scopes; Go uses `TestMain` and `t.Cleanup`).
- **Environment:** `node` for backend code; `jsdom` or `happy-dom` (simulated DOM in Node) for frontend components; a real browser for E2E or Vitest browser mode.
- **Reporters:** console, JUnit XML (for CI dashboards), JSON, HTML.

| Language | Common choice | Notes |
| --- | --- | --- |
| JS/TS (Vite, modern) | Vitest | Jest-compatible API, native ESM/TS, fast watch mode, browser mode |
| JS/TS (established) | Jest | huge ecosystem; ESM support still needs care |
| JS/TS (zero-dep) | `node:test` | built into Node; fine for libraries |
| E2E (browser) | Playwright Test, Cypress | separate runners from unit tests |
| Python | pytest | fixtures, parametrize, plugins (`pytest-xdist`, `pytest-asyncio`) |
| Java/Kotlin | JUnit 5 + AssertJ + Mockito | Spring Boot `@SpringBootTest`, `@WebMvcTest` slices |
| Go | `testing` + `testify` (optional) | table-driven tests, `go test -race` |
| C#/.NET | xUnit / NUnit + FluentAssertions | `WebApplicationFactory` for API tests |

### Engineering blueprint

Vitest config with separate projects for frontend and backend:

```ts
// vitest.config.ts
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    restoreMocks: true,
    coverage: { provider: 'v8', reporter: ['text', 'lcov'], thresholds: { lines: 80, branches: 75 } },
    projects: [
      { test: { name: 'web', include: ['web/**/*.test.{ts,tsx}'], environment: 'jsdom', setupFiles: ['web/test/setup.ts'] } },
      { test: { name: 'api', include: ['api/**/*.test.ts'], environment: 'node', setupFiles: ['api/test/setup.ts'] } },
    ],
  },
});
```

The same idea in other ecosystems:

```python
# pytest: fixture + parametrize
import pytest

@pytest.fixture
def cart():
    return Cart(items=[Item(price=50), Item(price=150)])

@pytest.mark.parametrize("coupon,expected", [("SAVE10", 180), (None, 200), ("BOGUS", 200)])
def test_total(cart, coupon, expected):
    assert cart.total(coupon) == expected
```

```go
// Go: table-driven test
func TestApplyDiscount(t *testing.T) {
    cases := []struct{ name string; subtotal int; coupon string; want int }{
        {"above threshold", 200, "SAVE10", 180},
        {"below threshold", 99, "SAVE10", 99},
        {"no coupon", 200, "", 200},
    }
    for _, tc := range cases {
        t.Run(tc.name, func(t *testing.T) {
            if got := ApplyDiscount(tc.subtotal, tc.coupon); got != tc.want {
                t.Fatalf("got %d, want %d", got, tc.want)
            }
        })
    }
}
```

Useful commands:

| Task | Command |
| --- | --- |
| Watch mode, only changed files | `vitest` / `jest --watch` |
| One file / one test | `vitest run cart.test.ts -t "below threshold"` |
| Coverage | `vitest run --coverage` / `jest --coverage` |
| Shard across CI machines | `vitest run --shard=1/4` / `jest --shard=1/4` |
| Detect leaked handles | `jest --detectOpenHandles` |
| Go race detector | `go test -race ./...` |
| Python parallel | `pytest -n auto` (pytest-xdist) |

### Production pitfalls

- Two runners in one repo (Jest and Vitest, or Mocha and Jest) with different configs, globals, and mocking semantics.
- `jsdom` used for backend tests (slow, wrong globals) or `node` env for components (`document is not defined`).
- Global setup that seeds shared state, making parallel workers interfere.
- Tests that pass locally (fast laptop, warm cache, your timezone) and fail in CI (2 vCPUs, UTC).
- Ignoring "open handle" warnings; the process hangs until CI kills it.

### SDE best practices

- One unit/integration runner per repo plus one E2E runner. Document the commands in the README.
- Pin `TZ=UTC` and a fixed locale in the test script; make tests independent of the machine.
- Output JUnit XML in CI so failures show up on the PR, with timings to find slow tests.
- Keep test files next to the code (`cart.ts` + `cart.test.ts`) for unit tests; put integration and E2E tests in their own folders.

### Resources

- [Vitest guide](https://vitest.dev/guide/)
- [Jest docs](https://jestjs.io/docs/getting-started)
- [Node.js test runner](https://nodejs.org/api/test.html)
- [pytest docs](https://docs.pytest.org/)
- [JUnit 5 user guide](https://junit.org/junit5/docs/current/user-guide/)
- [Go wiki: Table-driven tests](https://go.dev/wiki/TableDrivenTests)

---

## 5. TDD and BDD

**Test-Driven Development (TDD)** is a habit: write a failing test first, then the smallest code that makes it pass, then clean up. **Behavior-Driven Development (BDD)** is a team practice: describe behavior in business language ("Given... When... Then...") so developers, QA, and product agree on what "done" means before code is written.

### Core mechanism

TDD cycle (Kent Beck):

```text
   +-------+      write the smallest      +-------+      improve design,      +----------+
   |  RED  | ---- code to pass ---------> | GREEN | ---- tests stay green --> | REFACTOR |
   +-------+                              +-------+                           +----------+
       ^                                                                           |
       +-------------------- next small behavior ----------------------------------+
```

- Red proves the test can fail (a test that never failed may test nothing).
- Green proves the code does what the test says.
- Refactor is safe because the tests protect behavior.
- **Outside-in TDD:** start with an acceptance/API test for the feature, then drive inner units with unit tests. **Inside-out:** start with domain units, then compose.

BDD:

- Conversations first ("Three Amigos": dev, QA, product), then examples, then automation.
- Gherkin syntax (`Feature`, `Scenario`, `Given`, `When`, `Then`) executed by Cucumber, SpecFlow, behave, or Playwright-BDD.
- In plain code, BDD often just means `describe/it` names that read as behavior specs.

### Engineering blueprint

TDD in practice on a small rule ("password must be 12+ chars and not in the breached list"):

```ts
// 1. RED
it('rejects passwords shorter than 12 characters', () => {
  expect(validatePassword('short')).toEqual({ ok: false, reason: 'too_short' });
});
// 2. GREEN: simplest code
export const validatePassword = (p: string) =>
  p.length < 12 ? { ok: false, reason: 'too_short' } : { ok: true };
// 3. RED again: next behavior
it('rejects breached passwords', () => {
  expect(validatePassword('password1234', new Set(['password1234']))).toEqual({ ok: false, reason: 'breached' });
});
// 4. GREEN, then REFACTOR (extract rules into a list, rename, etc.)
```

Gherkin feature file:

```gherkin
Feature: Checkout discount
  Scenario: Coupon applies above the threshold
    Given a cart with a subtotal of 200
    When the shopper applies coupon "SAVE10"
    Then the total is 180

  Scenario: Coupon is ignored below the threshold
    Given a cart with a subtotal of 99
    When the shopper applies coupon "SAVE10"
    Then the total is 99
```

When TDD fits well and when it doesn't:

| Fits well | Fits poorly |
| --- | --- |
| business rules, parsers, calculations | exploratory UI spikes, prototypes you'll throw away |
| bug fixes (write the reproducing test first) | code whose shape you don't know yet (spike, then TDD the real version) |
| public API design (tests act as the first client) | thin glue over a framework |

### Production pitfalls

- "TDD" that writes all tests after the code; the tests mirror the implementation and miss design feedback.
- Skipping the refactor step; code passes but stays messy.
- Gherkin suites with hundreds of fragile step definitions that only developers read; all the BDD overhead with none of the collaboration value.
- Steps coupled to UI details ("When I click the blue button at the top") instead of intent ("When the shopper applies a coupon").

### SDE best practices

- Use TDD by default for bug fixes and pure logic; it's the cheapest way to prove the fix.
- Take very small steps; if a test takes more than a few minutes to turn green, the step is too big.
- Use BDD-style naming everywhere, and Gherkin only where non-developers actually read or write the scenarios.
- Keep scenario language at the business level; push UI details into step implementations or page objects.

### Resources

- Kent Beck, *Test-Driven Development: By Example* (Addison-Wesley, 2002)
- [Martin Fowler: Test Driven Development](https://martinfowler.com/bliki/TestDrivenDevelopment.html)
- [Cucumber: BDD docs](https://cucumber.io/docs/bdd/)
- [Dan North: Introducing BDD](https://dannorth.net/introducing-bdd/)

---

# Part B: Frontend Testing

## 6. Frontend Unit Testing (Logic, Hooks, State)

Most frontend bugs are not in the HTML; they are in the **logic around it**: formatting, validation, state transitions, data mapping. Pull that logic out of components into plain functions, hooks, or stores, and it becomes as easy to test as backend code.

### Core mechanism

- **Pure functions** (formatters, validators, selectors, reducers) are tested with plain input/output assertions; no DOM needed.
- **Reducers and state machines** are tested by feeding a state and an action and checking the next state.
- **Custom hooks** are tested with `renderHook` from Testing Library, which mounts a tiny invisible component that calls the hook.
- **Stores** (Redux Toolkit, Zustand, Pinia) are tested by creating a fresh store per test and dispatching real actions.
- Time, randomness, and locale are inputs. Control them (fake timers, injected `now`, fixed `Intl` locale).

### Engineering blueprint

Reducer test:

```ts
import { cartReducer, initialCart } from './cartReducer';

it('increments quantity when the same item is added twice', () => {
  const s1 = cartReducer(initialCart, { type: 'add', sku: 'A1', price: 500 });
  const s2 = cartReducer(s1, { type: 'add', sku: 'A1', price: 500 });
  expect(s2.items).toEqual([{ sku: 'A1', price: 500, qty: 2 }]);
  expect(s2.totalCents).toBe(1000);
});
```

Custom hook with fake timers (a debounced search input):

```ts
import { renderHook, act } from '@testing-library/react';
import { useDebouncedValue } from './useDebouncedValue';

it('only publishes the value after 300 ms of silence', () => {
  vi.useFakeTimers();
  const { result, rerender } = renderHook(({ v }) => useDebouncedValue(v, 300), { initialProps: { v: 'a' } });

  rerender({ v: 'ab' });
  act(() => vi.advanceTimersByTime(299));
  expect(result.current).toBe('a');

  act(() => vi.advanceTimersByTime(1));
  expect(result.current).toBe('ab');
  vi.useRealTimers();
});
```

Hook that needs a provider (React Query, router, theme):

```tsx
const wrapper = ({ children }: { children: React.ReactNode }) => (
  <QueryClientProvider client={new QueryClient({ defaultOptions: { queries: { retry: false } } })}>
    {children}
  </QueryClientProvider>
);
const { result } = renderHook(() => useUser('42'), { wrapper });
await waitFor(() => expect(result.current.data?.name).toBe('Ada'));
```

| Code | Best test level |
| --- | --- |
| `formatCurrency`, `parseDate`, Zod schemas | unit, many table-driven cases |
| reducers, XState machines, store slices | unit, state in / state out |
| custom hooks with side effects | `renderHook` + MSW [8] + fake timers |
| components that only lay out other components | component test [7] or none |

### Production pitfalls

- Logic buried inside JSX event handlers, so the only way to test it is through a full render.
- Sharing one `QueryClient` or store across tests; cached data leaks and tests depend on order.
- React Query's default retries (3 with backoff) making error tests take seconds; disable retries in tests.
- `act(...)` warnings ignored; they usually mean an async update happened after the assertion and the test is checking a stale UI.
- Date tests that pass in IST and fail in UTC, or around midnight / DST.

### SDE best practices

- Move non-trivial logic out of components into pure modules; keep components thin.
- Create a fresh store/client per test via a `renderWithProviders` helper.
- Use fake timers for debounce/throttle/polling; always restore real timers in `afterEach`.
- Validate API response shapes with the same schema (Zod/Valibot) in code and in tests.

### Resources

- [Testing Library: renderHook](https://testing-library.com/docs/react-testing-library/api#renderhook)
- [Vitest: Fake timers](https://vitest.dev/guide/mocking#timers)
- [Redux: Writing tests](https://redux.js.org/usage/writing-tests)
- [TanStack Query: Testing](https://tanstack.com/query/latest/docs/framework/react/guides/testing)

---

## 7. Component Testing (Testing Library)

A component test renders one component (or a small tree) and interacts with it **the way a user would**: find the button by its label, click it, check what appears on screen. It doesn't check internal state or which methods ran. The guiding rule from Testing Library: *"The more your tests resemble the way your software is used, the more confidence they can give you."*

### Core mechanism

- **Render** into a simulated DOM (`jsdom`/`happy-dom`) or a real browser (Vitest browser mode, Playwright component testing, Cypress component testing).
- **Query** elements through the accessibility tree, the same way assistive tech and users find them.
- **Interact** with `user-event`, which simulates real event sequences (focus, keydown, input, keyup, click), not just a single synthetic event like `fireEvent`.
- **Assert** on what the user can see or hear (`toBeInTheDocument`, `toHaveTextContent`, `toBeDisabled` from `@testing-library/jest-dom`).

Query priority (use the first that works):

| Priority | Query | Why |
| --- | --- | --- |
| 1 | `getByRole('button', { name: /save/i })` | matches what screen readers expose; also checks accessibility |
| 2 | `getByLabelText('Email')` | form fields as users find them |
| 3 | `getByPlaceholderText`, `getByText`, `getByDisplayValue` | visible text |
| 4 | `getByAltText`, `getByTitle` | images, tooltips |
| 5 | `getByTestId('cart-total')` | last resort, for things with no accessible name |

Query variants:

| Variant | No match | Many matches | Async | Use for |
| --- | --- | --- | --- | --- |
| `getBy` | throws | throws | no | elements that must be there now |
| `queryBy` | `null` | throws | no | asserting something is **absent** |
| `findBy` | rejects after timeout | throws | yes | elements that appear after async work |
| `*AllBy` | same rules, returns array | ok | | lists |

### Engineering blueprint

```tsx
// LoginForm.test.tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { LoginForm } from './LoginForm';

it('submits email and password and shows an error from the server', async () => {
  const user = userEvent.setup();
  const onSubmit = vi.fn().mockRejectedValue(new Error('Invalid credentials'));
  render(<LoginForm onSubmit={onSubmit} />);

  await user.type(screen.getByLabelText(/email/i), 'ada@example.com');
  await user.type(screen.getByLabelText(/password/i), 'correct horse');
  await user.click(screen.getByRole('button', { name: /sign in/i }));

  expect(onSubmit).toHaveBeenCalledWith({ email: 'ada@example.com', password: 'correct horse' });
  expect(await screen.findByRole('alert')).toHaveTextContent('Invalid credentials');
  expect(screen.getByRole('button', { name: /sign in/i })).toBeEnabled();
});

it('disables submit until both fields are filled', async () => {
  const user = userEvent.setup();
  render(<LoginForm onSubmit={vi.fn()} />);
  const submit = screen.getByRole('button', { name: /sign in/i });
  expect(submit).toBeDisabled();
  await user.type(screen.getByLabelText(/email/i), 'ada@example.com');
  expect(submit).toBeDisabled();
});
```

Shared render helper so every test gets fresh providers:

```tsx
// test/render.tsx
export function renderWithProviders(ui: React.ReactElement, { route = '/' } = {}) {
  const queryClient = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  return render(
    <QueryClientProvider client={queryClient}>
      <MemoryRouter initialEntries={[route]}>{ui}</MemoryRouter>
    </QueryClientProvider>,
  );
}
```

Equivalent libraries: `@testing-library/vue`, `@testing-library/svelte`, `@testing-library/angular`; Angular also has `TestBed` with component harnesses.

Debugging aids: `screen.debug()` prints the DOM; `screen.logTestingPlaygroundURL()` suggests the best query.

### Production pitfalls

- Querying by CSS class or DOM structure (`container.querySelector('.btn-primary > span')`); every styling change breaks tests.
- Testing implementation details: component state, number of renders, or props passed to children.
- Using `fireEvent.change` where `user.type` is needed; validation on keydown or blur never runs.
- Forgetting `await` on `user.*` calls or `findBy*`; the test asserts before the UI updates and passes or fails at random.
- Wrapping everything in `waitFor` with side effects inside it; side effects repeat on each retry. Put only assertions inside `waitFor`.
- Using `getBy` to assert absence (`expect(getByText('x')).not...` throws); use `queryBy`.
- jsdom has no layout: `getBoundingClientRect`, `IntersectionObserver`, scrolling, and CSS media queries are fake or missing. Test those in a real browser.

### SDE best practices

- Query by role and accessible name first; if you can't, the component probably has an accessibility bug, so fix it.
- Use `userEvent.setup()` once per test and `await` every interaction.
- Test behavior visible to the user: text, enabled/disabled, focus, navigation, network calls made.
- Keep one `renderWithProviders` helper; never let tests construct providers ad hoc.
- Run component tests in a real browser (Vitest browser mode / Playwright CT) for layout-heavy or canvas components.

### Resources

- [Testing Library: Guiding principles](https://testing-library.com/docs/guiding-principles)
- [Testing Library: Query priority](https://testing-library.com/docs/queries/about/#priority)
- [user-event docs](https://testing-library.com/docs/user-event/intro)
- [Kent C. Dodds: Common mistakes with React Testing Library](https://kentcdodds.com/blog/common-mistakes-with-react-testing-library)
- [jest-dom matchers](https://github.com/testing-library/jest-dom)
- [Vitest browser mode](https://vitest.dev/guide/browser/)

---

## 8. Mocking the Network in the Frontend (MSW)

Components fetch data. In tests, you don't want to call a real server (slow, flaky, shared state), but you also don't want to mock `fetch` or `axios` by hand (brittle, skips your real request code). **Mock Service Worker (MSW)** intercepts requests at the network layer and answers with responses you define, so your real `fetch`/axios/GraphQL client code still runs.

### Core mechanism

- In Node (Vitest/Jest), MSW patches the request modules (`http`, `fetch`, `XMLHttpRequest`) via `setupServer`.
- In the browser (Storybook, local dev, Playwright with a dev build), a real Service Worker intercepts requests via `setupWorker`.
- You declare **handlers**: method + URL pattern -> response. The same handlers can be reused in unit tests, Storybook, and local development.
- Per-test overrides (`server.use(...)`) let a single test simulate errors, empty states, or slow responses, and `resetHandlers()` removes them afterwards.

### Engineering blueprint

```ts
// test/msw/handlers.ts
import { http, HttpResponse, delay } from 'msw';

export const handlers = [
  http.get('/api/users/:id', ({ params }) =>
    HttpResponse.json({ id: params.id, name: 'Ada Lovelace', role: 'admin' })),
  http.post('/api/orders', async ({ request }) => {
    const body = (await request.json()) as { sku: string };
    return HttpResponse.json({ id: 'o_1', sku: body.sku, status: 'pending' }, { status: 201 });
  }),
];
```

```ts
// test/setup.ts
import { setupServer } from 'msw/node';
import { handlers } from './msw/handlers';

export const server = setupServer(...handlers);
beforeAll(() => server.listen({ onUnhandledRequest: 'error' }));   // fail on unexpected calls
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

Per-test override for error and loading states:

```tsx
it('shows a retry button when the profile request fails', async () => {
  server.use(http.get('/api/users/:id', () => new HttpResponse(null, { status: 500 })));
  renderWithProviders(<Profile id="42" />);
  expect(await screen.findByRole('button', { name: /retry/i })).toBeVisible();
});

it('shows a skeleton while loading', async () => {
  server.use(http.get('/api/users/:id', async () => { await delay('infinite'); return HttpResponse.json({}); }));
  renderWithProviders(<Profile id="42" />);
  expect(screen.getByTestId('profile-skeleton')).toBeInTheDocument();
});
```

States every data-driven component should be tested in:

| State | How to simulate |
| --- | --- |
| loading | `delay('infinite')` or a long delay |
| success, typical data | default handler |
| empty list | `HttpResponse.json([])` |
| validation error (400/422) | status + error body your API really returns |
| auth expired (401) | check redirect to login / token refresh |
| forbidden (403) | check "no access" UI, not a crash |
| server error (500), network error | `HttpResponse.error()` for network failure |

### Production pitfalls

- Handlers drift from the real API (field renamed on the backend); tests stay green. Generate mock data from the OpenAPI schema or back MSW with contract tests [16].
- `onUnhandledRequest: 'warn'` (or bypass) lets new, un-mocked requests hit the network or silently fail.
- Overrides leaking between tests because `resetHandlers` isn't called.
- Mocking `fetch` with `vi.fn()` directly: you skip headers, URL building, JSON parsing, and error handling in your real client.
- Relative URLs in Node: if your client uses `new URL(path, window.location.origin)`, make sure jsdom's URL is set (`testEnvironmentOptions.url`).

### SDE best practices

- One shared handler set describing the "happy" API, with per-test overrides for edge cases.
- `onUnhandledRequest: 'error'` in tests.
- Type handler responses with the same types or schemas as your API client (`HttpResponse.json<User>(...)`).
- Reuse the same handlers in Storybook so stories and tests show the same states.
- Assert on what the request contained when it matters (capture `request` in the handler), not on how many times `fetch` was called.

### Resources

- [MSW docs](https://mswjs.io/docs/)
- [MSW: Network behavior overrides](https://mswjs.io/docs/best-practices/network-behavior-overrides)
- [Kent C. Dodds: Stop mocking fetch](https://kentcdodds.com/blog/stop-mocking-fetch)
- [Storybook MSW addon](https://storybook.js.org/addons/msw-storybook-addon)

---

## 9. Snapshot and Visual Regression Testing

A **snapshot test** saves the output of something (serialized DOM, JSON) the first time, and fails if it changes later. A **visual regression test** does the same with **screenshots**, comparing pixels to catch CSS and layout changes that DOM snapshots and functional tests can't see.

### Core mechanism

- First run writes a baseline (`__snapshots__/*.snap` or `*.png`). Later runs diff against it.
- A diff means "something changed", not "something broke". A human decides: accept (update baseline) or fix.
- **DOM/inline snapshots:** cheap, but large snapshots are rarely reviewed carefully. Best for small, stable outputs (an error message object, a generated SQL string, a serialized config).
- **Visual snapshots:** render in a real browser at fixed viewport, fonts, and data; compare with a pixel tolerance (`maxDiffPixels`, `threshold`). Tools: Playwright `toHaveScreenshot`, Storybook + Chromatic, Percy, Applitools, BackstopJS, Lost Pixel.

### Engineering blueprint

Small inline snapshot (good use):

```ts
it('builds the search query string', () => {
  expect(buildQuery({ q: 'shoes', page: 2, sort: 'price' })).toMatchInlineSnapshot(`"q=shoes&page=2&sort=price"`);
});
```

Playwright visual test:

```ts
import { test, expect } from '@playwright/test';

test('pricing page looks right', async ({ page }) => {
  await page.goto('/pricing');
  await page.getByRole('heading', { name: 'Pricing' }).waitFor();
  await expect(page).toHaveScreenshot('pricing.png', {
    fullPage: true,
    mask: [page.getByTestId('live-chat-badge')],   // hide dynamic regions
    maxDiffPixelRatio: 0.01,
    animations: 'disabled',
  });
});
```

Update baselines on purpose: `npx playwright test --update-snapshots` (and review the image diff in the PR).

Component-level visual testing with Storybook: each story (button states, form errors, dark mode, RTL) becomes a visual test case; Chromatic or the Storybook test runner diffs them on every PR.

| Use | Snapshot type |
| --- | --- |
| design system components, all variants | visual (Storybook + Chromatic/Playwright) |
| key marketing / checkout pages | visual, full page, masked dynamic content |
| serializers, generated code, error payloads | small inline snapshot |
| whole component DOM trees | avoid; prefer explicit assertions |

### Production pitfalls

- Giant DOM snapshots that everybody updates with `-u` without reading; the test catches nothing.
- Visual baselines generated on a Mac and compared in Linux CI: font rendering differs, every test fails. Generate and compare in the same Docker image.
- Dynamic content (dates, avatars, ads, animations, carousels) causing constant false diffs.
- Snapshotting before fonts/images load, so screenshots are taken mid-render.
- Too many full-page screenshots; one CSS token change creates 400 diffs and reviewers approve all of them blind.

### SDE best practices

- Keep DOM snapshots tiny and inline; prefer explicit assertions for behavior.
- Run visual tests in a pinned container (`mcr.microsoft.com/playwright:<version>`), with fixed viewport, locale, timezone, and seeded data.
- Mask or freeze dynamic regions; disable animations; wait for a stable signal before capture.
- Test components visually at the design-system level (Storybook) and only a few critical full pages.
- Treat baseline updates as code review: the diff image must be in the PR.

### Resources

- [Playwright: Visual comparisons](https://playwright.dev/docs/test-snapshots)
- [Vitest: Snapshot](https://vitest.dev/guide/snapshot)
- [Storybook: Visual tests](https://storybook.js.org/docs/writing-tests/visual-testing)
- [Chromatic docs](https://www.chromatic.com/docs/)
- [Kent C. Dodds: Effective snapshot testing](https://kentcdodds.com/blog/effective-snapshot-testing)

---

## 10. Accessibility Testing

Accessibility (a11y) testing checks that people using screen readers, keyboards, voice control, or zoom can use the product. It is a quality requirement and in many markets a legal one (WCAG 2.1/2.2 AA is the usual target; the EU Accessibility Act applies from June 2025).

### Core mechanism

- **Automated rule engines** (axe-core, used by jest-axe, `@axe-core/playwright`, Lighthouse, Storybook a11y addon) scan the DOM for detectable issues: missing labels, low contrast, invalid ARIA, missing alt text, duplicate IDs. They catch a meaningful share of issues, but not all; many WCAG criteria need human judgment.
- **Role-based queries** in component tests [7] already act as a11y tests: if `getByRole('button', { name: 'Save' })` can't find your button, neither can a screen reader.
- **Keyboard tests:** Tab order, focus visible, focus trapped in modals and returned on close, Escape closes dialogs.
- **Manual checks:** screen reader passes (VoiceOver, NVDA), zoom to 200%/400%, reduced motion, high contrast.

### Engineering blueprint

Component level with jest-axe (works in Vitest too):

```tsx
import { axe } from 'jest-axe';      // plus expect.extend(toHaveNoViolations) in setup

it('signup form has no detectable a11y violations', async () => {
  const { container } = render(<SignupForm />);
  expect(await axe(container)).toHaveNoViolations();
});
```

Keyboard behavior:

```tsx
it('traps focus in the dialog and restores it on close', async () => {
  const user = userEvent.setup();
  render(<DeleteButtonWithConfirm />);
  const trigger = screen.getByRole('button', { name: /delete/i });
  await user.click(trigger);

  const dialog = screen.getByRole('dialog', { name: /confirm delete/i });
  expect(dialog).toContainElement(document.activeElement as HTMLElement);
  await user.keyboard('{Escape}');
  expect(screen.queryByRole('dialog')).not.toBeInTheDocument();
  expect(trigger).toHaveFocus();
});
```

Page level with Playwright:

```ts
import AxeBuilder from '@axe-core/playwright';

test('checkout page meets WCAG 2.2 AA rules axe can detect', async ({ page }) => {
  await page.goto('/checkout');
  const results = await new AxeBuilder({ page }).withTags(['wcag2a', 'wcag2aa', 'wcag22aa']).analyze();
  expect(results.violations).toEqual([]);
});
```

| Check | Automated? | Tool |
| --- | --- | --- |
| labels, alt text, ARIA validity, contrast | yes | axe |
| accessible names for controls | yes, through role queries | Testing Library |
| keyboard operability, focus management | partly (scripted) | user-event, Playwright |
| meaningful alt text, reading order, clear error messages | no | human review |
| screen reader experience | no | VoiceOver / NVDA passes |

### Production pitfalls

- Treating "axe passes" as "accessible"; automated rules miss many real barriers.
- Disabling rules globally to get green (`color-contrast` off) instead of fixing tokens in the design system.
- `div` with `onClick` instead of `button`: not focusable, no role, no keyboard activation.
- Testing components only in isolation; issues often come from composition (duplicate landmark names, heading order across the page).
- jsdom can't compute contrast reliably; run contrast checks in a real browser.

### SDE best practices

- Add axe to component tests for every design-system component and to E2E for key pages; fail CI on new violations.
- Use role-based queries everywhere; they double as a11y tests.
- Script keyboard flows for dialogs, menus, comboboxes, and forms.
- Schedule periodic manual audits with real assistive technology, especially before major releases.

### Resources

- [W3C: WCAG overview](https://www.w3.org/WAI/standards-guidelines/wcag/)
- [axe-core](https://github.com/dequelabs/axe-core)
- [jest-axe](https://github.com/nickcolley/jest-axe)
- [Playwright: Accessibility testing](https://playwright.dev/docs/accessibility-testing)
- [WAI-ARIA Authoring Practices Guide (patterns + keyboard behavior)](https://www.w3.org/WAI/ARIA/apg/)

---

## 11. End-to-End Testing (Playwright, Cypress)

An E2E test drives a **real browser** through the **real deployed app** (frontend + backend + DB) the way a user would: open the page, log in, add to cart, pay. It is the most convincing test and also the slowest and most fragile, so you keep a small number of them for the journeys that make or lose money.

### Core mechanism

- The runner launches Chromium/Firefox/WebKit, opens your app URL, and sends real input events.
- **Auto-waiting:** modern tools (Playwright, Cypress) wait for elements to be visible, enabled, and stable before acting, and retry assertions until a timeout. You almost never need `sleep`.
- **Locators** use the same accessibility-first strategy as Testing Library (`getByRole`, `getByLabel`).
- **Isolation:** each Playwright test gets a fresh browser context (cookies, storage), like a new incognito window.
- **Artifacts on failure:** trace (DOM snapshots, network, console, per-step timeline), screenshots, video.

| | Playwright | Cypress |
| --- | --- | --- |
| Browsers | Chromium, Firefox, WebKit | Chromium-family, Firefox (WebKit experimental) |
| Runs | out of process, via CDP/protocols | inside the browser alongside the app |
| Multi-tab, multi-origin | yes | limited (`cy.origin`) |
| Parallelism | built-in workers + sharding | via Cypress Cloud or 3rd-party |
| Languages | TS/JS, Python, Java, .NET | JS/TS |
| Debugging | Trace Viewer, UI mode | time-travel UI |

### Engineering blueprint

Config:

```ts
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './e2e',
  fullyParallel: true,
  retries: process.env.CI ? 2 : 0,
  reporter: [['html'], ['junit', { outputFile: 'results/e2e.xml' }]],
  use: { baseURL: process.env.BASE_URL ?? 'http://localhost:3000', trace: 'on-first-retry', screenshot: 'only-on-failure' },
  projects: [
    { name: 'setup', testMatch: /auth\.setup\.ts/ },
    { name: 'chromium', use: { ...devices['Desktop Chrome'], storageState: 'e2e/.auth/user.json' }, dependencies: ['setup'] },
    { name: 'mobile', use: { ...devices['Pixel 7'], storageState: 'e2e/.auth/user.json' }, dependencies: ['setup'] },
  ],
  webServer: { command: 'npm run start:test', url: 'http://localhost:3000', reuseExistingServer: !process.env.CI },
});
```

Log in once, reuse the session (fast, and avoids hammering the login page):

```ts
// e2e/auth.setup.ts
import { test as setup, expect } from '@playwright/test';

setup('authenticate', async ({ page }) => {
  await page.goto('/login');
  await page.getByLabel('Email').fill(process.env.E2E_USER!);
  await page.getByLabel('Password').fill(process.env.E2E_PASS!);
  await page.getByRole('button', { name: 'Sign in' }).click();
  await expect(page.getByRole('heading', { name: 'Dashboard' })).toBeVisible();
  await page.context().storageState({ path: 'e2e/.auth/user.json' });
});
```

A journey test with a page object and API-based data setup:

```ts
// e2e/checkout.spec.ts
import { test, expect } from '@playwright/test';

test('shopper can buy an item with a saved card', async ({ page, request }) => {
  const product = await (await request.post('/api/test/products', { data: { name: 'Mug', priceCents: 1200 } })).json();

  await page.goto(`/products/${product.id}`);
  await page.getByRole('button', { name: 'Add to cart' }).click();
  await page.getByRole('link', { name: 'Cart (1)' }).click();
  await page.getByRole('button', { name: 'Checkout' }).click();
  await page.getByRole('button', { name: 'Pay $12.00' }).click();

  await expect(page.getByRole('heading', { name: 'Order confirmed' })).toBeVisible();
  await expect(page).toHaveURL(/\/orders\/o_/);
});
```

Stubbing a third party you can't control in E2E:

```ts
await page.route('**/api.stripe.com/**', route => route.fulfill({ status: 200, json: { id: 'pi_test', status: 'succeeded' } }));
```

What belongs in E2E and what doesn't:

| E2E | Push lower |
| --- | --- |
| sign up, log in, log out, password reset | every validation message on the signup form [7] |
| core purchase / create / share flow | price calculation edge cases [6][13] |
| permissions visible in UI for 2 to 3 key roles | the full role x endpoint matrix [14][19] |
| smoke test after each deploy | API error mapping [8][14] |

### Production pitfalls

- Hard waits (`waitForTimeout(3000)`): slow when the app is fast, flaky when it is slow.
- CSS/XPath selectors tied to markup (`div:nth-child(3) > .btn`).
- Tests that depend on each other (test 2 uses the order created by test 1), so failures cascade and parallel runs break.
- Data setup through the UI (clicking through 5 screens to create a product) instead of an API or DB seed.
- Shared test accounts mutated by parallel workers (one test empties the cart another test is filling).
- Running E2E against a shared staging environment that other teams deploy to mid-run.
- Retries hiding real flakiness, so the suite gets slower and nobody notices.

### SDE best practices

- Keep E2E to critical journeys; aim for a PR suite under ~10 minutes with sharding.
- Locators by role/label/text; `data-testid` only as fallback. Web-first assertions (`await expect(locator).toBeVisible()`), never manual sleeps.
- Create data via API/DB per test with unique IDs; log in once via `storageState`; one account per worker if tests mutate account state.
- Run against an ephemeral environment per PR (preview deploy + seeded DB) when possible.
- Record traces on first retry; treat any retry-passed test as flaky [24].
- Use page objects or small helper functions for repeated flows, but keep assertions in the test.

### Resources

- [Playwright docs](https://playwright.dev/docs/intro)
- [Playwright: Best practices](https://playwright.dev/docs/best-practices)
- [Playwright: Authentication](https://playwright.dev/docs/auth)
- [Playwright: Trace Viewer](https://playwright.dev/docs/trace-viewer)
- [Cypress docs](https://docs.cypress.io/)
- [Cypress: Best practices](https://docs.cypress.io/app/core-concepts/best-practices)

---

## 12. Frontend Performance Testing

Frontend performance tests check that pages load and respond fast enough on realistic devices and networks, and that a new commit doesn't quietly make them slower. The usual yardstick is Google's **Core Web Vitals**.

### Core mechanism

| Metric | Measures | "Good" threshold (75th percentile of page loads) |
| --- | --- | --- |
| LCP (Largest Contentful Paint) | when the main content is visible | <= 2.5 s |
| INP (Interaction to Next Paint) | how quickly the page responds to input | <= 200 ms |
| CLS (Cumulative Layout Shift) | how much the layout jumps around | <= 0.1 |
| TTFB, FCP, TBT | diagnostics that explain the above | supporting metrics |

- **Lab testing** (synthetic): Lighthouse, WebPageTest, Playwright + CDP throttling. Repeatable; good for CI regression gates.
- **Field data (RUM):** the `web-vitals` library or CrUX reports what real users experience. It is the source of truth, but noisy and delayed.
- **Budgets:** limits on bundle size, request count, or metric values that fail the build when exceeded.

### Engineering blueprint

Lighthouse CI assertions:

```json
// lighthouserc.json
{
  "ci": {
    "collect": { "url": ["http://localhost:3000/", "http://localhost:3000/products/1"], "numberOfRuns": 3 },
    "assert": {
      "assertions": {
        "categories:performance": ["error", { "minScore": 0.9 }],
        "largest-contentful-paint": ["error", { "maxNumericValue": 2500 }],
        "cumulative-layout-shift": ["error", { "maxNumericValue": 0.1 }],
        "total-blocking-time": ["warn", { "maxNumericValue": 200 }],
        "resource-summary:script:size": ["error", { "maxNumericValue": 350000 }]
      }
    }
  }
}
```

Run with `npx @lhci/cli autorun` in CI after building and starting the app.

Bundle size budget (e.g. `size-limit`):

```json
// package.json
"size-limit": [{ "path": "dist/assets/index-*.js", "limit": "180 kB" }]
```

Field measurement:

```ts
import { onLCP, onINP, onCLS } from 'web-vitals';
const send = (m: { name: string; value: number; id: string }) =>
  navigator.sendBeacon('/rum', JSON.stringify({ ...m, page: location.pathname }));
onLCP(send); onINP(send); onCLS(send);
```

### Production pitfalls

- A single Lighthouse run in CI; results vary by +/-10%. Use median of 3 to 5 runs and assert on budgets with headroom.
- Testing on a fast dev laptop with no throttling; real users are on mid-range phones over 4G.
- Lab LCP looks fine, but field INP is bad because heavy third-party scripts (tag managers, chat widgets) run after load.
- Only testing the home page; the slow pages are search results and dashboards with real data volumes.
- Ignoring CLS from late-loading ads, fonts, or images without width/height.

### SDE best practices

- Gate PRs on bundle size and Lighthouse budgets for 3 to 5 key pages; track trends, not one-off scores.
- Collect RUM with `web-vitals` and alert on p75 regressions per page type.
- Test with throttling profiles that match your audience (Lighthouse "mobile" preset as a default).
- Include realistic data sizes in perf test fixtures (1,000-row tables, long names, many images).

### Resources

- [web.dev: Web Vitals](https://web.dev/articles/vitals)
- [Lighthouse CI](https://github.com/GoogleChrome/lighthouse-ci)
- [web-vitals library](https://github.com/GoogleChrome/web-vitals)
- [size-limit](https://github.com/ai/size-limit)
- [WebPageTest docs](https://docs.webpagetest.org/)

---

# Part C: Backend Testing

## 13. Backend Unit Testing (Services and Domain Logic)

Backend unit tests check business rules (pricing, permissions, state transitions, validation) without starting a server or touching a database. They should run in milliseconds, so you can run thousands on every save.

### Core mechanism

- Structure code in layers so logic can be tested apart from I/O:

```text
 HTTP route / controller  ->  Service / use case  ->  Domain (pure rules)  ->  Repository / adapters (DB, HTTP, queue)
      test via API [14]        unit test with fakes      pure unit tests           integration test [15]
```

- **Functional core, imperative shell:** pure functions decide *what* should happen (return a decision or new state); a thin shell performs I/O. The core needs no doubles at all.
- Dependencies (repo, gateway, clock, ID generator) are passed in through constructors or function parameters, so tests can supply fakes [3].
- Errors are part of the contract: test thrown domain errors or returned `Result` types as carefully as successes.

### Engineering blueprint

Pure domain rule (no doubles needed):

```ts
// domain/refund.ts
export type RefundDecision = { allowed: true; amountCents: number } | { allowed: false; reason: 'window_closed' | 'already_refunded' };

export function decideRefund(order: { paidAt: Date; totalCents: number; refunded: boolean }, now: Date): RefundDecision {
  if (order.refunded) return { allowed: false, reason: 'already_refunded' };
  const days = (now.getTime() - order.paidAt.getTime()) / 86_400_000;
  if (days > 30) return { allowed: false, reason: 'window_closed' };
  return { allowed: true, amountCents: order.totalCents };
}
```

```ts
describe('decideRefund', () => {
  const paidAt = new Date('2026-01-01T00:00:00Z');
  it.each([
    ['day 0', '2026-01-01T00:00:00Z', true],
    ['day 30 exactly', '2026-01-31T00:00:00Z', true],
    ['day 30 + 1 s', '2026-01-31T00:00:01Z', false],
  ])('%s -> allowed=%s', (_label, now, allowed) => {
    expect(decideRefund({ paidAt, totalCents: 1000, refunded: false }, new Date(now)).allowed).toBe(allowed);
  });

  it('never refunds twice', () => {
    expect(decideRefund({ paidAt, totalCents: 1000, refunded: true }, paidAt)).toEqual({ allowed: false, reason: 'already_refunded' });
  });
});
```

Service with a fake repository and a spy on an outgoing side effect:

```ts
it('issues a refund and publishes an event', async () => {
  const orders = new InMemoryOrderRepo([{ id: 'o1', paidAt: new Date('2026-01-01Z'), totalCents: 1000, refunded: false }]);
  const events = { publish: vi.fn() };
  const svc = new RefundService(orders, fakeGateway(), events, fixedClock('2026-01-10Z'));

  await svc.refund('o1');

  expect((await orders.get('o1'))!.refunded).toBe(true);
  expect(events.publish).toHaveBeenCalledWith('order.refunded', { orderId: 'o1', amountCents: 1000 });
});

it('throws NotFound for an unknown order', async () => {
  await expect(makeSvc().refund('nope')).rejects.toBeInstanceOf(OrderNotFoundError);
});
```

### Production pitfalls

- Business rules living in controllers or SQL, so the only way to test them is through HTTP or a DB.
- `new Date()` and `Math.random()` inside logic: tests become time-dependent and flaky around midnight, month ends, and DST.
- Money as floating point (`0.1 + 0.2`); tests pass with round numbers and fail in production. Use integer cents or a decimal library and test rounding rules explicitly.
- Testing the ORM's behavior with mocks ("`save` was called") instead of testing your rule.
- Mocking every collaborator so a refactor that moves logic between classes breaks dozens of tests.

### SDE best practices

- Keep domain logic pure and test it exhaustively with table-driven cases at boundaries.
- Inject clock, ID generator, and randomness. Use fixed values in tests.
- Test domain errors and invalid transitions as first-class behavior.
- Use fakes for repositories in service tests; verify the real repository separately with DB integration tests [15].
- If a unit test needs more than ~3 doubles, the class probably does too much; split it.

### Resources

- [Gary Bernhardt: Functional Core, Imperative Shell (talk)](https://www.destroyallsoftware.com/screencasts/catalog/functional-core-imperative-shell)
- [Martin Fowler: Unit Test](https://martinfowler.com/bliki/UnitTest.html)
- [Vitest: test.each](https://vitest.dev/api/#test-each)
- Vladimir Khorikov, *Unit Testing Principles, Practices, and Patterns*, ch. 7 (refactoring toward valuable unit tests)

---

## 14. API and Integration Testing

An API test sends real HTTP requests to your application and checks the real HTTP responses: status code, headers, body, and side effects in the database. Here the routing, middleware, validation, auth, serialization, and SQL all work together, which is exactly where many real bugs hide.

### Core mechanism

- **In-process:** start the app object in the test process and send requests without opening a port (`supertest(app)`, Fastify `app.inject`, Spring `MockMvc`/`WebTestClient`, .NET `WebApplicationFactory`, Go `httptest`). Fast, debuggable.
- **Out-of-process (black-box):** run the built service (often a Docker image) and hit it over the network. Closest to production; slower.
- Use **real infrastructure you own** (Postgres, Redis, a queue broker) in containers [15]; **double the third parties you don't own** (payment, email, SMS) with fakes or HTTP stubs (WireMock, MSW in Node, `nock`).
- Assert on the **contract**: status, response schema, error format, headers (e.g. `Location`, `Cache-Control`), and resulting state.

### Engineering blueprint

App factory so tests can inject dependencies:

```ts
// src/app.ts
export function buildApp(deps: { db: Pool; payments: PaymentGateway; clock: Clock }) {
  const app = express();
  app.use(express.json());
  app.use(authMiddleware(deps));
  app.use('/orders', ordersRouter(deps));
  app.use(errorHandler);
  return app;
}
```

Supertest suite:

```ts
import request from 'supertest';

let app: Express;
beforeAll(async () => { app = buildApp({ db: testDb, payments: new FakePayments(), clock: fixedClock('2026-01-01Z') }); });
beforeEach(async () => { await resetDb(testDb); });

describe('POST /orders', () => {
  it('creates an order and returns 201 with Location', async () => {
    const user = await userFactory.create();
    const res = await request(app)
      .post('/orders')
      .set('Authorization', `Bearer ${tokenFor(user)}`)
      .send({ sku: 'MUG-1', qty: 2 })
      .expect(201)
      .expect('Content-Type', /json/);

    expect(res.headers.location).toMatch(/^\/orders\/[0-9a-f-]{36}$/);
    expect(res.body).toMatchObject({ sku: 'MUG-1', qty: 2, status: 'pending' });
    expect(await testDb.query('SELECT count(*)::int AS n FROM orders WHERE user_id=$1', [user.id])).toMatchObject({ rows: [{ n: 1 }] });
  });

  it('returns 422 with field errors for invalid input', async () => {
    const res = await request(app).post('/orders').set(authHeader()).send({ sku: '', qty: 0 }).expect(422);
    expect(res.body).toEqual({
      type: 'https://example.com/problems/validation',
      title: 'Invalid request',
      status: 422,
      errors: expect.arrayContaining([expect.objectContaining({ field: 'qty' })]),
    });
  });

  it('returns 401 without a token', () => request(app).post('/orders').send({}).expect(401));

  it("returns 404 for another user's order (no existence leak)", async () => {
    const [alice, bob] = await userFactory.createList(2);
    const order = await orderFactory.create({ userId: alice.id });
    await request(app).get(`/orders/${order.id}`).set(authHeader(bob)).expect(404);
  });
});
```

Checklist per endpoint:

| Area | Cases |
| --- | --- |
| Happy path | correct status, body schema, headers, DB state |
| Validation | missing, wrong type, out of range, extra fields (mass assignment) |
| AuthN | no token, expired token, wrong audience |
| AuthZ | wrong role (403), other user's/tenant's object (404) |
| Idempotency | repeat with same `Idempotency-Key` returns same result, one side effect |
| Concurrency | two updates with stale `If-Match`/version -> 409/412 |
| Pagination | first, middle, last page, empty, limit bounds |
| Dependency failure | gateway timeout -> 502/503 with retry-safe semantics |

### Production pitfalls

- Starting the app with `app.listen(3000)` in tests; parallel files fight for the port. Pass the app object to Supertest instead.
- Tests that share DB rows and depend on execution order.
- Asserting the entire response body with `toEqual` including timestamps and generated IDs; use `toMatchObject` or `expect.any(String)`.
- Only testing 200s; most production incidents involve 4xx/5xx paths, timeouts, and retries.
- Hitting real third-party sandboxes in every CI run (rate limits, outages, slow).
- Auth middleware disabled in tests "for convenience", so authorization is never tested.

### SDE best practices

- Build the app through a factory with injectable dependencies; never read config from globals inside handlers.
- Use real DB/cache/broker in containers; fake only external vendors at the HTTP boundary.
- Generate tokens with a test signing key that the app trusts in test config; keep auth middleware on.
- Validate responses against your OpenAPI schema in tests (see [16]) so docs and code can't drift.
- Write an authorization matrix test (role x endpoint x expected status) generated from the route table.

### Resources

- [Supertest](https://github.com/ladjs/supertest)
- [Fastify: Testing with inject](https://fastify.dev/docs/latest/Guides/Testing/)
- [Spring Boot: Testing](https://docs.spring.io/spring-boot/reference/testing/index.html)
- [Go: net/http/httptest](https://pkg.go.dev/net/http/httptest)
- [WireMock](https://wiremock.org/docs/)
- [RFC 9457: Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc9457)

---

## 15. Database Testing (Testcontainers, Migrations, Isolation)

Database tests check that your SQL, ORM mappings, migrations, constraints, and transactions do what you think, against the **same database engine** you run in production. Many "works on my machine" bugs (case sensitivity, JSON operators, locking, collation, time zones) only show up on the real engine.

### Core mechanism

- **Testcontainers** starts a throwaway Docker container (Postgres, MySQL, Redis, Kafka, LocalStack) from your test code, gives you its connection string, and destroys it afterwards.
- **Migrations** run against the fresh container once per test run, which also tests that migrations apply cleanly from zero.
- **Isolation strategies** between tests:

| Strategy | How | Speed | Caveat |
| --- | --- | --- | --- |
| Transaction rollback | `BEGIN` before each test, `ROLLBACK` after | fastest | code under test that opens its own transactions or uses multiple connections won't see/stay inside it |
| Truncate tables | `TRUNCATE ... RESTART IDENTITY CASCADE` after each test | fast | must list or discover tables |
| Template database | `CREATE DATABASE t_x TEMPLATE app_template` per worker/test | fast in Postgres | Postgres-specific |
| Schema/DB per worker | one DB per parallel worker | parallel-safe | more setup |
| Unique data per test | random tenant/user IDs; no cleanup | trivially parallel | tables grow; queries must filter |

### Engineering blueprint

Global setup with Testcontainers (Node):

```ts
// test/globalSetup.ts
import { PostgreSqlContainer, type StartedPostgreSqlContainer } from '@testcontainers/postgresql';
import { execSync } from 'node:child_process';

let pg: StartedPostgreSqlContainer;

export async function setup() {
  pg = await new PostgreSqlContainer('postgres:16-alpine').start();   // match prod major version
  process.env.DATABASE_URL = pg.getConnectionUri();
  execSync('npm run migrate:latest', { env: process.env, stdio: 'inherit' });
}
export async function teardown() { await pg.stop(); }
```

Repository test with truncation:

```ts
beforeEach(() => db.query('TRUNCATE orders, users RESTART IDENTITY CASCADE'));

it('finds only the pending orders of one tenant, newest first', async () => {
  const t1 = await tenantFactory.create(); const t2 = await tenantFactory.create();
  await orderFactory.create({ tenantId: t1.id, status: 'pending', createdAt: '2026-01-01Z' });
  await orderFactory.create({ tenantId: t1.id, status: 'pending', createdAt: '2026-01-02Z' });
  await orderFactory.create({ tenantId: t1.id, status: 'paid' });
  await orderFactory.create({ tenantId: t2.id, status: 'pending' });

  const rows = await repo.pendingForTenant(t1.id);
  expect(rows.map(r => r.createdAt.toISOString())).toEqual(['2026-01-02T00:00:00.000Z', '2026-01-01T00:00:00.000Z']);
});

it('enforces one active subscription per user via a unique partial index', async () => {
  const u = await userFactory.create();
  await repo.createSubscription(u.id);
  await expect(repo.createSubscription(u.id)).rejects.toMatchObject({ code: '23505' });   // unique_violation
});
```

Migration tests worth having:

- Apply all migrations to an empty DB (every CI run).
- Apply new migration to a DB snapshot with realistic data; check run time and locks (e.g. `CREATE INDEX CONCURRENTLY`, no full-table rewrite on big tables).
- Run `down` then `up` if you support rollbacks.
- Check the app's old version still works against the new schema (expand/contract, for zero-downtime deploys).

Other stores, same idea: Redis (`@testcontainers/redis`), Kafka (`@testcontainers/kafka`), S3/SQS via LocalStack, Elasticsearch/OpenSearch containers.

### Production pitfalls

- SQLite or H2 standing in for Postgres/MySQL in tests: different SQL dialect, types, locking, and constraint behavior. Green tests, broken prod.
- Different DB major version in tests vs prod.
- Transaction-rollback isolation hiding bugs in code that commits, uses `SERIALIZABLE`, or relies on triggers firing at commit.
- Tests relying on auto-increment IDs (`id = 1`) or default row order without `ORDER BY`.
- Starting one container per test file; the suite spends minutes booting Postgres. Start once per run (global setup) and isolate by DB/schema.
- No tests for constraints and indexes, so a migration that drops a unique index goes unnoticed.

### SDE best practices

- Same engine, same major version, same extensions as prod; pin the image tag.
- One container per test run; per-worker databases (or template DBs) for parallelism.
- Test constraints (unique, FK, check), not just happy queries. Constraints are your last line of defense.
- Keep a "realistic volume" dataset for query-plan checks (`EXPLAIN`) on hot queries.
- Enable Testcontainers' reuse locally for fast iteration; always fresh in CI.

### Resources

- [Testcontainers (all languages)](https://testcontainers.com/)
- [Testcontainers for Node.js](https://node.testcontainers.org/)
- [Testcontainers Postgres module (Java docs, same concepts)](https://java.testcontainers.org/modules/databases/postgres/)
- [PostgreSQL: Template databases](https://www.postgresql.org/docs/current/manage-ag-templatedbs.html)
- [Martin Fowler: Evolutionary Database Design](https://martinfowler.com/articles/evodb.html)

---

## 16. Contract Testing (Pact, OpenAPI)

When a frontend calls a backend, or service A calls service B, both sides make assumptions about the API: URL, fields, types, status codes. A **contract test** checks those assumptions **without running both sides together**. It catches "the backend renamed `userId` to `user_id`" in the provider's CI, before deploy, instead of in production.

### Core mechanism

Two main styles:

| Style | How it works | Strength |
| --- | --- | --- |
| Consumer-driven (Pact) | consumer tests record the requests they make and responses they need into a *pact file*; the provider replays them against its real code | only tests what consumers actually use; providers can safely remove unused fields |
| Schema/spec-based (OpenAPI, JSON Schema, protobuf, GraphQL schema) | both sides are validated against a shared spec; breaking-change detectors diff spec versions | simple; works for public APIs with unknown consumers |

Pact flow:

```text
 Consumer CI                               Pact Broker / PactFlow                   Provider CI
 -----------                               ----------------------                   -----------
 unit test against Pact mock server  --->  publish pact (consumer v1.4.2)  --->  verify pact against real provider
 (records interactions)                                                           (with provider states set up)
                                           can-i-deploy? <-------------------- publish verification result
 deploy only if can-i-deploy says yes  <---
```

- **Provider states** (`given('user 42 exists')`) let the provider seed the data each interaction needs.
- **Matchers** (`like`, `eachLike`, `regex`) assert shape and type, not exact values, so contracts don't break on irrelevant data changes.
- `can-i-deploy` checks the broker's matrix: is this version compatible with what's deployed in the target environment?

### Engineering blueprint

Consumer side (frontend or calling service):

```ts
import { PactV3, MatchersV3 } from '@pact-foundation/pact';
const { like, eachLike, regex } = MatchersV3;

const provider = new PactV3({ consumer: 'web-app', provider: 'orders-api' });

it('fetches a user\'s orders', () => {
  provider
    .given('user 42 has orders')
    .uponReceiving('a request for user 42 orders')
    .withRequest({ method: 'GET', path: '/users/42/orders', headers: { Accept: 'application/json' } })
    .willRespondWith({
      status: 200,
      headers: { 'Content-Type': 'application/json' },
      body: eachLike({ id: like('o_1'), status: regex(/pending|paid|shipped/, 'paid'), totalCents: like(1200) }),
    });

  return provider.executeTest(async mockServer => {
    const orders = await new OrdersClient(mockServer.url).listForUser('42');   // real client code
    expect(orders[0].status).toBe('paid');
  });
});
```

Provider side:

```ts
import { Verifier } from '@pact-foundation/pact';

it('honours all consumer pacts', () =>
  new Verifier({
    providerBaseUrl: 'http://localhost:4000',
    pactBrokerUrl: process.env.PACT_BROKER_URL,
    provider: 'orders-api',
    providerVersion: process.env.GIT_SHA,
    publishVerificationResult: !!process.env.CI,
    stateHandlers: { 'user 42 has orders': () => seed.ordersForUser('42', 2) },
  }).verifyProvider());
```

Spec-based alternative (when you own an OpenAPI file):

- Validate every API test response against the spec (e.g. `jest-openapi`, `openapi-response-validator`, Schemathesis for property-based API fuzzing from the spec).
- Detect breaking spec changes in PRs with `oasdiff breaking old.yaml new.yaml`.
- Generate typed clients (openapi-typescript, orval) so frontend type-checks fail on drift.

### Production pitfalls

- Contracts with exact values (`totalCents: 1200`) instead of matchers; every data tweak breaks them.
- Consumers putting the provider's business logic in pacts ("returns 3 orders sorted by date"); contracts should test the *shape and meaning of the interface*, not functional behavior.
- Verifying pacts but not using `can-i-deploy`; incompatible versions still go out.
- Provider states that share data or aren't idempotent, causing flaky verification.
- Schema-only validation that allows anything (`additionalProperties: true`, everything optional) and catches nothing.

### SDE best practices

- Use Pact for internal service-to-service and frontend-to-BFF APIs you control on both sides; use OpenAPI validation + breaking-change checks for public APIs.
- Gate every deploy on `can-i-deploy` for the target environment.
- Version pacts with git SHA and branch; tag environments when deploying.
- Keep contracts minimal: only the fields the consumer reads.
- Combine with MSW [8]: frontend tests use handlers that are verified against the provider, closing the "mock drift" gap.

### Resources

- [Pact docs](https://docs.pact.io/)
- [Pact: can-i-deploy](https://docs.pact.io/pact_broker/can_i_deploy)
- [Martin Fowler: Consumer-Driven Contracts](https://martinfowler.com/articles/consumerDrivenContracts.html)
- [oasdiff (OpenAPI breaking-change detection)](https://github.com/oasdiff/oasdiff)
- [Schemathesis (property-based API testing from OpenAPI)](https://schemathesis.readthedocs.io/)

---

## 17. Testing Async Code, Queues, and Time

Backends do a lot of work that isn't request/response: background jobs, message consumers, scheduled tasks, retries, webhooks, and timeouts. These are hard to test because results show up **later** and **somewhere else**. The trick is to control time and wait on conditions, never on fixed delays.

### Core mechanism

- **Await promises properly.** An un-awaited promise means the test finishes before the work does (false pass) or the work fails after the test (unrelated test fails).
- **Fake timers** (`vi.useFakeTimers()`, `jest.useFakeTimers()`, Sinon, Python `freezegun`/`time-machine`, Java `Clock`) let you jump forward in time instantly to test timeouts, retries with backoff, TTLs, and cron schedules.
- **Polling assertions** (`vi.waitFor`, `expect.poll`, Awaitility in Java, `eventually` helpers) wait until a condition becomes true or a timeout passes, which is the right way to test eventual consistency.
- **Queue consumers** are tested at two levels: the handler as a plain function (unit), and the full produce -> broker -> consume path with a real broker in a container (integration).
- **Delivery semantics** to test: at-least-once delivery means duplicates happen, so handlers must be **idempotent**; failures must go to retry and finally a dead-letter queue (DLQ); ordering assumptions must be explicit.

### Engineering blueprint

Retry with exponential backoff using fake timers:

```ts
it('retries a failing call 3 times with backoff, then succeeds', async () => {
  vi.useFakeTimers();
  const call = vi.fn()
    .mockRejectedValueOnce(new Error('503'))
    .mockRejectedValueOnce(new Error('503'))
    .mockResolvedValue('ok');

  const p = withRetry(call, { retries: 3, baseMs: 100 });     // waits 100, 200 ms
  await vi.advanceTimersByTimeAsync(300);

  await expect(p).resolves.toBe('ok');
  expect(call).toHaveBeenCalledTimes(3);
  vi.useRealTimers();
});
```

Idempotent consumer (unit level):

```ts
it('processes a duplicate message only once', async () => {
  const ledger = new InMemoryLedger();
  const handler = makePaymentCapturedHandler({ ledger, processed: new InMemoryProcessedIds() });
  const msg = { id: 'evt_1', type: 'payment.captured', data: { orderId: 'o1', cents: 500 } };

  await handler(msg);
  await handler(msg);   // redelivery

  expect(ledger.entriesFor('o1')).toHaveLength(1);
});
```

End to end through a real broker:

```ts
it('publishes order.created and the email worker sends a confirmation', async () => {
  const outbox = fakeEmail.outbox();
  await request(app).post('/orders').set(authHeader()).send({ sku: 'MUG-1', qty: 1 }).expect(201);

  await vi.waitFor(() => expect(outbox).toContainEqual(expect.objectContaining({ template: 'order-confirmation' })),
                   { timeout: 5000, interval: 100 });
});
```

Async scenarios checklist:

| Scenario | Test |
| --- | --- |
| duplicate delivery | handler runs twice, side effect once |
| poison message | after N failures, message lands in DLQ; consumer keeps going |
| out-of-order events | `updated` before `created` handled or rejected safely |
| consumer crash mid-way | outbox/transaction ensures no lost or half-written work |
| timeout | slow dependency -> timeout error, no hung worker |
| scheduled job | fake clock at boundary times (month end, DST change, leap day) |

### Production pitfalls

- `await sleep(2000)` to "wait for the worker"; slow when it works, flaky when CI is slow.
- Fake timers left on after a test, freezing every later test's timeouts.
- Fake timers with promise-based code using `advanceTimersByTime` (sync) instead of `advanceTimersByTimeAsync`; the promise callbacks never run.
- Assuming exactly-once delivery; double-charges and double-emails in production.
- Background work started by a request but not awaited in tests, so errors appear in an unrelated test later.
- Testing consumers only with an in-memory queue that never redelivers, reorders, or times out.

### SDE best practices

- Wait on conditions (`waitFor`, `expect.poll`) with clear timeouts; never fixed sleeps.
- Make handlers idempotent with a processed-message-ID table or natural idempotency keys, and test it.
- Use the transactional outbox pattern for "write to DB and publish" and test both crash points.
- Run broker integration tests (Kafka/RabbitMQ/SQS via LocalStack) in containers for the critical flows.
- Inject a `Clock` for scheduled jobs; test DST transitions and month/year boundaries explicitly.

### Resources

- [Vitest: Timers](https://vitest.dev/guide/mocking#timers)
- [Playwright: expect.poll](https://playwright.dev/docs/test-assertions#expectpoll)
- [Awaitility (Java async assertions)](https://github.com/awaitility/awaitility)
- [microservices.io: Transactional outbox](https://microservices.io/patterns/data/transactional-outbox.html)
- [Testcontainers Kafka module](https://testcontainers.com/modules/kafka/)

---

## 18. Load and Performance Testing (k6)

Load testing sends many simultaneous requests to your backend to answer: *how fast is it under realistic traffic, how much traffic can it take, and how does it fail when overloaded?* Functional tests prove correctness for one user; load tests prove it for ten thousand.

### Core mechanism

| Test type | Shape | Question |
| --- | --- | --- |
| Smoke | 1 to 5 VUs, 1 min | does the script work; baseline latency |
| Load (average) | expected peak traffic, 15 to 60 min | do we meet SLOs at normal peak? |
| Stress | ramp beyond peak until it breaks | where is the limit; does it fail gracefully? |
| Spike | sudden 10x jump | do autoscaling, queues, and rate limits react? |
| Soak (endurance) | normal load, hours | memory leaks, connection leaks, disk growth |
| Breakpoint | slowly increasing load | maximum capacity for planning |

- **Virtual users (VUs)** run scripts in loops. **Open model** (fixed arrival rate, e.g. 200 req/s) matches real internet traffic better than **closed model** (fixed number of users waiting for each response), because real users don't slow down when your server does.
- Report **percentiles** (p50, p95, p99), not averages; tail latency is what users feel.
- **Thresholds** turn results into pass/fail for CI.
- Watch the server too (CPU, memory, DB connections, GC pauses, queue depth); the load tool only sees the outside.

### Engineering blueprint

k6 script with an arrival-rate scenario and SLO thresholds:

```js
// load/checkout.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  scenarios: {
    peak: {
      executor: 'ramping-arrival-rate',
      startRate: 10, timeUnit: '1s', preAllocatedVUs: 200, maxVUs: 1000,
      stages: [
        { target: 200, duration: '5m' },   // ramp up to 200 req/s
        { target: 200, duration: '20m' },  // hold peak
        { target: 0, duration: '2m' },     // ramp down
      ],
    },
  },
  thresholds: {
    http_req_failed: ['rate<0.01'],                       // < 1% errors
    'http_req_duration{name:list}': ['p(95)<300'],        // reads
    'http_req_duration{name:create}': ['p(95)<800', 'p(99)<1500'],
  },
};

export default function () {
  const headers = { Authorization: `Bearer ${__ENV.TOKEN}`, 'Content-Type': 'application/json' };
  const list = http.get(`${__ENV.BASE_URL}/orders?limit=20`, { headers, tags: { name: 'list' } });
  check(list, { 'list 200': r => r.status === 200 });

  const create = http.post(`${__ENV.BASE_URL}/orders`, JSON.stringify({ sku: 'MUG-1', qty: 1 }), { headers, tags: { name: 'create' } });
  check(create, { 'create 201': r => r.status === 201 });
  sleep(1);
}
```

Run: `k6 run -e BASE_URL=https://staging.example.com -e TOKEN=... load/checkout.js`.

Working out the target load (Little's Law: concurrency = throughput x latency):

```text
Peak:            1,200,000 requests/day, 20% of them in the busiest hour
Busiest hour:    240,000 / 3600 s  ≈ 67 req/s average
Peak factor x3:  ≈ 200 req/s  -> test target
At p95 300 ms:   in-flight requests ≈ 200 x 0.3 = 60  -> size pools and VUs accordingly
```

Other tools: Gatling (Scala/Java/JS), Locust (Python), JMeter, Artillery, Vegeta, wrk2.

### Production pitfalls

- Testing from one laptop; the load generator is the bottleneck, not the server.
- Testing against a staging environment that is 1/10th the size of prod, then extrapolating linearly.
- Same user, same product ID in every request: everything is cached, numbers look great, prod falls over.
- Coordinated omission: closed-model tools under-report latency when the server stalls. Use arrival-rate executors.
- Load testing with third-party calls enabled (real payment sandbox rate limits, real emails sent).
- Running once before launch, never again; performance regresses one commit at a time.

### SDE best practices

- Derive targets from real traffic data and SLOs; test peak x 2 to 3.
- Use realistic, varied data (many users, products, search terms) and realistic think time.
- Put a short smoke/load test with thresholds in CI for critical endpoints; run full load/soak nightly or pre-release on a production-like environment.
- Correlate client-side results with server metrics and traces to find the actual bottleneck.
- Stub external vendors with fast fakes (or vendor-provided load sandboxes) and say so in the report.

### Resources

- [Grafana k6 docs](https://grafana.com/docs/k6/latest/)
- [k6: Load test types](https://grafana.com/docs/k6/latest/testing-guides/test-types/)
- [k6: Open vs closed models](https://grafana.com/docs/k6/latest/using-k6/scenarios/concepts/open-vs-closed/)
- [Gil Tene: How NOT to Measure Latency (talk on coordinated omission)](https://www.youtube.com/watch?v=lJ8ydIuPFeU)
- [Gatling docs](https://docs.gatling.io/)
- [Locust docs](https://docs.locust.io/)

---

## 19. Security Testing

Security testing looks for ways an attacker could read, change, or destroy things they shouldn't. Some of it is automated in every build (scanners, dependency checks, authorization tests); some needs humans (threat modeling, penetration tests).

### Core mechanism

| Technique | What it does | Examples |
| --- | --- | --- |
| SAST (static analysis) | reads source code for dangerous patterns | Semgrep, CodeQL, SonarQube, ESLint security plugins |
| SCA (dependency scanning) | flags known-vulnerable packages (CVEs) | Dependabot, `npm audit`, Snyk, OSV-Scanner, Trivy |
| Secret scanning | finds committed keys/tokens | gitleaks, GitHub secret scanning, trufflehog |
| DAST (dynamic analysis) | attacks the running app over HTTP | OWASP ZAP, Burp Suite, Nuclei |
| Container/IaC scanning | vulnerable base images, risky config | Trivy, Grype, Checkov |
| Fuzzing | random/malformed inputs to find crashes | Schemathesis, RESTler, Jazzer.js |
| Security unit/integration tests | your own tests for authz, validation, headers | Vitest/Jest + Supertest |
| Pen test / red team | humans chaining issues the tools miss | internal or external specialists |

The most valuable tests you write yourself target the OWASP API Security Top 10 risks, especially **BOLA** (broken object-level authorization: user A reads user B's object by changing an ID).

### Engineering blueprint

Authorization matrix test generated from a table:

```ts
const matrix = [
  // role,     method,  path,                     expected
  ['viewer',  'GET',    '/projects/:own',          200],
  ['viewer',  'DELETE', '/projects/:own',          403],
  ['editor',  'PATCH',  '/projects/:own',          200],
  ['editor',  'GET',    '/projects/:otherTenant',  404],
  ['admin',   'DELETE', '/projects/:otherTenant',  404],
  [null,      'GET',    '/projects/:own',          401],
] as const;

describe.each(matrix)('%s %s %s', (role, method, path, expected) => {
  it(`returns ${expected}`, async () => {
    const ctx = await seedTwoTenants();
    const url = path.replace(':own', ctx.ownProject.id).replace(':otherTenant', ctx.foreignProject.id);
    const req = request(app)[method.toLowerCase() as 'get'](url);
    if (role) req.set(authHeader(ctx.userWithRole(role)));
    await req.expect(expected);
  });
});
```

Input-handling and header tests:

```ts
it('treats SQL meta-characters as data', async () => {
  await request(app).get('/search').query({ q: "' OR 1=1 --" }).set(authHeader()).expect(200).expect(res => {
    expect(res.body.items).toEqual([]);
  });
});

it('ignores privileged fields in the body (mass assignment)', async () => {
  const res = await request(app).patch('/me').set(authHeader()).send({ name: 'Ada', role: 'admin' }).expect(200);
  expect(res.body.role).toBe('member');
});

it('sets security headers', async () => {
  const res = await request(app).get('/').expect(200);
  expect(res.headers['strict-transport-security']).toMatch(/max-age=\d+/);
  expect(res.headers['x-content-type-options']).toBe('nosniff');
  expect(res.headers['content-security-policy']).toBeDefined();
});
```

Pipeline placement:

```text
pre-commit:   secret scan (gitleaks)
every PR:     SAST (Semgrep/CodeQL), SCA (fail on high/critical with a fix available), authz + input tests
nightly:      DAST baseline scan (ZAP) against staging, container image scan, API fuzzing from OpenAPI
quarterly:    threat model review; pen test before major launches
```

### Production pitfalls

- Scanners producing hundreds of unread findings; teams turn them off. Tune rules and fail only on new high-severity issues.
- Only testing "user without permission gets 403"; never testing "user with permission on *their* object can't touch *another user's* object" (BOLA).
- Tests running with authentication disabled or a superuser DB role that bypasses row-level security.
- Dependency alerts ignored because upgrades are scary; lack of tests makes upgrades scary. Good tests make patching cheap.
- DAST scans run against production without coordination (data pollution, alerts, rate limits).

### SDE best practices

- Write authorization tests for every endpoint: unauthenticated, wrong role, other user, other tenant.
- Run SAST, SCA, and secret scanning on every PR; fail on new high/critical issues, not on legacy backlog.
- Add a regression test for every security bug found by a scanner, bug bounty, or pen test.
- Test security headers, cookie flags, CORS allow-list, and rate limits as normal API tests.
- Keep a threat model per service and map each threat to the test that covers it.

### Resources

- [OWASP Web Security Testing Guide](https://owasp.org/www-project-web-security-testing-guide/)
- [OWASP API Security Top 10](https://owasp.org/API-Security/)
- [OWASP ZAP](https://www.zaproxy.org/docs/)
- [Semgrep docs](https://semgrep.dev/docs/)
- [GitHub CodeQL](https://codeql.github.com/docs/)
- [OSV-Scanner](https://google.github.io/osv-scanner/)
- [gitleaks](https://github.com/gitleaks/gitleaks)

---

## 20. Resilience and Chaos Testing

Resilience testing checks what your system does when its dependencies misbehave: the database is slow, the payment API times out, a pod is killed, the network drops packets. Chaos engineering does this deliberately and in a controlled way to find weak spots before a real outage does.

### Core mechanism

- **Failure modes to simulate:** latency, timeouts, errors (5xx), connection refused, partial responses, DNS failure, packet loss, process kill, disk full, clock skew, region/zone loss.
- **Patterns being tested:** timeouts on every outbound call, retries with backoff + jitter (only for idempotent operations), circuit breakers, bulkheads (separate pools per dependency), fallbacks/degraded mode, load shedding, graceful shutdown.
- **Levels:**
  - In tests: inject faults with fakes, MSW/WireMock delays, or a proxy like **Toxiproxy** between the app and a real container.
  - In staging/production: chaos tools (AWS Fault Injection Service, Chaos Mesh, LitmusChaos, Gremlin) run **experiments** with a hypothesis, a small blast radius, and an abort condition.
- **Principles of Chaos:** define steady state (e.g. checkout success rate), hypothesize it holds during the fault, inject the fault, compare, minimize blast radius.

### Engineering blueprint

Toxiproxy in an integration test (latency on the DB link):

```ts
import { Network } from 'testcontainers';
import { PostgreSqlContainer } from '@testcontainers/postgresql';
import { ToxiProxyContainer } from '@testcontainers/toxiproxy';

it('returns 503 quickly when the database is slow, instead of hanging', async () => {
  const network = await new Network().start();
  await new PostgreSqlContainer('postgres:16-alpine').withNetwork(network).withNetworkAliases('postgres').start();
  const toxiproxy = await new ToxiProxyContainer('ghcr.io/shopify/toxiproxy:2.11.0').withNetwork(network).start();
  const proxy = await toxiproxy.createProxy({ name: 'pg', upstream: 'postgres:5432' });   // app -> proxy -> postgres
  const app = buildApp({ db: poolFor(proxy.host, proxy.port, { statementTimeoutMs: 500 }), ...deps });

  await proxy.instance.addToxic({ type: 'latency', stream: 'downstream', attributes: { latency: 3000 } });

  const started = Date.now();
  await request(app).get('/orders').set(authHeader()).expect(503);
  expect(Date.now() - started).toBeLessThan(1000);
});
```

Circuit breaker behavior with a stubbed dependency:

```ts
it('opens the circuit after 5 failures and stops calling the gateway', async () => {
  const gw = { charge: vi.fn().mockRejectedValue(new Error('timeout')) };
  const svc = new PaymentsService(withBreaker(gw, { failureThreshold: 5, resetMs: 30_000 }));

  for (let i = 0; i < 5; i++) await expect(svc.pay('o1')).rejects.toThrow();
  await expect(svc.pay('o1')).rejects.toThrow(/circuit open/);
  expect(gw.charge).toHaveBeenCalledTimes(5);
});
```

Chaos experiment template (staging or prod):

| Field | Example |
| --- | --- |
| Steady state | checkout success >= 99.5%, p95 < 800 ms |
| Hypothesis | killing 1 of 3 API pods causes no SLO breach |
| Fault | terminate one pod every 5 min for 30 min |
| Blast radius | 1 service, 10% traffic, business hours, on-call aware |
| Abort | success rate < 98% for 2 min -> stop and roll back |
| Result | findings + tickets (e.g. readiness probe too slow) |

### Production pitfalls

- No timeout on outbound HTTP/DB calls; one slow dependency exhausts all threads/connections and takes the whole service down.
- Retries without backoff or jitter (retry storm) or on non-idempotent operations (double charges).
- Fallbacks that were never tested and throw their own errors.
- Running chaos in production without monitoring, an abort switch, or team awareness.
- Graceful shutdown not tested: deploys drop in-flight requests and partially processed messages.

### SDE best practices

- Every outbound call has a timeout, and a test proves the service responds within budget when the dependency hangs.
- Test retries, breakers, and fallbacks with injected faults in integration tests (Toxiproxy, WireMock faults, MSW delays).
- Start chaos in staging, with game days; move to production only with good observability and small blast radius.
- Turn every incident into a resilience test or chaos experiment so the same failure is caught next time.

### Resources

- [Principles of Chaos Engineering](https://principlesofchaos.org/)
- [Toxiproxy](https://github.com/Shopify/toxiproxy)
- [AWS Fault Injection Service](https://docs.aws.amazon.com/fis/)
- [Chaos Mesh](https://chaos-mesh.org/docs/)
- [AWS Builders' Library: Timeouts, retries, and backoff with jitter](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/)
- Michael Nygard, *Release It!* (2nd ed., Pragmatic Bookshelf)

---

# Part D: Quality and Delivery

## 21. Test Data Management (Fixtures, Factories, Seeds)

Every test needs data: a user, an order, a product in stock. How you create that data decides whether tests are readable, independent, and fast, or a tangle of shared JSON files nobody dares to edit.

### Core mechanism

| Approach | What it is | Good for | Weak at |
| --- | --- | --- | --- |
| Inline literals | build the object in the test | tiny unit tests | repetition for big objects |
| Static fixtures | JSON/SQL files loaded before tests | read-only reference data (countries, plans) | shared mutable data; hidden coupling |
| Factories / builders | functions that build valid objects with defaults, overridable per test | most tests | needs maintenance as models change |
| Seed scripts | populate a DB for dev/E2E/demo | E2E baseline, local dev | per-test variations |
| Production snapshots | (anonymized) copies of real data | perf tests, migration rehearsals | privacy risk if not masked |

Key ideas:

- **Only state what matters:** a test about expired subscriptions should only mention `expiresAt`; everything else comes from factory defaults.
- **Valid by default:** factories always produce a valid object; tests override to make it invalid when needed.
- **Unique by default:** sequences or random IDs avoid unique-constraint collisions in parallel runs.
- **Deterministic randomness:** if you use Faker, seed it (`faker.seed(123)`) so failures reproduce.

### Engineering blueprint

A small typed factory (libraries: `fishery`, `@faker-js/faker`, Python `factory_boy`, Ruby `factory_bot`, Java Instancio):

```ts
// test/factories/user.ts
import { Factory } from 'fishery';
import { faker } from '@faker-js/faker';

export const userFactory = Factory.define<User>(({ sequence, onCreate }) => {
  onCreate(user => usersRepo.insert(user));          // persist only when .create() is used
  return {
    id: crypto.randomUUID(),
    email: `user${sequence}@example.test`,
    name: faker.person.fullName(),
    role: 'member',
    tenantId: 't_default',
    createdAt: new Date('2026-01-01T00:00:00Z'),
  };
});

// in tests
const admin = userFactory.build({ role: 'admin' });          // in memory
const saved = await userFactory.create({ tenantId: t.id });  // persisted
const many  = await userFactory.createList(25);
```

Builder style for complex aggregates:

```ts
const order = anOrder().forUser(alice).withItem('MUG-1', 2).paidAt('2026-01-03Z').build();
```

E2E data: create through a test-only API (enabled only in test environments) or direct DB seeding per test; tag records with a run ID so cleanup jobs can delete them.

### Production pitfalls

- One giant `fixtures.json` used by 300 tests; changing one value breaks unrelated tests (the "mystery guest" smell).
- Tests that rely on seed data IDs (`user 1 is an admin`) that someone else changes.
- Unseeded Faker making a test fail once a month on a weird name with an apostrophe (that's actually a real bug, so seed it *and* add that case explicitly).
- Copies of production data with real PII in test environments, a privacy and compliance violation.
- Factories that build huge object graphs by default (user -> tenant -> 10 projects -> 100 tasks), making tests slow.

### SDE best practices

- Factories for everything you create in tests; static fixtures only for read-only reference data.
- Show the relevant values in the test body; hide the irrelevant ones in factory defaults.
- Keep factories minimal; add traits (`userFactory.params({ role: 'admin' })`, `.admin()`) for common variants.
- Never use raw production data in tests; anonymize or synthesize, and test the masking itself.
- Clean up by isolation (transactions, per-test tenants, truncation), not by hoping tests delete their own rows.

### Resources

- [fishery](https://github.com/thoughtbot/fishery)
- [Faker.js](https://fakerjs.dev/)
- [factory_boy (Python)](https://factoryboy.readthedocs.io/)
- [xUnit Patterns: Mystery Guest and test smells](http://xunitpatterns.com/Obscure%20Test.html)
- [Martin Fowler: Object Mother](https://martinfowler.com/bliki/ObjectMother.html)

---

## 22. Code Coverage and Mutation Testing

**Coverage** tells you which lines and branches ran during tests. It shows what is *not* tested, but it can't tell you whether what ran was actually *checked*. **Mutation testing** answers that: it makes small bugs in your code on purpose and checks whether any test fails.

### Core mechanism

Coverage kinds:

| Metric | Counts | Notes |
| --- | --- | --- |
| Line / statement | lines executed | easiest to game |
| Branch | both sides of every `if`, `?:`, `&&`, `??` taken | more meaningful |
| Function | functions called at least once | coarse |
| Condition / MC/DC | each boolean sub-condition affects the outcome | required in safety-critical domains |

Tools: V8 coverage (`--coverage` with provider `v8` in Vitest), Istanbul/nyc, JaCoCo (Java), coverage.py, `go test -cover`.

Mutation testing:

```text
 original:   if (age >= 18) return 'adult';
 mutant 1:   if (age >  18) return 'adult';     <- killed if a test checks age = 18
 mutant 2:   if (age <= 18) return 'adult';     <- killed by almost any test
 mutant 3:   if (true)      return 'adult';     <- killed if a test checks a minor
```

- **Killed** mutant: some test failed, which is good. **Survived** mutant: no test noticed the bug, which is a gap.
- **Mutation score** = killed / (total - equivalent mutants). Tools: Stryker (JS/TS, C#, Scala), PIT (Java), mutmut / cosmic-ray (Python), go-mutesting.

### Engineering blueprint

Coverage thresholds and diff coverage in CI:

```ts
// vitest.config.ts (excerpt)
coverage: {
  provider: 'v8',
  include: ['src/**'],
  exclude: ['src/**/*.stories.tsx', 'src/generated/**'],
  thresholds: { lines: 80, branches: 75, functions: 80 },
  reporter: ['text-summary', 'lcov', 'json-summary'],
}
```

Combine with a diff-coverage gate on PRs (Codecov/Coveralls/SonarQube "new code" coverage >= 80%) so new code is tested even if legacy code isn't.

Stryker on a critical module:

```json
// stryker.config.json
{
  "testRunner": "vitest",
  "mutate": ["src/domain/pricing/**/*.ts", "src/domain/refund.ts"],
  "thresholds": { "high": 85, "low": 70, "break": 65 },
  "incremental": true,
  "reporters": ["html", "clear-text", "progress"]
}
```

Run: `npx stryker run`. Open the HTML report, look at survived mutants, add the missing assertions.

How to read the numbers:

| Situation | Meaning |
| --- | --- |
| low coverage | definitely under-tested |
| high coverage, low mutation score | code runs in tests but results aren't asserted |
| high coverage, high mutation score | tests really protect behavior |
| 100% coverage target | usually leads to tests of trivial getters and generated code |

### Production pitfalls

- Coverage targets as KPIs: people write assertion-free tests to hit the number (Goodhart's law).
- Counting generated code, config, and type-only files, which distorts the number either way.
- Running mutation testing on the whole codebase on every PR; it is slow. Use incremental mode and focus on critical modules.
- Treating 100% coverage as "bug-free"; coverage says nothing about missing code (unhandled cases you never wrote).

### SDE best practices

- Use coverage to *find* untested code, not to prove quality. Gate on "no drop" and on diff coverage for new code.
- Prefer branch coverage over line coverage.
- Run mutation testing on high-risk logic (money, permissions, parsing) on a schedule or on changed files.
- Exclude generated and trivial code explicitly; review the exclusion list.

### Resources

- [Vitest: Coverage](https://vitest.dev/guide/coverage)
- [Stryker Mutator](https://stryker-mutator.io/)
- [PIT (Java mutation testing)](https://pitest.org/)
- [Martin Fowler: Test Coverage](https://martinfowler.com/bliki/TestCoverage.html)
- [Google Testing Blog: Code coverage best practices](https://testing.googleblog.com/2020/08/code-coverage-best-practices.html)

---

## 23. Property-Based Testing and Fuzzing

In normal (example-based) tests, *you* pick the inputs. In **property-based testing**, you describe a rule that must hold for **all** inputs, and the library generates hundreds of random inputs to try to break it. When it finds a failure, it **shrinks** the input to the smallest example that still fails. **Fuzzing** is the security-flavored cousin: throw huge numbers of malformed inputs at a parser or API to find crashes.

### Core mechanism

- **Arbitraries/generators** produce random values of a shape (`fc.integer()`, `fc.string()`, `fc.record({...})`).
- **Properties** are assertions that should be true for every generated value.
- **Shrinking** turns a failing 200-element array into `[0, -1]`, which is easy to debug.
- **Seeds** make runs reproducible; the library prints the seed and path of a failure.

Common property patterns:

| Pattern | Example |
| --- | --- |
| Round trip | `decode(encode(x)) == x` (JSON, URL params, serializers) |
| Invariant | sorting keeps the same length and elements |
| Idempotence | `normalize(normalize(x)) == normalize(x)` |
| Oracle / reference | fast implementation equals slow obvious implementation |
| Never crashes | parser returns a result or a typed error for any string |
| Metamorphic | adding an item never decreases the cart total |

### Engineering blueprint

fast-check with Vitest:

```ts
import fc from 'fast-check';

it('money formatting round-trips', () => {
  fc.assert(fc.property(fc.integer({ min: -1e12, max: 1e12 }), cents => {
    expect(parseMoney(formatMoney(cents, 'USD'), 'USD')).toBe(cents);
  }));
});

it('discount never makes total negative or larger than subtotal', () => {
  fc.assert(fc.property(
    fc.integer({ min: 0, max: 10_000_000 }),
    fc.constantFrom('SAVE10', 'HALF', 'BOGUS', ''),
    (subtotal, coupon) => {
      const total = applyDiscount({ subtotal, coupon });
      expect(total).toBeGreaterThanOrEqual(0);
      expect(total).toBeLessThanOrEqual(subtotal);
    },
  ));
});

it('parser never throws on arbitrary input', () => {
  fc.assert(fc.property(fc.string(), s => { expect(() => parseQuery(s)).not.toThrow(); }));
});
```

API-level fuzzing from the OpenAPI spec:

```bash
schemathesis run https://staging.example.com/openapi.json --checks all -H "Authorization: Bearer $TOKEN"
```

Language equivalents: Hypothesis (Python), jqwik (Java), Go native fuzzing (`func FuzzXxx(f *testing.F)` with `go test -fuzz=FuzzXxx`), proptest (Rust).

### Production pitfalls

- Properties that re-implement the function under test (the oracle has the same bug).
- Generators too narrow (only small positive ints) to find real edge cases, or too wide so inputs are always invalid and rejected.
- Non-reproducible failures because the seed wasn't logged or the property depends on time/IO.
- Slow properties (DB calls inside) run 100 times per test; keep them pure or lower `numRuns`.

### SDE best practices

- Use property tests for parsers, serializers, money/date math, validators, and data transformations.
- Add the shrunk counterexample as a regular example test once fixed.
- Log and pin seeds in CI; run more iterations nightly than on PRs.
- Fuzz public APIs and file/format parsers; they are where attackers send unexpected input.

### Resources

- [fast-check](https://fast-check.dev/)
- [Hypothesis (Python)](https://hypothesis.readthedocs.io/)
- [Go: Fuzzing tutorial](https://go.dev/doc/tutorial/fuzz)
- [Schemathesis](https://schemathesis.readthedocs.io/)
- [Scott Wlaschin: Choosing properties for property-based testing](https://fsharpforfunandprofit.com/posts/property-based-testing-2/)

---

## 24. Flaky Tests

A **flaky test** passes and fails on the same code without any change. Flaky tests are dangerous because they train the team to ignore red builds, and then real failures get ignored too. Google has reported that about 1.5% of its test runs showed a flaky result and that almost 16% of its tests had some level of flakiness. Every large suite has this problem; the difference is whether you manage it.

### Core mechanism

Common root causes:

| Cause | Typical symptom | Fix |
| --- | --- | --- |
| Async timing / race | fails on slow CI | wait on conditions, await everything |
| Shared state / order dependence | fails only when run with other tests | isolate data; fresh stores/mocks per test |
| Time and timezone | fails near midnight / month end / in UTC | inject clock, fake timers, `TZ=UTC` |
| Randomness | rare failure, can't reproduce | seed RNG/Faker, log seeds |
| External services | fails when vendor is slow/down | stub at boundary |
| Resource leaks | later tests time out | close servers, DB pools, timers in `afterAll` |
| Unordered results | list order differs | `ORDER BY` in SQL; sort before asserting or use `arrayContaining` |
| UI animations / layout | element "not clickable" | auto-waiting locators, disable animations |
| Infrastructure | OOM, port clash, slow container start | resource limits, random ports, health checks |

### Engineering blueprint

Detect:

```bash
# run one suspect test many times
npx vitest run src/cart.test.ts --repeats 50     # Vitest
npx playwright test checkout.spec.ts --repeat-each=30 --workers=4
# randomize order to surface hidden dependencies
npx vitest run --sequence.shuffle
```

Process:

```text
CI marks test as "passed on retry"  ->  flake tracker (e.g. CI analytics, BuildPulse, Trunk, Datadog CI Visibility)
       |
       v
Owner auto-assigned  ->  test quarantined (still runs, doesn't block merges)  ->  fixed or deleted within SLA
       |
       v
Fix merged with "repro: ran 200x green" evidence  ->  un-quarantine
```

Quarantine with Playwright annotations or a tag:

```ts
test('report export downloads a CSV', { tag: '@quarantine', annotation: { type: 'issue', description: 'JIRA-1234' } }, async ({ page }) => { /* ... */ });
// CI: blocking run uses --grep-invert @quarantine; a separate non-blocking job runs --grep @quarantine
```

### Production pitfalls

- Blanket retries (`retries: 3`) everywhere, so flakes are invisible and the suite slowly gets slower and less trustworthy.
- "Fixing" flakes by increasing timeouts or adding `sleep`.
- Quarantine without owners or deadlines; the quarantine folder becomes a graveyard of disabled tests and lost coverage.
- Treating flakiness as a test problem when it's a product bug (a real race condition in the app).

### SDE best practices

- Track flake rate per test; any test that passes on retry is reported, not hidden.
- Quarantine fast (same day), fix or delete within a set time window, and always assign an owner.
- Reproduce with repeat/shuffle runs and CI-like resource limits before declaring a fix.
- Ask "could this race happen for real users?" before blaming the test.
- Make determinism the default: fixed clock, seeded randomness, isolated data, no fixed sleeps.

### Resources

- [Google Testing Blog: Flaky tests at Google and how we mitigate them](https://testing.googleblog.com/2016/05/flaky-tests-at-google-and-how-we.html)
- [Martin Fowler: Eradicating non-determinism in tests](https://martinfowler.com/articles/nonDeterminism.html)
- [Playwright: Retries](https://playwright.dev/docs/test-retries)
- [Vitest CLI options (repeats, shuffle)](https://vitest.dev/guide/cli)

---

## 25. Testing in CI/CD

Continuous Integration runs your tests automatically on every push and pull request, so problems are found within minutes of being written. A good pipeline is **fast for the common case** (most PRs) and **thorough before release**.

### Core mechanism

- **Stages, fail fast:** cheapest checks first (lint, types, unit), expensive ones later (integration, E2E), heavy ones on a schedule (load, DAST, full cross-browser).
- **Parallelism and sharding:** split tests across machines (`--shard=i/n`); balance by historical timing, not file count.
- **Caching:** dependencies (`node_modules`, pnpm store), build outputs, Docker layers, Playwright browsers.
- **Test impact analysis / affected-only:** in monorepos, run only tests for projects touched by the change (Nx `affected`, Turborepo filters, Bazel).
- **Quality gates:** required checks that block merge (tests, coverage on new code, security scan, contract `can-i-deploy`).
- **Ephemeral environments:** a preview deploy per PR with a seeded DB for E2E.
- **Post-deploy verification:** smoke tests against each environment after deploy; automatic rollback if they fail.

### Engineering blueprint

GitHub Actions pipeline (trimmed):

```yaml
name: ci
on: [pull_request]

jobs:
  static:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: npm }
      - run: npm ci
      - run: npm run lint && npm run typecheck

  unit:
    needs: static
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: npm }
      - run: npm ci
      - run: npx vitest run --project web --project api --coverage --reporter=junit --outputFile=results/unit.xml
        env: { TZ: UTC }
      - uses: actions/upload-artifact@v4
        if: always()
        with: { name: unit-results, path: results/ }

  integration:
    needs: static
    runs-on: ubuntu-latest          # Docker available for Testcontainers
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: npm }
      - run: npm ci
      - run: npm run test:integration

  e2e:
    needs: [unit, integration]
    runs-on: ubuntu-latest
    strategy:
      matrix: { shard: [1, 2, 3, 4] }
    container: mcr.microsoft.com/playwright:v1.55.0-noble   # pin to the version in package.json
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npx playwright test --shard=${{ matrix.shard }}/4
      - uses: actions/upload-artifact@v4
        if: failure()
        with: { name: e2e-report-${{ matrix.shard }}, path: playwright-report/ }
```

Typical stage map:

| Trigger | Runs | Time budget |
| --- | --- | --- |
| pre-commit (local) | format, lint-staged, related unit tests | < 30 s |
| every PR | lint, types, unit, component, API integration, contract, E2E smoke, SAST/SCA | < 15 min |
| merge to main | above + full E2E, visual, build & deploy to staging, smoke | < 30 min |
| nightly | load/soak, DAST, full cross-browser, mutation testing, flake detection | hours |
| post-deploy (each env) | smoke + synthetic checks; auto-rollback on failure | < 5 min |

### Production pitfalls

- One long serial job; a 45-minute PR cycle makes people batch changes and skip running tests locally.
- Pipeline differs from local (different Node version, missing env vars) so "works locally" is meaningless. Pin versions; use the same scripts.
- Required checks that are flaky, so people get used to "re-run until green".
- E2E against shared staging, which breaks when another team deploys.
- Test results not published; failures are buried in raw logs.

### SDE best practices

- Fail fast: static checks and unit tests first; parallelize everything else.
- Keep the PR pipeline under ~15 minutes with sharding, caching, and affected-only runs.
- Publish JUnit results, coverage, traces, and screenshots as artifacts and PR annotations.
- Same commands locally and in CI (`npm run test:*`), same container images for browser tests.
- Gate deploys on tests passing *for that exact artifact*; run smoke tests after every deploy with auto-rollback.

### Resources

- [GitHub Actions docs](https://docs.github.com/en/actions)
- [Playwright: CI and sharding](https://playwright.dev/docs/test-sharding)
- [Nx: Affected](https://nx.dev/ci/features/affected)
- [Martin Fowler: Continuous Integration](https://martinfowler.com/articles/continuousIntegration.html)
- [Google SRE Book: Testing for Reliability](https://sre.google/sre-book/testing-reliability/)

---

## 26. Testing in Production

No staging environment matches production exactly: real data, real traffic patterns, real third parties, real scale. **Testing in production** is a set of safe practices that verify changes *with* real conditions while limiting how many users could be affected.

### Core mechanism

| Practice | What it does |
| --- | --- |
| Smoke tests after deploy | a handful of quick API/UI checks against the live environment |
| Synthetic monitoring | scripted journeys (login, search, checkout) run every few minutes from several regions; alert on failure |
| Feature flags | ship code dark, enable for internal users, then 1% -> 10% -> 100% |
| Canary releases | new version gets a small slice of traffic; compare error rate/latency to baseline; auto-rollback on regression |
| Blue/green | two identical environments; switch traffic, switch back instantly if needed |
| Shadow / dark traffic | copy real requests to the new version, discard its responses, compare results |
| A/B tests | measure business impact of a change with real users |
| Real User Monitoring + error tracking | Web Vitals, JS errors, backend error rates per release (Sentry, Datadog, OpenTelemetry) |

All of them depend on **observability**: if you can't see error rates and latency per version, you can't test in production safely.

### Engineering blueprint

Post-deploy smoke test (Playwright project pointed at prod with a dedicated synthetic account):

```ts
// e2e/smoke/prod.smoke.spec.ts
test('@smoke login and dashboard load', async ({ page }) => {
  await page.goto('/login');
  await page.getByLabel('Email').fill(process.env.SYNTH_USER!);
  await page.getByLabel('Password').fill(process.env.SYNTH_PASS!);
  await page.getByRole('button', { name: 'Sign in' }).click();
  await expect(page.getByRole('heading', { name: 'Dashboard' })).toBeVisible({ timeout: 10_000 });
});

test('@smoke health and version', async ({ request }) => {
  const res = await request.get('/healthz');
  expect(res.ok()).toBeTruthy();
  expect((await res.json()).version).toBe(process.env.EXPECTED_VERSION);
});
```

Canary analysis rule (Argo Rollouts / Flagger / Spinnaker-style):

```text
for 10 minutes at 5% traffic:
  error_rate(canary)  <= error_rate(stable) + 0.5%
  p95_latency(canary) <= p95_latency(stable) x 1.2
pass  -> 25% -> 50% -> 100%
fail  -> automatic rollback + alert
```

Feature-flagged rollout with a test for both paths:

```ts
it.each([true, false])('checkout works with newPricing=%s', async flag => {
  flags.override('newPricing', flag);
  await request(app).post('/checkout').set(authHeader()).send(cart).expect(201);
});
```

### Production pitfalls

- Synthetic users and test orders polluting analytics, invoices, or emails to real people. Mark them (`is_synthetic`) and filter them out everywhere.
- Feature flags that are never removed; the number of untested flag combinations explodes.
- Canary analysis on too little traffic to be statistically meaningful.
- Smoke tests that need write access to production data without safeguards.
- "We test in production" used as an excuse to skip pre-production tests.

### SDE best practices

- Treat production checks as an *addition* to the pyramid, not a replacement.
- Run synthetic checks for top journeys continuously and alert on them like any SLO.
- Release behind flags and canaries with automatic rollback on error/latency regression.
- Tag and isolate synthetic traffic and data; use dedicated test tenants.
- Remove flags after full rollout; test both flag states while both exist.

### Resources

- [Martin Fowler: Feature Toggles](https://martinfowler.com/articles/feature-toggles.html)
- [Cindy Sridharan: Testing in Production, the safe way](https://copyconstruct.medium.com/testing-in-production-the-safe-way-18ca102d0ef1)
- [Argo Rollouts: Canary](https://argo-rollouts.readthedocs.io/en/stable/features/canary/)
- [Checkly / Playwright synthetic monitoring docs](https://www.checklyhq.com/docs/)
- [OpenTelemetry docs](https://opentelemetry.io/docs/)

---

## 27. Production Checklist & Cheat Sheet

### Which test for which bug?

| Bug | Cheapest test that catches it |
| --- | --- |
| wrong calculation / rule | unit [6][13] |
| button disabled at wrong time, wrong error text | component [7] |
| UI breaks on 500/empty response | component + MSW [8] |
| CSS regression, layout broken | visual [9] |
| unlabeled input, keyboard trap | a11y [10] |
| SQL wrong, constraint missing, migration fails | DB integration [15] |
| wrong status code, auth bypass, validation gap | API test [14][19] |
| backend renamed a field the frontend uses | contract [16] |
| double processing of a message | async/idempotency test [17] |
| works for 1 user, dies at 500 rps | load [18] |
| one slow dependency takes down the service | resilience [20] |
| login → checkout broken in the real browser | E2E [11] |
| works in staging, broken in prod config | smoke / synthetic [26] |

### Recommended tooling (JS/TS stack)

| Need | Default | Alternatives |
| --- | --- | --- |
| Runner | Vitest | Jest, `node:test` |
| Component | Testing Library + user-event | Vitest browser mode, Playwright CT, Cypress CT |
| Network mocks | MSW | nock (Node), WireMock (any) |
| E2E | Playwright | Cypress, WebdriverIO |
| Visual | Playwright screenshots / Chromatic | Percy, Applitools, Lost Pixel |
| A11y | axe-core (jest-axe, @axe-core/playwright) | Lighthouse, Pa11y |
| API | Supertest / Fastify inject | Pactum, REST-assured (Java) |
| Real deps | Testcontainers | docker-compose in CI |
| Contracts | Pact | OpenAPI validation + oasdiff |
| Load | k6 | Gatling, Locust, Artillery |
| Security | Semgrep/CodeQL, OSV/Dependabot, gitleaks, ZAP | Snyk, Burp |
| Properties/fuzz | fast-check, Schemathesis | Jazzer.js |
| Mutation | Stryker | |
| Data | fishery + Faker | factory helpers |

### Pre-launch checklist

**Foundations**
- [ ] Tests follow Arrange/Act/Assert, one behavior each, descriptive names.
- [ ] Every test independent: runs alone, in any order, in parallel.
- [ ] Clock, randomness, IDs, timezone, and locale controlled in tests.
- [ ] Mocks reset automatically between tests; doubles only at boundaries you don't own.

**Frontend**
- [ ] Logic extracted from components and unit-tested.
- [ ] Component tests query by role/label; interactions via `user-event`; all `await`s present.
- [ ] Every data-driven component tested in loading, success, empty, error, 401/403 states (MSW, `onUnhandledRequest: 'error'`).
- [ ] axe checks on design-system components and key pages; keyboard flows for dialogs/menus.
- [ ] Visual tests for design-system variants and critical pages, in a pinned container.
- [ ] E2E covers critical journeys only; no fixed sleeps; data via API; traces on failure.
- [ ] Bundle-size and Lighthouse budgets in CI; RUM Web Vitals tracked.

**Backend**
- [ ] Domain rules pure and table-tested at boundaries.
- [ ] API tests run the real app in-process with auth on, real DB in a container.
- [ ] Each endpoint tested for validation, 401, 403, foreign-object 404, and error format.
- [ ] Same DB engine and version as prod; migrations applied from zero every run; constraints tested.
- [ ] Contract tests (or OpenAPI validation) between every consumer/provider pair; `can-i-deploy` gates deploys.
- [ ] Consumers idempotent; retries, DLQ, and outbox tested; no fixed sleeps.
- [ ] Timeouts on all outbound calls with tests proving bounded latency under dependency failure.
- [ ] Load test with SLO thresholds before launch and on a schedule.
- [ ] SAST, SCA, secret scan on every PR; DAST nightly; authz matrix tests in CI.

**Suite health and delivery**
- [ ] Coverage tracked; diff coverage gate on new code; mutation testing on critical modules.
- [ ] Flaky tests reported, quarantined, owned, and fixed within SLA; no hidden blanket retries.
- [ ] PR pipeline < 15 min (sharding, caching, affected-only); results published to the PR.
- [ ] Post-deploy smoke tests and synthetic monitoring with auto-rollback.
- [ ] Synthetic traffic and test data tagged and isolated from real users and analytics.

### Common smells -> fixes

| Smell | Fix |
| --- | --- |
| `sleep(2000)` / `waitForTimeout` | wait on a condition (`findBy`, `waitFor`, `expect.poll`, web-first assertions) |
| `container.querySelector('.btn')` | `getByRole('button', { name })` |
| mocking your own DB layer in API tests | real DB in Testcontainers |
| SQLite instead of Postgres | same engine via container |
| huge DOM snapshots | explicit assertions or visual tests |
| shared fixtures.json | factories with per-test overrides |
| tests break on every refactor | assert on behavior/output, not internals |
| retries everywhere | track flakes, quarantine, fix root cause |
| 100% coverage target | diff coverage + mutation score on critical code |
| E2E for every edge case | push down to component/API level |

### Quick glossary

| Term | One line |
| --- | --- |
| SUT | system under test |
| AAA / GWT | Arrange-Act-Assert / Given-When-Then structure |
| Test double | stand-in for a dependency: dummy, stub, spy, mock, fake |
| Fixture | state prepared for a test |
| Factory | function that builds valid test objects with overridable defaults |
| jsdom / happy-dom | simulated browser DOM running in Node |
| MSW | Mock Service Worker; intercepts network requests in tests |
| E2E | end-to-end test through a real browser and real backend |
| Locator | Playwright's lazy, auto-waiting element reference |
| Testcontainers | library that runs real dependencies in Docker for tests |
| Contract test | checks two services agree on an API without running both together |
| Provider state | data a provider seeds before verifying a pact interaction |
| VU | virtual user in a load test |
| p95 / p99 | 95th / 99th percentile latency |
| SAST / DAST / SCA | static code scan / running-app scan / dependency scan |
| BOLA | broken object-level authorization (accessing another user's object) |
| Mutation score | share of injected bugs your tests detect |
| Shrinking | reducing a failing random input to a minimal example |
| Flaky test | test that passes and fails on the same code |
| Canary | small slice of traffic sent to a new version first |
| Synthetic monitoring | scripted checks running continuously against production |

---

## 28. Master Resource List

### Foundations and strategy

- [Martin Fowler: The Practical Test Pyramid](https://martinfowler.com/articles/practical-test-pyramid.html)
- [Martin Fowler: Mocks Aren't Stubs](https://martinfowler.com/articles/mocksArentStubs.html)
- [Kent C. Dodds: The Testing Trophy](https://kentcdodds.com/blog/the-testing-trophy-and-testing-classifications)
- [Software Engineering at Google (free online), Ch. 11 to 14 on testing](https://abseil.io/resources/swe-book)
- [Google Testing Blog](https://testing.googleblog.com/)
- [xUnit Test Patterns (Meszaros), online catalog](http://xunitpatterns.com/)

### Frontend

- [Testing Library docs](https://testing-library.com/docs/)
- [MSW docs](https://mswjs.io/docs/)
- [Playwright docs](https://playwright.dev/docs/intro)
- [Cypress docs](https://docs.cypress.io/)
- [Storybook: Testing](https://storybook.js.org/docs/writing-tests)
- [axe-core](https://github.com/dequelabs/axe-core) and [WAI-ARIA Authoring Practices](https://www.w3.org/WAI/ARIA/apg/)
- [web.dev: Web Vitals](https://web.dev/articles/vitals) and [Lighthouse CI](https://github.com/GoogleChrome/lighthouse-ci)

### Backend

- [Supertest](https://github.com/ladjs/supertest)
- [Testcontainers](https://testcontainers.com/)
- [Pact docs](https://docs.pact.io/)
- [Grafana k6 docs](https://grafana.com/docs/k6/latest/)
- [OWASP Web Security Testing Guide](https://owasp.org/www-project-web-security-testing-guide/) and [OWASP API Security Top 10](https://owasp.org/API-Security/)
- [Principles of Chaos Engineering](https://principlesofchaos.org/)
- [AWS Builders' Library](https://aws.amazon.com/builders-library/)

### Runners and tools

- [Vitest](https://vitest.dev/) · [Jest](https://jestjs.io/) · [pytest](https://docs.pytest.org/) · [JUnit 5](https://junit.org/junit5/) · [Go testing](https://pkg.go.dev/testing)
- [fast-check](https://fast-check.dev/) · [Hypothesis](https://hypothesis.readthedocs.io/) · [Schemathesis](https://schemathesis.readthedocs.io/)
- [Stryker Mutator](https://stryker-mutator.io/) · [PIT](https://pitest.org/)
- [Semgrep](https://semgrep.dev/docs/) · [CodeQL](https://codeql.github.com/docs/) · [OWASP ZAP](https://www.zaproxy.org/docs/) · [gitleaks](https://github.com/gitleaks/gitleaks)

### Books

- Kent Beck, *Test-Driven Development: By Example* (2002)
- Steve Freeman & Nat Pryce, *Growing Object-Oriented Software, Guided by Tests* (2009)
- Gerard Meszaros, *xUnit Test Patterns* (2007)
- Vladimir Khorikov, *Unit Testing Principles, Practices, and Patterns* (2020)
- Lisa Crispin & Janet Gregory, *Agile Testing* and *More Agile Testing*
- Michael Nygard, *Release It!* (2nd ed., 2018)
- Titus Winters, Tom Manshreck, Hyrum Wright (eds.), *Software Engineering at Google* (2020)
