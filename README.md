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