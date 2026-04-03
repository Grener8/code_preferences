# `scrapy-pydoll`: Feature Specification

## Objective

Implement a Scrapy plugin that integrates **Pydoll** as an optional rendering backend. Requests can opt-in to browser automation while preserving full Scrapy APIs (`.css()`, `.xpath()`, middleware compatibility, retries, cookies).

---

## Runtime Requirements

Scrapy MUST run with asyncio reactor:

```python
TWISTED_REACTOR = "twisted.internet.asyncioreactor.AsyncioSelectorReactor"
```

Pydoll is asyncio-native and will not be wrapped in threads.

---

## Core Architecture

### Browser Model

```text
- 1 browser instance per crawler
- Tabs are created via browser.start()
- No native context abstraction → session handled via cookies
```

---

### Session Model (Critical)

Since Pydoll lacks contexts:

```text
- cookiejar → logical session
- session state persisted via cookies
- tabs are ephemeral
```

Flow:

1. Create new tab
2. Inject cookies from Scrapy cookiejar
3. Execute actions
4. Extract cookies
5. Update Scrapy cookiejar
6. Close tab

---

### Concurrency Model

Controlled via:

```python
asyncio.Semaphore(PYDOLL_CONCURRENCY)
```

* Limits number of **active tabs**
* Independent from Scrapy concurrency

---

## Settings

```python
PYDOLL_ENABLED = True
PYDOLL_CONCURRENCY = 4
PYDOLL_BROWSER_OPTIONS = {}
PYDOLL_DEFAULT_TIMEOUT = 30000  # ms
PYDOLL_ENABLE_CLOUDFLARE_SOLVER = True
```

---

## Downloader Middleware

### Activation

Intercept when:

```python
if request.meta.get("pydoll"):
```

Otherwise pass through unchanged.

---

### Execution Flow

For Pydoll requests:

1. Acquire semaphore
2. Create tab:

   ```python
   tab = await browser.start()
   ```
3. Inject cookies:

   ```python
   await tab.set_cookies(scrapy_cookies)
   ```
4. Navigate:

   ```python
   await tab.go_to(url)
   ```
5. Optional:

   ```python
   await tab.enable_auto_solve_cloudflare_captcha()
   ```
6. Execute actions
7. Extract HTML:

   ```python
   html = await tab.content()
   ```
8. Extract cookies:

   ```python
   cookies = await tab.get_cookies()
   ```
9. Close tab:

   ```python
   await tab.close()
   ```
10. Release semaphore

---

## Request API

### Meta-based

```python
meta = {
    "pydoll": {
        "actions": [...],
        "timeout": 15000,
        "solve_captcha": True
    }
}
```

---

### Helper Class

```python
class PydollRequest(scrapy.Request)
```

* Wraps meta automatically

---

## Action Engine

### Schema

```python
{
  "type": "click" | "type" | "scroll" | "wait",
  "selector": str,
  "text": str,
  "for": "selector" | "sleep",
  "value": str,
  "ms": int
}
```

---

### Execution

#### click

```python
el = await tab.find(selector, timeout=remaining)
await el.click()
```

#### type

```python
el = await tab.find(selector)
await el.insert_text(text)
```

#### scroll

```python
await tab.evaluate("window.scrollTo(0, document.body.scrollHeight)")
```

#### wait

* selector:

```python
await tab.find(value, timeout=remaining)
```

* sleep:

```python
await asyncio.sleep(ms / 1000)
```

---

## Timeout Model

* Single global timeout per request
* Each step consumes remaining time

On timeout:

```python
raise IgnoreRequest("Pydoll timeout")
```

---

## Error Handling

| Condition          | Behavior |
| ------------------ | -------- |
| Navigation failure | Retry    |
| Timeout            | Retry    |
| Selector not found | Fail     |
| Invalid action     | Fail     |
| Browser crash      | Retry    |

Errors attached:

```python
request.meta["pydoll_error"]
```

---

## Cookie Handling

### Before request

* Extract from Scrapy cookiejar
* Convert → Pydoll format
* Inject via `tab.set_cookies()`

### After request

* Extract via `tab.get_cookies()`
* Merge back into Scrapy cookiejar

---

## Response Construction

```python
HtmlResponse(
    url=request.url,
    body=html,
    encoding="utf-8",
    request=request,
)
```

* HTML from:

```python
await tab.content()
```

---

## Middleware Ordering

```python
DOWNLOADER_MIDDLEWARES = {
    "scrapy.downloadermiddlewares.cookies.CookiesMiddleware": 700,
    "scrapy_pydoll.middleware.PydollMiddleware": 750,
    "scrapy.downloadermiddlewares.retry.RetryMiddleware": 800,
}
```

---

## Browser Lifecycle

* Initialized in `from_crawler`
* Closed on `spider_closed`
* Single shared instance

---

## Logging

Must log:

* Request start/end
* Actions executed
* Failures with:

  * URL
  * actions
  * error type

---

## Success Criteria

* Pydoll requests return fully rendered HTML
* `.css()` / `.xpath()` work unchanged
* Sessions persist via cookies
* Concurrency is stable
* Errors propagate to RetryMiddleware
* No blocking behavior

