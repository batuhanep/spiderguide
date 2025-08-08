# Web Scraping & Spider Guide (Python + Scrapy)

> **Goal:** Start from scratch and build a **Scrapy** spider that can parse HTML, extract data via **XPath/CSS** selectors, follow links, and crawl across pages. You can drop this guide into your repo as **`WEB_SCRAPING_SPIDER_GUIDE_EN.md`**.

---

## Table of Contents
1. Prerequisites & Setup
2. HTML Crash Course
3. XPath Selectors: Basics to Advanced
4. CSS Selectors & XPath ↔ CSS Mappings
5. Extracting Attributes & Text
6. Practicing with Scrapy Selector (Standalone)
7. Working with response & Following Links
8. Write Your First Spider: Run with CrawlerProcess
9. Crawling: Collect Links, Follow, Gather Data
10. Mini Capstone: Course → Chapter Titles Dictionary
11. Debug Checklist & Tips
12. Common Pitfalls & Fixes
13. Stretch Tasks & Improvements

---

## Prerequisites & Setup

### 1) Python & Virtual Environment
```bash
python -V          # 3.10+ recommended
python -m venv .venv
# macOS/Linux
source .venv/bin/activate
# Windows
# .venv\Scripts\activate
```

### 2) Install Scrapy
```bash
pip install scrapy requests parsel
```
> `parsel` is the selector library used under Scrapy; we’ll use it for XPath/CSS.

### 3) PyCharm Tip
- Create a project with a **Virtualenv**; set interpreter to `.venv`.
- Choose the script you want to run in Run/Debug (e.g., `spider_dc.py`).

> **Ethics & Terms:** Always check a site’s **robots.txt** and **Terms of Service**. Be respectful: set reasonable concurrency, obey rate limits, and avoid disrupting services.

---

## HTML Crash Course
HTML is a tree of **elements (tags)** like `html`, `body`, `div`, `p`, `a`. Elements have **attributes** such as `id` (should be unique), `class` (can repeat across elements), `href` (link target), `src` (image source), etc.

**Key ideas:**
- Think in parent/child and sibling relationships.
- `<a>` contains links via `href`.
- `id` is unique; `class` can be shared.

---

## XPath Selectors: Basics to Advanced

### Core Symbols
- `/` : one step to a child
- `//` : search through all descendants
- `[]` : indexing or filtering among siblings (e.g., `p[2]`)
- `*` : wildcard (any tag)
- `@attr` : attribute access (`@href`, `@class`, `@id`)
- `contains(@attr, "part")` : partial match in attributes

### Examples
```xpath
/html/body/p                 # all p under body
/html/body/div/p[2]          # second p under body>div
//p                          # all p in the page
/html/body/*                 # all direct children of body
//*[@id="uid"]              # any element with id=uid
//div[@id="uid"]/p[2]       # second p under div#uid
//*[contains(@class,"class-1")] # class contains class-1
//div[@id="uid"]/a/@href     # href attribute values
```

**Note:** `'/html/body'` and `'/html[1]/body[1]'` target the same. XPath indices start at **1**.

---

## CSS Selectors & XPath ↔ CSS Mappings

### Quick Mappings
- `XPath /`  → CSS `>` (child)
- `XPath //` → CSS descendant (space) or `>` (child)
- `XPath [N]` → CSS `:nth-of-type(N)`

**Example mapping**
```
XPath: /html/body//div/p[2]
CSS  : html > body div > p:nth-of-type(2)
```

### Common CSS Patterns
```css
p.class-1                 /* p elements with class class-1 */
div#uid                   /* div with id=uid */
.class1                   /* any element with class1 */
div#uid > p.class1        /* p.class1 under div#uid */
```

### Attributes & Text via CSS
```css
a::attr(href)             /* href values of links */
p#p-example::text         /* direct text nodes of p */
p#p-example ::text        /* all descendant text nodes under p */
```

---

## Extracting Attributes & Text

### XPath for Attributes
```xpath
//div[@id='uid']/a/@href
```

### CSS for Attributes
```css
div#uid > a::attr(href)
```

### Extract Text
- XPath `text()` returns **direct** text nodes.
- `//text()` includes descendant text.
- In CSS, `::text` is direct text; ` ::text` (leading space) includes descendants.

```python
sel.xpath("//p[@id='p-example']/text()").extract()
sel.xpath("//p[@id='p-example']//text()").extract()
sel.css("p#p-example::text").extract()
sel.css("p#p-example ::text").extract()
```

---

## Practicing with Scrapy Selector (Standalone)
Sometimes you want to test selectors without a full project.

```python
from scrapy import Selector
html = """
<html>
  <body>
    <div class="hello datacamp">
      <p>Hello World!</p>
    </div>
    <p>Enjoy DataCamp!</p>
  </body>
</html>
"""
sel = Selector(text=html)

# XPath
sel.xpath("//p")                # returns a SelectorList
sel.xpath("//p").extract()      # ['<p>Hello World!</p>', '<p>Enjoy DataCamp!</p>']
sel.xpath("//p").extract_first()# '<p>Hello World!</p>'

# CSS
sel.css("div > p").extract()    # ['<p>Hello World!</p>']

# Grab a real page using requests
import requests
url = "https://en.wikipedia.org/wiki/Web_scraping"
html = requests.get(url).content
sel = Selector(text=html)
```

> **Tip:** `SelectorList` is indexable (e.g., `ps[1]`). `extract()` returns HTML strings; if you want plain text, target nodes via `::text`/`text()` first.

---

## Working with `response` & Following Links
Inside a Scrapy spider, `response` behaves like a Selector: use `response.xpath(...)` and `response.css(...)`, then `extract()` / `extract_first()` to get values.

Useful attributes/methods:
- **`response.url`**: the current page URL.
- **`response.follow(next_url, callback=...)`**: follow a link and send the new page to another callback.

```python
# chaining selectors
data = response.xpath('//div').css('span.bio').extract()
# following a link
yield response.follow(url=next_url, callback=self.parse_item)
```

---

## Write Your First Spider: Run with `CrawlerProcess`
The spider below requests a page and writes its HTML to a local file—perfect for a single-file demo.

```python
# file: spider_dc.py
import scrapy
from scrapy.crawler import CrawlerProcess

class DCSpider(scrapy.Spider):
    name = "dc_spider"

    def start_requests(self):
        url = "https://www.datacamp.com/courses/all"
        yield scrapy.Request(url=url, callback=self.parse)

    def parse(self, response):
        with open("DC_courses.html", "wb") as fout:
            fout.write(response.body)

if __name__ == "__main__":
    process = CrawlerProcess()
    process.crawl(DCSpider)
    process.start()
```

> `start_requests` must yield at least one `scrapy.Request`; `callback` declares which function will process the `response`.

---

## Crawling: Collect Links, Follow, Gather Data
Let’s collect course links from a page, write them to CSV, then follow each link and process details in a second parser.

```python
# file: spider_links.py
import scrapy
from scrapy.crawler import CrawlerProcess

class DCLinksSpider(scrapy.Spider):
    name = "dc_links"

    def start_requests(self):
        yield scrapy.Request(
            url="https://www.datacamp.com/courses/all",
            callback=self.parse
        )

    def parse(self, response):
        # one-liner
        links = response.css('div.course-block > a::attr(href)').extract()

        # verbose steps
        # blocks = response.css('div.course-block')
        # hrefs = blocks.xpath('./a/@href')
        # links = hrefs.extract()

        # write to CSV
        with open("DC_links.csv", "w", encoding="utf-8") as f:
            for link in links:
                f.write(link + "\n")

        # follow each link
        for link in links:
            yield response.follow(url=link, callback=self.parse_course)

    def parse_course(self, response):
        # process each course page here
        self.logger.info("Parsed course page: %s", response.url)

if __name__ == "__main__":
    process = CrawlerProcess()
    process.crawl(DCLinksSpider)
    process.start()
```

> **Note:** If you saw `"/n"` in some slides, Python’s newline is `"\n"`.

---

## Mini Capstone: Course → Chapter Titles Dictionary
From the index page, collect course links; on each course page, capture the **title** and **chapter titles**, then write a dictionary to JSON.

```python
# file: spider_capstone.py
import json
import scrapy
from scrapy.crawler import CrawlerProcess

dc_dict = {}

class DCChapterSpider(scrapy.Spider):
    name = "dc_chapter_spider"

    def start_requests(self):
        url = "https://www.datacamp.com/courses/all"
        yield scrapy.Request(url=url, callback=self.parse_front)

    def parse_front(self, response):
        course_blocks = response.css('div.course-block')
        course_links = course_blocks.xpath('./a/@href')
        for url in course_links.extract():
            yield response.follow(url=url, callback=self.parse_pages)

    def parse_pages(self, response):
        # course title
        title_sel = response.xpath('//h1[contains(@class, "title")]/text()')
        course_title = (title_sel.extract_first() or '').strip()

        # chapter titles
        ch_titles = response.css('h4.chapter__title::text')
        chapters = [t.strip() for t in ch_titles.extract()]

        if course_title:
            dc_dict[course_title] = chapters

    def closed(self, reason):
        with open('dc_chapters.json', 'w', encoding='utf-8') as f:
            json.dump(dc_dict, f, ensure_ascii=False, indent=2)

if __name__ == "__main__":
    process = CrawlerProcess()
    process.crawl(DCChapterSpider)
    process.start()
```

> `closed(self, reason)` is called when the spider finishes—handy to write aggregated data to disk.

---

## Debug Checklist & Tips
- **Inspect elements:** Use DevTools “Inspect Element” to see actual HTML.
- **Test selectors:** Try `scrapy shell URL` to interactively run `response.css(...)` and `response.xpath(...)`.
- **Iterate gradually:** First target the right element; then extract attribute/text; then loop; only then write to disk.
- **Relative vs absolute XPath:** `./a/@href` targets children of the current node; starting with `//` may grab too much.
- **Indices start at 1:** XPath uses `p[1]`, `p[2]`, …
- **`::text` vs `//text()`**: direct text vs all descendant text—choose intentionally.

---

## Common Pitfalls & Fixes
- **Empty lists/None:** Selector might be wrong or page is dynamic. Confirm HTML with `response.text[:500]`. Consider `selenium` or using an API when applicable.
- **Relative links:** `response.follow` usually resolves them for you. If building URLs manually, consider `urljoin`.
- **Encoding/escaping:** When writing files use `encoding='utf-8'`. Newlines are `"\n"`.

---

## Stretch Tasks & Improvements
- From each course page, also collect instructor names/tags.
- Handle pagination: follow the `next` link to traverse multiple index pages.
- Push scraped data via **Pipelines** to CSV/JSON/SQLite in a proper Scrapy project.

---

### Suggested Repo Layout (single-file spiders)
```
repo/
├─ WEB_SCRAPING_SPIDER_GUIDE_EN.md  # this guide (English)
├─ WEB_SCRAPING_SPIDER_GUIDE.md     # Turkish version (optional)
├─ spider_dc.py                      # first HTML dump example
├─ spider_links.py                   # collect links + follow + CSV
└─ spider_capstone.py                # course -> chapters dict (JSON)
```

> For larger projects, prefer `scrapy startproject yourproj` to get the full structure (`spiders`, `items`, `pipelines`, `settings`).

---

**Happy scraping!** 👋
