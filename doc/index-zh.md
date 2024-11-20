# Leveldb  

_作者：Jeff Dean, Sanjay Ghemawat_

Leveldb 是一个持久化的键值存储库库。键和值可以是任意的字节数组。键在存储库内按照用户指定的比较函数排序。

---

## 打开数据库  

Leveldb 数据库有一个名称，与文件系统中的某个目录对应。数据库的所有内容都存储在该目录中。以下示例展示了如何打开一个数据库，并在必要时创建它：  

```cpp
#include <cassert>
#include "leveldb/db.h"

leveldb::DB* db;
leveldb::Options options;
options.create_if_missing = true;
leveldb::Status status = leveldb::DB::Open(options, "/tmp/testdb", &db);
assert(status.ok());
...
```

如果希望在数据库已存在时引发错误，可以在调用 `leveldb::DB::Open` 前添加以下代码：  

```cpp
options.error_if_exists = true;
```

---

## 状态检查  

上文提到了 `leveldb::Status` 类型。Leveldb 中大多数可能发生错误的函数都会返回该类型的值。您可以通过检查返回值判断是否成功，并输出相关错误消息：  

```cpp
leveldb::Status s = ...;
if (!s.ok()) cerr << s.ToString() << endl;
```

---

## 关闭数据库  

使用完数据库后，只需删除数据库对象即可。示例如下：  

```cpp
... 按上述方式打开数据库 ...
... 对数据库进行操作 ...
delete db;
```

---

## 读写操作  

数据库提供了 `Put`、`Delete` 和 `Get` 方法用于修改或查询数据。例如，以下代码将存储在 `key1` 下的值移动到 `key2`：  

```cpp
std::string value;
leveldb::Status s = db->Get(leveldb::ReadOptions(), key1, &value);
if (s.ok()) s = db->Put(leveldb::WriteOptions(), key2, value);
if (s.ok()) s = db->Delete(leveldb::WriteOptions(), key1);
```

---

## 原子更新  

注意：如果在执行 `key2` 的 `Put` 操作后但在 `key1` 的 `Delete` 操作前进程崩溃，则同一个值可能会同时存储在多个键下。可以通过 `WriteBatch` 类原子性地应用一组更新来避免此类问题：  

```cpp
#include "leveldb/write_batch.h"
...
std::string value;
leveldb::Status s = db->Get(leveldb::ReadOptions(), key1, &value);
if (s.ok()) {
  leveldb::WriteBatch batch;
  batch.Delete(key1);
  batch.Put(key2, value);
  s = db->Write(leveldb::WriteOptions(), &batch);
}
```

`WriteBatch` 保存了需要对数据库执行的一系列编辑操作，并按顺序应用这些编辑。注意，我们先调用 `Delete` 再调用 `Put`，这样即使 `key1` 和 `key2` 相同，也不会错误地完全删除值。

除了原子性带来的好处外，`WriteBatch` 还可以通过将大量单独的操作放入同一个批次来加速批量更新。

## 同步写入  

默认情况下，对 Leveldb 的每次写入都是异步的：写入操作会在将数据从进程推送到操作系统后立即返回。数据从操作系统内存传输到底层持久化存储是异步进行的。可以通过开启同步标志使特定的写入操作在数据完全写入持久化存储后才返回结果。（在 POSIX 系统上，这通过调用 `fsync(...)`、`fdatasync(...)` 或 `msync(..., MS_SYNC)` 实现。）  

示例如下：  

```c++  
leveldb::WriteOptions write_options;  
write_options.sync = true;  
db->Put(write_options, ...);  
```  

异步写入的速度通常比同步写入快上千倍。异步写入的缺点是，如果机器崩溃，可能会导致最近几次更新丢失。不过，如果只是写入进程崩溃（即不涉及重启），不会导致数据丢失，因为即使 `sync` 为 `false`，更新在从进程内存推送到操作系统后才被视为完成。  

异步写入通常可以安全使用。例如，在将大量数据加载到数据库时，如果发生数据丢失，可以通过重新启动批量加载来恢复数据。一种折中的方法是，每第 N 次写入执行一次同步写入。在崩溃的情况下，只需从上次同步写入完成的位置重新启动批量加载即可。（同步写入可以更新一个标记，用来描述崩溃时从哪里重新启动。）  

`WriteBatch` 提供了异步写入的另一种替代方法。多个更新可以放入同一个 `WriteBatch`，然后通过一次同步写入（即将 `write_options.sync` 设置为 `true`）一起应用。同步写入的额外开销会分摊到批量中的所有写入操作中。  

---

## 并发  

数据库在同一时间只能由一个进程打开。Leveldb 的实现通过从操作系统获取锁来防止误用。在同一个进程中，多个并发线程可以安全地共享同一个 `leveldb::DB` 对象。例如，不同线程可以在同一个数据库中执行写入、获取迭代器或调用 `Get`，而无需额外的同步机制（Leveldb 的实现会自动处理所需的同步）。  

然而，其他对象（例如 `Iterator` 和 `WriteBatch`）可能需要外部同步。如果两个线程共享这些对象，必须使用自己的锁定协议来保护对这些对象的访问。更多细节可参考公共头文件中的说明。  

---

## 迭代  

以下示例展示如何打印数据库中的所有键值对：  

```c++  
leveldb::Iterator* it = db->NewIterator(leveldb::ReadOptions());  
for (it->SeekToFirst(); it->Valid(); it->Next()) {  
  cout << it->key().ToString() << ": "  << it->value().ToString() << endl;  
}  
assert(it->status().ok());  // 检查扫描期间是否存在错误  
delete it;  
```  

以下是一个处理指定范围 `[start, limit)` 内的键的示例：  

```c++  
for (it->Seek(start);  
   it->Valid() && it->key().ToString() < limit;  
   it->Next()) {  
  ...  
}  
```  

还可以按逆序处理条目。（注意：逆序迭代可能比正向迭代稍慢。）  

```c++  
for (it->SeekToLast(); it->Valid(); it->Prev()) {  
  ...  
}  
```  

## 快照（Snapshots）

快照提供了整个键值存储状态的**一致性只读视图**。通过设置 `ReadOptions::snapshot` 为非 NULL，表示读取操作应基于数据库状态的特定版本。如果 `ReadOptions::snapshot` 为 NULL，则读取操作将在当前状态的隐式快照上进行。

可以使用 `DB::GetSnapshot()` 方法创建快照：

```c++
leveldb::ReadOptions options;
options.snapshot = db->GetSnapshot();
... // 对数据库进行一些更新操作
leveldb::Iterator* iter = db->NewIterator(options);
... // 使用迭代器读取快照创建时的状态
delete iter;
db->ReleaseSnapshot(options.snapshot);
```

### 注意事项：
1. **快照释放**：当快照不再需要时，必须通过 `DB::ReleaseSnapshot` 接口释放它。这使得底层实现可以清理支持该快照读取所需的状态，释放相关资源。
2. **快照用途**：快照主要用于在更新数据库的同时提供一致的只读视图，确保读取过程中不会受到后续更新的影响。

---

## 切片（Slice）

在上面代码中的 `it->key()` 和 `it->value()` 返回值是 `leveldb::Slice` 类型的实例。`Slice` 是一个简单的数据结构，包含一个长度值和指向外部字节数组的指针。相比返回 `std::string`，返回 `Slice` 更高效，因为它避免了拷贝可能较大的键和值。此外，Leveldb 的方法不会返回以 NULL 结尾的 C 风格字符串，因为 Leveldb 的键和值允许包含 `'\0'` 字节。

### 示例

#### 字符串到 Slice 的转换：
```c++
leveldb::Slice s1 = "hello";

std::string str("world");
leveldb::Slice s2 = str;
```

#### Slice 到字符串的转换：
```c++
std::string str = s1.ToString();
assert(str == std::string("hello"));
```

---

### 使用 Slice 时的注意事项：

使用 `Slice` 时，**需要调用者确保 `Slice` 所指向的外部字节数组在其生命周期内有效**。以下代码是一个容易出错的示例：

```c++
leveldb::Slice slice;
if (...) {
  std::string str = ...; // str 是局部变量
  slice = str;           // slice 指向 str 的内存
}
Use(slice);              // 此时 str 已经被销毁，slice 的内存无效
```

当 if 语句结束后，`str` 被销毁，`slice` 的底层存储不再存在。这会导致后续对 `slice` 的操作产生未定义行为。

---

## 比较器 (Comparators)

前面的示例使用了默认的键排序函数，即按字节的字典序排序。然而，在打开数据库时，可以提供自定义的比较器。例如，假设数据库中的每个键由两个数字组成，并需要先按第一个数字排序，在第一个数字相同时按第二个数字排序。首先，定义一个表达此规则的 `leveldb::Comparator` 子类：

```c++
class TwoPartComparator : public leveldb::Comparator {
 public:
  // 三向比较函数：
  // 如果 a < b：返回负数
  // 如果 a > b：返回正数
  // 否则：返回 0
  int Compare(const leveldb::Slice& a, const leveldb::Slice& b) const {
    int a1, a2, b1, b2;
    ParseKey(a, &a1, &a2);
    ParseKey(b, &b1, &b2);
    if (a1 < b1) return -1;
    if (a1 > b1) return +1;
    if (a2 < b2) return -1;
    if (a2 > b2) return +1;
    return 0;
  }

  // 暂时忽略以下方法：
  const char* Name() const { return "TwoPartComparator"; }
  void FindShortestSeparator(std::string*, const leveldb::Slice&) const {}
  void FindShortSuccessor(std::string*) const {}
};
```

现在，使用这个自定义比较器创建数据库：

```c++
TwoPartComparator cmp;
leveldb::DB* db;
leveldb::Options options;
options.create_if_missing = true;
options.comparator = &cmp;
leveldb::Status status = leveldb::DB::Open(options, "/tmp/testdb", &db);
...
```

---

## 向后兼容性

比较器 `Name` 方法的返回值会在数据库创建时附加到数据库上，并在每次打开数据库时进行检查。如果名称发生变化，则 `leveldb::DB::Open` 调用将失败。因此，只有在新的键格式和比较函数与现有数据库不兼容并且可以丢弃所有现有数据库内容时，才应更改比较器名称。

然而，通过一些预先的规划，仍然可以逐步演进键格式。例如，可以在每个键的末尾存储一个版本号（一个字节足以满足大多数用途）。当需要切换到新的键格式时（例如为 `TwoPartComparator` 处理的键添加一个可选的第三部分），可以：
1. 保持比较器名称不变。
2. 对新键递增版本号。
3. 修改比较器函数，使其使用键中的版本号来决定如何解释键。

---

## 性能

可以通过更改 `include/options.h` 中定义的默认值来调整性能。这允许用户根据特定的用例对数据库进行优化。

## 块大小 (Block size)

LevelDB 将相邻的键分组到同一个块中，而该块是从持久存储中传输数据的基本单位。默认块大小约为 4096 字节（未压缩）。对于主要进行大规模扫描操作的应用程序，可以考虑增加块大小。而对于频繁执行小值点读取的应用程序，如果性能测试显示有所改进，则可以选择较小的块大小。通常情况下，块大小不建议小于 1KB 或大于几 MB。此外，较大的块大小通常能提升压缩效率。

---

## 压缩 (Compression)

每个块在写入持久存储之前都会被单独压缩。默认情况下启用了压缩，因为默认的压缩方法非常快，并会在数据无法压缩时自动禁用压缩。在极少数情况下，应用程序可能需要完全禁用压缩，但仅当基准测试显示性能有所提升时才应考虑：

```c++
leveldb::Options options;
options.compression = leveldb::kNoCompression;
... leveldb::DB::Open(options, name, ...) ....
```

---

## 缓存 (Cache)

数据库内容存储在文件系统中的一组文件中，每个文件包含一系列压缩的块。如果 `options.block_cache` 非空，则会使用它来缓存常用的未压缩块内容：

```c++
#include "leveldb/cache.h"

leveldb::Options options;
options.block_cache = leveldb::NewLRUCache(100 * 1048576);  // 100MB 缓存
leveldb::DB* db;
leveldb::DB::Open(options, name, &db);
... 使用数据库 ...
delete db;
delete options.block_cache;
```

注意，缓存保存的是未压缩的数据，因此缓存大小应根据应用级别的数据大小来设置，而不需要考虑压缩的影响。（压缩块的缓存通常交由操作系统的缓冲区缓存或用户自定义的 `Env` 实现处理。）

在进行大规模读取时，应用程序可能希望禁用缓存，以防止大规模读取的数据替换掉缓存中的大部分内容。这可以通过每个迭代器的选项来实现：

```c++
leveldb::ReadOptions options;
options.fill_cache = false;
leveldb::Iterator* it = db->NewIterator(options);
for (it->SeekToFirst(); it->Valid(); it->Next()) {
  ...
}
delete it;
```

---

## 键布局 (Key Layout)

磁盘传输和缓存的单位是块。相邻的键（根据数据库的排序顺序）通常会被放置在同一个块中。因此，应用程序可以通过将经常一起访问的键放在相邻的位置，以及将不常用的键放在键空间的单独区域中来提高性能。

例如，假设我们在 LevelDB 之上实现一个简单的文件系统。我们可能希望存储以下类型的条目：

- `filename -> 权限位, 长度, 文件块 ID 列表`
- `file_block_id -> 数据`

我们可以为文件名键添加一个前缀（例如 `'/'`），为 `file_block_id` 键添加另一个前缀（例如 `'0'`），以便仅扫描元数据时，不需要加载和缓存大量的文件内容。

## 过滤器 (Filters)

由于 LevelDB 在磁盘上的数据组织方式，单次 `Get()` 操作可能涉及多次磁盘读取。可以使用可选的 `FilterPolicy` 机制来显著减少磁盘读取次数。

```c++
leveldb::Options options;
options.filter_policy = NewBloomFilterPolicy(10);
leveldb::DB* db;
leveldb::DB::Open(options, "/tmp/testdb", &db);
... 使用数据库 ...
delete db;
delete options.filter_policy;
```

上述代码为数据库关联了一种基于布隆过滤器（Bloom Filter）的过滤策略。布隆过滤器通过为每个键在内存中保存一定数量的位（此处为每个键 10 位）来工作。这种过滤器可将 `Get()` 操作所需的不必要磁盘读取减少约 100 倍。增加每个键的位数可以进一步减少不必要的磁盘读取，但会增加内存使用量。对于工作集无法完全装入内存且执行大量随机读取的应用程序，建议设置过滤策略。

如果您使用的是自定义比较器，请确保所使用的过滤策略与您的比较器兼容。例如，假设某个比较器在比较键时忽略尾随空格。在这种情况下，不能直接使用 `NewBloomFilterPolicy`，而应该提供一个自定义的过滤策略，该策略同样忽略尾随空格。例如：

```c++
class CustomFilterPolicy : public leveldb::FilterPolicy {
 private:
  leveldb::FilterPolicy* builtin_policy_;

 public:
  CustomFilterPolicy() : builtin_policy_(leveldb::NewBloomFilterPolicy(10)) {}
  ~CustomFilterPolicy() { delete builtin_policy_; }

  const char* Name() const { return "IgnoreTrailingSpacesFilter"; }

  void CreateFilter(const leveldb::Slice* keys, int n, std::string* dst) const {
    // 在移除尾随空格后使用内置布隆过滤器
    std::vector<leveldb::Slice> trimmed(n);
    for (int i = 0; i < n; i++) {
      trimmed[i] = RemoveTrailingSpaces(keys[i]);
    }
    builtin_policy_->CreateFilter(trimmed.data(), n, dst);
  }
};
```

高级应用程序可以提供自定义过滤策略，不使用布隆过滤器，而是采用其他机制来概括键集。有关详细信息，请参阅 `leveldb/filter_policy.h`。

---

## 校验和 (Checksums)

LevelDB 为它在文件系统中存储的所有数据关联了校验和。可以通过以下两种方式控制校验和验证的严格程度：

- 设置 `ReadOptions::verify_checksums` 为 `true`，强制对特定读取操作涉及的所有文件系统数据进行校验和验证。默认情况下不执行这种验证。
  
- 在打开数据库之前，将 `Options::paranoid_checks` 设置为 `true`，以便在检测到内部数据损坏时立即引发错误。具体来说，如果某部分数据库已损坏，错误可能会在打开数据库时引发，也可能在后续的数据库操作中触发。默认情况下，此选项关闭，允许在部分持久存储已损坏的情况下继续使用数据库。

如果数据库已损坏（例如在启用严格检查时无法打开），可以使用 `leveldb::RepairDB` 函数尽可能恢复数据。

## 近似大小 (Approximate Sizes)

`GetApproximateSizes` 可以被用于获取储存了一个或者多个键的范围的文件系统空间占用字节的近似大小。

```c++
leveldb::Range ranges[2];
ranges[0] = leveldb::Range("a", "c");
ranges[1] = leveldb::Range("x", "z");
uint64_t sizes[2];
db->GetApproximateSizes(ranges, 2, sizes);
```

上述调用将会将 `sizes[0]` 设置为键范围 `[a..c)` 使用的文件系统空间的近似字节数，并将 `sizes[1]` 设置为键范围 `[x..z)` 使用的近似字节数。

## 环境 (Environment)

所有由 leveldb 发起的文件操作 (和其他的系统调用操作) 都是通过一个`leveldb::Env` 对象进行处理。高级用户可能会想通过他们自己提供的 Env 实现对系统更好的控制。例如, 一个应用可能会在文件IO路径中引入人工延迟去限制leveldb对系统中其他活动的影响.

```c++
class SlowEnv : public leveldb::Env {
  ... Env 的接口实现 ...
};

SlowEnv env;
leveldb::Options options;
options.env = &env;
Status s = leveldb::DB::Open(options, ...);
```

## Porting

要将 leveldb 移植到新的平台，可以通过实现 `leveldb/port/port.h` 中定义的类型、方法和函数的特定平台版本来完成。有关详细信息，请参考 `leveldb/port/port_example.h`。

此外，新平台可能需要一个新的默认 `leveldb::Env` 实现。请参阅 `leveldb/util/env_posix.h` 获取示例。