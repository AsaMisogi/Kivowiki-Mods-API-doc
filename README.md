# KivoWiki 自用API文档

本文档为个人学习研究使用的api文档，以便开发Kivowiki-Mods项目的相关模块，并非古书馆官方公开的正式文档，不具有绝对的稳定性和可靠性，仅供参考。

## 1. 快速开始

所有接口均为 `GET` 请求，公共基础地址为：

```text
https://api.kivo.wiki/api/v1
```

例如，搜索姓名中含有“星野”的角色：

```text
https://api.kivo.wiki/api/v1/data/students?page=1&page_size=10&character_data_search=星野
```

直接在浏览器地址栏打开该地址即可查看 JSON。命令行也可以使用：

```bash
curl "https://api.kivo.wiki/api/v1/data/students?page=1&page_size=10&character_data_search=%E6%98%9F%E9%87%8E"
```

JavaScript 示例：

```js
const url = new URL('https://api.kivo.wiki/api/v1/data/students');
url.search = new URLSearchParams({
  page: '1',
  page_size: '10',
  character_data_search: '星野',
});

const response = await fetch(url);
if (!response.ok) throw new Error(`HTTP ${response.status}`);

const result = await response.json();
if (!result.success) throw new Error(result.message || 'KivoWiki API 请求失败');

console.log(result.data.students);
```

浏览器中的网页可能受到跨域（CORS）限制。后端程序、命令行工具通常不受此限制；纯前端项目应使用自己可控的只读后端代理，并对允许访问的路径、参数和响应大小设置白名单。不要搭建不受限制的公共转发服务。

## 2. 先了解返回格式

接口通常返回一个统一外层：

```json
{
  "code": 2000,
  "data": {},
  "message": "OK",
  "success": true,
  "time": 1788173990
}
```

| 字段 | 含义 |
|---|---|
| `success` | 业务是否成功；读取 `data` 前必须检查。 |
| `code` | 业务状态码；已观察到成功值为 `2000`，但完整错误码未公开。 |
| `message` | 状态说明。 |
| `data` | 实际返回的数据。 |
| `time` | 服务端响应时间，Unix 秒时间戳。 |

同时检查 HTTP 状态码和 `success`。不存在的详情通常返回 HTTP 404；HTTP 200 也不应代替业务状态检查。

### 分页

列表接口通常接受：

| 参数 | 说明 |
|---|---|
| `page` | 页码，从 `1` 开始。 |
| `page_size` | 每页条数；没有公开最大值，建议按展示需要使用较小数值。 |

列表的 `data.max_page` 是“最大页码”，不是记录总数。每类列表的数组名不同，例如角色是 `students`，文章是 `article`，不要统一假设为 `items`。

### 图片和媒体地址

资源字段可能是以下形式：

```text
//static.kivo.wiki/images/...
https://static.kivo.wiki/images/...
images/...
```

`//` 开头时补上 `https:`；相对资源路径可拼接到静态资源主机。当前主机为 `static.kivo.wiki`，也可通过第 8.3 节的公开配置接口获取。只对明确的图片、封面、音频等资源字段这样处理，文章正文中的链接和 `url` 字段可能指向任意站内或外部页面。

```js
function toResourceUrl(value) {
  if (!value) return null;
  if (value.startsWith('//')) return `https:${value}`;
  if (/^https:\/\//i.test(value)) return value;
  return `https://static.kivo.wiki/${value.replace(/^\/+/, '')}`;
}
```

## 3. 接口总览

以下路径均接在基础地址 `https://api.kivo.wiki/api/v1` 后。

| 想实现的功能 | 接口 |
|---|---|
| 搜索角色、制作角色图鉴 | `/data/students`、`/data/students/{id}` |
| 制作本周生日提醒 | `/data/students/birthday/week` |
| 查询学校和组织 | `/data/schools`、`/data/schools/{id}`、`/data/relations`、`/data/relations/{id}` |
| 查询礼物、家具等物品 | `/data/items`、`/data/items/{id}` |
| 阅读文章和新闻 | `/articles`、`/articles/{id}`、`/news`、`/news/{id}` |
| 制作漫画阅读器 | `/comics`、`/comics/{id}`、`/comics/{comic_id}/chapters/{chapter_id}` |
| 浏览画廊 | `/galleries`、`/galleries/{id}` |
| 浏览或播放音乐 | `/musics`、`/musics/{id}` |
| 制作活动时间轴或日历 | `/timeline`、`/timeline/{id}` |
| 展示站内公告 | `/bulletins`、`/bulletins/{id}` |
| 展示当前卡池、活动和总力战 | `/data/pick_up`、`/data/event/now`、`/data/raid/now` |
| 制作随机推荐或幸运物 | `/data/lucky_item` |
| 展示 KivoWiki 统计 | `/statistics/index` |
| 获取当前静态资源主机 | `/upload/file_server` |

## 4. 角色、学校和物品

### 4.1 搜索和筛选角色

`GET /data/students`

效果：返回角色摘要和对应 ID，可用于搜索框、角色列表、筛选器和图鉴首页。完整技能、语音等信息需要再请求角色详情。

常用参数：

| 参数 | 可用值或示例 | 效果 |
|---|---|---|
| `page`、`page_size` | `1`、`20` | 分页。 |
| `character_data_search` | `星野` | 按角色姓名模糊搜索。 |
| `name_sort`、`id_sort` | `asc` / `desc` | 按姓名或 ID 排序。 |
| `birthday_sort` | `asc` / `desc` | 按生日排序。 |
| `release_date_sort` | `asc` / `desc` | 按日服发布日期排序。 |
| `release_date_global_sort` | `asc` / `desc` | 按国际服发布日期排序。 |
| `release_date_cn_sort` | `asc` / `desc` | 按国服发布日期排序。 |
| `updated_at_sort` | `asc` / `desc` | 按资料更新时间排序。 |
| `school` | 学校 ID，如 `1` | 筛选学校。 |
| `is_npc` | `true` / `false` | 是否为 NPC。 |
| `is_install` | `true` / `false` | 日服是否实装。 |
| `is_install_global` | `true` / `false` | 国际服是否实装。 |
| `is_install_cn` | `true` / `false` | 国服是否实装。 |
| `battlefield_position` | `STRIKER` / `SPECIAL` | 部队位置。 |
| `attack_attribute` | `Explosive` / `Piercing` / `Mystic` / `Vibration` | 攻击属性。 |
| `defensive_attributes` | `Light` / `Heavy` / `Special` / `Elastic` | 防御属性。 |
| `type` | `Tank` / `Dealer` / `Healer` / `Support` / `T.S.` | 战斗职能。 |
| `team_position` | `FRONT` / `MIDDLE` / `BACK` | 队伍站位。 |
| `weapon_type` | `SG`、`SMG`、`AR`、`GL`、`HG`、`RL`、`SR`、`RG`、`MG`、`MT`、`FT` | 武器类型。 |
| `rarity` | `1`、`2`、`3` | 初始稀有度。 |
| `limited` | `true` / `false` | 是否限定。 |
| `birthday` | `04-16` | 按生日筛选，格式为 `MM-DD`。 |

还观察到按身高、群控、换装、身材、设计师、原画师、装备和地形适应性筛选的参数，但其中部分历史拼写和取值尚未稳定确认，因此不建议社区项目依赖；见第 10 节。

```text
GET /data/students?page=1&page_size=20&school=1&is_npc=false&name_sort=asc
```

主要返回位置：`data.students[]`。常见字段包括 `id`、各语言姓名、`avatar`、`school` 和 `main_relation`。同一角色的不同换装可能拥有不同 ID，不要只按姓名去重。

### 4.2 获取角色详情

`GET /data/students/{id}`

效果：获取一个角色的完整公开资料，可用于详情页、技能查看器、礼物推荐和语音展示。

```text
GET /data/students/76
```

`data` 直接是详情对象。常用信息包括：

| 信息类型 | 常见字段 |
|---|---|
| 姓名与档案 | `family_name`、`given_name`、各语言姓名、`birthday`、`age`、`height`、`grade`、`hobby` |
| 所属关系 | `school`、`main_relation`、`relation` |
| 实装情况 | `is_npc`、`is_install`、`is_install_global`、`is_install_cn`、各服 `release_date` |
| 战斗信息 | `character_datas`，其中含攻击/防御属性、职能、站位、稀有度、装备、适应性、技能和武器 |
| 展示内容 | `introduction`、`more`、`gallery`、`avatar`、`sd_model_image`、`recollection_lobby_image` |
| 关联信息 | `gift_data`、`model`、`spine` |
| 语音 | `voice`、`voice_cn`、`voice_kr` |

字段可能为空、为 `null` 或随版本增加。`introduction` 和 `more` 可能包含 Markdown 或 HTML，显示前必须安全过滤。

### 4.3 本周生日

`GET /data/students/birthday/week`

效果：返回本周生日角色的 ID，适合机器人提醒和生日小组件。

以下为 2026-08-31 的响应结构示例，ID 会随日期变化：

```json
{
  "students": [68, 181, 78, 36, 560]
}
```

主要返回位置：`data.students[]`。数组内是 ID，不是角色对象；再调用 `/data/students/{id}` 获取姓名和生日。

### 4.4 学校

| 方法和路径 | 参数 | 效果 |
|---|---|---|
| `GET /data/schools` | `page`、`page_size`、`name` | 分页或按名称搜索学校；结果在 `data.school[]`。 |
| `GET /data/schools/{id}` | 路径中的学校 ID | 获取名称、介绍、徽标、预览图、地图、关联组织和角色等公开信息。 |

```text
GET /data/schools?page=1&page_size=20&name=阿比多斯
GET /data/schools/1
```

学校介绍可能使用扩展 Markdown/HTML 语法，显示前应使用有安全策略的渲染器。

### 4.5 关系、社团和组织

接口中的 `relations` 是学生关系、社团或组织的统一实体，不一定都能翻译成“社团”。

| 方法和路径 | 参数 | 效果 |
|---|---|---|
| `GET /data/relations` | `page`、`page_size`、`name` | 搜索关系或组织；结果在 `data.relation[]`。 |
| `GET /data/relations/{id}` | 路径中的关系 ID | 获取名称、图片、介绍和关联角色。 |

```text
GET /data/relations?page=1&page_size=20&name=圣三一
GET /data/relations/1
```

关联角色字段可能为 `null`，客户端应将其视为空数据而不是请求失败。

### 4.6 物品

| 方法和路径 | 参数 | 效果 |
|---|---|---|
| `GET /data/items` | `page`、`page_size`、`type`、`is_bind_article`、`id_sort` | 查询物品摘要；结果在 `data.item[]`。 |
| `GET /data/items/{id}` | 路径中的物品 ID | 获取物品名称、图标、稀有度、说明、关联文章及礼物/家具信息。 |

目前观察到的 `type` 包括 `gift`、`furniture` 和 `default`，以后可能增加。不要把类型写死为只有这几项。

```text
GET /data/items?page=1&page_size=20&type=furniture&id_sort=desc
GET /data/items/1742
```

`gift.students`、`furniture.students` 等嵌套值可能是 `null`。

## 5. 文章、新闻和公告

### 5.1 文章

| 方法和路径 | 参数 | 效果 |
|---|---|---|
| `GET /articles` | `page`、`page_size`、`summary_size`、`title` | 搜索文章并获取标题、封面和摘要；结果在 `data.article[]`。 |
| `GET /articles/{id}` | 路径中的文章 ID | 获取完整文章正文。 |

```text
GET /articles?page=1&page_size=10&summary_size=160&title=设定
GET /articles/83
```

详情中的 `body` 是 Markdown、HTML 和站内扩展语法混合文本，不是纯文本。社区应用应过滤脚本、事件属性和危险 URL 协议，并为外部链接增加清晰提示。

### 5.2 新闻

| 方法和路径 | 参数 | 效果 |
|---|---|---|
| `GET /news` | `page`、`page_size` | 获取新闻或首页资讯摘要；结果在 `data.news[]`。 |
| `GET /news/{id}` | 路径中的新闻 ID | 获取标题、图片、正文和原始链接。 |

```text
GET /news?page=1&page_size=10
GET /news/148
```

新闻也可能是活动横幅或外部宣传内容。不要默认信任 `url`；打开外链前只允许 `https:` 等安全协议。

### 5.3 公告

| 方法和路径 | 参数 | 效果 |
|---|---|---|
| `GET /bulletins` | `page`、`page_size` | 获取公告目录；结果在 `data.bulletin[]`。 |
| `GET /bulletins/{id}` | 路径中的公告 ID | 获取公告全文。 |

```text
GET /bulletins?page=1&page_size=10
GET /bulletins/39
```

`created_at` 和 `updated_at` 是 Unix 秒时间戳，JavaScript 中应乘以 `1000` 后传给 `Date`。

## 6. 漫画、画廊和音乐

这些接口适合制作索引、阅读器或播放器，但媒体素材不因 API 可访问而自动获得转载或重新托管许可。应优先链接原资源、标明来源、按需加载并遵守原作品授权。

### 6.1 漫画

| 方法和路径 | 参数 | 效果 |
|---|---|---|
| `GET /comics` | `page`、`page_size`、`title` | 搜索漫画合集；结果在 `data.comics[]`。 |
| `GET /comics/{id}` | `chapter_sort=asc` 或 `desc` | 获取合集介绍和章节目录 `data.chapter[]`。 |
| `GET /comics/{comic_id}/chapters/{chapter_id}` | 两个路径 ID | 获取章节信息和图片 `data.images[]`。 |

```text
GET /comics?page=1&page_size=10&title=官方
GET /comics/1?chapter_sort=asc
GET /comics/1/chapters/837
```

章节图片项使用服务端原始字段 `pagen_number` 表示页码，图片地址字段为 `file`。

### 6.2 画廊

| 方法和路径 | 参数 | 效果 |
|---|---|---|
| `GET /galleries` | `page`、`page_size`、`title` | 获取画廊标题和封面；结果在 `data.gallery[]`。 |
| `GET /galleries/{id}` | 路径中的画廊 ID | 获取介绍、分类和图片。 |

```text
GET /galleries?page=1&page_size=10&title=背景
GET /galleries/24
```

分类数组的原始字段名是 `categorys`，其中每项通常含 `name`、`introduction` 和 `images[]`。

### 6.3 音乐

| 方法和路径 | 参数 | 效果 |
|---|---|---|
| `GET /musics` | `page`、`page_size`、`s`、`id_sort` | 按标题搜索音乐；结果在 `data.music[]`。 |
| `GET /musics/{id}` | 路径中的音乐 ID | 获取介绍、音频和歌词地址。 |

```text
GET /musics?page=1&page_size=20&s=Theme&id_sort=asc
GET /musics/1
```

`file` 和 `lrc_file` 可能为空。播放前检查空值和 HTTP 状态，不要自动全量下载。

## 7. 时间轴和活动日历

### 7.1 Kivo 史书时间轴

`GET /timeline`

效果：查询剧情、活动、卡池、总力战、维护、直播等历史事件，适合制作日历、近期动态和历史检索工具。

| 参数 | 值 | 效果 |
|---|---|---|
| `page`、`page_size` | 正整数 | 分页。 |
| `type` | 类型英文值 | 按事件类型筛选；多类可重复传入，如 `type=Raid&type=Event`。 |
| `start_time_start` | Unix 秒 | 开始时间下界。 |
| `start_time_end` | Unix 秒 | 开始时间上界。 |
| `start_time_sort` | `asc` / `desc` | 按开始时间排序。 |
| `title` | 文本 | 标题模糊搜索。 |

已观察到的常用类型：

| 类型 | 含义 | 类型 | 含义 |
|---|---|---|---|
| `MainStory` | 主线故事 | `OtherStory` | 其他剧情 |
| `Event` | 活动 | `Gacha` | 卡池 |
| `Double` | 掉落加倍 | `MiniBattle` | 小型战役 |
| `Raid` | 总力战 | `BigRaid` | 大决战 |
| `AlliedOperation` | 联合作战 | `ContentImprovements` | 内容改进 |
| `Maintenance` | 维护 | `Live` | 直播 |
| `WebEvent` | 网页活动 | `OutsideGame` | 游戏外活动 |
| `Other` | 其他 | `Trivia` | 杂项知识 |

类型以后可能增加。保留 API 返回的原值，在显示层翻译。

```text
GET /timeline?page=1&page_size=20&type=Raid&type=Event&start_time_sort=desc
```

主要返回位置为 `data.timeline[]`；`data.type_num` 是各类型数量统计。列表项通常含 `title`、`image`、`body_summary`、`type`、`url`、`start_time` 和 `end_time`。

### 7.2 时间轴事件详情

`GET /timeline/{id}`

```text
GET /timeline/5919
```

详情中的 `body` 是完整正文。正文和 `url` 均可能含外部链接，渲染与跳转时应执行内容过滤和协议白名单检查。

### 7.3 当前卡池、总力战和活动

| 接口 | 参数 | 效果 |
|---|---|---|
| `GET /data/pick_up` | `server=jp` 或 `cn` | 当前卡池的开始时间、结束时间和卡池角色 ID。 |
| `GET /data/raid/now` | `server=jp` 或 `cn` | 当前总力战/大决战的开始时间、结束时间和横幅。 |
| `GET /data/event/now` | `server=jp` 或 `cn` | 当前活动的开始时间、结束时间和横幅。 |

```text
GET /data/pick_up?server=jp
GET /data/raid/now?server=jp
GET /data/event/now?server=jp
```

`/data/pick_up` 返回 `start_date`、`end_date` 和 `students[]`。`students` 中是角色 ID，可通过 `/data/students/{id}` 获取角色资料。

`/data/raid/now` 和 `/data/event/now` 返回 `start_date`、`end_date` 和 `banner`。横幅可能为空；没有当前数据时，时间也可能是负数等占位值。客户端应先检查时间是否为有效的正 Unix 秒，再决定是否展示，不要将占位值格式化为真实日期。接口没有提供活动标题时，不要根据图片文件名自行推断。

## 8. 其他公开数据

### 8.1 幸运物

`GET /data/lucky_item`

效果：返回一个资源类型和 ID，可用于每日推荐或随机展示。

```json
{
  "type": "item",
  "id": 559
}
```

主要返回位置：`data.type` 和 `data.id`。`type` 可能变化，应先判断类型，再决定是否能用已公开的详情接口读取。

### 8.2 站点统计

`GET /statistics/index`

效果：获取 KivoWiki 的用户、图片和角色统计快照。

以下为 2026-08-31 的响应结构示例，统计值会变化：

```json
{
  "users_number": 27434,
  "pictures_number": 20098,
  "students_number": 620
}
```

这些值不是各列表的精确记录总数，也不能代替分页中的 `max_page`。

### 8.3 静态资源主机

`GET /upload/file_server`

效果：返回 API 当前使用的静态资源主机，可避免在客户端永久写死主机名。

```json
{
  "server_host": "static.kivo.wiki"
}
```

主要返回位置：`data.server_host`。尽管路径中包含 `upload`，这个公开接口只是读取配置，不提供上传能力。不要据此猜测或探测任何写入接口。

## 9. 开发建议

### 安全解析

1. 同时检查网络错误、HTTP 状态和 `success`。
2. 将未知字段视为正常扩展，不因 API 增加字段而报错。
3. 将 `null`、空数组和空字符串视为可能的合法业务状态。
4. 文章、公告、学校说明、时间轴正文等富文本必须经过安全过滤后才能插入网页。
5. 对接口返回的外链只允许明确的安全协议，建议默认仅允许 `https:`。
6. 不要将 API 代理做成可请求任意主机或任意路径的开放代理。

### 友好调用

1. 根据界面需要选择较小的 `page_size`，不要用超大页面探测服务上限。
2. 缓存不常变化的角色、学校、物品和文章数据。
3. 控制并发，遇到 429 或 5xx 时进行有限次数的指数退避；不要无限重试 4xx。
4. 全量分页确有必要时，从第一页读取 `max_page`，依次获取，并在空页或错误时停止。
5. 不要通过猜测并遍历巨大 ID 区间查找内容，也不要批量抓取媒体文件。

一个通用请求函数示例：

```js
const API_BASE = 'https://api.kivo.wiki/api/v1';

async function kivoGet(path, params = {}) {
  if (!path.startsWith('/')) throw new Error('path 必须以 / 开头');

  const url = new URL(`${API_BASE}${path}`);
  for (const [key, value] of Object.entries(params)) {
    if (Array.isArray(value)) {
      for (const item of value) url.searchParams.append(key, String(item));
    } else if (value !== undefined && value !== null) {
      url.searchParams.set(key, String(value));
    }
  }

  const response = await fetch(url, { headers: { Accept: 'application/json' } });
  if (!response.ok) throw new Error(`KivoWiki HTTP ${response.status}`);

  const result = await response.json();
  if (!result?.success) throw new Error(result?.message || 'KivoWiki API 请求失败');
  return result.data;
}

const students = await kivoGet('/data/students', {
  page: 1,
  page_size: 10,
  character_data_search: '星野',
});
```

## 10. 本文没有收录的内容


- 3D 模型资源列表与详情
- Spine 动画资源列表与详情 
- 重定向/别名数据
- 配队方案与攻略
- 贡献者数据
- 文章补充内容和声明关联

