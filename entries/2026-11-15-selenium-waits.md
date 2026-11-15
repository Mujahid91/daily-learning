# Explicit, implicit and fluent waits in Selenium

**Date:** 2026-11-15
**Area:** Selenium / UI automation

## Key idea

Most "flaky" UI tests are timing problems, not product bugs. The page is still
loading, an element is in the DOM but hidden, or a spinner covers the button.
The fix is to **wait for a condition**, never for a fixed amount of time.

Rule I follow: **implicit wait = 0, explicit waits everywhere they are needed,
`Thread.sleep()` never.**

## The three kinds of waits

| Wait | Scope | Waits for | Typical use |
|---|---|---|---|
| Implicit | Whole driver session | Element to be *present* in the DOM | Legacy suites; better left at 0 |
| Explicit (`WebDriverWait`) | One call | Any condition (visible, clickable, text, URL...) | Default choice |
| Fluent (`FluentWait`) | One call | Any condition, with custom polling and ignored exceptions | Slow or unusual conditions |

`WebDriverWait` is actually a `FluentWait<WebDriver>` with sensible defaults
(poll every 500 ms, ignore `NotFoundException`).

## Implicit wait

```java
driver.manage().timeouts().implicitlyWait(Duration.ofSeconds(10));
```

Every `findElement` now retries for up to 10 s. Problems:

- It only checks **presence**, not visibility or clickability.
- It slows down negative checks: "element should not exist" takes the full 10 s.
- **Do not mix it with explicit waits.** The two timeouts can stack in
  unpredictable ways (a 10 s implicit + 15 s explicit can wait much longer than
  15 s), and the Selenium docs warn against it.

## Explicit wait

```java
WebDriverWait wait = new WebDriverWait(driver, Duration.ofSeconds(10));

WebElement submit = wait.until(
        ExpectedConditions.elementToBeClickable(By.id("submit")));
submit.click();
```

Useful `ExpectedConditions`:

| Condition | When |
|---|---|
| `visibilityOfElementLocated(by)` | Before reading text or asserting it's shown |
| `elementToBeClickable(by)` | Before clicking |
| `invisibilityOfElementLocated(by)` | Wait for a spinner/overlay to disappear |
| `textToBePresentInElementLocated(by, text)` | Async content updates |
| `urlContains(fragment)` | After navigation or redirect |
| `numberOfElementsToBe(by, n)` | Table or list finished loading |
| `frameToBeAvailableAndSwitchToIt(by)` | Iframes that load late |
| `alertIsPresent()` | JS alerts that appear after an action |
| `stalenessOf(element)` | Old element replaced after a re-render |

## Fluent wait

Use it when you need a different polling interval or must ignore extra
exceptions:

```java
Wait<WebDriver> wait = new FluentWait<>(driver)
        .withTimeout(Duration.ofSeconds(30))
        .pollingEvery(Duration.ofSeconds(2))
        .ignoring(NoSuchElementException.class)
        .ignoring(StaleElementReferenceException.class)
        .withMessage("Report status never became COMPLETED");

wait.until(d -> d.findElement(By.id("report-status"))
                 .getText().equals("COMPLETED"));
```

The lambda returns `true` / a non-null object when done. `withMessage` makes
the `TimeoutException` readable in the test report.

## Putting it in a base page

Keep waits in one place so page objects stay clean:

```java
public abstract class BasePage {
    protected final WebDriver driver;
    protected final WebDriverWait wait;

    protected BasePage(WebDriver driver) {
        this.driver = driver;
        this.wait = new WebDriverWait(driver, Duration.ofSeconds(10));
    }

    protected void click(By locator) {
        wait.until(ExpectedConditions.invisibilityOfElementLocated(By.cssSelector(".spinner")));
        wait.until(ExpectedConditions.elementToBeClickable(locator)).click();
    }

    protected void type(By locator, String text) {
        WebElement field = wait.until(ExpectedConditions.visibilityOfElementLocated(locator));
        field.clear();
        field.sendKeys(text);
    }

    protected String textOf(By locator) {
        return wait.until(ExpectedConditions.visibilityOfElementLocated(locator)).getText();
    }
}
```

## Why not `Thread.sleep()`?

- Too short on a slow CI agent: the test still fails.
- Too long on a fast machine: wasted time multiplied by hundreds of tests.
- It hides the real condition the test depends on.

## Takeaways

- Keep implicit wait at 0 and use explicit waits; never mix the two.
- Wait for the right condition: visible to read, clickable to click, invisible for spinners.
- Use `FluentWait` for long-running or polling scenarios, with a clear `withMessage`.
- Centralize waits in a base page so tests don't repeat wait code.
- `Thread.sleep()` in a test is a bug waiting to happen.
