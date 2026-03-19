## 数据库设计

### 元信息存储（mysql、sqlite）

- 知识库（knowledge_database）

| 字段      | 字段类型 | 字段含义         |
| --------- | -------- | ---------------- |
| knowledge_id        | bigint   | 知识库主键       |
| title     | varchar  | 名称             |
| category  | varchar  | 类型             |
| author_id | bigint   | 用户主键（外键） |
| create_dt | datetime | 创建时间         |
| update_dt | datetime | 更新时间         |

- 知识文档（knowledge_document）

| 字段         | 字段类型 | 字段含义           |
| ------------ | -------- | ------------------ |
| document_id           | bigint   | 文档主键           |
| title        | varchar  | 文档名称           |
| category     | varchar  | 文档类型           |
| knowledge_id | bigint   | 知识库主键（外键） |
| file_path    | varchar  | 储存地址           |
| file_type    | varchar  | 数据类型           |
| create_dt    | datetime | 创建时间           |
| update_dt    | datetime | 更新时间           |


### 文档全文与向量存储（es）

- **document_meta** 存储知识文档的元信息，包括文件路径、名称、摘要和全部内容等。

| 字段名           | 数据类型     | 字段说明               |
|------------------|--------------|------------------------|
| `document_id`    | `int`       | 文档元信息的唯一标识符   |
| `document_name`      | `text`       | 文档的名称              |
| `knowledge_id`   | `text`       | chunk 所属知识库的标识符 |
| `file_path`      | `text`       | 文档的存储路径          |
| `abstract`       | `text`       | 文档的摘要信息          |

- **chunk_info** 存储分块的文档内容，包含 chunk 的文字内容、图片、表格地址等。

| 字段名           | 数据类型     | 字段说明               |
|------------------|--------------|------------------------|
| `chunk_id`       | `text`       | chunk 的唯一标识符      |
| `document_id`    | `int`       | 文档元信息的唯一标识符   |
| `knowledge_id`   | `text`       | chunk 所属知识库的标识符 |
| `page_number`    | `integer`    | chunk 所在文档的页码    |
| `chunk_content`  | `text`       | chunk 的文字内容        |
| `chunk_images`   | `text`       | chunk 相关图片的路径     |
| `chunk_tables`   | `text`       | chunk 相关表格的路径     |
| `embedding_vector` | `array<float>` | chunk 的语义编码结果   |

### 知识节点与图片存储（neo4j）

TODO

## 开发资料

- https://elasticsearch-py.readthedocs.io/en/latest/

## RAG选择

一.选模型的依据是什么？
METB（Massive Text Embedding Benchmark）榜单：可以在这个榜单上进行参考https://huggingface.co/spaces/mteb/leaderboard

但看榜单也有技巧：
1.不要只看总分。 MTEB 的总分是多个任务的平均值，包括分类、聚类、句对匹配等。RAG 场景最关心的是 Retrieval 子任务的得分，要单独筛选看这一项。
2.注意语言。 MTEB 有英文榜和中文榜（C-MTEB），如果你的场景是中文，要看 C-MTEB 的排名，不要看英文榜。
3.注意模型大小。 榜单上排名靠前的可能是几十亿参数的大模型，部署成本很高。在实际选型时要综合考虑精度和推理速度，不能只追排名。

二.Embedding模型怎么选？
1.一般情况下不知道选什么 → BGE-M3， 不会错 （BAAI智源出品）
2.纯中文或中英混合 → BGE-M3 或 BGE-large-zh
3.多语言场景 → GTE-multilingual-base 或 BGE-M3，看MTEB榜单最新排名 （阿里达摩院出品，跟BGE-M3是直接竞品关系）
4.资源紧张/边缘设备 → E5-small 或 MiniLM (微软出品)
5.长文本 >=8K token → Jina Embeddings v2(德国 AI 公司 Jina AI 出品)


三.为什么还需要Rerank模型？
有个对比实验：query 是"世界上最高的山峰是哪个"，候选文本有三条：珠穆朗玛峰、乔戈里峰(K2)、干城章嘉峰。Bi-Encoder 把 K2 排在了第一位（因为"最高的山峰"这几个字跟三条文本都很相似，细微差别区分不出来）；Cross-Encoder 正确地把珠穆朗玛峰排在了第一位，因为它能理解"最高"和具体山峰名称之间的事实关系。这就是为什么两阶段检索是标配：Bi-Encoder 负责从百万文档中快速召回 Top 100，Cross-Encoder 负责对这 100 条做精排取 Top 5 给 LLM。

四.Rerank模型是怎么选择的？
1.经典流水线：BGE-base检索Top 100 → BGE-Reranker-base精排
2.多语言场景：GTE-multilingual-base检索Top 100 →GTE-multilingual-reranker精排
3.GPU紧张：E5-small → MiniLM-L6-cross-encoder（batch 推理）
4.长文档 / 8K：Jina-embeddings-v2 + Jina-ColBERT-v2，段内匹配更稳

选型的核心原则就一条：Embedding 和 Rerank 尽量选同一系列的，因为它们在训练时的数据分布和语义空间更一致，搭配效果最好。

五.Embedding模型在这个项目中是怎么选的？

在这个项目中，我们的场景是百姓问答

1.选型维度。 我们从三个维度评估：语种支持（中文/多语言）→ 中文、上下文长度（chunk 长度匹配） → 500 token、部署资源（GPU 显存和推理速度）。

2.对比过程。 在 MTEB/C-MTEB 榜单上筛选了 Retrieval 任务得分靠前的候选模型，包括 GTE-small-zh、BGE-small-zh。在我们自己的测试集上跑了 MRR 和 Precision@5 对比，BGE-small-zh 综合最优。

3.搭配方案。 Embedding 用 BGE-small-zh，Rerank 用同系列的 bge-reranker-base，检索 Top 100 后精排取 Top 5。选同系列是因为语义空间更一致，搭配效果最好。

最后选择的是 bge-small-zh-v1.5







