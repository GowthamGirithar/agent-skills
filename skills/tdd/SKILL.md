---
name: tdd
description: "Use this skill whenever implementing any task that involves writing or modifying code. Enforces Test Driven Development using the Red-Green-Refactor cycle. Trigger when the user asks to implement a feature, fix a bug, add a function, build a service, create an endpoint, or do any development work — even if they don't mention tests or TDD. This skill ensures every implementation starts with a failing test, proceeds to the minimal passing implementation, and finishes with a refactor pass."
---

# Test Driven Development

Every implementation task follows the Red-Green-Refactor cycle — not as bureaucracy, but because it produces code that is correct by construction, safe to change, and documented through its tests.

## The Core Cycle: Red-Green-Refactor

Work through each requirement **one at a time**. Do not batch multiple tests before implementing — the discipline of the cycle is what makes TDD effective.

For each requirement:

### 1. RED — Write a failing test first (Prove It)

Write a test that describes the behavior you want. Run it. It must fail. This is the **Prove It** step — you are proving that your test actually detects the absence of the behavior. A test you haven't seen fail is a test you can't trust.

The test should fail because the function doesn't exist yet, returns the wrong value, or the behavior isn't implemented. If the test passes without any new code, either the behavior already exists (great, move on) or the test is wrong (fix it).

### 2. GREEN — Write the minimum code to pass

Write just enough production code to make the failing test pass. Resist the urge to write the "complete" solution — only satisfy the current test. Run the test again — it should pass now. Also run the broader test suite to make sure you haven't broken anything.

### 3. REFACTOR — Clean up while green

Now that tests are passing, look at both the test code and production code. 
- Is there duplication? 
- Can naming be clearer? 
- Is the structure right? 
- Too many lines of code inside single method?

Refactor with confidence — your tests will catch any regressions. Run the tests again after refactoring to confirm everything still passes.

Then move on to the next requirement and repeat the cycle.

## Writing Good Tests

### DAMP — Descriptive And Meaningful Phrases

Test names and descriptions should read like specifications. Someone reading just the test names should understand what the system does without looking at the implementation.

- Use descriptive test names that state the scenario and expected outcome
- Favor readability over brevity in test code — a little repetition in tests is fine if it makes each test self-contained and clear
- Avoid cryptic abbreviations or generic names like `test1`, `testHelper`, `testBasic`
- It should describe WHAT instead HOW

**Good test names:**
- `TestCalculateTotal_WithEmptyCart_ReturnsZero`
- `test_pop_on_empty_stack_raises_index_error`
- `it("returns -5 when dividing -10 by 2")`

**Bad test names:**
- `TestCalc`, `test_it_works`, `it("works")`

### Arrange-Act-Assert

Structure every test body in three distinct phases:

1. **Arrange** — Set up the preconditions (create objects, prepare input, configure mocks)
2. **Act** — Call the function or method under test
3. **Assert** — Verify the result matches expectations

Keep these phases visually separate. When a test fails, this structure makes it immediately clear what was set up, what was called, and what went wrong.

## What to Test — and What Not To

### Test the public interface

Write tests against public functions, exported methods, and API contracts — the interface your module/package/class exposes to consumers. This means:

- Tests break when **behavior** changes (a real problem you want to catch)
- Tests survive **refactoring** of internals (you can freely restructure private code)
- Tests document the module's API for other developers

### Do not test private/internal functions

Private or internal functions are implementation details. Testing them couples your tests to the internal structure and makes refactoring painful — exactly the opposite of what TDD should give you. If a private function has complex logic worth testing, test it indirectly through the public function that calls it. The public function's tests should exercise all the important paths through the private code.

## When Specs or Test Cases Are Provided

If the user provides specific requirements, acceptance criteria, or test cases — each one becomes a cycle. Work through them in order:

1. Pick the first spec/test case
2. Write a failing test for it (RED — Prove It)
3. Implement to pass (GREEN)
4. Refactor (REFACTOR)
5. Pick the next spec/test case
6. Repeat

Do not skip any. Each spec should have at least one corresponding test.

## Adapt to the Language and Project

Before writing the first test, look at the existing project to understand:

- **Test framework in use** — use whatever the project already uses (e.g., `testing` in Go, `pytest` in Python, `jest`/`vitest` in JS/TS, `JUnit` in Java). Don't introduce a new framework unless the project has none.
- **Test file conventions** — follow the project's naming and placement patterns (e.g., `_test.go` next to code in Go, `__tests__/` directories in JS, `test_*.py` in Python).
- **Test patterns** — if the project uses table-driven tests, parameterized tests, or a specific assertion library, match that style.
- **Test runner command** — use the project's existing test runner (e.g., `go test ./...`, `pytest`, `npm test`, `cargo test`).

If there are no existing tests, pick the idiomatic defaults for the language.

## Handling Dependencies

When the function under test depends on external services (database, HTTP APIs, other modules):

- **Define interfaces/abstractions** at the consumer side for the dependencies you need to mock
- **Use dependency injection** so tests can pass in fakes/mocks/stubs
- **Keep interfaces small** — accept the narrowest interface that satisfies the need
- **Use the project's existing mocking approach** — don't introduce a new mocking library if one is already in use

## Error Path Testing

Don't only test the happy path. For each function, consider:
- What happens with invalid input?
- What happens when a dependency fails?
- Are error messages/types correct?

These are separate cycles — write a failing test for the error case, then implement the error handling.

## Advisory Mode

TDD is the recommended approach for all implementation work. If the user explicitly asks to skip tests or implement without TDD, explain briefly why TDD would help for their specific case, then comply with their request. Don't block them — the goal is to make TDD the path of least resistance, not a gatekeeping exercise.

If a task is genuinely trivial (renaming a variable, fixing a typo, updating a constant), TDD overhead isn't justified. Use judgment — the cycle is for behavioral changes, not cosmetic ones.

## Checklist Before Moving to Next Task

Before declaring a requirement complete, verify:
- [ ] All tests pass
- [ ] No test was written for private/internal functions
- [ ] Tests use descriptive names (DAMP) and Arrange-Act-Assert structure
- [ ] Tests cover both happy path and error cases for the requirement
- [ ] Each spec/test case from the user has a corresponding test
- [ ] Refactoring pass is done (no obvious duplication or naming issues)