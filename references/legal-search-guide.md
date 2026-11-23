# 法律数据库检索指南

本文件用于**模块 D"法律问题清单与检索方向"的落地检索执行**，以及**模块 C 涉及法条/类案时的引用核验**。所有法条与类案必须通过真实工具检索获得，禁止凭记忆编造条号、案号或裁判要旨。

检索关键词组合由模块 D 产出（含同义表达与法院实务表述，见 `module-d-research.md`），本文件负责具体的工具调用、参数填写与失败处理。引用时须遵守全局铁律：法条写"名称+条号+是否现行有效+生效日期"，案例写"案号+审理法院+裁判日期"，不确定即标"需核实/待核实"。

## 一、可用数据库与优先级

| 优先级 | 数据库 | 加载方式 | 状态说明 |
|--------|--------|----------|----------|
| 1 | 北大法宝（pkulaw） | `@skill:pkulaw` 加载 | 主用法条精确查询与案例检索 |
| 2 | 元典（yuandian-mcp） | ToolSearch 加载 `mcp__yuandian-mcp__*` schema | 主用语义检索与幻觉核查 |
| 3 | 其他（qcc-legal / fyopen-lawsearch） | 对应 skill | 若处于未授权状态则跳过，不阻塞流程 |

**纪律**：检索前先加载对应 skill 或工具 schema，确保工具可调用。不要凭工具名猜测参数。

## 二、北大法宝工具选用速查

| 场景 | 工具 | 关键参数 |
|------|------|----------|
| 已知道法名+条号，取原文 | `get_law_article` | `law_title`（简称可用，如「民法典」）、`article_number`（232 或「第二百三十二条」；「之一」写 120.1） |
| 读连续几条 | `get_law_articles_by_range` | 法名 + 起止条号 |
| 按关键词找法条 | `search_law_articles_by_keyword` | `fulltext`（必填，空格=且、\|=或）、`title`（法规名收窄）、`timeliness=["现行有效"]`（只要现行条文时必传）、`effectiveness`（效力位阶） |
| 只有案情、写不出条文用词 | `search_law_articles` | 自然语言案情描述，语义检索 |
| 有案号/案由/当事人/关键词找案例 | `search_cases_by_keyword` | `case_number`、`category`（案由数组）、`party`、`gist`（裁判要点）、`fulltext`、`case_grade`（如「指导性案例」）、`court`、`court_level` |
| 只有案情、找相似案例 | `search_cases` | 自然语言案情，语义检索 |
| 核验引用真伪 | `verify_law_citations` | 传入文本中的法条与案号引用 |
| 识别文本中的案号 | `recognize_case_numbers` | 传文本 |
| 识别文本中的法条引用 | `recognize_law_citations` | 传文本 |

**重要参数说明**：
- `search_law_articles_by_keyword` **不会默认只查现行有效**，要现行条文必须传 `timeliness=["现行有效"]`。
- `get_law_article` 取的是现行有效版；法名后可写版本（如「刑法(2011修正)」）取特定版。
- `get_law_article` 判断适用时点看 `implement_date`（施行日期），不看公布日期。
- `search_cases_by_keyword` 的 `final_result` 已废弃，改用 `result_type`。
- 不同字段之间是"并且"，同一列表参数内多个值是"或者"。
- 检索结果均为"可信级别 B"，是一页召回而非穷尽检索，引用前须打开链接核对原文。

**翻页纪律**：默认只看第 1 页。用户明确要更多时，用返回结果的 `next_cursor` 原样回填翻页；一次最多 50 条、5 页，再多就收窄条件。用户没要求更多时不要主动翻页。

## 三、元典工具选用速查

| 场景 | 工具 | 关键参数 |
|------|------|----------|
| 自然语言案情找类案 | `yuandian_semantic_search_cases` | `query`（必填，自然语言）、`limit`、`case_document_filter`（按 case_category / cause_of_action / court_level / authoritative_only 等过滤） |
| 关键词找法条 | `yuandian_search_legal_articles` | `keyword`（必填）、`search_mode`（AND/OR）、`regulation_name`、`primary_effect_level`、`region` |
| 取法条原文详情 | `yuandian_get_legal_article_detail` | 按检索返回的 ID 取详情 |
| 取案例详情与分享链接 | `yuandian_get_case_detail` | 按检索返回的 ID 取详情，返回 `share_access` 说明 |
| 权威案例定向检索 | `yuandian_search_authoritative_cases` | 按指导性/公报/典型案例定向检索 |
| 幻觉核查 | `yuandian_hall_detect` | 传入分析文本，检测结论是否偏离检索依据 |

**注意**：`yuandian_semantic_search_cases` 与 `yuandian_search_legal_articles` 返回的 url 需登录且不附分享凭证。需要可转发的原文链接时，调用 `yuandian_get_case_detail` / `yuandian_get_legal_article_detail` 按 `share_access` 说明获取，不要在检索结果上自行拼接 shareCode。

## 四、检索执行流程

对每个法律问题，独立走完以下流程（不合并、不同问题不用同一批案例）：

1. **语义检索起步**：用通俗案情描述调 `search_cases`（法宝）或 `yuandian_semantic_search_cases`（元典），拿到初步相关案例。
2. **法条定位**：
   - 从案例的"裁判依据"字段提取被引用的法条 → 用 `get_law_article` 取原文核对。
   - 或用 `search_law_articles_by_keyword` 按法律概念关键词检索（记得传 `timeliness=["现行有效"]`）。
   - 仍找不到时用 `search_law_articles` 语义兜底。
3. **类案深度确认**：对高相关案例，用案号调 `search_cases_by_keyword` 获取拆好的"本院查明、本院认为、争议焦点、裁判依据、裁判结果"字段；需要全文时用元典 `yuandian_get_case_detail`。
4. **滚雪球**：把裁判依据中引用的新法条、判决理由中提及的关联案号，立刻作为新一轮检索关键词。
5. **停止条件**：每个问题找到 2–3 个直接相关案例 + 对应法条原文，或所有路径均穷尽。

## 五、失败与异常处理

- **0 结果**：换关键词、换同义词、换角度重试至少 2 轮；语义检索与关键词检索交叉使用。
- **工具调用失败/超时**：换另一个数据库重试；两个都失败则标注"数据库暂不可用，待人工核实"。
- **数据库未授权**：跳过，改用另一个；不要卡住流程，更不要因此编造依据。
- **查无此条**：如实说明"该法条在现行有效版本中未检索到"，不得用旧版顶替而不说明。
- **前后矛盾**：以工具返回的原文为准，不依赖记忆。

## 六、引用规范

- 法条写法：《法律名称》第 X 条（第 Y 款），附核心要旨一句话。例：《民法典》第 563 条（法定解除权：一方迟延履行主要债务经催告后仍未履行）。
- 类案写法：（YYYY）XX……民终 XX 号，XX 法院，裁判要点一句话。
- 所有引用在交付前用 `verify_law_citations`（法宝）或 `yuandian_hall_detect`（元典）跑一次核验。
- 关键结论交付前，用 `yuandian_hall_detect` 检测分析文本是否有偏离检索依据的幻觉。
- 无法核验的引用标注"待核实"，不得作为确定结论呈现。
