---
title: 【爬虫】使用baiduspider实现一个异步爬虫
date: 2022-5-14
tags: [python,爬虫]
cover: https://kakaluoto-hexo-blog.oss-cn-guangzhou.aliyuncs.com/img/202212171656832.webp
mathjax: true
---
代码仓库：https://github.com/Kakaluoto/asnyc_spider
# 爬虫数据收集
## 1. 各文件说明
- img:存放爬取的图片
- async_spider:异步爬虫类
- data_collector:爬虫执行脚本，从这里启动
- ip_pool:代理IP池,管理可用的代理IP
- image_selector:图片筛选器，将爬取的不符合要求的图片删除

## 2. 涉及到的一些工具和项目
- 爬虫工具:  BaiduSpider: https://github.com/BaiduSpider/BaiduSpider
- 代理IP池:  ProxyPool 代理IP池: https://github.com/jhao104/proxy_pool
- 随机生成useragent:  pip install fake-useragent
- 异步http请求,异步文件读写:  aiohttp, aiofiles, asyncio



## 3. 相关环境依赖

### 3.1 安装baiduspider

```shell
pip install baiduspider
```

在路径site-packages/baiduspider下找到\__init__.py文件源代码第506行的_

content = self._get_response(url, proxies)  # original version

```python
try:
    url = "http://image.baidu.com/search/flip?tn=baiduimage&word=%s&pn=%d" % (
        quote(query),
        (pn - 1) * 20,
    )
    content = self._get_response(url, proxies)  # original version
    result = self.parser.parse_pic(content)
    result = result if result is not None else self.EMPTY
```

添加输入参数verify: bool = True，修改为content = self._get_response(url, proxies,verify: bool = True)

修改结果如下

```python
try:
    url = "http://image.baidu.com/search/flip?tn=baiduimage&word=%s&pn=%d" % (
        quote(query),
        (pn - 1) * 20,
    )
    content = self._get_response_by_hy(url, proxies,verify=verify)  # original version
    result = self.parser.parse_pic(content)
    result = result if result is not None else self.EMPTY
```

在路径site-packages/baiduspider下找到\_spider.py文代码的第63，77，79行

```python
def _get_response(
    self, url: str, proxies: dict = None, encoding: str = None
) -> str:
    """获取网站响应，并返回源码

        Args:
            url (str): 要获取响应的链接
            proxies (dict): 代理相关设置
            encoding (Union[str, None]): 目标网页编码

        Returns:
            str: 获取到的网站HTML代码
        """
    if proxies is not None:
        response = requests.get(url, headers=self.headers, proxies=proxies)
        else:
            response = requests.get(url, headers=self.headers)
```

在对应位置添加verify参数,修改为如下结果

```python
def _get_response(
    self, url: str, proxies: dict = None, encoding: str = None, verify=True
) -> str:
    """获取网站响应，并返回源码

        Args:
            url (str): 要获取响应的链接
            proxies (dict): 代理相关设置
            encoding (Union[str, None]): 目标网页编码

        Returns:
            str: 获取到的网站HTML代码
        """
    if proxies is not None:
        response = requests.get(url, headers=self.headers, proxies=proxies, verify=verify)
        else:
            response = requests.get(url, headers=self.headers)
```

附自定义header：
spider.headers["Connection"] = "close"
此时可以正常使用search_pic

### 3.2 windows安装ProxyPool

主要参考[python爬虫添加代理ip池ProxyPool (Windows) - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/333433627)

#### 3.2.1 将代码仓库下载至本地

```
git clone git@github.com:jhao104/proxy_pool.git
```
#### 3.2.2 安装代理IP池所需依赖

```
pip install -r requirements.txt
```

#### 3.2.3 安装代理ip存储数据库Redis

- 下载[Redis-x64-3.0.504.zip](https://github.com/microsoftarchive/redis/releases/download/win-3.0.504/Redis-x64-3.0.504.zip)，[Releases · microsoftarchive/redis (github.com)](https://github.com/microsoftarchive/redis/releases)

- 启动服务：cmd进入文件位置(文件夹地址栏输入cmd)，启动服务

  执行redis-server.exe redis.windows.conf

- 安装服务：另外打开一个cmd窗口，输入命令安装Redis到windows服务

  redis-server --service-install redis.windows.conf

- 启动服务：关闭第一个“启动服务”cmd窗口，另开一个cmd窗口，启动服务

  redis-server --service-start (启动之后这个窗口就可以关闭了)

- 测试是否可以使用：在文件夹下另开一个cmd窗口，输入命令进行测试

  redis-cli.exe -h 127.0.0.1 -p 6379

  输出

  ```bash
  E:\Redis\Redis-x64-3.2.100>redis-cli.exe -h 127.0.0.1 -p 6379
  127.0.0.1:6379> set name 123
  OK
  127.0.0.1:6379>
  ```

​		测试成功，Redis安装完成，ProxyPool相关依赖全部完成，接下来修改ProxyPool配置，启动ProxyPool。

#### 3.2.4 修改Proxypool配置文件setting.py：主要需要修改两处

- DB_CONN：Redis数据库位置，注意ip和端口，127.0.0.1:6379

- FROXY_FEYCHER：可用的代理ip地址，参考github上实时代理源有效信息          

  ```python
  
  # ############### server config ###############
  HOST = "0.0.0.0"
  
  PORT = 5010
  
  # ############### database config ###################
  # db connection uri
  # example:
  #      Redis: redis://:password@ip:port/db
  #      Ssdb:  ssdb://:password@ip:port
  DB_CONN = 'redis://@127.0.0.1:6379'
  
  # proxy table name
  TABLE_NAME = 'use_proxy'
  
  
  # ###### config the proxy fetch function ######
  PROXY_FETCHER = [
      "freeProxy01",
      "freeProxy02",
      "freeProxy03",
      "freeProxy04",
      "freeProxy05",
      "freeProxy06",
      "freeProxy07",
      "freeProxy08",
      "freeProxy09",
      "freeProxy10"
  ]
  ```

  

#### 3.2.4 启动代理池服务：在proxypool文件夹下，分别打开两个cmd窗口运行命令

启动调度程序：python proxyPool.py schedule

启动webApi服务：python proxyPool.py server



## 4. 可以愉快地开始啦

执行python data_collect.py可以爬取图片