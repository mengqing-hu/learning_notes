# RAG后台数据库处理流程

从上传文档 → 切片 → 入库 → 用户聊天 → RAG 检索 → 存聊天记录，

我用最简单的场景：

- 文档：一份《员工手册》PDF
- 数据库：MySQL（关系型）
- 向量库：Milvus（你换成 Chroma/Pinecone 流程一样）
- 模型：随便一个 embedding 模型 + LLM

------

## 0. 先定义三张核心表（MySQL）

#### 1）document 文档表

```
CREATE TABLE document (
  id INT PRIMARY KEY AUTO_INCREMENT,
  title VARCHAR(255) COMMENT '文档标题',
  file_url VARCHAR(500) COMMENT '文件存储路径',
  file_type VARCHAR(20) COMMENT 'pdf/txt/docx',
  create_time DATETIME DEFAULT NOW()
);
```

#### 2）chunk 切片表

```
CREATE TABLE chunk (
  id INT PRIMARY KEY AUTO_INCREMENT,  -- 关键！全局唯一 chunk_id
  doc_id INT,                         -- 属于哪个文档
  chunk_order INT,                    -- 第几个切片
  content TEXT,                       -- 切片文本
  create_time DATETIME DEFAULT NOW()
);
```

#### 3）chat_message 聊天记录表

```
CREATE TABLE chat_message (
  id INT PRIMARY KEY AUTO_INCREMENT,
  user_id INT,
  question TEXT,
  answer TEXT,
  hit_chunk_ids VARCHAR(255),  -- 命中哪些切片，逗号分隔
  create_time DATETIME DEFAULT NOW()
);
```

------

## 1. 第一步：上传《员工手册.pdf》

假设文件内容简化为：

```
第一章 考勤制度
1. 上班时间：周一至周五 9:00-18:00
2. 迟到15分钟以上算迟到，迟到三次算一次旷工

第二章 请假制度
1. 事假需提前1天申请
2. 病假需提供医院证明
```

#### 实际存储：

- PDF 文件存在本地或 OSS：

  ```
  /upload/2026/employee_handbook.pdf
  ```

- MySQL 插入一条 document：

|  id  |  title   |              file_url              | file_type |
| :--: | :------: | :--------------------------------: | :-------: |
|  1   | 员工手册 | /upload/2026/employee_handbook.pdf |    pdf    |

------

## 2. 第二步：读取文本 → 切片（chunk）

我们切成两段：

#### chunk 1

```
第一章 考勤制度
1. 上班时间：周一至周五 9:00-18:00
2. 迟到15分钟以上算迟到，迟到三次算一次旷工
```

#### chunk 2

```
第二章 请假制度
1. 事假需提前1天申请
2. 病假需提供医院证明
```

#### 存入 MySQL chunk 表：

|  id  | doc_id | chunk_order |  content  |
| :--: | :----: | :---------: | :-------: |
|  1   |   1    |      1      | 第一章... |
|  2   |   1    |      2      | 第二章... |

**到这里：文本内容已经安全存在 MySQL**

------

## 3. 第三步：向量化 → 存入向量数据库

对每个 chunk 的 content 做 embedding，得到向量。

#### 向量库每条数据结构（Milvus）

```
chunk_id: 1
doc_id: 1
content: "第一章 考勤制度..."
vector: [0.123, 0.456, 0.789, ...]  # 比如 768/1536 维
```



```
chunk_id: 2
doc_id: 1
content: "第二章 请假制度..."
vector: [0.234, 0.567, 0.890, ...]
```

重点：

- **向量只存在向量库**
- **content 两边都存**（MySQL + 向量库）

------

## 4. 第四步：用户开始聊天（真实 RAG 流程）

#### 用户提问：

```
我迟到了会怎么样？
```

#### ① 后端把问题向量化

问题 → embedding → 得到一个查询向量 `query_vector`

#### ② 去向量库做相似度检索

向量库计算距离，返回最相似的 chunk：

```
chunk_id: 1
content: "第一章 考勤制度...迟到三次算一次旷工"
score: 0.89
```

#### ③ 把检索内容拼给 LLM

```
根据以下资料回答问题：

资料：
第一章 考勤制度
1. 上班时间：周一至周五 9:00-18:00
2. 迟到15分钟以上算迟到，迟到三次算一次旷工

问题：我迟到了会怎么样？
```

#### ④ LLM 返回回答

```
根据员工手册，迟到15分钟以上记为迟到，迟到三次将计为一次旷工。
```

------

## 5. 第五步：把聊天记录存入 MySQL

插入 `chat_message`：

|  id  | user_id |      question      |                 answer                 | hit_chunk_ids |
| :--: | :-----: | :----------------: | :------------------------------------: | :-----------: |
|  1   |  1001   | 我迟到了会怎么样？ | 根据员工手册... 迟到三次计为一次旷工。 |       1       |

------

## 6. 整个数据流总结

#### 上传文档时

文件 → OSS / 本地

文本 → 切片 →

- 存 MySQL（chunk 表）
- 向量化 → 存向量库（vector + content + chunk_id）

#### 用户聊天时

问题 → 向量 → 向量库检索 → 拿到 chunk 文本 → LLM 生成回答 →

- 问答记录存入 MySQL（chat_message）
- 命中的 chunk_id 也一起存，方便溯源

------

## 7. 一张表看懂所有东西存在哪

|      数据      | 存在 MySQL | 存在 向量库 | 存在 文件存储 |
| :------------: | :--------: | :---------: | :-----------: |
| 文档标题、信息 |     ✅      |             |               |
| 整篇 PDF/Word  |            |             |       ✅       |
|   chunk 文本   |     ✅      |      ✅      |               |
|   chunk 向量   |            |      ✅      |               |
|    聊天记录    |     ✅      |             |               |
|    用户信息    |     ✅      |             |               |

------

## 8. 一些问题

#### Q：为什么 chunk 内容要存两遍？

- 向量库里存：检索完**直接用**，快
- MySQL 里存：方便后台管理、编辑、重新向量化、溯源

#### Q：能不能只存一边？

可以，但不推荐。

只存向量库：管理麻烦、丢了就没了。

只存 MySQL：检索完还要再查一次库，多一次 IO。

#### Q：向量库能不能存聊天记录？

绝对不要。

向量库不适合做业务增删改查、分页、关联查询。