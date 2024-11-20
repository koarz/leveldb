### LevelDB 文件格式

```
<文件开始>
[data block 1]
[data block 2]
...
[data block N]
[meta block 1]
...
[meta block K]
[metaindex block]
[index block]
[Footer]        （固定大小；位于文件末尾，偏移量为 file_size - sizeof(Footer)）
<文件结束>
```

文件中包含内部指针，每个指针称为 `BlockHandle`，包含以下信息：

- **offset**: varint64  
- **size**: varint64  

关于 varint64 格式的详细说明，请参考 [varints](https://developers.google.com/protocol-buffers/docs/encoding#varints)。

### 文件结构描述

1. **数据块**  
   文件中的键值对按排序顺序存储，并划分为一系列数据块。这些数据块按顺序位于文件的开头。每个数据块根据 `block_builder.cc` 中的代码进行格式化，然后可以选择进行压缩。

2. **元数据块**  
   数据块之后存储了一系列元数据块。目前支持的元数据块类型如下所述，未来可能会增加更多元数据块类型。每个元数据块同样由 `block_builder.cc` 格式化，并可选择压缩。

3. **Metaindex 块**  
   包含每个元数据块的条目，其中键为元数据块的名称，值为指向该元数据块的 `BlockHandle`。

4. **索引块**  
   包含每个数据块的条目。条目的键是一个字符串，该字符串大于等于该数据块的最后一个键且小于下一个数据块的第一个键。条目的值为该数据块的 `BlockHandle`。

5. **Footer**  
   文件末尾为固定长度的 Footer，包含 Metaindex 和索引块的 `BlockHandle`，以及一个魔数。

   ```
   metaindex_handle: char[p];     // Metaindex 的块句柄
   index_handle:     char[q];     // 索引块的块句柄
   padding:          char[40-p-q];// 填充字节，以固定长度
                                  // (40==2*BlockHandle::kMaxEncodedLength)
   magic:            fixed64;     // 魔数：0xdb4775248b80fb57 (小端格式)
   ```

---

### "filter" 元数据块

如果在打开数据库时指定了 `FilterPolicy`，每个表都会存储一个过滤器块。`Metaindex` 块包含一个条目，将 `filter.<N>` 映射到过滤器块的 `BlockHandle`，其中 `<N>` 是过滤器策略的 `Name()` 方法返回的字符串。

过滤器块存储了一系列过滤器，其中过滤器 i 包含 `FilterPolicy::CreateFilter()` 方法生成的结果，作用于文件偏移量位于以下范围内的所有键：

```
[i*base ... (i+1)*base-1]
```

当前，`base` 值为 2KB。例如，如果块 X 和块 Y 的起始偏移量位于 `[0KB .. 2KB-1]` 范围内，则 X 和 Y 中的所有键会通过调用 `FilterPolicy::CreateFilter()` 转换为过滤器，并将结果存储为过滤器块中的第一个过滤器。

过滤器块的格式如下：

```
[filter 0]
[filter 1]
[filter 2]
...
[filter N-1]

[filter 0 的偏移量]                : 4 字节
[filter 1 的偏移量]                : 4 字节
[filter 2 的偏移量]                : 4 字节
...
[filter N-1 的偏移量]              : 4 字节

[偏移量数组的起始位置]              : 4 字节
lg(base)                           : 1 字节
```

过滤器块末尾的偏移量数组支持高效地将数据块偏移量映射到对应的过滤器。

---

### "stats" 元数据块

此元数据块包含一系列统计信息，其中键为统计项名称，值为统计值。

**TODO（发布后）**：记录以下统计信息：
- 数据大小
- 索引大小
- 键大小（未压缩）
- 值大小（未压缩）
- 条目数量
- 数据块数量