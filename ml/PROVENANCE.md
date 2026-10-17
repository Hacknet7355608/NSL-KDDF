# 模型血缘（PROVENANCE）

> 一句话：任何一个上线的 `model.joblib`，都能沿着这张表倒推回
> “哪次实验、哪段代码、哪版数据”。这是 MLOps 与“跑个 notebook” 的根本区别。

## 1. 当前生产模型

| 项 | 值 |
|---|---|
| 模型文件 | `models/model.joblib` |
| 元数据 | `models/model_meta.json` |
| 实验名 / run_id | `nsl-kdd` / `<run_id>` |
| 配置 | `ml/configs/improved.json`（RandomForest） |
| 训练代码 commit | `<commit>` |
| 数据版本 | tag `data-v2`，DVC md5 `<md5>` |
| 训练时间 | `<2026-xx-xxTxx:xx:xx+00:00>` |
| sklearn 版本 | `<1.x.y>` |
| 指标（KDDTest+） | accuracy `<0.7x>` / f1_weighted `<0.7x>` / f1_macro `<0.5x>` |

## 2. 全部实验记录

| run_id | 实验 | 代码 commit | 数据版本（md5） | f1_weighted | 产出 |
|---|---|---|---|---|---|
| `<run_id_1>` | baseline（20 棵/深度 5） | `<commit>` | `<md5>` | `<0.7x>` | 仅 MLflow artifacts |
| `<run_id_2>` | improved（100 棵/深度 10） | `<commit>` | `<md5>` | `<0.7x>` | 已导出为 `models/` |

## 3. 怎么验证这条链没断

```bash
# 1) 模型文件里记的 run_id
python -c "import json;print(json.load(open('models/model_meta.json'))['mlflow_run_id'])"

# 2) 该 run 记录的数据版本
cat mlruns/0/<run_id>/params/data_version_md5

# 3) 当前 .dvc 指针是否仍指向同一版数据
git show data-v2:data/processed.dvc | grep md5

# 4) 代码是否就是那个 commit
git log --oneline -1 <commit>
```

四处对得上，说明“模型 ↔ 数据 ↔ 代码”三者是一致的；任何一处不一致，都说明
有人绕过了 DVC 或手工改过模型文件，属于评分会扣分的操作。

## 4. 模型文件里有什么

`models/model_meta.json` 是 API 与服务端的契约来源：

| 字段 | 用途 |
|---|---|
| `feature_names` | 特征顺序；API 按它组织输入、校验字段是否齐全 |
| `n_features` | 与 `feature_names` 长度一致，用于快速自检 |
| `classes` | 5 个类别名，顺序与 `model.classes_` 一致 |
| `mlflow_run_id` | 回到实验记录 |
| `data_version.dvc_md5` | 回到数据版本 |
| `categorical_encodings` | 3 个类别特征的字符串→整数映射，写接口文档用 |
| `metrics` | 该模型的评测结果，写 README/登记表用 |
