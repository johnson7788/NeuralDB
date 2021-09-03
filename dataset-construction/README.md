# 从kelm生成神经数据库训练数据集

该资源库通过KELM映射的事实模板，为NDB训练数据集生成训练实例。

在一些脚本中，使用了KELM映射和Wikidata图。这些脚本假定它们已经被加载到MongoDB数据库中。

设置以下环境变量，将脚本指向适当的数据库。
```
MONGO_USER=
MONGO_PASSWORD=
MONGO_HOST=
MONGO_PORT=
MONGO_DB=
```

## 1. 将数据加载到数据库中

### 1.1 Wikidata
维基数据可以通过`wikidata_index.py`脚本导入MongoDB，但这需要几天时间来解压、导入和索引。
最简单的方法是恢复Mongo数据库的导出。

```
mongorestore --archive --gzip < mongo_wikidata_dump.gz
```

所有的索引都应该在恢复DB时生成。但是，如果不能这样做，应该为wiki_graph集合创建以下索引。
`wikidata_id, english_name, english_wiki, sitelinks.title`，以及为wiki_redirect集合`title`创建以下索引

### 1.2 KELM
从KELM导入映射是相当快的，通过运行`kelm_data.py`脚本完成。

```
python -m ndb_data.data_import.kelm_data <path_to_mapped_kelm>.jsonl
```
导入后，应该创建以下索引。`实体，关系，（实体，关系）`。

## 2. Dataset construction

问题和答案的生成分几个阶段进行，使用来自KELM的数据 对于ACL的实验。数据已经被检查过，并在ACL论文中发布了确定为`v2.4`的版本。

用于ACL论文的确切选择被存储在`scripts/`文件夹中的bash脚本中。


### 2.1 Build Initial Database
对数据库的初始事实集进行采样。这个过程的前半段很慢，因为它需要缓存KELM中的所有主题和实体。
这可以通过将KELM分割成更小的块并将结果汇总来更快地完成。

首先，我们需要对主体/实体/关系进行缓存，因为索引这些东西需要一段时间，而且这些东西可以被重复使用。

```
python -m ndb_data.construction.make_database_initial_cache kelm_file.jsonl
```

有了所有的主题对象和关系，这个文件产生了一个主题的分割，分为训练、开发集和测试，并输出3个文件。每个拆分都有一个。

```
python -m ndb_data.construction.make_database_initial kelm_file.jsonl out_file_prefix
```

这将挑选一个随机的关系，然后挑选参与这个关系的事实来添加到数据库。
其它参数:
```
    --num_dbs_to_make # How many databases to make (50,000). 
    --sample_rels # How many relations to sample per database (2)
    --sample_per_rel # How many entities to sample per relation (10)
    --sample_extra # How many to randomly sample
```

### 2.2 Finalize Database
这一步会在数据库中的事实基础上增加一跳的关系。这需要将KELM和Wikidata导入MongoDB（使用`dataset_creation/extraction/kelm_data.py`脚本）。

这将查询KELM数据集，以找到与DB中的实体和内容共享的事实。

这一步也将规范数据库中的事实，因为KELM经常会破坏实体的名称，从而破坏评分。

如果有太多的事实，这将放弃这些事实，这个文件可以用来设置数据库的大小。

```
python -m ndb_data.construction.make_database_finalize [out_file_prefix]_train second_phase_train.jsonl
python -m ndb_data.construction.make_database_finalize [out_file_prefix]_dev second_phase_dev.jsonl
python -m ndb_data.construction.make_database_finalize [out_file_prefix]_test second_phase_test.jsonl
``` 

### 2.3 Generate Questions
这需要数据库中的事实，并相应地生成问题。这也会产生复杂的问题。

```
python -m ndb_data.construction.make_questions second_phase_train.jsonl questions_train.jsonl
python -m ndb_data.construction.make_questions second_phase_dev.jsonl questions_dev.jsonl
python -m ndb_data.construction.make_questions second_phase_test.jsonl questions_test.jsonl
``` 

### 2.4 Stratified Sample Queries
问题生成脚本将为每个严重不平衡的查询生成所有问题。这种平衡在分档的支持集上进行多变量分层抽样，并试图在支持集大小上获得均匀分布。
```
python -m ndb_data.sample_questions questions_train.jsonl balanced_train.jsonl
python -m ndb_data.sample_questions questions_dev.jsonl balanced_dev.jsonl
python -m ndb_data.sample_questions questions_test.jsonl balanced_test.jsonl
``` 

### v2.5 [extra step for small databases]: Filter databases of more than 900 tokens
在基线实验中，我们过滤掉了包含900多个token的数据库，以评估在1024个token限制下可以在标准transformer模型中编码的数据库。
```
python -m ndb_data.generation.fiter_db_facts in_folder/ out_folder/
```