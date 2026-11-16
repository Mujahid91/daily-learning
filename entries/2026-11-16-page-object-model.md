# Page Object Model: structure and common mistakes

**Date:** 2026-11-16
**Area:** Selenium / UI automation / test design

## Key idea

A Page Object is a class that represents **one page (or one component) of the
application** and exposes what a user can *do* there, not how the HTML is built.
Tests talk to page objects; only page objects talk to WebDriver.

When a locator changes, you fix it in exactly one place, and the tests keep
reading like a description of user behaviour.

## Responsibilities: who does what

| Layer | Knows about | Must not contain |
|---|---|---|
| Test class | Scenario, test data, assertions | Locators, `driver.findElement`, waits |
| Page object | Locators, waits, user actions on that page | Assertions about business rules, test data |
| Base page | Shared helpers (click, type, wait) | Page-specific locators |
| Component object | A reusable part (header, modal, table) | Navigation to unrelated pages |

## A clean example

```java
public class LoginPage extends BasePage {

    private final By username = By.id("username");
    private final By password = By.id("password");
    private final By submit   = By.cssSelector("button[type='submit']");
    private final By error    = By.cssSelector(".alert-error");

    public LoginPage(WebDriver driver) {
        super(driver);
    }

    public LoginPage open(String baseUrl) {
        driver.get(baseUrl + "/login");
        return this;
    }

    // Happy path returns the next page
    public DashboardPage loginAs(String user, String pass) {
        type(username, user);
        type(password, pass);
        click(submit);
        return new DashboardPage(driver);
    }

    // Negative path stays on the same page
    public LoginPage loginExpectingError(String user, String pass) {
        type(username, user);
        type(password, pass);
        click(submit);
        return this;
    }

    public String errorMessage() {
        return textOf(error);
    }
}
```

`BasePage` (from yesterday's note on waits) provides `click`, `type` and
`textOf` with explicit waits built in.

The test then reads like a scenario:

```java
public class LoginTest extends BaseTest {

    @Test
    public void validUserSeesDashboard() {
        DashboardPage dashboard = new LoginPage(driver)
                .open(baseUrl)
                .loginAs("qa_user", "Secret123!");

        Assert.assertEquals(dashboard.welcomeText(), "Welcome, QA User");
    }

    @Test
    public void wrongPasswordShowsError() {
        String error = new LoginPage(driver)
                .open(baseUrl)
                .loginExpectingError("qa_user", "wrong")
                .errorMessage();

        Assert.assertEquals(error, "Invalid username or password");
    }
}
```

## Component objects for repeated UI

Headers, side menus, modals and data tables appear on many pages. Model them once
and compose them into pages instead of copying locators:

```java
public class HeaderComponent extends BasePage {
    private final By userMenu = By.cssSelector("[data-test='user-menu']");
    private final By logout   = By.cssSelector("[data-test='logout']");

    public HeaderComponent(WebDriver driver) { super(driver); }

    public LoginPage logout() {
        click(userMenu);
        click(logout);
        return new LoginPage(driver);
    }
}

public class DashboardPage extends BasePage {
    private final By welcome = By.cssSelector("h1.welcome");
    public final HeaderComponent header;

    public DashboardPage(WebDriver driver) {
        super(driver);
        this.header = new HeaderComponent(driver);
    }

    public String welcomeText() { return textOf(welcome); }
}
```

## Common mistakes

| Mistake | Why it hurts | Fix |
|---|---|---|
| Assertions inside page objects | Page can't be reused for negative tests | Return data (`errorMessage()`), assert in the test |
| Exposing `WebElement` or `By` publicly | Tests start doing `findElement` again | Expose actions and values, keep locators `private` |
| One giant "God page" for the whole app | Hard to navigate, merge conflicts | One class per page, components for shared parts |
| Methods named after HTML (`clickButton2`) | Tests read like DOM scripts | Name by intent: `submitOrder()`, `searchFor(text)` |
| `Thread.sleep` in actions | Slow and still flaky | Explicit waits in the base page |
| Static `WebDriver` shared everywhere | Breaks parallel runs | Pass the driver in the constructor (or `ThreadLocal`) |
| Test data hardcoded in pages | Can't run other datasets | Pass data as method parameters |
| Returning `void` from every method | No fluent flow, unclear navigation | Return the page the user lands on |

## Takeaways

- Page objects model user actions; tests hold the scenario and the assertions.
- Keep locators private and name methods by intent, not by HTML.
- Return the next page object to make navigation explicit and the test fluent.
- Extract headers, modals and tables into component objects.
- Inject the driver through the constructor so the suite can run in parallel.
