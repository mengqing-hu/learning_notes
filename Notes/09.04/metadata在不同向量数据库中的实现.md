## metadata 在不同向量数据库中的实现

#### Pinecone

- metadata = JSON
- 支持复杂 filter（$eq, $gte, $in）

#### Weaviate

- metadata = schema（更结构化）
- 支持 GraphQL 查询

#### FAISS

- 不原生支持 metadata
- 通常要“自己维护”（比如 list / dict）

#### Chroma

- 内置 metadata 支持
- 很适合 LangChain



## 常见坑

1. metadata 太多会影响：查询速度和内存

2. metadata 不结构化

```
"info": "this is a paper from 2023 about AI"
```

不可过滤，应该拆成：

```
{
    "year": 2023,
    "topic": "AI"
}
```

3. 忘记用 metadata

很多人只用 similarity → 精度低