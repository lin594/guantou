# 乡声集盒 API v1 设计说明

> 状态：预期设计，尚未冻结。字段级契约以同目录 [`openapi.yaml`](openapi.yaml) 为准。

## 1. 设计目标

v1 支持“可查、可录、可校验”的方言语音资料闭环：用户按写法或义项检索，查看不同方言下的读音与实录；也可以先保存录音，再由社区通过铭牌补充写法、义项和证据。

核心关系如下：

```text
Package（规范化写法） N ─ N Flavor（标准化义项）
          │                    │
          └──── Pronunciation ─┘
                    │
                 Dialect
                    │
                  Can[]

Can 1 ─ N Nameplate（用户对录音的 Package + Flavor 主张）
Shelf N ─ N Flavor / Can
```

`Package`、`Flavor`、`Dialect` 和 `Pronunciation` 是可复用的词典资料；`Can` 是一次具体录音；`Nameplate` 是可追溯的社区判断。规范化资料不能覆盖或替代原始证据。

## 2. 通用约定

### 路径与数据格式

- 业务资源直接挂在根路径，统一保留尾部 `/`。
- 请求与响应使用 UTF-8 JSON；文件上传使用 `multipart/form-data`。
- 时间使用 RFC 3339，例如 `2026-08-06T10:30:00+08:00`。
- 创建成功返回 201，删除成功返回 204，普通读取或更新返回 200。
- 分页查询使用 `page` 和 `page_size`，`page_size` 默认 15、最大 100。

```json
{
  "count": 1,
  "next": null,
  "previous": null,
  "results": []
}
```

### 认证与错误

写接口使用：

```http
Authorization: Bearer <token>
```

统一错误结构只保留一个人类可读字段：

```json
{
  "code": 401,
  "message": "请先登录",
  "data": {},
  "request_id": "7f2c1a90-..."
}
```

- `code` 是数字，必须等于 HTTP 状态码。
- `message` 供用户或开发者阅读。
- `data` 保存字段校验错误或冲突上下文；没有附加信息时为空对象。需要区分同一状态码下的业务原因时使用稳定的 `data.reason`，字段错误统一放入 `data.fields`，不再给顶层增加第二套 code。
- `request_id` 用于排障，并通过响应头 `X-Request-ID` 同步返回。
- 401 表示未认证或凭证失效；403 表示身份有效但无权操作；409 表示资源状态或唯一约束冲突。

## 3. 方言树

`Dialect` 只表达方言之间的从属关系，不额外维护行政区划树。地区名称只有在代表一种可区分的地方话时才进入树：

```text
闽语
└─ 莆仙语
   ├─ 莆田片
   └─ 仙游片
      └─ 游洋话
```

每个节点包含稳定 `id`、同级唯一 `code`、`parent_id` 和只用于展示的 `kind`。规范限定名按从具体到宽泛的顺序生成，例如 `youyang.xianyou.puxian.min`。限定名可用于解析和展示，但外键始终使用稳定 ID；改名或重新归类后，旧限定名保留为 alias。

- `GET /dialects/?parent_id=...` 只返回直接子节点。
- `GET /dialects/{id}/?expand=ancestors,children` 按需展开关系。
- `GET /dialects/resolve/?qualified_code=...` 把限定名或 alias 解析为稳定节点。
- 其他资源按 `dialect_id` 筛选时默认精确匹配；显式传 `dialect_scope=subtree` 才包含全部后代。

资料只能确定到上层方言时直接关联上层节点，不能为了填满层级虚构县镇节点。详见方言与读音模型 [ADR](../../adr/0001-dialect-pronunciation-model.md)。

## 4. 写法、义项与读音

### Package：写法入口

`Package` 保存规范化书写形式，例如“行”“月娘”“hing2”。`package_type` 区分正字、借字、俗写、拟音、罗马字和未定写法。同一写法可以连接多个义项。

### Flavor：义项

`Flavor` 保存跨方言比较使用的标准化语义，例如“行走动作”“银行机构”“月亮”。同一个义项可以连接多个写法。可选 `concepticon_id` 用于与外部概念集对齐，但外部编号不能替代本项目的稳定 ID 和中文定义。

Flavor 与 Package 之间通过 `FlavorPackage` 关联，并保留 `primary / synonym / borrowed / disputed`（主写法、同义写法、假借、争议）及说明。写入 Flavor 时使用结构化的 `package_links`，不能把这一关系降级成没有语义的 ID 数组。这个中间表只负责写法—义项关系；Pronunciation 仍按下述三个外键直接存储。

### Pronunciation：带语义消歧的方言读音

`Pronunciation` 替代含义不清的 `FlavorVariant`，准确表示：

> 这个 `Package` 在表达这个 `Flavor` 时，在这个 `Dialect` 下读作什么。

```json
{
  "package_id": 12,
  "flavor_id": 34,
  "dialect_id": 56,
  "ipa": "hiŋ²³",
  "romanization": "hing2",
  "tone_value": "23",
  "reading_type": "colloquial",
  "sandhi_info": {},
  "source_citation": "田野调查记录"
}
```

虽然 Pronunciation 分别保存三个外键，服务端仍须验证 `package_id + flavor_id` 已存在关联。该三元组不唯一：同一方言可以存在文读、白读、代际差异或争议读音。每个 `reading_type` 最多有一条 `is_canonical=true` 的推荐记录，其他记录继续保留。

Pronunciation 不保存音频 URL。多条 `Can` 可以关联同一条 Pronunciation，作为它的实录证据；尚未完成词典分析的 Can 可以只关联 Dialect，稍后通过 Nameplate 归入已有或新建的 Pronunciation。

### “行”的示例

```text
Package「行」
├─ Flavor「行走动作」
│  ├─ Pronunciation（莆田片，读音 A）
│  │  └─ Can #101 / #102
│  └─ Pronunciation（仙游片，读音 B）
│     └─ Can #205
└─ Flavor「行业类别」
   └─ Pronunciation（另一读音）
```

这样既能表达“同一个义项在不同方言中读音不同”，也能表达“同一个写法因义项不同而读音不同”。

## 5. 罐头与铭牌

### 创建罐头

```http
POST /cans/
```

自由装罐只要求录音、普通话概念和方言。用户可以同时提交 `initial_nameplate`；服务端必须在同一事务中创建或关联 Package、Flavor、Nameplate，避免生成半成品数据。

```json
{
  "audio_url": "https://example.com/audio.mp3",
  "dialect_id": 56,
  "concept_text": "行走",
  "source_note": "本人记忆，家中长辈确认",
  "initial_nameplate": {
    "text_content": "行",
    "definition": "走路",
    "package_type": "orthodox",
    "evidence_level": 1,
    "source_citation": "本人记忆"
  }
}
```

为已有词典资料补录音时传 `pronunciation_id`；如果只确定义项而无法确定写法或读音，则传 `flavor_id` 和 `dialect_id`，录音先进入待贴牌状态。

### 铭牌

铭牌保留 `package_id` 与 `flavor_id`，因为它记录用户对某条 Can 的主张，而不是已确认的词典事实。第一张有效铭牌可以成为主铭牌；不同用户提出的写法、义项、释义和证据可并存。

```http
POST /cans/{can_id}/nameplates/
```

```json
{
  "package_id": 12,
  "flavor_id": 34,
  "text_content": "行",
  "definition": "走路",
  "evidence_level": 1,
  "source_citation": "本人记忆"
}
```

用户支持铭牌使用幂等资源：

```http
PUT    /nameplates/{id}/support/
DELETE /nameplates/{id}/support/
```

## 6. 查询与响应深度

- `*Ref`：嵌入其他资源的最小引用。
- `*Card`：列表、搜索和集盒中的展示数据。
- `*Detail`：资源详情页所需数据。

嵌套对象只能使用 Ref 或 Card；Detail 不允许递归嵌套 Detail。例如 FlavorDetail 可以返回 PronunciationCard 列表，但 PronunciationCard 中的 Flavor、Package 和 Dialect 必须是 Ref。

聚合搜索：

```http
GET /search/?q=行&limit=8
GET /search/suggest/?q=行&limit=8
GET /search/hot/?limit=10
```

搜索按 `flavors`、`packages`、`cans` 分组。单资源筛选继续使用资源列表，例如：

```http
GET /pronunciations/?package_id=12&flavor_id=34&dialect_id=56
GET /pronunciations/?flavor_id=34&dialect_id=2&dialect_scope=subtree
GET /cans/?pronunciation_id=78
```

## 7. 状态与权限

Can 的状态保持为 `unlabeled / pending / tentative / verified / disputed / rejected`，通过以下端点流转：

```http
POST /cans/{id}/transition/
```

Pronunciation 使用 `draft / verified / disputed / rejected`，认证与驳回由 reviewer 或 admin 执行。非法状态转换返回 409，权限不足返回 403。

审核 Pronunciation 使用 `POST /pronunciations/{id}/transition/`。`verify` 时 reviewer 可传 `is_canonical=true`；服务端须在同一事务中取消相同 `package_id + flavor_id + dialect_id + reading_type` 下旧记录的 canonical 标记，保证最多一条推荐读音。

通用权限：

- 匿名用户可读公开资料。
- 登录用户可创建 Can、Nameplate、Shelf，以及提交 Package、Flavor、Pronunciation 候选。
- 创建者可修改自己的用户生成内容；共享词典资料进入 verified 后只允许 reviewer 或 admin 修改。
- Dialect 树的新增、改名和重新归类只允许 admin，避免限定名和后代查询被任意破坏。

## 8. 用户隐私

`GET /users/{id}/` 只返回公开档案。邮箱、电话、生日、微信绑定状态和登录时间只从 `GET /users/me/` 返回；修改本人资料也统一使用 `/users/me/`。不提供公开的邮箱筛选或通过用户名返回完整邮箱的接口。
