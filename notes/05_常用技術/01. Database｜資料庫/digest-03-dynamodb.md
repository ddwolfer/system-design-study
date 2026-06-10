# 05_常用技術 / 01. Database｜資料庫 — DynamoDB — digest (pre-read cache)
> 2026-06-07 pre-read。來源:DynamoDB.pdf。**尚未入庫 KG**(預讀快取,日後上課時才蒸餾)。
> 若內容是簡體或檔名與內容不符,在此註記,但**不要**刪改任何檔案。
>
> ⚠️ **內容為簡體中文**:Gemini 回傳的逐字原文是簡體(资料/键值/纲要/复制 等)。檔名與內容相符(DynamoDB)。**未改檔**,僅註記。

---

## Slide 1
- **Verbatim text**:
DynamoDB
DynamoDB 是 AWS 提供的一个全受管(fully-managed)、高度可扩展的键值服务。
听起来都是行销术语,但这到底是什么意思,为什么它很重要?
• 全受管(Fully-Managed):AWS 负责资料库所有的运维方面,像是硬体配置、
设定、修补、扩展,开发者不需要触碰任何这些配置,可以专注在应用程式开发
上。
• 高度可扩展(Highly Scalable) : DynamoDB 能处理庞大的资料量和流量。它会
自动根据你的应用程式需求向上或向下扩展,不会有任何停机或效能下降。
• 键值(Key-Value): DynamoDB 是 NoSQL 资料库,不使用传统的关联式资料库
模型,而是采用键值模型,让资料的储存和取得更有弹性。
简单说,DynamoDB 非常好上手,又能扩展支援各式各样的应用程式。就系统设计面
试而言,它几乎涵盖了你对资料库的所有需求。它甚至现在已经支援交易
(transactions)了,这让过去对 DynamoDB 最大的批评失去了根据。
值得一提的是,DynamoDB 不是开源的,所以我们没办法像分析 Kafka 或 Redis 那样
深入剖析它的内部原理。我们的重点会放在如何使用它,以及 AWS 透过官方文件和
DynamoDB 论文所揭露的有限资讯。

面试者常问:「面试里可以用 DynamoDB吗?」
答案很简单:问你的面试官!很多面试官会说可以,并期待你知道怎么用它。也有
些面试官希望避开供应商锁定,期待你用开源的替代方案。永远直接问就对了。

资料模型
DynamoDB 的资料组织成表格(table),每个表格包含多个项目(item),代表各别
的记录。这和关联式资料库类似,但有几个为了扩展性和弹性而设计的关键差异。
• Table(表格): DynamoDB 的最顶层资料结构,每个表格都有一个必填的主键
(primary key)来唯一识别其中的项目。表格支援次要索引(secondary
index),让你可以用非主键的属性来查询资料。
• Item(项目):相当于关联式资料库里的「列(row)」,包含一组属性
(attribute)。每个 item 必须有主键,最多可以储存 400KB 的资料(包含所有属
性,像是属性名称就算作大小限制的一部分。)。
• Attribute(属性):构成 item 内容的键值对。类型可以是纯量类型(字串、数
字、布林值)或集合类型(字串集合、数字集合)。属性也可以巢状嵌套,让单一
item 里能有复杂的资料结构。
- **Diagram**: This slide does not contain a diagram.

## Slide 2
- **Verbatim text**:
设定 DynamoDB 非常直观:你可以直接在 AWS 控制台建立表格,然后马上开始插入
资料。和传统的关联式资料库不同,DynamoDB 是无纲要(schema-less)的——你不
需要在插入资料之前先定义纲要。这表示同一个表格里的 item 可以有不同的属性集,
任何时候都可以新增属性,不影响现有的 item。这种无纲要设计提供了极高的弹性,
但需要在应用程式层做仔细的资料验证,因为 DynamoDB 本身不会强制 item 之间的
属性一致性。
```json
{
    "PersonID": 101,
    "LastName": "Smith",
    "FirstName": "Fred",
    "Phone": "555-4321"
},
{
    "PersonID": 102,
    "LastName": "Jones",
    "FirstName": "Mary",
    "Address": {
        "Street": "123 Main",
        "City": "Anytown",
        "State": "OH",
        "ZIPCode": 12345
    }
},
{
    "PersonID": 103,
    "LastName": "Stephens",
    "FirstName": "Howard",
    "Address": {
        "Street": "123 Main",
        "City": "London",
        "PostalCode": "ER3 5K8"
    },
    "FavoriteColor": "Blue"
}
```
每个 item 代表一个用户,有各自的属性。注意有些用户有其他人没有的属性,例如
`FavoriteColor`,这就展示了 DynamoDB 在属性管理上的弹性。
- **Diagram**: JSON 結構展示三個 item,示範 schema-less:101 有 Phone;102 有巢狀 Address 但無 Phone;103 有 Address + FavoriteColor。同一表格的 item 不需要相同屬性集。

## Slide 3
- **Verbatim text**:
虽然 DynamoDB 用 JSON 格式传输资料,但那只是传输格式。DynamoDB 的实际储
存格式是私有的,让用户可以专注在资料建模上,而不需要管底层的物理储存细节。

分区键和排序键
DynamoDB 的表格由主键定义,主键可以由一个或两个属性组成:
分区键(Partition Key):一个单一属性,加上排序键(如果有的话),唯一识别表
格中的每个 item。DynamoDB 用分区键的值来决定 item 在资料库中的物理位置——这
个值会被杂凑(hash),用来决定 item 储存在哪个分区。
排序键(Sort Key,选填):一个额外的属性,和分区键组合在一起形成复合主键。
排序键用来对同一个分区键值下的 item 排序,让你可以在分区内做高效的范围查询和
排序。
在面试中,介绍 DynamoDB 时一定要指定分区键,以及视情况指定排序键。这个选择
对查询效能和资料取得效率至关重要。和任何其他资料库一样,要选择能优化应用程
式最常见查询模式的分区键,并让资料均匀分散在各个分区。如果你需要做范围查询
或排序,就额外指定排序键。
举个例子:如果你在设计一个群组聊天应用,用 `chat_id` 作为分区键、`message_id` 作
为排序键就很合理。这样你可以高效地查询特定聊天群组的所有讯息,并依时间顺序
排列后显示给用户。
注意这里我们用的是单调递增的 `message_id`,而不是时间戳记作为排序键。虽然时间
戳记看起来很直觉,但它不保证唯一性—————同一毫秒内可能有多则讯息被建立。单调递
增的 ID 同时提供了时间顺序和唯一性。ID 可以用以下技术产生:
• 每个分区的自动递增计数器
• UUID v7 (优于UUID v1———时间戳记优先的格式让它作为字串自然可排序,且不
暴露机器的 MAC 地址)
• Snowflake ID
• ULID

底层实际发生了什么?
DynamoDB 结合了杂凑分区和 B-tree 来高效管理资料分散和取得:
分区键的杂凑分区:资料的物理位置由分区键的杂凑值决定。请求路由器查询分区
metadata 服务,把杂凑后的键对应到正确的储存节点。这在概念上类似一致性杂凑
(consistent hashing),但 DynamoDB 使用的是中心化的分区映射和配置服务(而
不是 2007 年原始 Dynamo 论文里描述的 peer-to-peer hash ring)。分区 metadata
服务也负责处理资料成长时分区的自动拆分和合并。
- **Diagram**: This slide does not contain a diagram.

## Slide 4
- **Verbatim text**:
排序键的 B-tree:在每个分区内,DynamoDB 用以排序键为索引的 B-tree 资料结构
来组织 item,让分区内的范围查询和排序取得更有效率。
复合键操作:同时使用两个键查询时,DynamoDB 先用分区键的杂凑找到正确的节
点,再用排序键遍历 B-tree 找到特定的 item。
这个两层架构让 DynamoDB 同时达到水平可扩展性(透过分区)和分区内的高效查询
(透过 B-tree 索引)。这个组合使 DynamoDB 能在处理庞大资料量的同时,仍然对
使用分区键和排序键的查询提供快速、可预测的效能。

次要索引
如果你需要用不是分区键的属性来查询资料呢?这就是次要索引(secondary index)
发挥功用的地方。DynamoDB 支援两种类型的次要索引:
全域次要索引(Global Secondary Index,GSI):使用与表格分区键不同的分区键
(和选填的排序键)建立的索引。GSI 让你能用非表格分区键的属性来查询 item。由
于 GSI 使用不同的分区键,资料会储存在与基底表格完全不同的物理分区上,并分别
被复制。
本地次要索引(Local Secondary Index,LSI):使用与表格主键相同的分区键,但
不同排序键的索引。LSI 让你能在分区内做范围查询和排序。由于 LSI 使用和基底表格
相同的分区键,它们和被索引的 item 储存在相同的物理分区上。
理解 GSI 和 LSI 在物理储存上的差异很重要。GSI 维护自己独立的分区和副本,提供更
大的查询弹性,但需要额外的储存和处理开销。LSI 则和基底表格的 item 储存在一
起,让分区内的查询更有效率,但灵活度有限。
在实务上,这两种索引都只需要在 AWS 控制台或 AWS SDK 里设定就好,DynamoDB
会负责在资料变更时维护和更新这些索引。
什么时候用 GSI?当你需要用非分区键的属性来高效查询资料时。例如,你的聊天应
用有一个讯息表格,主表格的分区键是 `chat_id`,排序键是 `message_id`,这让你能轻松
取得特定聊天群组的所有讯息并按时间排序。但如果你想显示用户在所有聊天室里发
送的所有讯息呢?你就需要一个以 `user_id` 为分区键、`message_id` 为排序键的 GSI。
- **Diagram**: This slide does not contain a diagram.

## Slide 5
- **Verbatim text**:
Main Table
| chat_id | message_id | user_id | num_attachments | content |
| :--- | :--- | :--- | :--- | :--- |
| 1 | 5322 | 123 | 0 | Hello! |
| 2 | 7411 | 124 | 1 | What time is it? |
| 1 | 8425 | 125 | 3 | How are you? |
| 1 | 9327 | 126 | 2 | lol |
Partition Key (under `chat_id`)
Sort Key (under `message_id`)

GSI created from main table with different
primary key and optional sort key

Global Secondary Index
| user_id | message_id |
| :--- | :--- |
| 123 | 5322 |
| 124 | 7411 |
| 125 | 8425 |
| 126 | 9327 |
Partition Key (under `user_id`)
Sort Key (under `message_id`)
Optionally Projected

什么时候用LSI?当你需要在分区内,用与主要排序键不同的属性做范围查询或排序
时。回到聊天应用的例子,我们已经可以在聊天群组内按 `message_id` 排序,但如果我
们想查询某个聊天群组里附件最多的讯息呢?我们可以在 `num_attachments` 属性上建立
一个 LSI,来快速找出附件很多的讯息。一个重要的限制:LSI 只能在建立表格时定
义,之后无法新增或移除,所以要提前规划好。
- **Diagram**: GSI 圖解:Main Table(chat_id=PK, message_id=SK)→ 從主表建立 GSI,改以 user_id=PK、message_id=SK,可選擇投影其他屬性。示範用 user_id 重新組織以支援不同查詢模式。

## Slide 6
- **Verbatim text**:
Main Table
| chat_id | message_id | user_id | num_attachments | content |
| :--- | :--- | :--- | :--- | :--- |
| 1 | 5322 | 123 | 0 | Hello! |
| 2 | 7411 | 124 | 1 | What time is it? |
| 1 | 8425 | 125 | 3 | How are you? |
| 1 | 9327 | 126 | 2 | lol |
Partition Key (under `chat_id`)
Sort Key (under `message_id`)

Local Secondary Index
| chat_id | num_attachments |
| :--- | :--- |
| 1 | 0 |
| 2 | 1 |
| 1 | 3 |
| 1 | 2 |
Partition Key (under `chat_id`)
Sort Key (under `num_attachments`)
Optionally Projected

| | 全域次要索引(GSI) | 本地次要索引(LSI) |
| :--- | :--- | :--- |
| 定义 | 使用与主表格不同的分区键 | 使用与主表格相同的分区键,但不同的排序键 |
| 使用时机 | 需要用非主键属性查询时 | 需要在同一个分区键下使用额外的排序键时 |
| 大小限制 | 无限制 | 每个分区键最多 10GB |
| 吞吐量 | 与基底表格分开计算读写容量 | 与基底表格共用读写容量 |
| 一致性 | 仅支援最终一致性 | 支援最终一致性(预设)和强一致性读取 |
| 建立时机 | 可随时新增或移除 | 只能在建立表格时定义,无法移除 |
| 数量上限 | 每张表格最多20个 | 每张表格最多5个 |
- **Diagram**: LSI 圖解:Main Table 同上(chat_id=PK, message_id=SK);LSI 用相同 PK(chat_id)但不同 SK(num_attachments),可選擇投影。允許在同一 partition 內依別的屬性排序/篩選。

## Slide 7
- **Verbatim text**:
| | 全域次要索引(GSI) | 本地次要索引(LSI) |
| :--- | :--- | :--- |
| 使用范例 | 跨所有分区的全域搜寻,例如在用户资料库里用 email 搜寻 | 分区内的局部搜寻,例如查找某个客户最近的订单 |

底层实际发生了什么?
次要索引由系统自动维护。GSI 作为独立的内部表格实作;LSI则与基底表格共址
(co-located):
GSI:每个 GSI 本质上是一个有自己分区方式的独立表格。当主表格的 item 被新增、
更新或删除,DynamoDB 会非同步地更新 GSI。这允许对所有分区的非主键属性做高
效查询,但意味著 GSI 是最终一致的。
LSI:LSI 和主表格的分区共址,使用相同的分区键。它们在每个分区内维护一个独立
的B-tree 结构,以 LSI 的排序键为索引。LSI 的更新和主表格的写入同步进行,支援
最终一致性(预设)和强一致性读取。

存取资料
DynamoDB 有两种主要的资料存取方式:
扫描操作(Scan):读取表格或索引中的每一个item,以分页的方式返回结果。扫描
在你需要读取整张表格或索引的所有 item 时很有用,但对大型资料集非常低效,因为
需要读取每一个 item,应该尽可能避免使用。
查询操作(Query):根据主键或次要索引键属性来取得 item。查询比扫描更有效
率,因为只读取符合指定键条件的item。查询也可以用来对排序键做范围查询。
和传统 SQL 资料库不同,DynamoDB 的主要介面是透过 AWS SDK 或 AWS 控制台,
而不是一个独立的查询语言。不过 DynamoDB 确实支援 PartiQL,一个 SQL 相容的查
询语言,让你可以使用熟悉的 SELECT、INSERT、UPDATE、DELETE 语法。在底
层,PartiQL 操作会被转换成同样的 DynamoDB 操作,所以它是个方便的语法层,而
不是本质上不同的能力。
SQL 查询:
```sql
SELECT * FROM users WHERE user_id = 101
```
等效的 DynamoDB 查询操作:
```javascript
const params = {
    TableName: 'users',
    KeyConditionExpression: 'user_id = :id',
```
- **Diagram**: This slide does not contain a diagram.

## Slide 8
- **Verbatim text**:
```javascript
    ExpressionAttributeValues: {
        ':id': 101
    }
};

dynamodb.query(params, (err, data) => {
    if (err) console.error(err);
    else console.log(data);
});
```
SQL 全表扫描等效的 DynamoDB 扫描操作:
```javascript
const params = {
    TableName: 'users'
};

dynamodb.scan(params, (err, data) => {
    if (err) console.error(err);
    else console.log(data);
});
```
在使用 DynamoDB 时,你通常要尽一切可能避免昂贵的扫描操作。这正是谨慎的资料
建模发挥作用的地方,像是选对分区键和排序键,确保查询是高效的。
一个重要的细节:查询 DynamoDB 时,预设会读取整个 item。虽然 DynamoDB 支援
`ProjectionExpression` 只回传特定属性,但这只能减少网路传输量,底层仍然读取了完
整的item,你也要支付以 item 完整大小计算的读取容量费用。这和 SQL 的栏位选取
是不同的概念。对于大型 item,要适当地正规化资料,避免每次读取都带回超过你需
要的内容。

CAP 定理
你通常会在面试的非功能性需求阶段做一些关于一致性和可用性的早期决策。因此,
选择符合这些需求的资料库非常重要。
我遇到的大多数面试者,在需要高可用性和扩展性时都选 DynamoDB。这没有错,但
就像传统的「SQL vs NoSQL」之争,这个观念已经有点过时了。
DynamoDB 支援两种读取一致性模型,例如最终一致性和强一致性。重要的是,这不
是表格层级的设定,而是在每个读取请求上个别指定,透过在 `GetItem`、`Query` 或
`Scan` 呼叫里设定 `ConsistentRead=true` 来实现。
- **Diagram**: This slide does not contain a diagram.

## Slide 9
- **Verbatim text**:
最终一致性(预设):除非明确要求,否则每次读取都是最终一致的。这提供了最高
的可用性和最低的延迟,但你可能不会立刻看到最近的写入。DynamoDB 在这个模式
下通常表现为 AP 系统,具备 BASE 特性。
强一致性:设定 `ConsistentRead=true` 后,DynamoDB 确保读取会反映在此次读取之前
所有成功的写入。这需要两倍的读取容量(每 4KB 消耗 1 RCU,而最终一致性只消耗
0.5 RCU),延迟也可能稍高,但保证你看到的是最新资料。
这种每个请求都可以选择的灵活性,意味著你可以在需要强一致性的场景(例如订位
系统)使用 DynamoDB,同时对读取频繁、延迟敏感的路径保持最终一致性。
DynamoDB 也透过 `TransactWriteItems` 和 `TransactGetItems` 支援 ACID 交易,在跨多张
表格的最多 100 个 item 上提供可序列化隔离(serializable isolation)。
重要限制:强一致性读取只支援基底表格和本地次要索引(LSI)。全域次要索引
(GSI)只支援最终一致性读取,在设计需要强一致性的存取模式时要牢记这一点。

底层实际发生了什么?
DynamoDB 的一致性模型透过其分散式架构和复制机制来实现:
最终一致性读取:读取可以由分区复制群组中的任意三个副本之一来服务。由于
leader 节点在确认了 quorum(多数)之后,才非同步地把写入复制给 follower,
follower 可能还没有最新的写入。消耗较少的读取容量(每4KB 0.5 RCU),延迟也
较低。
强一致性读取:读取请求直接路由到分区的 leader 节点。由于所有写入都先经过
leader,它永远有最新的资料。消耗较多的读取容量(每4KB 1 RCU),延迟可能稍
高,且不支援 GSI。

架构与可扩展性
扩展性
DynamoDB 透过自动分片(auto-sharding)和负载平衡来扩展。当一个分区达到容量
上限(大小或吞吐量),DynamoDB 会自动把它拆分,并重新分配资料。杂凑分区确
保了节点之间的均匀分配。
AWS 的全球基础设施进一步强化了这种可扩展性。全域资料表(Global Tables)让
资料可以在多个 Region 之间即时复制,让全球用户都能就近做读写操作,降低延迟、
改善用户体验。DynamoDB 也整合了每个 Region 的多个可用区域(Availability
Zone),确保资料的持久性和服务的连续性。
在面试中设计全球性应用时,提到用 Global Tables 做跨区域复制通常就够了。

容错和可用性
- **Diagram**: This slide does not contain a diagram.

## Slide 10
- **Verbatim text**:
DynamoDB 透过分散式架构和资料复制机制提供高可用性和容错能力。服务自动在一
个 Region 内的多个可用区域之间复制资料,确保资料在硬体故障或网路中断时仍然可
存取。
DynamoDB 自动在一个 Region 内的三个可用区域之间复制资料——这不是用户可设定
的。每个分区维护三个副本(一个 leader 和两个 follower),完全由 AWS 管理。对于
跨 Region 的复制,可以启用 Global Tables 在其他 AWS Region 新增副本。
在底层,每个分区使用 Multi-Paxos 共识,由一个有三个节点的 leader-based 复制群
组管理。Leader 处理所有写入:产生一笔 WAL 记录,发送给 peer 节点,等到
quorum(3个中的2个)确认持久化后,写入就算完成。对于强一致性读取,
DynamoDB 把请求直接路由到 leader;对于最终一致性读取,三个副本的任何一个都
能服务请求,延迟更低,但可能返回稍旧的资料。

安全性
DynamoDB 预设对所有资料静态加密(at rest),资料在不被存取时也是安全的。
DynamoDB 也强制所有 API 呼叫使用 TLS 加密传输中的资料,不需要任何额外设定。
DynamoDB 整合了 AWS IAM,提供对资料的细粒度存取控制。你可以建立 IAM 策
略,指定谁可以存取资料、以及可以执行哪些操作。你也可以使用 VPC 端点(VPC
Endpoint),让你在 VPC 内部安全地存取 DynamoDB,而不需要把资料暴露到公共网
路上。
在面试中,处理敏感用户资料时,提到你了解 DynamoDB 预设静态加密、并透过 TLS
强制传输加密就够了。超过这个范围大概就过头了。

定价模型
定价听起来好像跟面试毫不相干,但请继续看下去,理解定价模型会带出对架构的明
确限制。
DynamoDB 有两种定价模式:随需计费(on-demand)和预置容量(provisioned
capacity)。随需计费按请求次数收费,适合流量不可预测的工作负载。预置容量则
需要用户指定读取和写入容量单位,按小时计费,对于流量可预测的工作负载更划
算,但在低流量期间可能有闲置容量的浪费。
计费基于 AWS 所说的读取容量单位(Read Capacity Unit,RCU)和写入容量单位
(Write Capacity Unit,WCU):
• 1 RCU = 每秒读取最多 4KB 的资料(强一致性读取);最终一致性读取消耗 0.5
RCU
• 1 WCU = 每秒写入最多 1KB 的资料
- **Diagram**: This slide does not contain a diagram.

## Slide 11
- **Verbatim text**:
| 功能 | 费用 | 说明 |
| :--- | :--- | :--- |
| 读取容量单位 (RCU) | 每百万次读取约1.12 美元 (每次4KB) | 每秒提供一次强一致性读取 (≤4KB), 或两次最终一致性读取 |
| 写入容量单位 (WCU) | 每百万次写入约5.62 美元 (每次1KB) | 每秒提供一次写入 (≤1KB) |

费用本身在面试中不是重点,但对数字有基本概念是有用的。每个 DynamoDB 分区最
多支援 3,000 RCU 和 1,000 WCU。这意味著单个分区可以处理每秒 12MB 的读取
(3,000 × 4KB)和每秒1MB 的写入(1,000 × 1KB)。DynamoDB 自动处理分片和
自动扩展,但这些数字对粗略估算很有帮助。
举个例子:假设你要在 DynamoDB 储存 YouTube 的观看次数。每次写入(不管多
小)至少消耗 1 WCU,因为 DynamoDB 的计费会无条件进位到最近的 1KB。每个分
区支援约 1,000 次写入/秒。如果你预期每秒 1,000 万次观看,大约需要 10,000 个分
区。以预置容量定价计算(约 0.00065 美元/WCU-小时),那就是 10,000,000 WCU
× 0.00065 美元/小时 × 24 小时 ≈ 每天 156,000 美元。随需计费会更贵。这种估算能
帮你快速判断你的方案在成本上是否合理。

进阶功能
DAX (DynamoDB Accelerator)
DynamoDB 有一个专门设计的记忆体快取,叫做 DAX (DynamoDB
Accelerator)。这意味著你可能不需要在架构里引入额外的服务(Redis、
Memcached)。
DAX 是专为提升 DynamoDB 效能而设计的快取服务,对读取密集型工作负载提供微秒
级的回应时间。使用 DAX 需要把你的 DynamoDB 客户端换成 DAX 客户端 SDK(支援
Java、.NET、Node.js、Python、Go)——API 是相容的,所以改动很小,但并不是完
全透明的。
DAX 同时作为读取式快取(read-through cache)和写入式快取(write-through
cache)运作:它快取读取结果直接提供给应用程式,也把资料同时写入快取和
DynamoDB。一个重要的细节是:DAX 只对透过 DAX 本身执行的写入自动让快取失
效。如果你直接更新 DynamoDB(绕过 DAX),那些快取的条目可能持续保留到 TTL
过期或被逐出为止。
DAX 维护两个快取:一个是 item 快取(给 GetItem/BatchGetItem 结果),另一个是
查询快取(给 Query 和 Scan 结果)。这两个快取永远都是启用的。重要限制:DAX
不快取强一致性读取——当你透过 DAX 请求强一致性读取时,它会直接把请求传给
DynamoDB,回传结果但不快取。
- **Diagram**: This slide does not contain a diagram.

## Slide 12
- **Verbatim text**:
Streams (串流)
DynamoDB 也内建支援变更资料撷取(Change Data Capture,CDC),透过
DynamoDB Streams 实现。Streams 会捕获表格中 item 的变更,让下游应用程式能
即时处理。表格中发生的任何变更事件(插入、更新、删除),都会被记录为一笔串
流记录供下游消费。
几个常见的使用场景:
与 Elasticsearch 保持一致:DynamoDB Streams 可以用来让 Elasticsearch 索引和
DynamoDB 表格保持同步,这对在 DynamoDB 资料上建构搜寻功能很有帮助。
即时分析:你可以在 DynamoDB 表格上启用 Kinesis Data Streams,然后透过
Kinesis Data Firehose 把变更资料汇入 S3、Redshift 或 OpenSearch 做即时分析。
(注意:Firehose 无法直接读取 DynamoDB Streams,需要 Kinesis Data Streams 或
Lambda 函式作为中介。)
变更通知:你可以用 DynamoDB Streams 触发 Lambda 函式,回应资料库的变更—————
例如发送通知、更新快取,或根据资料变更执行其他操作。

DynamoDB 在面试中的应用
什么时候用它
在面试中,几乎任何持久化层的需求你都可以合理地选择 DynamoDB。它高度可扩
展、持久、支援交易,并提供个位数毫秒的延迟(有了 DAX 甚至是微秒级)。DAX 的
快取功能和 DynamoDB Streams 的跨系统一致性让它更加强大。所以如果面试官允
许,它很可能是个很好的选择。
知道它的限制
不过,知道什么时候不用 DynamoDB 同样重要。以下是几个你可能选择其他资料库的
理由(除了只是对其他技术更熟悉之外):
成本效益:DynamoDB 的定价基于读写操作和储存的资料量,在高流量工作负载下可
能很昂贵。如果你需要每秒数十万次的写入,成本可能远超过好处。
复杂的查询模式:如果你的系统需要复杂的查询,例如 JOIN 或临时聚合(ad-hoc
aggregation),DynamoDB 可能力不从心。DynamoDB 确实支援跨多张表格的交易
(每次交易最多 100 个 item),但缺乏 SQL 资料库灵活的查询能力。
资料建模的限制:DynamoDB 要求仔细的资料建模才能有好的效能,它为键值和文件
结构做了优化。如果你发现自己频繁地使用 GSI 和 LSI,像 PostgreSQL 这样的关联式
资料库可能更合适。
- **Diagram**: This slide does not contain a diagram.

## Slide 13
- **Verbatim text**:
供应商锁定(Vendor Lock-in):选择 DynamoDB 意味著绑定在 AWS 上。很多面试
官希望你保持云端供应商中立,所以你可能需要考虑开源的替代方案来避免被绑定。

总结
DynamoDB 功能多元、强大,而且非常好用。在面试中,它是大多数使用案例的稳固
选择,但你要清楚它的限制,并对它的运作方式有扎实的理解——包括如何选择正确的
资料模型、分区键、排序键、次要索引,以及知道什么时候要启用 DAX 和 Streams 等
进阶功能。
记住:DynamoDB 现在已经支援交易(包括跨多张表格的交易),所以「NoSQL 就是
没有交易」这个旧评论已经过去了。
- **Diagram**: This slide does not contain a diagram.
