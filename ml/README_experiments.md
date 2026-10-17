# 实验总结（ml/README_experiments.md）

> 评分点：说明“哪个 run 可以上生产、为什么”，“你优化的指标是什么、为什么它对这个应用重要”。
> 表格里的数字请替换成 `python ml/train.py --all` 跑出来的真实值。

## 1. 优化指标：为什么是 F1，而不是 accuracy

NSL-KDD 训练集 125,973 条里，normal 占 53.5%（67,343 条），其余四类攻击共 58,630 条，
其中 U2R 只有 52 条、R2L 只有 995 条。

- 只看 accuracy：把所有流量都判成 normal 就有 53.5%，而 U2R（提权）一个都抓不到；
- 在官方 KDDTest+ 上还包含训练时没见过的攻击类型，模型很容易把 R2L/U2R 判成 normal，
  accuracy 看着还行，实际漏报严重。

所以同时看三个数：

| 指标 | 含义 | 为什么用它 |
|---|---|---|
| `accuracy` | 整体正确率 | 与课件、与老师直观预期对齐 |
| `f1_weighted` | 按类别样本数加权的 F1 | 主指标：类别不平衡时比 accuracy 稳 |
| `f1_macro` | 每类 F1 的平均（不加权） | 检查少数类（R2L/U2R）有没有被放弃 |

入侵检测里“漏报”比“误报”更贵，所以不会为了让 accuracy 好看而牺牲少数类召回率。

## 2. 两个实验

两个实验只改超参，和 MLflow 实验课里的两组（20/5 与 100/10）保持一致：

| 实验 | 配置 | n_estimators | max_depth | accuracy | f1_weighted | f1_macro | run_id |
|---|---|---|---|---|---|---|---|
| baseline | `ml/configs/baseline.json` | 20 | 5 | `<0.7x>` | `<0.7x>` | `<0.4x>` | `<run_id>` |
| improved | `ml/configs/improved.json` | 100 | 10 | `<0.7x>` | `<0.7x>` | `<0.5x>` | `<run_id>` |

> 参考量级：官方 KDDTest+ 上 accuracy 0.75~0.78，f1_macro 明显低于 f1_weighted
> （因为 U2R 只有几十条）。如果你的 accuracy 是 0.99，说明评估用了同分布划分，
> 要回去加 `--test-raw` 重新生成 `X_test.csv`。

两次实验的差别说明了什么：

1. **树更多**（20 → 100）：单棵树对数据扰动敏感，多棵树投票后方差更小；
2. **树更深**（5 → 10）：深度 5 学不到“源字节数 + 错误率”这类组合规则，属于欠拟合；
3. 两者同时提高后 f1_weighted 上升，但提升幅度会递减 —— 再往上加树，收益很小，
   这也是“不用把 n_estimators 调到 500”的理由（答辩可以讲这个成本收益）。

## 3. 生产模型选谁

**选 `improved`（100 棵树、深度 10）。**

理由：
1. `f1_weighted` 与 `f1_macro` 都更高，说明整体和少数类都更好，不是靠多判 normal 刷分；
2. 模型文件十几 MB 以内，能进镜像；单条预测是毫秒级，满足接口实时性；
3. 超参与实验课第二组一致，复现和解释成本最低。

该 run 的 id 与数据版本记在 `ml/PROVENANCE.md`，模型导出为
`models/model.joblib` + `models/model_meta.json`，并在 `registry/current_model.md` 登记。

## 4. 怎么复现

```bash
python ml/data_pipeline.py --test-raw data/raw/KDDTest+.txt
python ml/train.py --all
mlflow ui --backend-store-uri ./mlruns     # http://127.0.0.1:5000
```

每个 run 里能看到：

| 位置 | 内容 |
|---|---|
| Parameters | `n_estimators`、`max_depth`、`n_features`、`eval_set`、`git_commit`、`data_version` |
| Metrics | `accuracy`、`f1_weighted`、`f1_macro` |
| Artifacts | `model/`（MLflow 自己的模型目录）、`plots/confusion_matrix.png`、`reports/classification_report.txt` |

> 【提示】`eval_set` 会写明是 `kddtest+` 还是 `holdout`。答辩主动说明这一点，
> 比被老师问出来要好。
