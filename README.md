# 电商经营数据问答（NL2SQL Agent）

用自然语言查数仓：用户问"上个月华东的 GMV 是多少"，系统自动召回相关的表/字段/指标/枚举值，生成 SQL、校验后执行，并把结果流式返回。

---

## 要解决的问题

LLM 直接写 SQL 会犯三类错误，而且**越大的库越容易犯**：

| 错误类型 | 典型表现 | 后果 |
| --- | --- | --- |
| **表选错** | 问订单金额，却去查 `dim_product` | 结果看似正常，实际全错 |
| **列选错** | 把"客单价"映射到 `order_quantity` | 数值离谱但 SQL 合法 |
| **值写错** | 用户说"华东"，写成 `region_name = '华东大区'` | 返回 0 行，**不报错** |

所以核心不是"让模型更聪明"，而是**在生成前把正确的候选（表/列/指标/值域）喂给它**，生成后**用数据库自己的能力校验**。

## 为什么不把全库 DDL 直接塞进 prompt

1. **塞不下**：真实数仓上千张表，DDL 动辄几十万 token，超出上下文且每次请求都付费。
2. **塞不准**：候选表越多，模型选错的概率越高——这是 Text2SQL 最主要的失败来源，不是"token 不够"这么简单。

**边界**：如果整个库只有个位数张表（本项目 5 张维表 + 1 张事实表，DDL 约 2000 token），直接全塞反而是更简单正确的做法。RAG 式元数据召回是为"库会变大"准备的通用设计。

---

## 架构

```
                    ┌── 元数据（离线构建，跑一次）──────────┐
meta_config.yaml ──►│ ① MySQL meta 库：表/列/指标全量        │
dw 库真实数据  ────►│ ② Qdrant：字段按 name+description      │
                    │    各一条向量（语义召回）              │
                    │ ③ ES：列的真实枚举值（精确召回）        │
                    └────────────────────────────────────────┘

                        ┌── 查询链路（12 节点 / 4 阶段）──────────┐
Query ──► ① 理解 ──► ② 召回（三路并行）──► ③ 加工 ──► ④ 生成与校验 ──► SSE
         关键词抽取    字段 / 指标 / 值召回      合并·裁剪·上下文    SQL → EXPLAIN → 执行/纠错
```

### 12 个节点（7 个用 LLM，5 个纯代码）

| # | 节点 | 类型 | 职责 |
| --- | --- | --- | --- |
| 1 | `extract_keywords` | 代码 | jieba TF-IDF 提关键词，并把**整句**也塞进关键词表（防分词丢信息） |
| 2 | `recall_column` | LLM | 扩写关键词 → 向量检索字段 |
| 3 | `recall_metric` | LLM | 扩写关键词 → 向量检索指标口径 |
| 4 | `recall_value` | LLM | 扩写关键词 → ES 精确检索枚举值 |
| 5 | `merge_retrieved_info` | 代码 | 按字段 id 去重、补主外键、把值写回 `examples` |
| 6 | `filter_table` | LLM | 裁掉不相关的表与列 |
| 7 | `filter_metric` | LLM | 裁掉不相关的指标 |
| 8 | `add_extra_context` | 代码 | 注入当前日期/星期/季度 + 数据库版本与方言 |
| 9 | `generate_sql` | LLM | 拼 YAML 上下文 → 生成 SQL |
| 10 | `validate_sql` | 代码 | `EXPLAIN` 校验，失败把错误写回 state |
| 11 | `correct_sql` | LLM | 带报错信息修正一次 |
| 12 | `execute_sql` | 代码 | 真正执行并推结果 |

编排用 LangGraph `StateGraph`：2–4 三路并行、6–7 两路并行，分支靠条件边——`not state["error"] → execute_sql`，否则 `→ correct_sql`。

---

## 五个关键设计取舍

### 1. 三路召回为什么按"检索对象"拆，而不是按"检索方式"拆

常见拆法是"向量 + 全文"，但这里真正的区分是**你要找的是什么**：

- **字段名**：`order_amount` 对应"金额/销售额"，语义相近即可 → Qdrant 向量
- **指标口径**：GMV 的定义写在描述里，也是语义匹配 → Qdrant 向量
- **枚举值**：用户说"华东"，库里存的必须是完全一致的字符串 → **ES 精确匹配**

值走向量会怎样？「华东」「华东大区」「华南区」的向量几乎同向，模型可能拿到「华东大区」并写进 SQL——**不报错、静默返回错误数据**，比报错更危险。一句话：**语义交给 Qdrant，精确交给 ES。**

### 2. 为什么一个字段要建两条向量

用户可能按字面名问（"金额"对 `order_amount`），也可能按业务含义问（"销售额"对 description）。两种问法落在不同文本上，所以同一字段按 `name` 与 `description` **各生成一个 point**。

不拼成一条的原因：name 很短、description 很长，拼接后向量会被长文本主导，字面匹配能力下降。代价是存储翻倍、合并时要去重——值得。

### 3. 为什么主外键不进向量索引（`sync: false`）

`id`、`xxx_id` 这类字段语义区分度极低，5 张表的主键编码后几乎同向量：既召不回有用的东西，又会把 `xxx_id` 噪声塞进 prompt，诱导模型写出 `WHERE id = 3`。

但 **JOIN 需要它们**。所以：向量索引里不放（`sync: false`），由 `merge` 阶段按表分组后从 MySQL 元数据库**结构性地补齐**。本质区别是——**语义检索是概率问题，主外键是契约问题，用概率手段拿契约信息一定会漏。**

### 4. 为什么 `validate_sql` 用 `return {"error": ...}` 而不是 `raise`

`raise` 会穿透 LangGraph，`correct_sql` 收不到报错、SSE 流中断。返回错误对象则让错误成为 state 的一部分，条件边一看 `state["error"]` 就知道往哪走。**错误是一等公民，不是异常。**

### 5. 为什么用 YAML 注入表结构

JSON 的花括号、引号是纯 token 开销（约多 20–30%），且 `json.dumps` 默认 `ensure_ascii=True` 会把中文转成 `\uXXXX`；表结构本质是嵌套清单，YAML 缩进天然对应。用 `yaml.dump(..., allow_unicode=True, sort_keys=False)` 保留中文字段描述与字段顺序。

---

## 快速开始

```bash
# 1. 起依赖：MySQL(3306) / Elasticsearch(9200) / Kibana(5601) / Qdrant(6333) / Embedding(8081)
docker compose -f docker/docker-compose.yaml up -d

# 2. 配置（示例文件里 api_key 为空，需自行填入）
cp conf/app_config.example.yaml conf/app_config.yaml

# 3. 建表与元数据
mysql -h127.0.0.1 -uroot -p < docker/mysql/meta.sql
mysql -h127.0.0.1 -uroot -p < docker/mysql/dw.sql
python -m app.scripts.build_meta_knowledge -c conf/meta_config.yaml   # 写 meta 库 + Qdrant + ES

# 4. 启动
uv run main.py                       # FastAPI，POST /api/query 返回 SSE 流
```

请求示例：

```bash
curl -N -X POST http://127.0.0.1:8000/api/query \
  -H "Content-Type: application/json" \
  -d '{"query":"上个月华东地区的GMV是多少"}'
```

返回是 SSE 事件流：每进一个节点推一次 `stage`（可做进度条），最后推一次 `result`（结果集）。

## 已知短板（写在明处）

1. **单轮对话**：`StateGraph` 未挂 checkpointer，"那华东呢"这类追问会丢上下文。补法是按 `thread_id` 加记忆，并把上轮的表/日期带进关键词抽取做指代消解。
2. **只纠错一次**：`correct_sql` 后直接执行，没有二次校验与重试上限。稳妥做法是纠错后回到 `validate_sql`，设 2 次上限 + 规则护栏（空结果、行数异常、量级校验）。
3. **EXPLAIN 只能拦"语法/结构错"**：拦不住指标算错、条件写反、JOIN 接错维度、笛卡尔积。需要建评估集衡量**执行准确率**与**有效 SQL 率**。
4. **日期算术交给模型**：`add_extra_context` 只给"今天是哪天"，"上周六"要靠 LLM 自己算，应改为代码预计算。
5. **阈值硬编码**：Qdrant `score_threshold=0.6` 写在代码里，换 embedding 模型不可复用。
6. **枚举值同义词缺失**：ES 精确匹配对"华东 / 华东区"这类写法会漏，需要同义词词典。

## 目录结构

```
app/
├── agent/            # 12 节点 + graph.py + state.py
├── api/              # FastAPI 路由（SSE）
├── clients/          # MySQL / Qdrant / ES / Embedding 客户端
├── conf/             # 配置与元数据模型
├── core/             # 日志、生命周期
├── models/           # ORM 与检索模型
├── repositories/     # 数据访问层
├── scripts/          # 元数据构建入口 build.py
└── services/         # 业务服务（元数据知识库、查询服务）
conf/
├── app_config.example.yaml   # 配置模板
└── meta_config.yaml          # 表/列/指标元数据定义
prompts/                       # 8 个提示词（扩词 / 裁剪 / 生成 / 纠错）
docker/                        # compose + meta.sql + dw.sql
```

## License

代码仅作学习与作品展示用途。
