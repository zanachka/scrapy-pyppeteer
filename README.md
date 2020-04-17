# Pyppeteer integration for Scrapy
[![actions](https://github.com/elacuesta/scrapy-pyppeteer/workflows/Build/badge.svg)](https://github.com/elacuesta/scrapy-pyppeteer/actions)
[![codecov](https://codecov.io/gh/elacuesta/scrapy-pyppeteer/branch/master/graph/badge.svg)](https://codecov.io/gh/elacuesta/scrapy-pyppeteer)


This project provides a Scrapy Download Handler which performs requests using
[Pyppeteer](https://github.com/miyakogi/pyppeteer). It can be used to handle
pages that require JavaScript. This package does not interfere with regular
Scrapy workflows such as request scheduling or item processing.


## Motivation

After the release of [version 2.0](https://docs.scrapy.org/en/latest/news.html#scrapy-2-0-0-2020-03-03),
which includes partial [coroutine syntax support](https://docs.scrapy.org/en/2.0/topics/coroutines.html)
and experimental [asyncio support](https://docs.scrapy.org/en/2.0/topics/asyncio.html), Scrapy allows
to integrate `asyncio`-based projects such as `Pyppeteer`.


## Installation

This package is not (yet) available on PyPI, but it can be installed from GitHub:
```
$ pip install https://github.com/elacuesta/scrapy-pyppeteer
```

## Configuration

Replace the default `http` and `https` Download Handlers through
[`DOWNLOAD_HANDLERS`](https://docs.scrapy.org/en/latest/topics/settings.html):

```python
DOWNLOAD_HANDLERS = {
    "http": "scrapy_pyppeteer.ScrapyPyppeteerDownloadHandler",
    "https": "scrapy_pyppeteer.ScrapyPyppeteerDownloadHandler",
}
```

Note that the `ScrapyPyppeteerDownloadHandler` class inherits from the default
`http/https` handler, and it will only use Pyppeteer for requests that are
explicitly marked (see the "Basic usage" section for details).

Also, be sure to [install the `asyncio`-based Twisted reactor](https://docs.scrapy.org/en/latest/topics/asyncio.html#installing-the-asyncio-reactor):

```python
TWISTED_REACTOR = "twisted.internet.asyncioreactor.AsyncioSelectorReactor"
```

`scrapy-pyppeteer` accepts the following settings:

* `PYPPETEER_LAUNCH_OPTIONS` (type `dict`, default `{}`)

    A dictionary with options to be passed when launching the Browser.
    See the docs for [pyppeteer.launcher.launch](https://miyakogi.github.io/pyppeteer/reference.html#pyppeteer.launcher.launch)

* `PYPPETEER_NAVIGATION_TIMEOUT` (type `Optional[int]`, default `None`)

    The timeout used when requesting pages by Pyppeteer. If `None` or unset,
    the default value will be used (30000 ms at the time of writing this).
    See the docs for [pyppeteer.page.Page.setDefaultNavigationTimeout](https://miyakogi.github.io/pyppeteer/reference.html#pyppeteer.page.Page.setDefaultNavigationTimeout)


## Basic usage

Set the `pyppeteer` [Request.meta](https://docs.scrapy.org/en/latest/topics/request-response.html#scrapy.http.Request.meta)
key to download a request using Pyppeteer:

```python
import scrapy

class AwesomeSpider(scrapy.Spider):
    name = "awesome"

    def start_requests(self):
        yield scrapy.Request("https://example.org", meta={"pyppeteer": True})

    def parse(self, response):
        return response.follow_all(css="a", meta={"pyppeteer": True})
```

### Receiving the Page object in the callback

Specifying `pyppeteer.page.Page` as the type for a callback argument will result
in the corresponding `Page` object being injected in the callback.

```python
import scrapy
import pyppeteer

class AwesomeSpiderWithPage(scrapy.Spider):
    name = "page"

    def start_requests(self):
        yield scrapy.Request("https://example.org", meta={"pyppeteer": True})

    async def parse(self, response, page: pyppeteer.page.Page):
        title = await page.title()  # "Example Domain"
        yield {"title": title}
        await page.close()
```

**Notes:**

* In order to avoid memory issues, it is recommended to manually close the page
  by awaiting the `Page.close` coroutine. To do this, the callback needs to be
  defined as a coroutine function (`async def`):
* Any network operations resulting on awaiting a coroutine on a `Page` object
  (`goto`, `goBack`, etc) will be executed directly by Pyppeteer, bypassing the
  Scrapy request workflow (Scheduler, Middlewares, etc).


## Page coroutines

A sorted iterable could be passed in the `pyppeteer_page_coroutines`
[Request.meta](https://docs.scrapy.org/en/latest/topics/request-response.html#scrapy.http.Request.meta)
key to request coroutines to be awaited on the `Page` before returning the final
`Response` to the callback.

This is useful when you need to perform certain actions on a page, like scrolling
down or clicking links, and you want everything to count as a single Scrapy
Response, containing the final result.

### Supported actions

* `scrapy_pyppeteer.page.PageCoroutine(method: str, *args, **kwargs)`:

    _Represents a coroutine to be awaited on a `pyppeteer.page.Page` object,
    such as "click", "screenshot", "evaluate", etc.
    `method` should be the name of the coroutine, `*args` and `**kwargs`
    are passed to the function call._

    For instance,
    ```python
    PageCoroutine("screenshot", options={"path": "quotes.png", "fullPage": True})
    ```

    produces the same effect as:
    ```python
    # 'page' is a pyppeteer.page.Page object
    await page.screenshot(options={"path": "quotes.png", "fullPage": True})
    ```

* `scrapy_pyppeteer.page.NavigationPageCoroutine(method: str, *args, **kwargs)`:

    _Subclass of `PageCoroutine`. It waits for a navigation event: use this when you know
    a coroutine will trigger a navigation event, for instance when clicking on a link.
    This forces a `Page.waitForNavigation()` call wrapped in `asyncio.gather`, as recommended in
    [the Pyppeteer docs](https://miyakogi.github.io/pyppeteer/reference.html#pyppeteer.page.Page.click)._

    For instance,
    ```python
    NavigationPageCoroutine("click", selector="a")
    ```

    produces the same effect as:
    ```python
    # 'page' is a pyppeteer.page.Page object
    await asyncio.gather(
        page.waitForNavigation(),
        page.click(selector="a"),
    )
    ```


## Examples

**Click on a link, save the resulting page as PDF**

```python
class ClickAndSavePdfSpider(scrapy.Spider):
    name = "pdf"

    def start_requests(self):
        yield Request(
            url="https://example.org",
            meta=dict(
                pyppeteer=True,
                pyppeteer_page_coroutines=[
                    NavigationPageCoroutine("click", selector="a"),
                    PageCoroutine("pdf", options={"path": "iana.pdf"}),
                ],
            ),
        )

    def parse(self, response):
        yield {"url": response.url}  # response.url is "https://www.iana.org/domains/reserved"
```

**Scroll down on an infinite scroll page, take a screenshot of the full page**

```python
class ScrollSpider(scrapy.Spider):
    name = "scroll"

    def start_requests(self):
        yield Request(
            url="http://quotes.toscrape.com/scroll",
            meta=dict(
                pyppeteer=True,
                pyppeteer_page_coroutines=[
                    PageCoroutine("waitForSelector", "div.quote"),
                    PageCoroutine("evaluate", "window.scrollBy(0, 2000)"),
                    PageCoroutine("waitForSelector", "div.quote:nth-child(11)"),  # 10 per page
                    PageCoroutine("screenshot", options={"path": "quotes.png", "fullPage": True}),
                ],
            ),
        )

    def parse(self, response):
        yield {"quote_count": len(response.css("div.quote")}  # 100 quotes
```


## Acknowledgements

This project was inspired by:

* https://github.com/scrapy/scrapy/pull/1455
* https://github.com/michalmo/scrapy-browser
* https://github.com/lopuhin/scrapy-pyppeteer
* https://github.com/clemfromspace/scrapy-puppeteer
