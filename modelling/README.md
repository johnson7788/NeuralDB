# Neural database implementation for Database Reasoning over Text ACL paper

## Package Structure

```
resources/                       # DB files go here

src/
    neuraldb/                    # Python package containing NeuralDB project
       
        dataset/                 # Components pertaining to reading and loading the DB files
            instance_generator/  # Generates model inputs from different DB formats

        evaluation/              # Helper functions that allow in-line evaluation of the models from the training script
        modelling/               # Extra models/trainers etc
        retriever/               # TF-IDF and DPR baselines
        util/                    # Other utils
        
tests/                           # Unit tests for scorer
```

## TF-IDF and DPR retrieval

可以先运行基线检索方法，收集将被下游模型使用的数据

```
bash scripts/baselines/retrieve.sh dpr
bash scripts/baselines/retrieve.sh tfidf
```

## 重复 ACL论文中实验

这些将为包含多达25个事实的v2.4数据库训练和生成预测。
脚本使用task spooler来管理一个作业队列。如果你没有这个功能，请从脚本中删除`tsp`。

```
export SEED=1
bash scripts/experiments_ours.sh v2.4_25
bash scripts/experiments_baseline.sh v2.4_25
```
最后的评分脚本将采用这些脚本产生的预测结果，并对照参考预测结果进行评估。

```
python -m neuraldb.final_scoring
``` 
根据数据库大小绘制答案准确性的图是由以下内容产生的
```
python -m neuraldb.final_scoring_with_db_size
``` 

### Larger databases
该评分脚本有几个变体，可用于评估更大的数据库（v2.4_50、v2.4_100、v2.4_250、v2.4_500和v2.4_1000）。
这将涉及在更大的数据库中运行在25个事实上训练的模型。

```
bash scripts/ours/predict_spj_rand_sweep.sh
python -m neuraldb.final_scoring_with_db_size_sweep
```

### Fusion in decoder baseline

是使用从https://github.com/facebookresearch/FiD 改编的FiD代码的修改版进行的，其输出可以转换为NeuralDB格式，用

```
python -m neuraldb.convert_legacy_predictions <fid_output> <output_file>
```