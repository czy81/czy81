# Scrapling 使用指南

> 基于源码版本 **v0.4.9** 整理。原项目：[D4Vinci/Scrapling](https://github.com/D4Vinci/Scrapling) · 文档：https://scrapling.readthedocs.io
>
> 本目录的 `scrapling.zip` 为该项目完整源码归档。

## 它是什么

Scrapling 是一个**自适应网页抓取框架（adaptive web scraping framework）**，核心卖点有三个：

- **自适应解析**：解析器会记住元素特征，网页改版后能自动重新定位元素。
- **反爬绕过**：内置 fetcher 开箱即可绕过 Cloudflare Turnstile / Interstitial 等反爬。
- **完整爬虫框架**：Scrapy 风格的并发爬虫，支持多会话、断点续爬、流式输出。

要求 **Python ≥ 3.10**。

它分四层，按需要由轻到重选用：

| 层级 | 类 | 适用场景 |
|------|-----|----------|
| 纯 HTTP | `Fetcher` | 静态页面，最快，能伪装浏览器 TLS 指纹 |
| 真浏览器 | `DynamicFetcher` | 需要 JS 渲染的动态页面（Playwright/Chrome） |
| 隐身反检测 | `StealthyFetcher` | 有反爬墙的站点，自动过 Cloudflare |
| 大规模爬虫 | `Spider` | 成千上万页、并发、断点续爬 |

## 安装

```bash
pip install scrapling                 # 只装解析器引擎
pip install "scrapling[fetchers]"     # 要用 fetcher / spider 必须装这个
scrapling install                     # 下载浏览器内核（用 Dynamic/Stealthy 时必需）

# 全功能（含命令行 shell 和 AI/MCP server）：
pip install "scrapling[all]" && scrapling install
```

> ⚠️ 只 `pip install scrapling` 后再 `import scrapling.fetchers` 会报 `ModuleNotFoundError` —— 这是设计如此，必须安装 `[fetchers]` 额外依赖。

额外功能：

```bash
pip install "scrapling[ai]"      # MCP server（配合 Claude/Cursor 等 AI 使用）
pip install "scrapling[shell]"   # 交互式抓取 shell 和 extract 命令
```

Docker（含所有浏览器和额外功能）：

```bash
docker pull pyd4vinci/scrapling
# 或
docker pull ghcr.io/d4vinci/scrapling:latest
```

## 最常用的三种方式

### 1. 普通 HTTP 抓取（最快）

```python
from scrapling.fetchers import Fetcher

page = Fetcher.get('https://quotes.toscrape.com/')
quotes = page.css('.quote .text::text').getall()                  # CSS 选择器
authors = page.xpath('//small[@class="author"]/text()').getall()  # 或 XPath
```

带会话（保持 cookie / 状态，伪装 Chrome 指纹）：

```python
from scrapling.fetchers import FetcherSession

with FetcherSession(impersonate='chrome') as session:
    page = session.get('https://quotes.toscrape.com/', stealthy_headers=True)
    quotes = page.css('.quote .text::text').getall()
```

### 2. 反爬隐身抓取（自动过 Cloudflare）

```python
from scrapling.fetchers import StealthyFetcher

page = StealthyFetcher.fetch(
    'https://nopecha.com/demo/cloudflare',
    headless=True,
    solve_cloudflare=True,
)
data = page.css('#padded_content a::text').getall()
```

### 3. 全浏览器自动化（JS 渲染页面）

```python
from scrapling.fetchers import DynamicFetcher

page = DynamicFetcher.fetch('https://quotes.toscrape.com/', network_idle=True)
data = page.css('.quote .text::text').getall()
```

## 自适应抓取（网页改版后仍能定位元素）

```python
from scrapling.fetchers import StealthyFetcher
StealthyFetcher.adaptive = True

p = StealthyFetcher.fetch('https://example.com', network_idle=True)
products = p.css('.product', auto_save=True)   # 第一次：记住这些元素的特征
# 之后网站改版了，再加 adaptive=True 自动重新定位：
products = p.css('.product', adaptive=True)
```

## 灵活的选择与导航 API

```python
from scrapling.fetchers import Fetcher
page = Fetcher.get('https://quotes.toscrape.com/')

quotes = page.css('.quote')                       # CSS
quotes = page.xpath('//div[@class="quote"]')      # XPath
quotes = page.find_all('div', class_='quote')     # BeautifulSoup 风格
quotes = page.find_by_text('quote', tag='div')    # 按文本查找

first = page.css('.quote')[0]
author = first.next_sibling.css('.author::text')  # 兄弟节点
parent = first.parent                             # 父节点
similar = first.find_similar()                    # 查找结构相似的元素
```

不抓网页、直接解析已有 HTML：

```python
from scrapling.parser import Selector
page = Selector("<html>...</html>")   # 用法完全一致
```

## 不写代码，直接命令行抓取

```bash
scrapling shell                                              # 交互式抓取 shell（IPython）

scrapling extract get 'https://example.com' out.md          # 存成 Markdown
scrapling extract get 'https://example.com' out.txt --css-selector '#main'
scrapling extract fetch 'https://example.com' out.md --no-headless
scrapling extract stealthy-fetch 'https://站点' out.html --solve-cloudflare
```

输出文件后缀决定格式：`.txt` = 纯文本，`.md` = Markdown，`.html` = 原始 HTML。

## 大规模爬虫（Scrapy 风格，支持并发 / 断点续爬）

```python
from scrapling.spiders import Spider, Response

class QuotesSpider(Spider):
    name = "quotes"
    start_urls = ["https://quotes.toscrape.com/"]
    concurrent_requests = 10

    async def parse(self, response: Response):
        for q in response.css('.quote'):
            yield {
                "text": q.css('.text::text').get(),
                "author": q.css('.author::text').get(),
            }
        nxt = response.css('.next a')
        if nxt:
            yield response.follow(nxt[0].attrib['href'])

result = QuotesSpider(crawldir="./crawl_data").start()  # 传 crawldir 可 Ctrl+C 暂停、再启动续爬
print(f"抓取了 {len(result.items)} 条")
result.items.to_json("quotes.json")
```

## 一句话上手建议

先用 `Fetcher`（够快，覆盖约 80% 场景）；遇到 JS 渲染的页面换 `DynamicFetcher`；被反爬墙挡了换 `StealthyFetcher`；要爬成千上万页再上 `Spider`。

## 免责声明

> 该库仅供学习与研究用途。使用时请遵守目标网站的 robots.txt、服务条款以及当地与国际的数据抓取/隐私法律。作者与贡献者不对任何滥用行为负责。
