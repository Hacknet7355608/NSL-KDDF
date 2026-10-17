# 数据说明（ml/README_data.md）

本文件是老师评分点“DVC 数据版本”的书面证据，缺一项都会扣分。
A 同学填完后，请把 `<>` 占位符替换成真实内容。

## 1. 数据来源

| 项 | 内容 |
|---|---|
| 数据集 | NSL-KDD（KDD'99 的去冗余改进版） |
| 下载地址 | https://github.com/jmnwong/NSL-KDD-Dataset （UNB 官方站点已停止提供下载，此仓库为公开镜像） |
| 备用地址 | https://github.com/defcom17/NSL_KDD （含 KDDTrain+.csv / KDDTest+.txt） |
| 原始文件 | `data/raw/KDDTrain+.txt`（训练，125,973 行）、`data/raw/KDDTest+.txt`（官方测试，22,544 行） |
| 文件格式 | 无表头 CSV，43 列 = 41 个特征 + `label` + `difficulty` |
| 许可证 | 允许再分发与镜像，但使用必须引用原文（见下） |
| 引用 | Tavallaee, Bagheri, Lu, Ghorbani, "A Detailed Analysis of the KDD CUP 99 Data Set", CISDA 2009 |

## 2. 为什么用这份数据

- 41 个特征全部是数值或可枚举的类别，不需要复杂特征工程，适合 2 周工期；
- 标签天然是 5 大类（normal / DoS / Probe / R2L / U2R），正好是“多分类 + 类别不平衡”的真实案例；
- 类别极不平衡（U2R 只有 52 条），可以顺理成章地讲清“为什么不能只看 accuracy”。

## 3. v1 → v2 到底改了什么

| 版本 | 内容 | Git tag |
|---|---|---|
| v1 | 官方原始 txt 文件，一个字节都没改 | `data-v1` |
| v2 | 5 类标签合并 + 类别特征整数编码 + 删除全常量列 | `data-v2` |

v2 的三处处理及其理由：

1. **标签合并**：原始 22~39 种攻击名（`neptune.`、`smurf.`、`satan.` ……）合并为 5 大类。
   粒度太细会让每类样本过少，也无法与文献结果对比。
   映射表写在 `ml/data_pipeline.py` 的 `ATTACK_MAP`，覆盖 KDDTest+ 里额外出现的 17 种攻击；
   遇到未映射的攻击名直接报错，绝不静默丢掉样本。
2. **类别特征编码**：`protocol_type` / `service` / `flag` 三个字符串列编码成整数。
   用“排序后固定映射表”而不是 `LabelEncoder`，映射写进 `data/processed/encoders.json`，
   并随模型导出到 `models/model_meta.json`，API 端必须使用同一份映射。
3. **删除全常量列**：`num_outbound_cmds` 在整份数据里恒为 0（`is_host_login` 亦近乎恒为 0），
   对分类没有任何信息量，删掉可以简化 API 的输入契约。实际删除的列记录在
   `data/processed/dataset_stats.json` 的 `dropped_constant_columns` 字段。

> 【注意】v2 不是“随便改改”。必须做到“同一个 Git commit + 同一份 DVC 指针 → 一定能复现出同一份 X.csv”，
> 所以预处理脚本本身也要提交，参数要写死或来自配置文件。

## 4. 版本 ↔ commit ↔ DVC 哈希 对照表

> 这张表是评分点“which commit hash corresponds to which dataset version”的直接答案。
> 填法：`git log --oneline -- data/raw.dvc`，再看 `data/raw.dvc` 里 `md5:` 的值。

| 版本 | Git tag | 对应 commit | DVC 指针文件 | DVC 内容哈希（md5） | 行数 | 特征数 |
|---|---|---|---|---|---|---|
| v1 | data-v1 | `<commit>` | `data/raw.dvc` | `<md5>` | 125,973 + 22,544 | 41 |
| v2 | data-v2 | `<commit>` | `data/processed.dvc` | `<md5>` | `<行数>` | `<特征数>` |

补充：v2 的 DVC 哈希会被 `ml/train.py` 自动写进每次 MLflow run 的
`data_version_md5` 参数，从而可以反向回答“这个模型是用哪版数据训练的”。

## 5. 怎么复现

```bash
git checkout data-v2
dvc pull                                  # 从 Azure 拉回 v2 数据
python ml/data_pipeline.py                # 若想从原始数据重建 v2
python ml/train.py --all
```

```bash
git checkout data-v1
dvc pull                                  # 切回 v1
python ml/data_pipeline.py                # v1 是原始 txt，需要自己再跑一遍预处理才有 v2
```

## 6. 什么进了 Git，什么没进

| 内容 | 位置 | 说明 |
|---|---|---|
| 19MB 原始 txt | Azure 容器 `wut2026/dvc` | 只通过 `dvc push/pull` 传输 |
| `.dvc` 指针（几百字节） | Git | 记录内容哈希，不含数据 |
| `encoders.json`、`dataset_stats.json` | Git（在 v2 里一起被 DVC 跟踪） | 保证 API 编码一致、可追溯 |
| SAS token | `.dvc/config.local` | 本地文件，被 DVC 默认忽略，绝不提交 |
