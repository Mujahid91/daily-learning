# Locators: XPath vs CSS selectors

**Date:** 2026-11-14
**Area:** Selenium / UI automation

## Key idea

A test is only as stable as its locators. Prefer locators that are tied to
meaning (id, name, `data-testid`, visible label) over locators tied to layout
(position in the DOM, long absolute paths, generated class names).

Priority I follow:

1. `id` or a dedicated test attribute (`data-testid`, `data-qa`)
2. `name`
3. Short CSS selector
4. XPath, when I need text or need to walk up the tree

## CSS selectors

Fast, readable, and supported everywhere.

| Goal | CSS |
|---|---|
| By id | `#login` |
| By class | `.btn.primary` |
| By attribute | `input[name='email']` |
| Attribute starts with | `a[href^='/products']` |
| Attribute contains | `div[class*='alert']` |
| Direct child | `form > button` |
| Nth child | `ul.menu > li:nth-child(3)` |

Limits: CSS **cannot** select by visible text and **cannot** go to a parent.

## XPath

More powerful, a bit harder to read.

| Goal | XPath |
|---|---|
| By exact text | `//button[text()='Sign in']` |
| Contains text | `//h2[contains(text(),'Order')]` |
| Normalize spaces | `//span[normalize-space()='Total']` |
| Parent | `//input[@id='email']/..` |
| Ancestor | `//td[text()='Alice']/ancestor::tr` |
| Following sibling | `//label[text()='Email']/following-sibling::input` |
| Combine conditions | `//input[@type='text' and @name='q']` |

Avoid absolute XPath like `/html/body/div[2]/div/form/input[1]`. It breaks on
any layout change.

## Example: the table-row problem

Click "Edit" in the row where the name is "Alice". CSS can't do this, XPath
can:

```java
By editForAlice = By.xpath(
    "//td[normalize-space()='Alice']/ancestor::tr//button[text()='Edit']"
);
driver.findElement(editForAlice).click();
```

A reusable version:

```java
private By editButtonForRow(String name) {
    return By.xpath(String.format(
        "//td[normalize-space()='%s']/ancestor::tr//button[text()='Edit']", name));
}
```

## Checking locators quickly

In Chrome DevTools console:

```js
$$("input[name='email']")            // CSS, returns all matches
$x("//button[text()='Sign in']")     // XPath, returns all matches
```

A good locator returns **exactly one** element.

## Takeaways

- Ask developers to add `data-testid` attributes; it's the cheapest stability win.
- Use CSS by default, XPath for text matching and moving up the tree.
- Never copy absolute XPath from DevTools into a test.
