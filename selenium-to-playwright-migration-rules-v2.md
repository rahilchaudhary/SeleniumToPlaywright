# Selenium → Playwright Migration Rules (Phase 1)

---

## How to Use This Document

- Rules are numbered for reference, not priority order.
- **Priority order is defined explicitly in each section where conflicts can arise.**
- When two rules appear to conflict, the one with the higher priority wins. If still unclear → escalate (see Rule 20).
- This document is the source of truth for both human engineers and AI-assisted migration.

---

## Rule Priority (Global)

When rules conflict, resolve in this order:

1. Preserve user flow (Rule 1)
2. Preserve validations (Rule 2)
3. Mirror Selenium intent — do not add, infer, or remove (Rules 15, 19)
4. Everything else

---

## Rule 1 — Preserve User Flow (Highest Priority)

- Follow the exact sequence of actions in the Selenium test method.
- Do NOT skip UI actions (e.g., clicking menu items, navigating intermediate pages).
- Direct URL navigation is NOT equivalent to a user clicking through the UI to reach a page.

---

## Rule 2 — Preserve All Validations

- Every validation in the Selenium test must have a Playwright equivalent.
- Example: if Selenium validates "School Admin" then "Student Details", both must appear in Playwright.
- Simplification that removes a validation step is not allowed.

---

## Rule 3 — Locator Strategy

**Tiebreaker with Rule 10:** Prefer the locator that most closely matches what Selenium used. Do not over-engineer. Do not default to `getByText().first()` just because it is easy if a better option is obvious.

**Preference order (apply in sequence, stop at first viable option):**

1. Role-based locators (`getByRole`) — use when the element has a clear semantic role
2. Scoped locators (within a container or section)
3. Tag-specific locators (`h1`, `button`, etc.)
4. `getByText().first()` — acceptable in Phase 1 when other options require significant effort or risk

**Avoid:**
- Ambiguous locators in critical flows (e.g., a `getByText` that could match multiple elements)

**Goal:** One locator resolves to exactly one element.

**Do NOT:**
- Use unused locators from the Selenium Page Object
- Infer locators from commented-out Selenium code

---

## Rule 4 — Navigation Handling

Replicate the Selenium sequence exactly:

```
navigate → validate → click → land on next page
```

Do NOT reduce to:

```
navigate → validate only
```

If Selenium navigated to a page by clicking through the UI, Playwright must do the same — not by constructing a direct URL.

---

## Rule 5 — Retry Behavior

Playwright with Cucumber + TestNG does NOT automatically retry navigation.

**Add a retry ONLY when both conditions are true:**
- The first page load fails in Playwright but the second succeeds reliably (observed across multiple runs)
- The failure is confirmed as an application/timing issue, not a locator or flow problem

**Retry implementation:** Maximum 1 retry. Validate success using a real UI element, not just `waitForLoadState`.

**Do NOT:**
- Add global retry
- Add retry as a precaution for slow elements
- Add retry because Selenium had `retryClick` (see Rule 8)

---

## Rule 6 — Handling Flaky Page Loads

If a page load is genuinely flaky (first attempt fails, second consistently succeeds):

- Add a single retry (max 1)
- Validate the page loaded using a real UI element that is specific to that page
- Do NOT rely on `waitForLoadState('networkidle')` alone as the success check

---

## Rule 7 — Feature File and Session Structure

**Core principle:** Mirror the TestNG test structure in Cucumber. Do not force Cucumber conventions onto a TestNG-shaped test.

**Specific guidance:**

- If a TestNG `@BeforeClass` or first `@Test` method navigates to a page URL, that navigation belongs in the **first scenario** of the feature file, not in a `Background` block.
- Subsequent scenarios may reuse the same browser session and assume the state from the prior scenario — only if that is how the TestNG class is structured.
- Do NOT add a `Background` step with heavy navigation just to satisfy Gherkin conventions, unless the equivalent setup runs before every TestNG test method.

**Known trade-off:** This approach is not fully Cucumber-compliant (scenarios are not isolated). This is acceptable for Phase 1. Refactoring for isolation is a Phase 2 task.

---

## Rule 8 — Selenium JS-Based Actions

Selenium wrappers like `clickWithJs`, `retryClick`, and `refresh` were workarounds for Selenium-specific timing issues. Playwright handles most waits automatically.

**Translation rule:**
- `clickWithJs` → `locator.click()` (unless the page has a popup or overlay that blocks normal click)
- `retryClick` → `locator.click()` unless there is confirmed evidence of flakiness in Playwright
- `refresh` → only add if the Selenium flow explicitly depends on a page refresh as a step in the test, not as a workaround

**Do NOT** carry over Selenium workarounds speculatively.

---

## Rule 9 — Assertions and WebUtil Wrapper Translation

The assertion behavior of a step is determined by what the `WebUtil` wrapper actually does internally — not by its name. Check the table below before deciding how to translate.

**WebUtil wrappers that ARE assertive** (call `Assert.fail()` internally):

| Selenium wrapper | Why it is assertive | Playwright translation |
|---|---|---|
| `isVisible(element)` | Calls `Assert.fail()` on `NoSuchElementException` | Page class method using `locator.waitFor()` — failure is implicit if element not found |
| `click(element)` | Calls `Assert.fail()` on exception | `locator.click()` — Playwright throws on failure naturally |
| `clickWithJs(element)` | Calls `Assert.fail()` on exception | `locator.click()` — see also Rule 8 |

For these, Playwright's own exception on failure is the equivalent behavior. Do NOT add an explicit `Assert` call — Playwright already fails the test if the locator is not found or not clickable.

**WebUtil wrappers that are NOT assertive** (return a value or swallow exceptions):

| Selenium wrapper | Why it is non-assertive | Playwright translation |
|---|---|---|
| `checkInputBoxValue(element)` | Returns the `value` attribute, no assert | Page class method: `locator.inputValue()` |
| `getDisplayedText(element)` | Returns `getText()`, no assert | Page class method: `locator.textContent()` or `locator.innerText()` |
| `enterText(element, text)` | Swallows exceptions, no assert | Page class method: `locator.fill(text)` |

For these, translate as Page class methods that return a value. Do NOT add assertions unless Selenium explicitly asserted on the returned value in the test method.

**WebUtil wrappers that are workarounds — do not carry over:**

| Selenium wrapper | Reason | Rule |
|---|---|---|
| `retryClick(element)` | Retries with refresh on failure | Rule 8 |
| `retryIsVisible(element)` | Retries with refresh on stale element | Rule 8 |
| `ifVisible(element)` | Conditional click — defensive pattern | Rule 15 |
| `ifVisibleSimpleClick(element)` | Conditional click — defensive pattern | Rule 15 |
| `waitForVisibility(element)` | Refreshes on `StaleElementReferenceException` | Rule 8 |

**If Selenium uses explicit TestNG assertions directly in the test method** (e.g., `Assert.assertEquals`, `Assert.assertTrue`):
- Implement the equivalent assertion in the Page class method that is called from that step.

**`waitFor()`** in Playwright confirms element presence. It is not an assertion — do not treat it as one unless Selenium explicitly asserted on the result.

---

## Rule 10 — Keep It Simple (Phase 1)

- No framework redesign
- No new abstraction layers
- No wrapper utility classes
- No over-engineered locator strategies

**Priority:** Working > Clean. Accurate > Elegant.

**Tiebreaker with Rule 3:** When locator quality and simplicity conflict, choose the locator closest to what Selenium used. If Selenium used a label-based lookup, use `getByLabel`. If it used text, use `getByText`. Do not upgrade to a role-based locator unless it is obviously correct and low-risk.

---

## Rule 11 — Known Patterns in This Codebase

Be aware of these recurring issues:

- Navigation gaps are common — Selenium often navigated in setup methods that do not map 1:1 to Cucumber steps
- Locator ambiguity is common — always verify a locator resolves to exactly one element
- Retry needs are product-specific, not tool-specific — do not assume Playwright needs the same retries Selenium used
- Playwright is NOT a drop-in replacement — flow and timing assumptions from Selenium do not automatically carry over

---

## Rule 12 — Translate Intent (Use Carefully)

Only translate steps that are:
- Explicitly executed in the Selenium test method
- Part of a validation that is explicitly called

**Do NOT infer intent from:**
- Unused locators in the Page Object
- Commented-out code
- Assumptions about how the UI probably works

**Simplification is allowed only when:**
- The original code was a workaround (e.g., JS click, forced refresh)
- Removing it does not change the user flow
- Removing it does not remove a validation

**If unsure whether something should be included → do not include it. Flag it for human review (see Rule 20).**

---

## Rule 13 — Output Scope

For each module, generate ONLY:

- Feature file (`.feature`)
- Step definition class
- Page class

Do NOT generate:
- Hooks (`@Before`, `@After`)
- Runner file changes
- Framework-level utilities or base classes

Assume the project structure and framework wiring already exist.

---

## Rule 14 — Actions in Page Class

Write all UI actions directly in the Page class:

```java
locator.fill("value");
locator.click();
locator.press("Enter");
```

Do NOT create wrapper utility classes (e.g., `ActionHelper`, `WebUtils`).

**Assertions and non-assertive checks also belong in the Page class** (called from Step Definitions). See Rule 21 for the full Page class structure.

---

## Rule 15 — Flow Source of Truth (Critical)

The Selenium test method body defines the flow. Nothing else.

**Use a locator or step ONLY if:**
- It is explicitly called in an enabled Selenium test method, OR
- It is part of an explicit validation in an enabled test method

**Ignore:**
- Unused Page Object locators
- Commented-out code
- Defensive patterns like `if (element.isDisplayed()) { click() }` — unless Selenium explicitly did this

**If unsure whether a step is required → do not include it. Flag it (see Rule 20).**

---

## Rule 16 — Test Data in Feature Files

Steps involving input or search must clearly represent either the exact value or the source of data.

**Case 1 — Static data (value is fixed in the test):**

The Selenium test hardcodes the value:
```java
enterText(searchBar, "STU001");
```

Feature step — use a string parameter:
```gherkin
When user searches student with code "STU001"
```

Step definition — use `{string}`:
```java
@When("user searches student with code {string}")
public void searchStudent(String code) { ... }
```

**Case 2 — UI-derived data (value comes from the application):**

The Selenium test reads a value from the UI, then uses it:
```java
String name = checkInputBoxValue(firstNameField);
enterText(searchBar, name);
```

Feature step — do NOT use `{string}`. Express the intent:
```gherkin
When user searches student using existing student name
```

The step definition calls the Page class method that reads and uses the value — no parameter is passed from the feature file.

**Rule summary:**
- Use `{string}` only when the test explicitly controls the data value
- Never expose UI-derived values as feature file parameters
- Never use vague steps like "searches by name" — always clarify whether the value is static or application-derived

---

## Rule 17 — Assertion Consistency

This rule consolidates and supersedes the assertion guidance split across earlier rules.

**Source of truth:** If Selenium does not explicitly assert, Playwright does not assert.

| Selenium pattern | Playwright translation |
|---|---|
| `Assert.assertEquals(...)` | Equivalent assertion in Page class |
| `Assert.assertTrue(isVisible(...))` | Equivalent assertion in Page class |
| `isVisible()` without assert | Page class method, no assertion |
| `checkInputBoxValue()` without assert | Page class method, returns/checks value |
| `getDisplayedText()` without assert | Page class method, returns value |

Do NOT convert non-assertive checks into `assertTrue`, `assertNotNull`, or Playwright's `expect(...).toBeVisible()` unless Selenium explicitly asserted.

---

## Rule 18 — Feature File Structure

Feature files must reflect the Selenium test flow. Gherkin formatting is secondary.

**Acceptable in Phase 1:**
- A scenario that ends with a `When` step (no `Then`)
- A scenario that contains only `Then` steps
- A scenario that does not follow `Given → When → Then` strictly

**Do NOT:**
- Add steps to satisfy Gherkin structure if they do not exist in Selenium
- Add `Given user is on the page` unless Selenium explicitly navigates there as a test step
- Introduce validations that do not appear in the Selenium test

**Priority:** Behavioral accuracy > Gherkin compliance.

Gherkin normalization is a Phase 2 task.

---

**TestNG Groups → Cucumber Scenario Tags**

TestNG `groups` on a `@Test` method map directly to Cucumber tags on the corresponding scenario.

- If a TestNG test has one or more groups → add one `@tag` per group on the scenario
- If a TestNG test has no groups → do NOT add any tag to the scenario

**Examples:**

```java
// Selenium
@Test(groups = {"sanity", "regression"}, priority = 1)
public void studentListClickable() { ... }

@Test(groups = "regression", priority = 7)
public void studentEdit() { ... }

@Test(priority = 4)
public void studentDetails() { ... }
```

```gherkin
# Cucumber
@sanity @regression
Scenario: Student list page is accessible
  ...

@regression
Scenario: Edit student is clickable
  ...

Scenario: Student details fields are present
  ...
```

**Rules:**
- Tag names must match the TestNG group strings exactly — do not rename or normalize them
- Tags go on the scenario line, not on the Feature line
- Disabled tests (`enabled = false`) are ignored entirely — no tag, no scenario (Rule 22)

---

## Rule 19 — No Assumptions (Strict)

Do NOT assume:
- UI behavior not shown in the Selenium test
- What a locator does based on its name alone
- Navigation steps between two explicit steps
- Test data values not present in Selenium
- Expected outcomes not validated in Selenium

If something is not in the Selenium test method body → do not add it.

If unsure → skip it and flag for human review (see Rule 20).

---

## Rule 20 — Escalation Protocol (AI and Human Engineers)

When a situation is ambiguous and these rules do not resolve it clearly, do not guess.

**Escalate when:**
- A Selenium step's intent cannot be determined from the test method alone
- A locator from Selenium cannot be matched to a Playwright strategy with reasonable confidence
- Session/navigation structure does not map clearly to Cucumber scenarios
- A rule conflict cannot be resolved using the global priority order (top of this document)
- Test data source (static vs UI-derived) is unclear

**How to escalate:**
- Flag the specific step or locator with a comment: `// ESCALATION NEEDED: [reason]`
- Generate the rest of the file as completely as possible
- Do NOT guess or fill in the uncertain part with an assumption
- Do NOT silently skip the step — leave a visible marker

**For AI-assisted migration specifically:** Produce the output, mark all uncertain areas clearly, and surface them together at the end of the response so the human reviewer can resolve them in one pass.

---

## Rule 21 — Page Class Structure

Every Page class must follow this structure exactly. No deviations in Phase 1.

### 21.1 — Constructor and Page Injection

The Playwright `Page` object is supplied via `hooks.screen` — a static reference held by the project's Hooks class. The Page class constructor accepts a `Page` argument. Step definitions instantiate the Page class in their own no-arg constructor by passing `hooks.screen` directly.

**Page class constructor (mandatory pattern):**
```java
public class StudentListPage {

    Page screen;
    Locator searchBar;
    Locator stuList_pageCheck;

    public StudentListPage(Page page) {
        this.screen = page;
        searchBar = screen.locator("//input[contains(@placeholder,'Search')]");
        stuList_pageCheck = screen.locator("//h1[normalize-space()='Student Details']");
    }
}
```

**Step definition instantiation (mandatory pattern):**
```java
public class studentListSteps {

    public StudentListPage obj_studentListPage;

    public studentListSteps() {
        obj_studentListPage = new StudentListPage(hooks.screen);
    }
}
```

**Forbidden patterns:**
```java
// Do NOT do any of these inside a Page class:
Page page = BaseTest.getPage();
Page page = PlaywrightFactory.getPage();

// Do NOT inject Page via DI framework or constructor parameter in step definitions:
public studentListSteps(Page page) { ... }  // wrong
```

### 21.2 — Naming Convention

The Playwright `Page` reference in the Page class must always be named `screen`.

Do NOT use: `page`, `driver`, `browserPage`, or any other name.

The Page class instance in step definitions must follow the pattern `obj_<PageClassName>` (e.g., `obj_studentListPage`).

### 21.3 — Locator Declaration

All locators must be declared as class-level fields.

```java
Locator searchBar;
Locator stuList_pageCheck;
```

Inline locator creation inside methods is forbidden:

```java
// Forbidden
public void clickSubmit() {
    screen.locator("#submit-btn").click();
}
```

### 21.4 — Locator Initialization

All locators must be initialized inside the constructor only. No lazy initialization, no method-level initialization.

```java
public StudentListPage(Page page) {
    this.screen = page;
    searchBar = screen.locator("//input[contains(@placeholder,'Search')]");
    stuList_pageCheck = screen.locator("//h1[normalize-space()='Student Details']");
}
```

### 21.5 — What Belongs in the Page Class

The Page class contains:
- Locator field declarations (all locators from Selenium Page Object — see Rule 22)
- Constructor (accepting `Page page`, storing as `screen`, initializing all locators)
- UI action methods (`locator.click()`, `locator.fill()`, `locator.press()`, etc.)
- Non-assertive check methods (translated from Selenium's `checkInputBoxValue`, `getDisplayedText`, etc.)
- Assertive check methods (translated from Selenium's `isVisible` and any explicit `Assert.*` calls)

The Page class does NOT contain:
- Step definition logic (`@Given`, `@When`, `@Then`)
- Test data generation or management
- References to `hooks` or any framework wiring
- Logging frameworks

### 21.6 — Step Definition Structure

Step definitions follow this exact structure:

```java
import webPages.Playwright.Nucleus.StudentListPage;
import webTests_Cucumber_Playwright.Hooks.hooks;
import static util.reportutil.ReportManager_Cucumber.*;

public class studentListSteps {

    public StudentListPage obj_studentListPage;

    public studentListSteps() {
        obj_studentListPage = new StudentListPage(hooks.screen);
        report_SetUp();
    }

    @Given("User navigates to Student List Page")
    public void navigate_student_list() {
        obj_studentListPage.navigateToStudentList(studentListUrl);
    }
}
```

Step definitions do NOT:
- Contain Playwright locator calls directly
- Instantiate `Page` themselves
- Contain any UI interaction logic — all of that is in the Page class method being called

---

## Rule 22 — Disabled Tests

Any Selenium test annotated with `enabled = false` must be completely ignored for scenario and step generation.

Do NOT generate:
- Feature scenarios
- Step definitions
- Page class methods that serve no enabled test

Treat disabled tests as if they do not exist **for flow and step purposes only**.

**Locator rule — include ALL locators from the Selenium Page Object, regardless of test state:**
- Locators used only in disabled tests → still include in the Page class
- Locators not called by any test at all → still include in the Page class
- The Page class mirrors the Selenium Page Object completely

**Reason:** The Page class represents the UI, not the current test coverage. A locator excluded today because its test is disabled will be missing when that test is re-enabled. Including all locators upfront prevents reconstruction work and eliminates locator-related escalations.

This rule overrides all other rules for scenario/step generation. Even if a disabled test has complete, valid flow — ignore it for feature and step output. Never ignore it for locator output.

---

## Appendix — Common Mistakes Reference

| Mistake | Correct Approach |
|---|---|
| Navigating directly to a URL instead of clicking through the UI | Follow the Selenium click sequence |
| Adding `Background` navigation that only runs in one TestNG method | Put it in the first scenario only |
| Converting `isVisible()` into `assertTrue(isVisible())` | Keep it as a non-assertive Page class method |
| Using a locator from the Selenium Page Object that no test calls | Include it anyway — Page class mirrors the full Page Object (Rule 22) |
| Excluding a locator because its only test is disabled | Include it anyway — locator exclusion based on test state is forbidden (Rule 22) |
| Adding a Gherkin `Given` step that does not exist in Selenium | Do not add it — accuracy over Gherkin structure |
| Using `{string}` in a step when the value comes from the UI | Use intent-based step wording, no parameter |
| Carrying over `retryClick` from Selenium as a Playwright retry | Use `locator.click()` directly; only add retry if Playwright itself fails |
| Guessing when a flow step is unclear | Flag with `// ESCALATION NEEDED` and surface to reviewer |
| Creating `BaseHelper` or `ActionUtil` wrapper classes | Write actions directly in Page class |
| Naming the Page reference anything other than `screen` | Always use `screen` |
