# mockup说明文档

##概要
```
    * 配host，可参考我上次群里发的那个。
    HOSTS文件加一个本地映射  ===》  127.0.0.1 local-newapi.ddicar.com

    * 启动UI服务
    终端切换到项目根目录。执行 ===》 NODE_ENV=local npm start
    apiBase变为local-newapi.ddicar...
    local个性化位于 =》 platform-ui-v3/config.json、mock-api script

    * 启动API服务
    然后定位到终端窗口快捷键 ⌘ + N 来新开终端窗口。两个窗口，一个查看代理日志，一个查看UI日志。
    连接测试服务器【staging】执行： npm run mock-api
    连接其他同事机器 执行：./node_modules/.bin/mockup-server -x http://同事IP:4080【这条命令已实测】

    [这里不指定端口会默认去请求https的443端口，会报connect ECONNREFUSED ip:443 这类的错误。mock的会报80的错。本地局域网和后端连调还是不要走https。否则v3/api/下的请求都会报404, mock日志报socket hang up。]
    [此方法启动很快🤣🤗，只是执行api域变更操作，不涉及UI。一旦启动，不需要重启这个服务，新建文件夹、r/w本地api文件神马的。刷新页面，自动生效]

    ***
    目前发现的 常见 http状态码
    200 OK  ==> 服务器成功处理了请求（这个是我们见到最多的）
    204 No Content  ==>  响应中包含一些响应头和一个状态行，不存在消息主体。
    304 Not Modified ==> 就是说要客户端使用缓存。当前客户端的缓存资源是最新的
    400 Bad Request ==>  告诉客户端，它发送了一个错误的请求。
    401 Unauthorized ==>   需要客户端对自己认证[授权]
    403 Forbidden ==>  请求被服务器拒绝了
    404 Not Found ==> 未找到资源,请求了一个服务端没有的不知道神马的东西。
    409 Conflict ==>  发出的请求在资源上造成了一些冲突
    5xx ==> 服务端【api或者nginx】无法处理，完后就抛出错误，就是无法对客户端发起的请求提供服务的意思，服务器看不懂请求。
    常见的502表示代理服务器遇到了上游的无效响应, 无效响应意味着重试过后的超时。
```

---

## 详细说明

##### 请在项目根目录执行  `./node_modules/.bin/mockup-server` 的相关命令。
---

#### 命令行参数的用法

 > `-p, --port` ->   ***[字符串类型]***      本地服务监听端口   [默认: 4080]

 > `-o, --origin` ->   这个主要是突破浏览器的限制。以便可以从脚本内发起跨域的HTTP请求。使用请求头里面的 Origin 和 服务端响应头的 Access-Control-Allow-Origin 来完成最简单的访问控制。星号表示允许来自各种域的各种无语请求的访问不受限制。 [默认: "*"]

 > `-b, --base` ->   用于定位API文件所在的根目录。相对于当前工作目录 => process.cwd() [默认API文件夹根目录名称为 "mockup-data"]

 > `-c, --context` ->   ***[字符串类型]***      API上下文    [默认值: ""]
 >
 > >  _比如该参数值是 mock , 并且请求路径是 /mock/foo/bar.json。那么服务器就会定位到api文件夹里面「-b 对应的目录」的 "foo/bar.json" 这个json文件。_
 > >>>> 多说一句。还是在api根目录mkdir -p /v3/api 比较好。因为如果启动参数添加了-c '/v3/api' 这个东西。那之前比如测服已有的参数接口的路径就乱啦。

 > `-f, --fallback` ->  ***[字符串类型]***   请求路径中出现不匹配节点的的降级解决方案。[默认是“_”]。   如果路径中有某个节点匹配不到，那么mock服务会自动将这个不存在节点的名字替换为 -f 参数指定的名称【fallback name】。然后会在api文件夹【默认是mockup-data】里尝试去定位文件，看看替换后的文件路径是否存在。
 >
 > > _e.g.比如请求"/foo/wuyu/bar.json"发现`wuyu`这个文件夹不存在，这里将会去自动请求/foo/\_/bar.json这个文件。如果/foo/\_/bar.json也找不到，然后呢？那就没有然后啦。😒_
 >
 > `-e, --enable-query` -> ***[布尔值类型]*** 是否接受来自请求的查询字符串【querystring】 默认不允许
 >
 > 如果此处设为允许，举个🌰，"/foo/bar.json?a=1&b=2" 这个查询字符串将会被处理并定位到 api目录中的"foo/bar/a=1&b=2.json" 这个文件上。否则url查询无效, 会被忽略。
 >
 > `-i, --ignore-queries` -> ***[默认空数组]***  将按照 请求url中 查询字符串参数的名字 来进行忽略，参数名字用空格分开。
 >
 > `-t, --content-type` -> ***[字符串类型]***  这里是设置http响应头里面的content-type实体头部。 是要告诉接收的一方，实际返回数据的内容类型是什么，需要用什么方式去处理这种类型的数据内容【media-type、charset、boundary】。默认请求头为"application/json; charset=utf-8"。
 >
 > `-x, --proxy-host` ->  未命中的请求【miss matched request】将被代理到代理主机。多个主机名，用空格分开。
 >
 > 默认的mock-api启动参数里面定义的是去连接测试服务器【staging】。如果和局域网backend连跳。-x ip:4080
 >
 >>> 举个🌰说下。
 >>>
 >>> proxy - 304 - http://192.168.3.38:4080 - GET - http://local-newapi.ddicar.com:4080/v3/api/transport_reports?date_range=2017-06-01T00%3A00%3A00%2B08%3A00%2C2017-06-19T23%3A59%3A59%2B08%3A00&page=1&limit=20
 >>> 👆  上面这个表示客户端被代理到192.168.3.38这个代理服务器上面。代理服务器的功能各有不同。目前backend会在本地构建数据库和服务供ui访问，已经算一个比较完整的服务端。
 >>> ******
 >>> 204 - 0ms - OPTIONS - http://local-newapi.ddicar.com:4080/v3/api/wuyu -
 >>> 200 - 0ms - GET - http://local-newapi.ddicar.com:4080/v3/api/wuyu - /Users/songzhuolun/工作库/DDicar/platform-ui-v3/mockup-data/v3/api/wuyu
 >>> 👆  上面这个是说我请求 api/wuyu。但是代理服务搞不了。所以就读到了本地的API文件夹里面的对应文件。然后返回200。【本地模拟了请求返回的报文】

 >
 > `-h, --help` -> 帮助
 >
 > `-v, --version` -> 查看版本

 ***
 ---
 >  如果请求体的类型[type/method]不是`GET`。mockup服务会优先尝试定位到api文件夹里的带请求类型*「比如GET（SELECT）、POST（CREATE）、PUT（UPDATE）、PATCH（UPDATE）、DELETE（DELETE）、HEAD（获取资源的元数据）、OPTIONS（获取信息，关于资源的哪些属性是客户端可以改变的）等等」*后缀的文件上。这里多唠叨一句，请求貌似都是跨域的，先option探路，然后get/post大军后面再跟上。
 >  `mockup默认支持GET,HEAD,PUT,PATCH,POST,DELETE`
 >
 > 举个🌰～ 👀
 >
 > 比如请求类型是`put`, 请求路径是"/foo/bar.json"。那么如果API文件夹里有“/foo/bar.put.json”这个文件，那么就会优先被命中。 如果这个api文件不存在，将降级请求到api文件夹里面的/foo/bar.json 这个文件上。


## No picture no BB 🙄
