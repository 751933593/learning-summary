### Scarpy

#### 1.爬取数据的流程

![](img\Scrapy\scrapy的爬取流程.jpg)

#### 2.项目实践

- 创建项目

  ```shell
  scrapy startproject 项目名
  ```

- 创建爬虫

  ```powershell
  scrapy genspider 爬虫名 域名
  ```

- 执行爬虫

  ```powershell
  scrapy crawl 爬虫名
  ```

- 编写py脚本执行爬虫

  ```python
  from scrapy import cmdline
  
  cmdline.execute("scrapy crawl shanghai_drug".split())
  ```
  
- 编写item

  ```python
  # 药品名称
  name = scrapy.Field()
  # 剂型
  jixing = scrapy.Field()
  # 医保类型
  jiayilei = scrapy.Field()
  # 城镇职工支付比例
  zhifu = scrapy.Field()
  # 备注
  remark = scrapy.Field()
  ```
  
- 编写spider

  ```python
  def parse(self, response):
  
      data_list = response.xpath("//tr[@bgcolor='#FFFFFF']")
      for data in data_list:
          item = KnowledgeSpiderItem()
          item["name"]=data.xpath(".//td[1]/text()").extract_first()
          item["jixing"] = data.xpath(".//td[2]/text()").extract_first()
          item["jiayilei"] = data.xpath(".//td[3]/text()").extract_first()
          item["zhifu"] = data.xpath(".//td[4]/text()").extract_first()
          item["remark"] = data.xpath(".//td[5]/text()").extract_first()
          yield item
  ```
  
- 编写管道

  - 配置文件settings.py

    ```python
    # Configure item pipelines
    # See https://docs.scrapy.org/en/latest/topics/item-pipeline.html
    # 数字越小，优先级越高
    ITEM_PIPELINES = {
       'KNOWLEDGE_SPIDER.pipelines.KnowledgeSpiderPipeline': 300,
    }
    ```

  - 处理数据

    ```python
    def process_item(self, item, spider):
        print(item)
        return item
    ```
    
#### 3.反爬措施

##### 3.1 随机User-Agent

```python
class UADownloaderMiddleware(object):
    
    # 可以去www.useragentstring.com找需要的UA，到httpbin.org/user-agent去验证UA
    UA_LIST = ['Mozilla/5.0 (compatible; U; ABrowse 0.6; Syllable) AppleWebKit/420+ (KHTML, like Gecko)',
               'Mozilla/5.0 (compatible; U; ABrowse 0.6;  Syllable) AppleWebKit/420+ (KHTML, like Gecko)',
               'Mozilla/5.0 (compatible; MSIE 8.0; Windows NT 6.0; Trident/4.0; Acoo Browser 1.98.744; .NET CLR 3.5.30729)',
               'Mozilla/5.0 (compatible; MSIE 8.0; Windows NT 6.0; Trident/4.0; Acoo Browser 1.98.744; .NET CLR   3.5.30729)',
               'Mozilla/4.0 (compatible; MSIE 8.0; Windows NT 6.0; Trident/4.0;   Acoo Browser; GTB5; Mozilla/4.0 (compatible; MSIE 6.0; Windows NT 5.1;   SV1) ; InfoPath.1; .NET CLR 3.5.30729; .NET CLR 3.0.30618)',
               'Mozilla/4.0 (compatible; MSIE 8.0; Windows NT 5.1; Trident/4.0; SV1; Acoo Browser; .NET CLR 2.0.50727; .NET CLR 3.0.4506.2152; .NET CLR 3.5.30729; Avant Browser)',
               'Mozilla/4.0 (compatible; MSIE 7.0; Windows NT 6.0; Acoo Browser; SLCC1;   .NET CLR 2.0.50727; Media Center PC 5.0; .NET CLR 3.0.04506)',
               'Mozilla/4.0 (compatible; MSIE 7.0; Windows NT 6.0; Acoo Browser; GTB5; Mozilla/4.0 (compatible; MSIE 6.0; Windows NT 5.1; SV1) ; Maxthon; InfoPath.1; .NET CLR 3.5.30729; .NET CLR 3.0.30618)',
               'Mozilla/4.0 (compatible; Mozilla/5.0 (compatible; MSIE 8.0; Windows NT 6.0; Trident/4.0; Acoo Browser 1.98.744; .NET CLR 3.5.30729); Windows NT 5.1; Trident/4.0)',
               'Mozilla/4.0 (compatible; Mozilla/4.0 (compatible; MSIE 8.0; Windows NT 5.1; Trident/4.0; GTB6; Acoo Browser; .NET CLR 1.1.4322; .NET CLR 2.0.50727); Windows NT 5.1; Trident/4.0; Maxthon; .NET CLR 2.0.50727; .NET CLR 1.1.4322; InfoPath.2)',
               'Mozilla/4.0 (compatible; MSIE 8.0; Windows NT 6.0; Trident/4.0; Acoo Browser; GTB6; Mozilla/4.0 (compatible; MSIE 6.0; Windows NT 5.1; SV1) ; InfoPath.1; .NET CLR 3.5.30729; .NET CLR 3.0.30618)']


    def process_request(self, request, spider):

        # 随机user-agent
        request.headers['User-Agent'] = random.choice(self.UA_LIST)
        return None
```

##### 3.2 IP代理池

购买代理

- [芝麻代理](http.zhimaruanjian.com)

- [太阳代理](http.taiyangruanjian.com)
- [快代理](www.kuaidaili.com)
- [讯代理](www.xdaili.com)
- [蚂蚁代理](www.mayidaili.com)

```python
class IPProxyPoolDownloaderMiddleware(object):
    
    # 可以去www.useragentstring.com找需要的UA，到httpbin.org/user-agent去验证UA
    IPProxyPool = ['112.12.37.196:53281','122.136.212.132:53281']
    
    def process_request(self, request, spider):

        # 随机ip
        request.meta['Proxy'] = random.choice(self.IPProxyPool)
        return None
```

#### 4.分布式爬虫



​    