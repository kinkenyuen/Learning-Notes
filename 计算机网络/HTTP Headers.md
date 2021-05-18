# 目录

   * [HTTP首部](#http首部)
      * [首部概览](#首部概览)
      * [End-to-end首部和Hop-by-hop首部](#end-to-end首部和hop-by-hop首部)
      * [通用首部字段](#通用首部字段)
         * [Cached-Control](#cached-control)
            * [表示是否能缓存的指令](#表示是否能缓存的指令)
            * [控制可执行缓存的对象的指令](#控制可执行缓存的对象的指令)
            * [指定缓存期限和认证的指令](#指定缓存期限和认证的指令)
         * [Connection](#connection)
         * [Trailer](#trailer)
         * [Transfer-Encoding](#transfer-encoding)
         * [Upgrade](#upgrade)
         * [Via](#via)
      * [请求首部字段](#请求首部字段)
         * [Accept](#accept)
         * [Accept-Charset](#accept-charset)
         * [Accept-Encoding](#accept-encoding)
         * [Authorization](#authorization)
         * [Host](#host)
         * [If-Match](#if-match)
      * [响应首部字段](#响应首部字段)
         * [Accept-Ranges](#accept-ranges)
         * [Age](#age)
         * [ETag](#etag)
         * [Location](#location)
         * [Vary](#vary)
         * [WWW-Authenticate](#www-authenticate)
      * [实体首部字段](#实体首部字段)
         * [Allow](#allow)
         * [Content-Encoding](#content-encoding)
         * [Content-Length](#content-length)
         * [Content-MD5](#content-md5)
         * [Content-Range](#content-range)
      * [Cookie使用的首部字段](#cookie使用的首部字段)
         * [Set-Cookie](#set-cookie)
         * [Cookie](#cookie)

# HTTP首部

HTTP Headers 译为 HTTP 首部，在HTTP协议中的请求和响应中必定包含HTTP首部

HTTP首部在通信过程中，起到传递额外重要信息的作用

HTTP首部字段类型可分为以下4种：

* 通用首部字段

  请求或响应报文都可能使用的首部

* 请求首部字段

  客户端发送请求报文使用的首部

* 响应首部字段

  服务器端返回响应报文时使用的首部

* 实体首部字段

  针对请求报文和响应报文的实体部分使用的首部

## 首部概览

通用首部字段

| 首部字段名        | 说明                       |
| ----------------- | -------------------------- |
| Cache-Control     | 控制缓存的行为             |
| Connection        | 逐跳首部、连接的管理       |
| Date              | 创建报文的日期时间         |
| Pragma            | 报文指令                   |
| Trailer           | 报文末端的首部一览         |
| Transfer-Encoding | 指定报文主体的传输编码方式 |
| Upgrade           | 升级为其他协议             |
| Via               | 代理服务器的相关信息       |
| Warning           | 错误通知                   |

请求首部字段

| 首部字段名          | 说明                                        |
| ------------------- | ------------------------------------------- |
| Accept              | 用户代理可处理得媒体类型                    |
| Accept-Charset      | 优先的字符集                                |
| Accept-Encoding     | 优先的内容编码                              |
| Accept-Language     | 优先的语言(自然语言)                        |
| Authorization       | Web认证信息                                 |
| Expect              | 期待服务器的特定行为                        |
| From                | 用户的电子邮箱地址                          |
| Host                | 请求资源所在服务器                          |
| If-Match            | 比较实体标记(ETag)                          |
| If-Modified-Since   | 比较资源的更新时间                          |
| If-None-Match       | 比较实体标记(与If-Match相反)                |
| If-Range            | 资源未更新时发送实体Byte的范围请求          |
| If-Unmodified-Since | 比较资源的更新时间(与If-Modified-Since相反) |
| Max-Forwards        | 最大传输逐跳数                              |
| Proxy-Authorization | 代理服务器要求客户端的认证信息              |
| Range               | 实体的字节范围请求                          |
| Referer             | 对请求中URI的原始获取方                     |
| TE                  | 传输编码的优先级                            |
| User-Agent          | HTTP客户端程序的信息                        |

响应首部字段

| 首部字段名         | 说明                         |
| ------------------ | ---------------------------- |
| Accept-Ranges      | 是否接受字节范围请求         |
| Age                | 推算资源创建经过时间         |
| ETag               | 资源的匹配信息               |
| Location           | 令客户端重定向至指定URI      |
| Proxy-Authenticate | 代理服务器对客户端的认证信息 |
| Retry-After        | 对再次发起请求的时机要求     |
| Server             | HTTP服务器的安装信息         |
| Vary               | 代理服务器缓存的管理信息     |
| WWW-Authenticate   | 服务端对客户端的认证信息     |

实体首部字段

| 首部字段名       | 说明                       |
| ---------------- | -------------------------- |
| Allow            | 资源可支持的HTTP方法       |
| Content-Encoding | 实体主体适用的编码方式     |
| Content-Language | 实体主体的自然语言         |
| Content-Length   | 实体主体的大小(单位：字节) |
| Content-Location | 替代对应资源的URI          |
| Content-MD5      | 实体主体的报文摘要         |
| Content-Range    | 实体主体的位置范围         |
| Content-Type     | 实体主体的媒体类型         |
| Expires          | 实体主体过期的日期时间     |
| Last-Modified    | 资源的最后修改日期时间     |

> 还有一些非正式的HTTP首部字段，例如Cookie、Set-Cookie和Content-Disposition，它们的使用频率很高。这些非正式的首部字段统一归纳在RFC4229中

## End-to-end首部和Hop-by-hop首部

HTTP首部字段根据缓存代理和非缓存代理的行为，分为2种类型

* 端到端首部(End-to-end Header)

  分在此类别的首部会转发给请求/响应对应的最终接收目标，且必须保存在由缓存生成的响应中，另外规定它必须被转发

* 逐跳首部(Hop-by-hop Header)

  分在此类别中的首部只对单次转发有效，会因通过缓存或代理而不再转发。HTTP/1.1和之后的版本中，如果要使用hop-by-hop首部，需要提供Connection首部字段

除了以下8个逐跳首部字段外，其他的首部字段均属于端到端首部

* Connection
* Keep-Alive
* Proxy-Authenticate
* Proxy-Authorization
* Trailer
* TE
* Transfer-Encoding
* Upgrade

## 通用首部字段

### Cached-Control

缓存请求指令

| 指令               | 参数   | 说明                         |
| ------------------ | ------ | ---------------------------- |
| no-cache           | 无     | 强制向源服务器再次验证       |
| no-store           | 无     | 不缓存请求或响应的任何内容   |
| max-age = [秒]     | 必需   | 响应的最大Age值              |
| max-stale( = [秒]) | 可省略 | 接收已过期的响应             |
| min-fresh = [秒]   | 必需   | 期望在指定时间内的响应仍有效 |
| no-transform       | 无     | 代理不可更改媒体类型         |
| only-if-cached     | 无     | 从缓存获取资源               |
| cache-extension    | -      | 新指令标记(token)            |

缓存响应指令

| 指令            | 参数   | 说明                                           |
| --------------- | ------ | ---------------------------------------------- |
| public          | 无     | 可向任意方提供响应的缓存                       |
| private         | 可省略 | 仅向特定用户返回响应                           |
| no-cache        | 可省略 | 缓存前必须先确认其有效性                       |
| no-store        | 无     | 不缓存请求或响应的任何内容                     |
| no-transform    | 无     | 代理不可更改媒体类型                           |
| must-revaildate | 无     | 要求中间缓存服务器对缓存的响应有效性再进行确认 |
| max-age = [秒]  | 必需   | 响应的最大Age值                                |
| s-maxage = [秒] | 必需   | 公共缓存服务器响应的最大Age值                  |
| cache-extension | -      | 新指令标记(token)                              |

#### 表示是否能缓存的指令

* public

  当使用public指令时，表明其他用户也可利用缓存

* private

  缓存服务器会对该特定用户提供资源缓存的服务，对于其他用户发送过来的请求，代理服务器则不会返回缓存

* no-cache

  为了防止从缓存中返回过期的资源，

  如果客户端发送的请求中包含该指令，则表示客户端将不会接收缓存过的响应。于是，缓存服务器必须把客户端请求转发给源服务器

  如果服务器返回的响应中包含no-cached指令，那么缓存服务器不能对资源进行缓存。源服务器以后也将不再对缓存服务器请求中提出的资源有效性进行确认，且禁止其对响应资源进行缓存操作

  `Cache-Control: no-cache=Location`

  服务器返回的响应中，若报文首部字段Cache-Control中对no-cache字段名具体指定参数值，那么客户端在接收到这个被指定参数值的首部字段对应的响应报文后，就不能使用缓存。换言之，无参数值的首部字段可以使用缓存。只能在响应指令中指定该参数

#### 控制可执行缓存的对象的指令

* no-store 

  当使用no-store指令时，暗示请求(和对应的响应)或响应中包含机密信息，因此该指令规定缓存不能在本地存储请求或响应的任一部分

> 从字面意思上很容易把no-cache误解称为不缓存，但事实上no-cache代表不缓存过期的资源，缓存会向源服务器进行有效期确认后处理资源。no-store才是真正的不进行缓存

#### 指定缓存期限和认证的指令

* s-maxage

  它和max-age指令功能相同，不同点是s-maxage指令只适用于供多位用户使用的公共缓存服务器(一般指代理)，也就是说对于向同一用户重复返回响应的服务器来说，这个指令没有任何作用。另外，使用该指令后，则直接忽略对Expires首部字段及max-age指令的处理

* max-age

  当客户端请求包含该指令时，如果判定缓存资源的缓存时间数值比指定时间更小，那么客户端接收缓存的资源。当该值为0，那么缓存服务器通常将请求转发给源服务器

  当服务器返回的响应包含该指令时，缓存服务器将不对资源的有效性再作确认，而max-age数值代表资源保存为缓存的最长时间

* min-fresh

  min-fresh指令要求缓存服务器返回至少还未过指定时间的缓存资源，比如，指定60秒后，过了60秒的资源都无法作为响应返回

* max-stale

  可指示缓存资源，即使过期也照常接收

  如果未指定参数值，那么无论经过多久，客户端都会接收响应

  如果指令中指定了具体数值，那么即使资源过期，只要仍处于max-stale指定的时间内，仍旧会被客户端接收

* only-if-cached

  该指令要求缓存服务器不重新加载响应，也不会再次确认资源有效性，若缓存服务器无响应，则返回状态码504 Gateway Timeout

* must-revalidate

  代理会向源服务器再次验证即将返回的响应是否仍然有效

  若代理无法连通源服务器，缓存必须给客户端一条504 Gateway Timeout状态码

  使用该指令会忽略请求的max-stale指令

* proxy-revalidate

  要求所有的缓存服务器在接收到客户端带有该指令的请求返回响应之前，必须再次验证缓存的有效性

* no-transform

  请求还是响应中，缓存都不能改变实体主体的媒体类型，防止缓存或代理压缩图片等类似操作

### Connection

该字段有以下两个作用：

* 控制不再转发给代理的首部字段

  `Connection:不再转发的首部字段名`

  在客户端发送请求和服务器返回的响应内，使用Connection首部字段，可控制不再转发给代理的首部字段(即Hop-by-hop首部)

* 管理持久连接

  HTTP/1.1版本的默认连接均为持久连接，客户端在持久连接上连续发送请求，当服务器想明确断开连接时，则指定Connection首部字段值为Close

### Trailer

事先说明在报文主体后记录了哪些首部字段，该首部字段可应用在HTTP/1.1版本分块传输编码时

### Transfer-Encoding

规定了传输报文主体时使用的编码方式

HTTP/1.1的传输编码方式仅对分块传输编码有效

### Upgrade

首部字段Upgrade用于检测HTTP协议及其他协议是否可使用更高的版本进行通信

### Via

使用Via是为了追踪客户端与服务器之间的请求和响应报文的传输路径

报文经过代理或网关时，会先在首部字段Via中附加该服务器的信息，然后再进行转发

它不仅用于追踪报文的转发，还可避免请求回环发生

## 请求首部字段

### Accept

通知服务器，用户代理能够处理得媒体类型以及媒体类型的相对优先级

用type/subtype这种形式来表示，一次指定多种媒体类型

* 文本文件

  text/html，text/plain，text/css ...

  application/xhtml+xml，application/xml ...

* 图片文件

  image/jpeg，image/gif，image/png ...

* 视频文件

  video/mpeg，video/quicktime ...

* 应用程序二进制文件

  application/octet-stream，application/zip ...

要给显示的媒体类型增加优先级，则使用q=来额外表示权重，用分号(;)分隔，权重值范围是0~1，默认值为1，当服务器提供多种内容时，将首先返回权重值最高的媒体类型

### Accept-Charset

通知服务器，用户代理支持的字符集以及字符集的相对顺序

可以一次性指定多种字符集

与Accept一样，可以使用权重值来表示优先级

### Accept-Encoding

通知服务器，用户代理支持的内容编码以及内容编码的优先级顺序，可一次性指定多种内容编码

同样也可以使用权重值来表示相对优先级，也可以使用星号(*)作为通配符，指定任意的编码格式

### Authorization

通知服务器，用户代理的认证信息(证书值)，通常，想要通过服务器认证的用户代理会在接收到返回的401状态码响应后，把首部字段Authorization加入请求中

### Host

虚拟主机运行在同一个IP上，因此使用首部字段Host加以区分

Host告知服务器，请求的资源所处的互联网主机名和端口号

Host字段是HTTP/1.1规范内是唯一一个必须被包含在请求内的首部字段

### If-Match

形如If-xxx这种形式的请求首部字段，都可称为条件请求

服务器接收到附带条件的请求后，只有判断指定条件为真时，才会执行处理请求

If-Match通知服务器匹配资源所用的实体标记(ETag)值，这时的服务器无法使用弱ETag值

服务器会比对If-Match的字段值和资源的ETag值，仅当两者一致时，才会执行请求，反之则返回状态码412 Precondition Failed的响应

还可以使用星号(*)指定If-Match的字段值，表示如果服务器资源存在就处理请求

## 响应首部字段

### Accept-Ranges

首部字段Accept-Ranges是用来告知客户端服务器是否能处理范围请求

当字段值为bytes表示可以处理范围请求，当为none则不能处理范围请求

### Age

通知客户端，源服务器在多久前创建了响应

### ETag

实体标识，一种将资源以字符串形式做唯一性标识的方式

### Location

将响应接收方引导至某个与请求URI位置不同的资源

基本上，该字段会配合3xx: Redirection的响应，提供重定向的URI

几乎所有的浏览器在接收到包含该字段的响应后，都会强制性地尝试地对已提示的重定向资源的访问

### Vary

当代理服务器接收到带有Vary首部字段指定获取资源的请求时，如果使用的Accept-Language字段的值相同，那么就直接从缓存返回响应。反之则需要先从源服务器端获取资源后才能作为响应返回

首部字段Vary可对缓存进行控制，源服务器会向代理服务器传达关于本地缓存使用方法的命令

从代理服务器接收到源服务器返回包含Vary指定项的响应之后，若再要进行缓存，仅对请求中含有相同Vary指定首部字段的请求返回缓存。即使对相同资源发起请求，但由于Vary指定的首部字段不相同，因此必须要从源服务器重新获取资源

### WWW-Authenticate

告知客户端适用于访问请求URI所指定资源的认证方案和带参数提示的质询(challenge)

## 实体首部字段

包含在请求报文和响应报文中的实体部分所使用的首部，用于补充内容的更新时间等与实体相关的信息

### Allow

通知客户端能够支持Request-URI指定资源的所有HTTP方法

### Content-Encoding

通知客户端，服务器对实体的主体部分选用的内容编码方式，内容编码是指在不丢失实体信息的前提下所进行的压缩

### Content-Length

表明实体主体部分的大小（单位：字节）

对实体主体进行内容编码传输时，不能再使用Content-Length首部字段

### Content-MD5

目的在于检查报文主体在传输过程中是否保持完整，以及确认传输到达

对报文主体执行MD5算法获得128位二进制数，再通过Base64编码后将结果写入Content-MD5字段值，由于HTTP首部无法记录二进制值，所以要经过Base64编码处理

为确保报文的有效性，作为接收方的客户端会对报文主体再执行一次相同的MD5算法，计算出的值与字段值作比较后，即可判断出报文主体的准确性

采用这种方法，对内容上的偶发性改变是无从查证的，也无法检测出恶意篡改。原因在于，内容如果被篡改，那么同时意味着Content-MD5也可重新计算然后被篡改。所以处在接收阶段的客户端是无法意识到报文主体以及首部字段Content-MD5是已经被篡改过的

### Content-Range

用于范围请求，字段值以字节为单位，表示当前发送部分以及整个实体的大小

## Cookie使用的首部字段

这部分首部没有编入标准化文档，但是在Web网站中广泛使用

Cookie的工作机制是用户识别及状态管理，Web网站为了管理用户的状态会通过Web浏览器，把一些数据临时写入用户的计算机内，接着当用户访问该Web网站时，可通过通信方式取回之前发放的Cookie

调用Cookie时，由于可校验Cookie的有效期，以及发送方的域、路径、协议等信息，所以正规发布的Cookie内的数据不会因来自其他Web站点和攻击者的攻击而泄漏

| 首部字段名 | 说明                           | 首部类型     |
| ---------- | ------------------------------ | ------------ |
| Set-Cookie | 开始状态管理所使用的Cookie信息 | 响应首部字段 |
| Cookie     | 服务器接收到的Cookie信息       | 请求首部字段 |

### Set-Cookie

| 属性         | 说明                                                         |
| ------------ | ------------------------------------------------------------ |
| NAME=VALUE   | 赋予Cookie的名称和其值（必需项）                             |
| expires=DATE | Cookie的有效期（若不明确指定则默认为浏览器关闭前为止）       |
| path=PATH    | 将服务器上的文件目录作为Cookie的适用对象(若不指定则默认为文档所在的文件目录) |
| domain=域名  | 作为Cookie适用对象的域名（若不指定则默认为创建Cookie的服务器的域名） |
| Secure       | 仅在HTTPS安全通信时才会发送Cookie                            |
| HttpOnly     | 加以限制，使Cookie不能被JavaScript脚本访问                   |

### Cookie

首部字段Cookie通知服务器，当客户端想获得HTTP状态管理支持时，就会在请求中包含从服务器接收到的Cookie。接收到多个Cookie时，同样可以以多个Cookie形式发送

