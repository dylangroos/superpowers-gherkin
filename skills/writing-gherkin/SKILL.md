---
name: writing-gherkin
description: Use when writing acceptance criteria in a spec or turning them into tests in a plan - defines the scenario style, the runner detection rule, and the two TDD task templates
---

# Writing Gherkin

## Overview

Acceptance criteria written as prose are not countable. "The exporter should handle empty results gracefully" cannot be checked off, cannot be mapped to a task, and cannot be argued about precisely. The same criterion written as a scenario can be.

This skill is the shared vocabulary between two others:

- **superpowers:brainstorming** writes scenarios into the spec.
- **superpowers:writing-plans** turns each scenario into a task with a test.

Neither skill invokes the other through this one. Both read this file for the rules below.

**Announce at start:** "I'm using the writing-gherkin skill to write the acceptance criteria."

## Scenario Style

**One behavior per scenario.** If a scenario needs a second `When`, it is two scenarios.

**Declarative, not imperative.** Steps describe intent, not mechanics. The mechanics change; the intent is the requirement.

```gherkin
# Bad - describes the UI, breaks when the UI moves
When the user clicks the "Export" button in the toolbar
And selects "CSV" from the dropdown
And clicks "Confirm"

# Good - describes the intent, survives a redesign
When the user exports the results as CSV
```

**Unique scenario names across the whole spec.** The plan refers to scenarios by name. Two scenarios called "Invalid input" make the plan ambiguous.

**Concrete values in the Then.** `Then the file has a header row and no data rows` is checkable. `Then the export behaves correctly` is not.

**Background only for setup every scenario in the feature shares.** If two of five scenarios share setup, repeat it. A Background that half the scenarios ignore is a lie about the fixture.

**Scenario Outline when the behavior is one rule over a table of values.** Not as a way to cram unrelated cases together.

```gherkin
Scenario Outline: Row limit rejects oversized exports
  Given a query returns <rows> rows
  When the user exports the results as CSV
  Then the export <outcome>

  Examples:
    | rows      | outcome              |
    | 999999    | succeeds             |
    | 1000000   | succeeds             |
    | 1000001   | fails with E_TOO_BIG |
```

## What Does Not Become a Scenario

Scenarios cover observable behavior. They are not the whole spec. Keep these as prose:

- Architecture and component boundaries
- Data flow and storage decisions
- Performance and scale targets that are not per-request assertions
- Rationale, alternatives considered, and trade-offs

A spec that is nothing but Gherkin has lost the reasoning. A spec with no Gherkin has lost the acceptance criteria. Write both.

## Runner Detection

Run this before writing plan tasks. The answer changes the task template.

Search the project's dependency manifests and test tree for a BDD runner that is **already installed**:

| Ecosystem | Look for | In |
|---|---|---|
| Python | `pytest-bdd`, `behave`, `radish-bdd` | `pyproject.toml`, `requirements*.txt`, `setup.cfg`, `uv.lock` |
| JS / TS | `@cucumber/cucumber`, `cucumber-js`, `jest-cucumber`, `@amiceli/vitest-cucumber` | `package.json` |
| Go | `github.com/cucumber/godog` | `go.mod` |
| Ruby | `cucumber` | `Gemfile` |
| Java / Kotlin | `io.cucumber` | `pom.xml`, `build.gradle*` |
| .NET | `Reqnroll`, `SpecFlow` | `*.csproj` |
| Rust | `cucumber` | `Cargo.toml` |

Also glob for `**/*.feature`. Existing feature files mean a runner exists even if the dependency name is unfamiliar - find how they run before assuming.

<HARD-RULE>
Use a BDD runner ONLY if the project already depends on one. NEVER add a
BDD runner, a step-definition library, or a `features/` directory to a
project that does not have one. Adding a test framework is an
architectural change that needs its own brainstorming and its own
approval - it is not a side effect of writing acceptance criteria.
</HARD-RULE>

The fallback is not a lesser option. A unit test named after the scenario carries the same traceability with none of the new dependency.

State the result explicitly in the plan header:

```markdown
**Runner:** pytest-bdd (found in pyproject.toml:42)
```

```markdown
**Runner:** none - plain unit tests named after scenarios
```

## Template A: Runner Present

The feature file is the test. Red/green runs against it.

````markdown
### Task 3: CSV export row limit

**Satisfies:** "Empty result set", "Row limit rejects oversized exports"

**Files:**
- Create: `features/csv_export.feature`
- Create: `tests/steps/test_csv_export.py`
- Modify: `src/export/csv.py`

- [ ] **Step 1: Write the failing scenario**

```gherkin
# features/csv_export.feature
Feature: CSV export

  Scenario: Empty result set
    Given a query returns 0 rows
    When the user exports the results as CSV
    Then the file has a header row and no data rows
```

- [ ] **Step 2: Write the step definitions**

```python
# tests/steps/test_csv_export.py
from pytest_bdd import scenarios, given, when, then

scenarios("../../features/csv_export.feature")

@given("a query returns 0 rows", target_fixture="rows")
def _():
    return []

@when("the user exports the results as CSV", target_fixture="output")
def _(rows):
    return export_csv(rows, columns=["id", "name"])

@then("the file has a header row and no data rows")
def _(output):
    assert output.splitlines() == ["id,name"]
```

- [ ] **Step 3: Run to verify it fails**

Run: `pytest tests/steps/test_csv_export.py -v`
Expected: FAIL with `NameError: name 'export_csv' is not defined`
````

Steps 4-6 follow the standard implement / verify green / commit cycle.

## Template B: No Runner

The scenario lives in the spec. The test name carries it into the code.

````markdown
### Task 3: CSV export row limit

**Satisfies:** "Empty result set", "Row limit rejects oversized exports"

**Files:**
- Create: `tests/export/test_csv.py`
- Modify: `src/export/csv.py`

- [ ] **Step 1: Write the failing test**

```python
# Scenario: Empty result set
def test_empty_result_set_writes_header_only():
    # Given a query returns 0 rows
    rows = []
    # When the user exports the results as CSV
    output = export_csv(rows, columns=["id", "name"])
    # Then the file has a header row and no data rows
    assert output.splitlines() == ["id,name"]
```

- [ ] **Step 2: Run to verify it fails**

Run: `pytest tests/export/test_csv.py::test_empty_result_set_writes_header_only -v`
Expected: FAIL with `NameError: name 'export_csv' is not defined`
````

Steps 3-5 follow the standard implement / verify green / commit cycle.

The test name is the scenario name in snake case. The Given/When/Then comments are not decoration - they are how a reviewer checks the test against the spec without a runner to do it for them.

## Red Flags

| Thought | Reality |
|---------|---------|
| "I'll add pytest-bdd, it's a small dependency" | Adding a test framework is architectural. Use Template B. |
| "This scenario needs a second When" | That is two scenarios. Split it. |
| "The steps should say which button to click" | Imperative steps break on redesign. Describe intent. |
| "Two scenarios can share a name, the feature differs" | The plan refers to scenarios by name. Make them unique. |
| "I'll write the whole spec in Gherkin" | Scenarios are the acceptance criteria, not the architecture. Write both. |
| "There's a features/ dir but no dependency I recognize" | A runner exists. Find how they run it before falling back. |
| "The scenario is obvious, I'll skip the Then" | A scenario with no assertion cannot fail. It is not a criterion. |
