# Selenium → Playwright Migration Rules (Phase 1)

A structured, rule-based system for migrating Selenium + TestNG test suites to Playwright + Cucumber + TestNG — designed to be applied consistently by both engineers and AI-assisted tooling.

---

## The Problem

Migrating from Selenium to Playwright is not just a syntax swap. The real challenges are:

- Selenium workarounds (JS clicks, forced refreshes, retry loops) that shouldn't carry over
- Assertion behavior that lives inside utility wrappers, not in the test methods themselves
- TestNG session structure that doesn't map cleanly to Cucumber's scenario model
- Locator strategies that need translation without breaking the original user flow
- Disabled tests, unused locators, and commented code that pollute the migration scope

Without a clear ruleset, every engineer migrates differently. Reviews become inconsistent. Bugs get introduced silently.

---

## The Solution

A single markdown rules document that defines exactly how every pattern in a Selenium codebase maps to Playwright + Cucumber output — covering:

- Feature file structure and Gherkin tagging (from TestNG groups)
- Page class structure, locator declaration, and constructor injection via `hooks.screen`
- Step definition patterns
- Assertion vs non-assertive check translation (based on actual wrapper behavior, not method names)
- Session sharing strategy mirroring the TestNG class structure
- Disabled test handling
- Escalation protocol for ambiguous cases

---

## What Gets Generated Per Module

For each Selenium module, the rules produce exactly three output files:

| File | Purpose |
|---|---|
| `.feature` | Cucumber scenarios with tags from TestNG groups |
| `Page Class` | Playwright locators + UI actions + checks |
| `Step Definitions` | Glue between feature steps and page methods |

No hooks, no runners, no framework utilities — the existing project structure is assumed to be in place.

---

## Current Status

- Phase 1 rules: complete
- Validated against: 6 modules
- Confidence level: rules handle ~80% of migration patterns automatically for validated modules
- Remaining iteration estimate: 3–4 rule refinements to reach 80%+ automation across the full suite

---

## Stack

| Layer | Technology |
|---|---|
| Source | Selenium WebDriver + TestNG |
| Target (Browser) | Playwright (Java) |
| Target (BDD) | Cucumber |
| Target (Runner) | TestNG |

---

## How to Use

1. Read `selenium-to-playwright-migration-rules.md` fully before generating any output
2. Provide the rules document alongside the Selenium test class and Page Object to your engineer or AI tool
3. Generate the three output files per module
4. Flag any `// ESCALATION NEEDED` comments in the output for human review before merging

---

## Rule Highlights

- **Flow is sacred** — the Selenium test method body is the only source of truth. Nothing is inferred.
- **All locators are included** — the Page class mirrors the full Selenium Page Object, regardless of test state.
- **Wrappers are translated by behavior, not name** — `isVisible()` calls `Assert.fail()` internally; it is assertive. `checkInputBoxValue()` returns a value silently; it is not.
- **TestNG groups become Cucumber tags** — exact match, scenario-level, no normalization.
- **Disabled tests are ignored for scenarios and steps** — but their locators are still included.

---
