<center><h2 style="margin-top: 2em; padding-top: 0.5em; border-top: 1px solid #eee;">一.数据库设计</h2></center>


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

<center><h2 style="margin-top: 2em; padding-top: 0.5em; border-top: 1px solid #eee;">二.开发资料</h2></center>


- https://elasticsearch-py.readthedocs.io/en/latest/




<center><h2 style="margin-top: 2em; padding-top: 0.5em; border-top: 1px solid #eee;">三.RAG模型的选择</h2></center>

---

### 1\.选模型的依据是什么？

METB（Massive Text Embedding Benchmark）榜单：可以在这个榜单上进行参考https://huggingface\.co/spaces/mteb/leaderboard

但看榜单也有技巧：

1. 不要只看总分。 MTEB 的总分是多个任务的平均值，包括分类、聚类、句对匹配等。RAG 场景最关心的是 Retrieval 子任务的得分，要单独筛选看这一项。

2. 注意语言。 MTEB 有英文榜和中文榜（C\-MTEB），如果你的场景是中文，要看 C\-MTEB 的排名，不要看英文榜。

3. 注意模型大小。 榜单上排名靠前的可能是几十亿参数的大模型，部署成本很高。在实际选型时要综合考虑精度和推理速度，不能只追排名。
   
---

### 2\.Embedding模型怎么选？

1. 一般情况下不知道选什么 → BGE\-M3， 不会错 （BAAI智源出品）

2. 纯中文或中英混合 → BGE\-M3 或 BGE\-large\-zh

3. 多语言场景 → GTE\-multilingual\-base 或 BGE\-M3，看MTEB榜单最新排名 （阿里达摩院出品，跟BGE\-M3是直接竞品关系）

4. 资源紧张/边缘设备 → E5\-small 或 MiniLM (微软出品)

5. 长文本 >=8K token → Jina Embeddings v2(德国 AI 公司 Jina AI 出品)

---

### 3、为什么还需要 Rerank 模型？

有个对比实验：query 是「世界上最高的山峰是哪个」，候选文本有三条：珠穆朗玛峰、乔戈里峰（K2）、干城章嘉峰。

- **Bi-Encoder** 把 K2 排在了第一位（因为「最高的山峰」这几个字跟三条文本都很相似，细微差别区分不出来）；
- **Cross-Encoder** 正确地把珠穆朗玛峰排在了第一位，因为它能理解「最高」和具体山峰名称之间的事实关系。

这就是为什么两阶段检索是标配：
- **Bi-Encoder** 负责从百万文档中快速召回 Top 100；
- **Cross-Encoder** 负责对这 100 条做精排，取 Top 5 给 LLM。

---


### 4\.Rerank模型是怎么选择的？

1. 经典流水线：BGE\-base检索Top 100 → BGE\-Reranker\-base精排

2. 多语言场景：GTE\-multilingual\-base检索Top 100 →GTE\-multilingual\-reranker精排

3. GPU紧张：E5\-small → MiniLM\-L6\-cross\-encoder（batch 推理）

4. 长文档 / 8K：Jina\-embeddings\-v2 \+ Jina\-ColBERT\-v2，段内匹配更稳

选型的核心原则就一条：Embedding 和 Rerank 尽量选同一系列的，因为它们在训练时的数据分布和语义空间更一致，搭配效果最好。

---

### 5\.Embedding模型在这个项目中是怎么选的？

在这个项目中，我们的场景是百姓问答

1.选型维度。 我们从三个维度评估：
- 语种支持（中文/多语言）→ 中文;
- 上下文长度（chunk 长度匹配） → 500 token;
- 部署资源（GPU 显存和推理速度）

2.对比过程。 在 MTEB/C-MTEB 榜单上筛选了 Retrieval 任务得分靠前的候选模型，包括 GTE-small-zh、BGE-small-zh。在我们自己的测试集上跑了 MRR 和 Precision@5 对比，BGE-small-zh 综合最优。

3.搭配方案。 Embedding 用 BGE-small-zh，Rerank 用同系列的 bge-reranker-base，检索 Top 100 后精排取 Top 5。选同系列是因为语义空间更一致，搭配效果最好。

最后选择的是 bge-small-zh-v1.5



<center><h2 style="margin-top: 2em; padding-top: 0.5em; border-top: 1px solid #eee;">四.混合检索</h2></center>


---

### 1、为什么需要混合检索？

向量检索的核心弱点：对**精确关键词、产品编号、专有名词**，它的召回能力明显不足。

**BM25（全称：Best Match 25）**，核心包含四个参数：

- IDF（逆文档频率）

- tf（词频）

- k1（词频饱和参数，默认1\.5）

- b（长度归一化，默认0\.75）

工程实践：若线上系统已部署 ElasticSearch（ES），可直接使用ES自带的BM25实现；ES从5\.0版本开始，默认使用BM25替代TF\-IDF。

---

### 2、RRF融合算法：k=60从哪来的？

**注：这是整篇文档的核心，也是面试高频追问点。**

RRF的全称是Reciprocal Rank Fusion，由Cormack、Clarke和Buettcher在2009年的SIGIR论文《Reciprocal Rank Fusion outperforms Condorcet and individual Rank Learning Methods》中提出。

#### 1\. 核心公式理解

理解RRF的关键，在于掌握其分数计算公式：

```latex
score = 1/(k + rank + 1)
```
补充说明：

- rank（排名）从0开始计数：排名第1的文档得分是 $1/(k+1)$，排名第2的文档得分是 $1/(k+2)$，以此类推。

- 多源融合逻辑：若一个文档同时出现在BM25结果和向量检索结果中，其最终分数为两路结果各自得分 $\frac{1}{k+\text{rank}+1}$ 之和。

#### 2\. k=60的由来

（1）实验依据

Cormack等人在2009年的论文中，通过大量实验，在多个IR基准数据集上测试了不同k值的表现，最终结论：

k=60在大多数场景下能取得最佳融合效果，且对k值的选取不敏感——当k在40\~80之间时，性能差异通常在1%以内。

（2）数学直觉理解

k的核心作用：控制排名靠前的文档与后续文档的权重差距，不同k值的效果对比如下：

- k很小（如k=1）：排名第1得分1/2=0\.5，排名第2得分1/3≈0\.33，权重差距巨大，算法对最高排名文档极度敏感。

- k很大（如k=1000）：排名第1得分1/1001≈0\.001，排名第20得分1/1020≈0\.00098，权重差距极小，几乎拉平所有文档。

- k=60（最优）：排名第1得分1/61≈0\.0164，排名第20得分1/81≈0\.0125，既能保持合理的排名差距，又不会对噪声过度敏感。

---

### 3、为什么用RRF而不是加权求和？

加权求和存在一个致命缺陷：**BM25分数与向量相似度分数的量纲完全不同**。

- BM25分数：受文档库大小、词频分布影响，取值范围可在0\.1\~50之间波动。

- 向量相似度分数：经过归一化后，取值范围固定在0\~1之间。

直接加权求和前，必须对两路分数做归一化处理；而归一化方式的选择会引入新的超参数，且不同批次的检索结果中，分数范围可能发生变化，导致稳定性差。

RRF则完全绕开了这一问题——它**只用排名，不用具体分数**。无论BM25返回的分数是0\.3还是30，只要排名第1，得分就固定为$\frac{1}{k+1}$。这使得RRF在跨批次、跨模型更新时表现稳定，无需重新校准归一化参数。

补充：标准RRF默认两路结果等权融合，但实际业务中，有时需要让某一路检索的贡献更大，由此引出下一个问题。

---

### 4、如何手动调整权重？

RRF默认两路检索结果等权融合，但业务场景中存在权重调整需求（例如：理赔查询类问题中，BM25更可靠；产品咨询类问题中，向量检索更可靠）。

加权RRF的公式如下：

```latex
score = α * 1/(K+rank_bm25+1) + (1-α) * 1/(K+rank_vector+1)
```

---

### 5、混合检索的速度分析

混合检索对检索延迟的影响极小，各环节延迟拆解如下：

- BM25检索：速度极快，处于毫秒级。

- 向量检索（HNSW索引）：P99延迟约5ms。

- RRF融合：纯Python列表操作，延迟不到1ms。

整体端到端检索延迟：从约7ms增加至约9ms，用户无感知。

---

### 6、一些踩坑经验

1. **中文分词质量影响BM25效果**：Jieba默认词典不包含保险行业专有词汇，可能出现“免赔额”被切分为“免/赔额”、“等待期”被切分为“等待/期”的情况。解决方案：给Jieba添加自定义词典，收录5000份文档中出现的所有保险专业术语，此举可使BM25的精确词召回率提升约8个百分点。

2. **候选集大小的选择**：我们每路检索取Top20，该数值经实验验证：Top5太少，两路结果交集极小，无法体现RRF融合价值；Top100太多，会引入大量噪声。对于5000份文档、每份平均切分16个块的场景，每路Top20是效果与效率的最佳平衡点。

3. **BGE模型查询前缀不可遗漏**：BGE系列模型训练时，对查询和文档采用不同前缀格式——查询需添加前缀“为这个句子生成表示以用于检索相关文章：”，文档编码无需添加。若遗漏前缀，召回率会下降约3\~5个百分点。

---







