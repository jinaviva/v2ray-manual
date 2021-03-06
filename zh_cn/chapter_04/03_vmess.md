# VMess 协议

VMess 是 V2Ray 原创的加密通讯协议。

## 版本
当前版本号为 1。

## 依赖
### 底层协议
VMess 是一个基于 TCP 的协议，所有数据使用 TCP 传输。

### 用户 ID
ID 等价于 [UUID](https://en.wikipedia.org/wiki/Universally_unique_identifier)，是一个 16 字节长的随机数，它的作用相当于一个令牌（Token）。
一个 ID 形如：de305d54-75b4-431b-adb2-eb6b9e546014，几乎完全随机，可以使用任何的 UUID 生成器来生成，比如[这个](https://www.uuidgenerator.net/)。

用户 ID 可在[配置文件](../chapter_02/01_overview.md)中指定。

## 通讯过程
VMess 是一个无状态协议，即客户端和服务器之间不需要握手即可直接传输数据，每一次数据传输对之前和之后的其它数据传输没有影响。
VMess 的客户端发起一次请求，服务器判断该请求是否来自一个合法的客户端。如验证通过，则转发该请求，并把获得的响应发回给客户端。
VMess 使用非对称格式，即客户端发出的请求和服务器端的响应使用了不同的格式。

## 客户端请求
| 16 字节 | X 字节 | 余下部分 |
|---------|----------|--------|
| 认证信息| 指令部分 | 数据部分|

### 认证信息
认证信息是一个 16 字节的哈希（hash）值，它的计算方式如下：
* H = MD5
* K = 用户 ID (16 字节)
* M = UTC 时间，精确到秒，取值为当前时间的前后 30 秒随机值(8 字节, Big Endian)
* Hash = HMAC(H, K, M)

### 指令部分
指令部分经过 AES-128-CFB 加密：
* Key：md5(用户 ID + 'c48619fe-8f02-49e0-b9e9-edf763e17e21')
* IV：md5(X + X + X + X)，X = []byte(认证信息生成的时间) (8 字节, Big Endian)

| 1 字节 | 16 字节   | 16 字节 | 1 字节 | 1 字节 | 2 字节 | 1 字节 | 2 字节 | 1 字节 | N 字节 | 4 字节 |
|---------|----------|---------|--------|--------|--------|--------|--------|--------|--------|--------|
|版本号 Ver|数据加密 IV|数据加密 Key|响应认证 V|选项 Opt|保留|指令 Cmd|端口 Port|地址类型 T|地址 A|校验 F|

其中：
* 版本号 Ver：始终为 1；
* 数据加密 IV：随机值；
* 数据加密 Key：随机值；
* 响应认证 V：随机值；
* 选项 Opt：
  * 0x01：带校验的数据流；
* 指令 Cmd：
  * 0x01：TCP 数据；
  * 0x02：UDP 数据；
* 端口 Port：Big Endian 格式的整型端口号；
* 地址类型 T：
  * 0x01：IPv4
  * 0x02：域名
  * 0x03：IPv6
* 地址 A：
  * 当 T = 0x01 时，A 为 4 字节 IPv4 地址；
  * 当 T = 0x02 时，A 为 1 字节长度（L） + L 字节域名；
  * 当 T = 0x03 时，A 为 16 字节 IPv6 地址；
* 校验 F：指令部分除 F 外所有内容的 FNV1a hash；

### 数据部分
数据部分使用 AES-128-CFB 加密，Key 和 IV 在指令部分中指定

数据部分有两种格式，默认为基本格式。

#### 基本格式
所有数据均认为是请求的实际内容。这些内容将被发往指令部分所指定的地址。当 Cmd = 0x01 时，这些数据将以 TCP 的形式发送；当 Cmd = 0x02 时，这些数据将以 UDP 形式发送。

#### 校验格式
当 Opt = 0x01 时，数据部分使用此格式。实际的请求数据被分割为若干个小块，每个小块的格式如下。服务器校验完所有的小块之后，再按基本格式的方式进行转发。

| 2 字节 | 4 字节 | L 字节 |
|---------|----------|--------|
| 长度 L | 校验信息 | 实际数据|

其中：
* 长度 L：Big Endian 格式的整型；
* 校验信息：实际数据的 FNV1a hash；

## 服务器应答
所有应答数据使用 AES-128-CFB 加密，IV 为 md5(数据加密 IV)，Key 为 md5(数据加密 Key)

| 1 字节 | 1 字节    | 1 字节   | 1 字节 | M 字节 | 余下部分 |
|---------|----------|----------|--------|--------|----------|
|响应认证 V| 保留  |指令 Cmd |指令长度 M|指令内容| 实际应答数据|

其中
* 响应认证 V：必须和客户端请求中的响应认证 V 一致；
* 指令 Cmd：
  * 0x01：动态端口指令
* 实际应答数据：如果请求中的 Opt = 0x01，则使用校验格式，否则使用基本格式。格式均和请求数据相同。

### 动态端口指令
| 1 字节 | 2 字节    | 16 字节   | 2 字节 | 1 字节 | 1 字节 |
|---------|----------|----------|--------|--------|----------|
| 保留  |端口 Port |用户 ID| AlterID | 用户等级 | 有效时间 T |

其中：
* 端口 Port：Big Endian 格式的整型端口号；
* 有效时间 T：分钟数；

客户端在收到动态端口指令时，服务器已开放新的端口用于通信，这时客户端可以将数据发往新的端口。在 T 分钟之后，这个端口将失效，客户端必须重新使用主端口进行通信。
