---
title: Scarpy基本使用
date: 2022-11-26 22:33:01
tags: 爬虫
---

## 基础介绍

### 架构

![img](Scarpy基本使用/20180502174530976.png)

Engine:图中最中间的部分，中文可以称为引擎，用来处理整个系统的数据流和事件，是整个框架的核心，可以理解为整个框架的中央处理器，负责数据的流转和逻辑的处理。
Item:它是一个抽象的数据结构，所以图中没有体现出来，它定义了爬取结果的数据结构，爬取的数据会被赋值成Item对象。每个Iltem就是一个类，类里面定义了爬取结果的数据字段，可以理解为它用来规定爬取数据的存储格式。°
Scheduler:图中下方的部分，中文可以称为调度器，它用来接受Engine发过来的Request 并将其加入队列中，同时也可以将 Request 发回给Engine供 Downloader执行，它主要维护Request的调度逻辑，比如先进先出、先进后出、优先级进出等等。
Spiders:图中上方的部分,中文可以称为蜘蛛,Spiders是一个复数统称,其可以对应多个Spider,每个Spider 里面定义了站点的爬取逻辑和页面的解析规则，它主要负责解析响应并生成Item和新的请求然后发给Engine进行处理。
Downloader:图中右侧部分，中文可以称为下载器，即完成“向服务器发送请求，然后拿到响应”的过程,得到的响应会再发送给Engine处理。
Item Pipelines:图中左侧部分，中文可以称为项目管道，这也是一个复数统称，可以对应多个Item Pipeline。Item Pipeline主要负责处理由Spider 从页面中抽取的 Item，做一些数据清洗、验证和存储等工作，比如将Item的某些字段进行规整，将Item存储到数据库等操作都可以由Item Pipeline来完成。
Downloader Middlewares:图中 Engine和 Downloader 之间的方块部分，中文可以称为下载器中间件，同样这也是复数统称，其包含多个Downloader Middleware，它是位于 Engine和Downloader之间的Hook框架,负责实现Downloader和 Engine之间的请求和响应的处理过程。

Spider Middlewares:图中 Engine和Spiders之间的方块部分，中文可以称为蜘蛛中间件，它是位于Engine和 Spiders之间的Hook框架，负责实现Spiders 和 Engine之间的Item.
请求和响应的处理过程。



### 数据流处理过程

(1)启动爬虫项目时，Engine根据要爬取的目标站点找到处理该站点的Spider，Spider 会生成最初需要爬取的页面对应的一个或多个Request，然后发给Engine。
(2) Engine 从 Spider中获取这些Request，然后把它们交给Scheduler等待被调度。
(3)Engine向 Scheduler索取下一个要处理的Request，这时候Scheduler根据其调度逻辑选择合适的Request发送给Engine。
(4) Engine将Scheduler发来的 Request转发给Downloader进行下载执行，将Request发送给Downloader的过程会经由许多定义好的 Downloader Middlewares 的处理。
(5) Downloader将Request 发送给目标服务器，得到对应的Response，然后将其返回给Engine。将Response返回 Engine的过程同样会经由许多定义好的 Downloader Middlewares的处理。
(6) Engine 从 Downloder处接收到的 Response里包含了爬取的目标站点的内容，Engine会将此Response发送给对应的Spider进行处理，将Response发送给Spider 的过程中会经由定义好的SpiderMiddlewares的处理。
(7)Spider处理Response，解析Response的内容，这时候Spider会产生一个或多个爬取结果Item或者后续要爬取的目标页面对应的一个或多个Request，然后再将这些Item或Request 发送给Engine进行处理，将Item或Request发送给Engine的过程会经由定义好的Spider Middlewares的处理。
(8)Engine将Spider发回的一个或多个Item转发给定义好的Item Pipelines进行数据处理或存储的一系列操作，将Spider 发回的一个或多个Request转发给Scheduler等待下一次被调度。
重复第(2)步到第(8)步，直到Scheduler中没有更多的Request，这时候Engine会关闭Spider,整个爬取过程结束。



### 项目结构

创建项目

```
scrapy startproject project_name
```

进入项目，创建spider

```
scrapy genspider spider_name spider_domain
```

目录结构

```
|
|___project_name
|	|
|	|___
|	|
|	|___
|
|
|
|
|
```



## Spider使用

功能：定义爬取网站的动作和分析爬取下的网页

Spider爬取循环

1. 以初始的URL初始化Request并设置回调方法。当该Request成功请求并返回时，将生成Response并将其作为参数传给该回调方法。
2. 在回调方法内分析返回的网页内容。返回结果可以有两种形式，一种是将解析到的有效结果返回字典或Item对象，下一步可直接保存或者经过处理后保存;另一种是解析的下一个链接，可以利用此链接构造Request并设置新的回调方法，返回Request
3. 如果返回的是字典或Item对象，可以保存到文件或者经由Pipeline处理（如过滤、修正等)并保存。
4. 如果返回的是Reqeust，那么Request 执行成功得到Response之后会再次传递给Request 中定义的回调方法，可以再次使用选择器来分析新得到的网页内容，并根据分析的数据生成Item.

#### 基础属性

- name:爬虫名称，是定义Spider名字的字符串。Spider的名字定义了Scrapy如何定位并初始化 Spider，所以它必须是唯一的。不过我们可以生成多个相同的Spider实例，这没有任何限制。name是Spider最重要的属性，而且是必须的。如果该Spider爬取单个网站，一个常见的做法是以该网站的域名名称来命名Spider。例如 Spider爬取mywebsite.com ，该Spider通常会被命名为mywebsite。
- allowed_domains:允许爬取的域名，是一个可选的配置，不在此范围的链接不会被跟进爬取。
- start_urls:起始URL列表，当我们没有实现start_requests方法时，默认会从这个列表开始抓取。
- custom_settings:一个字典，是专属于本Spider 的配置，此设置会覆盖项目全局的设置，而且此设置必须在初始化前被更新，所以它必须定义成类变量。
- crawler:此属性是由from_crawler方法设置的，代表的是本Spider类对应的Crawler对象，Crawler对象中包含了很多项目组件，利用它我们可以获取项目的一些配置信息，常见的就是获取项目的设置信息，即 Settings。
- settings:一个Settings对象，利用它我们可以直接获取项目的全局设置变量。

#### 常用方法

- start_requests:此方法用于生成初始请求，它必须返回一个可迭代对象，此方法会默认使用start_urls里面的URL来构造Request，而且Request是GET请求方式。如果我们想在启动时以POST方式访问某个站点，可以直接重写这个方法，发送 POST请求时我们使用FormRequest即可。
- parse:当Response没有指定回调方法时，该方法会默认被调用，它负责处理Response，并从或Item的中提取想要的数据和下一步的请求，然后返回。该方法需要返回一个包含Request可迭代对象。
- closed:当Spider关闭时，该方法会被调用，这里一般会定义释放资源的一些操作或其他收尾操作。

##### Request

- url: Request的页面链接，即Request URL。
- callback:Request的回调方法，通常这个方法需要定义在Spider类里面，并且需要对应一个response参数，代表Request执行请求后得到的Response对象。如果这个callback参数不指定，默认会使用Spider类里面的parse方法。
- method: Request的方法，默认是GET，还可以设置为POST、PUT、DELETE等。
- meta:Request 请求携带的额外参数，利用meta，我们可以指定任意处理参数，特定的参数经由Scrapy各个组件的处理,可以得到不同的效果。另外,meta还可以用来向回调方法传递信息。body: Request的内容，即 Request Body，往往Request Body对应的是POST请求,可以使用FormRequest或JsonRequest更方便地实现POST请求。
- headers: Request Headers，是字典形式。
- cookies: Request携带的Cookie，可以是字典或列表形式。encoding: Request的编码，默认是utf-8。
- prority:Request优先级，默认是0，这个优先级是给Scheduler做Request调度使用的，数值越大，就越被优先调度并执
- dont_filter:Request不去重，Scrapy默认会根据Request的信息进行去重，使得在爬取过程中不会出现重复请求，设置为True代表这个Request会被忽略去重操作，默认是False.
- errback:错误处理方法，如果在请求处理过程中出现了错误，这个方法就会被调用。flags:请求的标志，可以用于记录类似的处理。
- cb_kwargs:回调方法的额外参数，可以作为字典传递。

##### Response

- url: Request URL。
- status : Response状态码，如果请求成功就是200。
- headers: Response Headers，是一个字典，字段是一一对应的。
- body:Response Body，这个通常就是访问页面之后得到的源代码结果了，比如里面包含的是HTML或者JSON字符串，但注意其结果是bytes类型。
- request: Response对应的 Request对象。
- certificate:是twisted.internet.ssl.Certificate类型的对象，通常代表一个SSL证书对象。
- ip_address:是一个ipaddress.IPv4Address或ipaddress.IPv6Address类型的对象，
  代表服务器的IP地址。
- urljoin:是对URL的一个处理方法，可以传入当前页面的相对URL，该方法处理后返回的就是绝对URL。
- follow/follow all:是一个根据URL来生成后续Request的方法，和直接构造Request不同的是，该方法接收的url可以是相对URL，不必一定是绝对URL。

 **方法**

- text:同body属性，但结果是str类型。
- encoding: Response的编码，默认是utf-8。
- selector:根据Response的内容构造而成的Selector对象，Selector在上一节我们已经了解过，利用它我们可以进一步调用xpath、css 等方法进行结果的提取。
- xpath:传入XPath进行内容提取，等同于调用selector的xpath 方法。
- css:传入CSS选择器进行内容提取，等同于调用selector的css方法。
- json:是Scrapy 2.2新增的方法，利用该方法可以直接将text属性转为JSON对象

#### 举例

```python
import scrapy
from scrapy import Request


class HttpbinSpider(scrapy.Spider):
    name = 'httpbin2'
    allowed_domains = ['httpbin.org']
    start_url = 'https://httpbin.org/get'
    headers = {
        'User-Agent': 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_4) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/83.0.4103.116 Safari/537.36'
    }
    cookies = {'name': 'germey', 'age': '26'}
    
    def start_requests(self): # 默认有隐式定义
        for offset in range(5):
            url = self.start_url + f'?offset={offset}'
            yield Request(url, headers=self.headers,
                          cookies=self.cookies,
                          callback=self.parse_response,
                          meta={'offset': offset})
    
    def parse_response(self, response):
        print('url', response.url)
        print('request', response.request)
        print('status', response.status)
        print('headers', response.headers)
        print('text', response.text)
        print('meta', response.meta)
```

## Downloader Middleware

**作用**

- Engine从Scheduler获取Request发送给Downloader，在Request被Engine发送给Downloader执行下载之前，Downloader Middleware可以对Request进行修改。
- Downloader执行Request后生成Response，在Response被Engine发送给Spider之前，也就是在Resposne被Spider解析之前，Downloder Middleware可以对Response进行修改

功能：修改Agent、重定向、设置代理、失败重试、设置Cookie

每个DownloaderMiddleware都可以通过定义process_request和process_response方法来分别处理Request和Response,被开启的Downloader Middleware的process_request方法和process_response方法会根据优先级被顺次调用，数字越小越靠近Engine，越大越靠近Downloader Middleware

核心方法：

- process_request(request，spider)
- process_response(request，response，spider)
- process_exception(request，exception，spider)

定义其中一个可以实现一个Downloader Middleware

### process_request

Request从Scheduler里被调度出来发送到Downloader下载执行之前，可以用process_request方法对Request进行处理。

参数：

- request对象
- spider，request对应的spider对象

返回值：

- None：执行其他Downloader Middleware的process_requst方法，知道执行得到Response

- Response对象：不调用低优先级的process_request和process_exception方法，调用process_response方法，完成后发送到Spider

- Request对象：低优先级的process_request对象停止执行，重新回到调度队列。若被调度，所有的process_request方法重新按照顺序执行

- 抛出IgnoreRequest异常：执行process_exception，如果不存在异常处理方法，则errorback方法会回调

### process_response

Downloader 执行 Request下载之后，会得到对应的 Response。Engine便会将Response发送给Spider进行解析。在发送给Spider之前，我们都可以用process_response方法来对Response进行处理

参数：

- request: Request对象，即此Response对应的Request。
- response: Response对象，即被处理的Response。
- spider: Spider对象，即此 Response对应的Spider对象。

返回值：

- Request对象：低优先级的process_response对象停止执行，重新回到调度队列。若被调度，所有的process_request方法重新按照顺序执行
- Response对象：更低优先级的Downloader Middleware的process_response继续被调用，对该Response对象进行处理。
- IgnoreRequest异常：Request 的errorback方法会回调。如果该异常还没有被处理,那么它会被忽略。

### process_exception

参数：

- request: Request对象，即异常的Request。
- exception: Exception对象，即抛出的异常。
- spider: Spider对象，即Request对应的Spider对象。

返回值：

None：更低优先级的Downloader Middleware的process_exception继续被调用到执行完毕

Response：不调用低优先级的process_exception方法，调用process_response方法

Request：低优先级的process_exception对象停止执行，重新回到调度队列。若被调度，所有的process_request方法重新按照顺序执行

### 举例

```python
#middlewares.py配置useragent、proxy、返回码
class RandomUserAgentMiddleware(object):
    def __init__(self):
        self.user_agents = [
            'Mozilla/5.0 (Windows; U; MSIE 9.0; Windows NT 9.0; en-US)',
            'Mozilla/5.0 (Windows NT 6.1) AppleWebKit/537.2 (KHTML, like Gecko) Chrome/22.0.1216.0 Safari/537.2',
            'Mozilla/5.0 (X11; Ubuntu; Linux i686; rv:15.0) Gecko/20100101 Firefox/15.0.1'
        ]
    
    def process_request(self, request, spider):
        request.headers['User-Agent'] = random.choice(self.user_agents)


class ProxyMiddleware(object):
    def process_request(self, request, spider):
        request.meta['proxy'] = 'http://157.100.12.138:999'


class ChangeResponseMiddleware(object):
    def process_response(self, request, response, spider):
        response.status = 201
        return response
    
# setting设置优先级
DOWNLOADER_MIDDLEWARES = {
    'scrapydownloadermiddlewaredemo.middlewares.RandomUserAgentMiddleware': 543,
    'scrapydownloadermiddlewaredemo.middlewares.ChangeResponseMiddleware': 544,
    'scrapydownloadermiddlewaredemo.middlewares.ProxyMiddleware': 544
}
```

返回结果

```json
Text: {
  "args": {},
  "headers": {
    "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8",
    "Accept-Encoding": "gzip, deflate",
    "Accept-Language": "en",
    "Host": "httpbin.org",
    "User-Agent": "Mozilla/5.0 (Windows NT 6.1) AppleWebKit/537.2 (KHTML, like Gecko) Chrome/22.0.1216.0 Safari/537.2",
    "X-Amzn-Trace-Id": "Root=1-635f9ebb-5b6b8f61460ed7d05370e77d",
    "X-Proxy-Id": "1875528327"
  },
  "origin": "122.193.87.110, 157.100.12.138",
  "url": "http://httpbin.org/get"
}
```

## Spider Middleware

处于Spider和Engine之间的处理模块。当Downloader生成Response之后，Response会被发送给Spider,在发送给Spider之前，Response会首先经过Spider Middleware的处理,当Spider处理生成item和Request之后，item和Request还会经过Spider Middleware的处理

*https://docs.scrapy.org/en/latest/topics/spider-middleware.html*

**作用：**

- Downloader 生成Response之后，Engine 会将其发送给Spider进行解析，在Response发送给Spider之前，可以借助Spider Middleware对 Response进行处理。
- Spider生成Request之后会被发送至Engine，然后Request会被转发到Scheduler，在Request被发送给Engine之前，可以借助Spider Middleware对Request进行处理。
- Spider生成 Item之后会被发送至Engine，然后Item会被转发到 ItemPipeline，在 Item被发送给Engine之前，可以借助Spider Middleware对Item进行处理。

**核心方法**

- process_spider_input(response，spider)

- process_spider_output(response，result，spider)

- process_spider_exception(response,exception，spider)

- process_start_requests(start_requests,spider)

与Downloader Middleware类似

### 举例

spider

```python
from scrapy import Spider, Request

from scrapyspidermiddlewaredemo.items import DemoItem


class HttpbinSpider(Spider):
    name = 'httpbin'
    allowed_domains = ['httpbin.org']
    start_url = 'https://httpbin.org/get'
    
    def start_requests(self):
        for i in range(5):
            url = f'{self.start_url}?query={i}'
            yield Request(url, callback=self.parse)
    
    def parse(self, response): # 将Response转化为DemoItem
        print('Status', response.status)
        item = DemoItem(**response.json())
        yield item
```

items

```python
import scrapy

class DemoItem(scrapy.Item):
    origin = scrapy.Field()
    headers = scrapy.Field()
    args = scrapy.Field()
    url = scrapy.Field()
```

middlewares

```python
from scrapyspidermiddlewaredemo.items import DemoItem


class CustomizeMiddleware(object):
    
    def process_spider_input(self, response, spider):
        response.status = 201
        return None
    
    def process_spider_output(self, response, result, spider):
        for i in result:
            if isinstance(i, DemoItem):
                i['origin'] = None
                yield i
    
    def process_spider_exception(self, response, exception, spider):
        # Called when a spider or process_spider_input() method
        # (from other spider middleware) raises an exception.
        
        # Should return either None or an iterable of Request or item objects.
        pass
    
    def process_start_requests(self, start_requests, spider):
        for request in start_requests:
            url = request.url
            url += '&name=germey'
            request = request.replace(url=url)
            yield request
```

输出结果

```
2022-10-31 18:38:30 [scrapy.core.scraper] DEBUG: Scraped from <201 https://httpbin.org/get?query=4&name=germey>
{'args': {'name': 'germey', 'query': '4'},
 'headers': {'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8',
             'Accept-Encoding': 'gzip, deflate',
             'Accept-Language': 'en',
             'Host': 'httpbin.org',
             'User-Agent': 'Scrapy/2.7.0 (+https://scrapy.org)',
             'X-Amzn-Trace-Id': 'Root=1-635fa5a6-52b6778b2d2f7bbe158b468c'},
 'origin': None,
 'url': 'https://httpbin.org/get?query=4&name=germey'}
Status 201
```

url属性被替换，改写了Request

response被替换，改写了Response

相关内置Spider Middelware

- HttpErrorMiddleware：过滤需要忽略的Response
- OffisteMiddleware：过滤不符合allow_domains的Request
- UrlLengthMiddleware：根据URL长度过滤

## Item Pipeline

调用发生在Spider产生Item之后。当Spider解析完Response,ltem就会被Engjne传递到Item Pipeline,被定义的Item Pipeline组件会顺次被调用

功能

- 清洗HTML数据。
- 验证爬取数据，检查爬取字段。
- 查重并丢弃重复内容。
- 将爬取结果储存到数据库中。

核心方法：

- process_item(item, spider)：必须实现的方法，被定义的 Item Pipeline 会默认调用这个方法对Item进行处理,比如进行数据处理或者将数据写入数据库等操作。process item方法必须返回Item类型的值或者抛出一个 Dropltem异常。返回Item类型的值会被低优先级的process_item处理

可选方法：

- open_spider(spider)：自动调用，可以做初始化工作，如数据库连接
- close_spider(spider)：自动调用，做一些收尾工作，如关闭数据库连接
- from_crawler(cls, crawler)：一个类方法，用@classmethod标识，通过crawler对象，我们可以拿到Scrapy的所有核心组件，如全局配置的每个信息。然后可以在这个方法里面创建一个Pipeline实例。参数cls就是Class，最后返回一个Class实例。

### 举例

scrapy

定义爬取逻辑

```python
from scrapy import Request, Spider

from scrapyitempipelinedemo.items import MovieItem


class ScrapeSpider(Spider):
    name = 'scrape'
    allowed_domains = ['ssr1.scrape.center']
    base_url = 'https://ssr1.scrape.center'
    max_page = 10
    
    def start_requests(self):
        for i in range(1, self.max_page + 1):
            url = f'{self.base_url}/page/{i}'
            yield Request(url, callback=self.parse_index)
    
    def parse_index(self, response):
        for item in response.css('.item'):
            href = item.css('.name::attr(href)').extract_first()
            url = response.urljoin(href)
            yield Request(url, callback=self.parse_detail)
    
    def parse_detail(self, response):
        item = MovieItem()
        item['name'] = response.xpath('//div[contains(@class, "item")]//h2/text()').extract_first()
        item['categories'] = response.xpath('//button[contains(@class, "category")]/span/text()').extract()
        item['score'] = response.css('.score::text').re_first('[\d\.]+')
        item['drama'] = response.css('.drama p::text').extract_first().strip()
        item['directors'] = []
        directors = response.xpath('//div[contains(@class, "directors")]//div[contains(@class, "director")]')
        for director in directors:
            director_image = director.xpath('.//img[@class="image"]/@src').extract_first()
            director_name = director.xpath('.//p[contains(@class, "name")]/text()').extract_first()
            item['directors'].append({
                'name': director_name,
                'image': director_image
            })
        item['actors'] = []
        actors = response.css('.actors .actor')
        for actor in actors:
            actor_image = actor.css('.actor .image::attr(src)').extract_first()
            actor_name = actor.css('.actor .name::text').extract_first()
            item['actors'].append({
                'name': actor_name,
                'image': actor_image
            })
        yield item
```

items

```
class MovieItem(Item):
    name = Field()
    categories = Field()
    score = Field()
    drama = Field()
    directors = Field()
    actors = Field(
```

pipelines

定义MongoPipeline和重写Image下载的Pipeline（未保存到本地，有问题？）

```python
from itemadapter import ItemAdapter
from elasticsearch import Elasticsearch

import pymongo

from scrapyitempipelinedemo.items import MovieItem

class MongoDBPipeline(object):
    
    @classmethod
    def from_crawler(cls, crawler):
        cls.connection_string = crawler.settings.get('MONGODB_CONNECTION_STRING')
        cls.database = crawler.settings.get('MONGODB_DATABASE')
        cls.collection = crawler.settings.get('MONGODB_COLLECTION')
        return cls()
    
    def open_spider(self, spider):
        self.client = pymongo.MongoClient(self.connection_string)
        self.db = self.client[self.database]
    
    def process_item(self, item, spider):
        self.db[self.collection].update_one({
            'name': item['name']
        }, {
            '$set': dict(item)
        }, True)
        return item
    
    def close_spider(self, spider):
        self.client.close()

from scrapy import Request
from scrapy.exceptions import DropItem
from scrapy.pipelines.images import ImagesPipeline


class ImagePipeline(ImagesPipeline):
    def file_path(self, request, response=None, info=None):
        movie = request.meta['movie']
        type = request.meta['type']
        name = request.meta['name']
        file_name = f'{movie}/{type}/{name}.jpg'
        print("filename")
        return file_name
    
    def item_completed(self, results, item, info):
        image_paths = [x['path'] for ok, x in results if ok]
        if not image_paths:
            raise DropItem('Image Downloaded Failed')
        return item
    
    def get_media_requests(self, item, info):
        for director in item['directors']:
            director_name = director['name']
            director_image = director['image']
            yield Request(director_image, meta={
                'name': director_name,
                'type': 'director',
                'movie': item['name']
            })
        
        for actor in item['actors']:
            actor_name = actor['name']
            actor_image = actor['image']
            yield Request(actor_image, meta={
                'name': actor_name,
                'type': 'actor',
                'movie': item['name']
            })
```

setting

设置优先级和部分参数

```python
ITEM_PIPELINES = {
    'scrapyitempipelinedemo.pipelines.ImagePipeline': 300,
    'scrapyitempipelinedemo.pipelines.MongoDBPipeline': 301
    # 'scrapyitempipelinedemo.pipelines.ElasticsearchPipeline': 302,
}

MONGODB_CONNECTION_STRING = "10.182.61.118"
MONGODB_DATABASE = "movies"
MONGODB_COLLECTION = "movies"


IMAGES_STORE = './images'
```



## Extension

自定义添加和扩展部分功能，如：

- LogStats：记录基本爬取信息

- CoreStats：统计爬取的核心统计信息

https://docs.scrapy.org/en/latest/topics/extensions.html

通过settings.py控制启用

实现自定义Extension：

- 实现一个Python类，然后实现对应的处理方法，如实现一个 spider_opened方法用于处理Spider开始爬取时执行的操作，可以接收一个spider参数并对其进行操作。
- 定义from_crawler类方法，其第一个参数是cls类对象，第二个参数是crawler。利用crawler的signals对象将Scrapy的各个信号和已经定义的处理方法关联起来。

### 举例

新建extension.py

```python
from scrapy import signals


NOTIFICATION_URL = 'http://localhost:5000/notify'


class NotificationExtension(object):

    @classmethod
    def from_crawler(cls, crawler): #将其他方法与对应的Scrapy信号关联
        ext = cls()
        crawler.signals.connect(
            ext.spider_opened, signal=signals.spider_opened)
        crawler.signals.connect(
            ext.spider_closed, signal=signals.spider_closed)
        crawler.signals.connect(ext.item_scraped, signal=signals.item_scraped)
        return ext

    def spider_opened(self, spider):
        requests.post(NOTIFICATION_URL, json={
            'event': 'SPIDER_OPENED',
            'data': {
                'spider_name': spider.name
            }
        })

    def spider_closed(self, spider):
        requests.post(NOTIFICATION_URL, json={
            'event': 'SPIDER_OPENED',
            'data': {
                'spider_name': spider.name
            }
        })

    def item_scraped(self, item, spider):
        requests.post(NOTIFICATION_URL, json={
            'event': 'ITEM_SCRAPED',
            'data': {
                'spider_name': spider.name,
                'item': dict(item)
            }
        })
```

设置完成之后在settings.py修改优先级，如100

## Scrapy对接Selenuim

### 原理

Downloader Middleware的process_request方法中，当返回为Response对象时，更低优先级的Downloader Middleware的process_request和 process_exception方法不会被继续调用，每个Downloader Middleware的process_response方法转而被依次调用。调用完之后直接将Response对象发送给Spider来处理。

即返回Response对象后，process_request接收的Request对象不传给Spider处理，而是由process_response方法处理，Spider只解析结果。

### 举例

spider

```python
class BookSpider(Spider):
    name = 'book2'
    allowed_domains = ['spa5.scrape.center']
    base_url = 'https://spa5.scrape.center'

    def start_requests(self):
        """
        first page
        :return:
        """
        start_url = f'{self.base_url}/page/1'
        logger.info('crawling %s', start_url)
        yield Request(start_url, callback=self.parse_index)

    def parse_index(self, response):
        """
        extract books and get next page
        :param response:
        :return:
        """
        items = response.css('.item')
        for item in items:
            href = item.css('.top a::attr(href)').extract_first()
            detail_url = response.urljoin(href)
            yield Request(detail_url, callback=self.parse_detail, priority=2)

        # next page
        match = re.search(r'page/(\d+)', response.url)
        if not match:
            return
        page = int(match.group(1)) + 1
        next_url = f'{self.base_url}/page/{page}'
        yield Request(next_url, callback=self.parse_index)

    def parse_detail(self, response):
        """
        process detail info of book
        :param response:
        :return:
        """
        name = response.css('.name::text').extract_first()
        tags = response.css('.tags button span::text').extract()
        score = response.css('.score::text').extract_first()
        price = response.css('.price span::text').extract_first()
        cover = response.css('.cover::attr(src)').extract_first()
        tags = [tag.strip() for tag in tags] if tags else []
        score = score.strip() if score else None
        item = BookItem(name=name, tags=tags, score=score,
                        price=price, cover=cover)
        yield item
```

item

```python
from scrapy.item import Item,Field

class BookItem(Item):
    name = Field()
    tags = Field()
    score = Field()
    cover = Field()
    price = Field()
```

middlewares

```python
from itemadapter import is_item, ItemAdapter
from scrapy.http import HtmlResponse
from selenium import webdriver
import time


class SeleniumMiddleware(object):
    
    def process_request(self, request, spider):
        url = request.url
        browser = webdriver.Chrome()
        browser.get(url)
        time.sleep(5)
        html = browser.page_source
        browser.close()
        return HtmlResponse(url=request.url,
                            body=html,
                            request=request,
                            encoding='utf-8',
                            status=200)
```

settings

```python
DOWNLOADER_MIDDLEWARES = {
    'gerapy_selenium.downloadermiddlewares.SeleniumMiddleware': 543,
}
```

### GerapySelenium

https://github.com/Gerapy/GerapySelenium

scrapy的selenium支持包

安装

```
pip3 install gerapy-selenium
```

**使用**

在Downloader Middleware和Spider中将Request改为SelenuimRequest

设置代理可以在Spider的SelenuimRequest中添加proxy参数

wait_for参数可以等待某个指定节点加载出

如`yield SeleniumRequest(start_url, callback=self.parse_index,wait_for='.item .name' , proxy='127.0.0.1:7890')`

setting相关配置举例

- GREAPY_SELENIUM_HEADLESS = True #无头模式
- GREAPY_SELENIUM_IGNORE_HTTPS_ERRORS = True #忽略https错误
- GREAPY_SELENIUM_PRETEND = Fasle #webdriver反屏蔽
- GREAPY_SELENIUM_DOWNLOAD_TIMEOUT = 60 #超时时间

 

