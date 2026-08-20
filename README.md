# D3：预测下一小时家庭用电功率

本仓库是 Day 3 独立提交：根据一户家庭的真实历史用电记录，比较"下一小时≈最近一小时"的持续性基线与一个固定种子随机森林模型，预测下一小时平均有功功率（kW）。新同学只读本文件即可从零开始运行。

## 1. 问题

- **使用者：** 希望提前看到用电变化、但不能依赖昂贵系统的家庭能源研究小组。
- **真实输入：** 一户家庭连续记录的分钟级用电数据（`Global_active_power` 等）。
- **需要的输出：** 每个整点到来之前，估计下一小时平均有功功率（kW）。
- **与使用者最相关的错误：** 预测与真实下一小时平均值相差很大的时刻，尤其是晚间突增或突然低谷。
- **本日产品边界：** 只做单户历史数据实验，不直接控制电器，也不承诺推广到其他家庭或用于电网控制/安全告警。

## 2. 真实数据

- **数据集：** Individual Household Electric Power Consumption（家庭用电功耗）
- **所有者/发布者：** UCI Machine Learning Repository
- **下载入口（Kaggle）：** https://www.kaggle.com/datasets/uciml/electric-power-consumption-data-set
- **原始说明（UCI）：** https://archive.ics.uci.edu/dataset/235/individual+household+electric+power+consumption
- **许可标签：** Kaggle 页面需账号同意下载条款；原始 UCI 条目说明许可与变量缺失值（详见上面的 UCI 链接）。
- **内容：** 一户家庭 2006-12 至 2010-11 约 207 万条分钟级记录。
- **本日使用窗口：** 取文件前 150,000 条分钟记录，聚合成 2,501 条小时平均（2006-12-16 17:00 至 2007-03-30 21:00）。
- **下载位置：** 必须放在 `data/raw/household_power_consumption.txt`，不改名、不生成替代信号、不把原始大数据提交进仓库（已被 `.gitignore` 排除）。

## 3. 环境

- Windows PowerShell
- Python 3.14（本项目在 Python 3.14.7 下验证）
- 依赖见 `requirements.txt`：`numpy`、`pandas`、`scikit-learn`、`matplotlib`

## 4. 安装

在仓库根目录（本文件夹）运行：

```powershell
python -m pip install -r requirements.txt
```

## 5. 运行

第一次运行先准备小时级真实数据（把分钟记录聚合为小时）：

```powershell
python analyze.py --prepare
```

预期输出：

```text
REAL DATA PREPARATION PASSED
source_rows_requested: 150000
hourly_rows: 2501
start: 2006-12-16 17:00:00
end: 2007-03-30 21:00:00
output: data\processed\hourly_power.csv
```

然后运行主程序，比较基线与候选模型：

```powershell
python analyze.py
```

程序会输出 `metrics.json`（同条件对比结果）、`forecast.png`（首个留出周的曲线图）和 `largest_errors.csv`（误差最大的 12 个真实小时）。

## 6. 预期输出（本机实测）

`metrics.json` 中的关键数字（命令可复现）：

| 项目 | 基线（持续性） | 候选（随机森林） |
| --- | ---: | ---: |
| MAE（kW） | 0.794 | 0.577 |
| RMSE（kW） | 1.112 | 0.796 |
| 高需求召回率（阈值 3.162 kW） | 0.184 | 0.132 |

- 训练 / 测试行数：1,980 / 496（严格按时间顺序划分，测试为后 20%）。
- 候选在平均误差上更好（MAE 降约 27%），但高需求召回率更低——平均更准，却仍漏报大部分高需求小时。
- 最大误差案例：`2007-03-28 17:00` 真实 0.386 kW、预测 3.461 kW（绝对误差 3.076 kW，低谷被误报为高需求）。

## 7. 测试

```powershell
python -m unittest discover -s tests -v
```

预期：2 个测试全部通过（`chronological_split` 保持时间顺序；`make_lagged` 只用过去数据预测下一小时）。

## 8. 限制

- 只有一户家庭的历史数据，不能代表其他家庭，也不能外推到其他时段或地区。
- 不能用于电网控制、安全告警或任何真实决策——模型会误报低谷、漏报高需求。
- 平均指标（MAE/RMSE）掩盖了高需求时段的漏报；评估时必须同时看最大误差与召回率。
- 本实验是单日窗口，不代表完整 2006-2010 数据集的表现。

## 9. 文件结构

```text
data/raw/household_power_consumption.txt   # 原始真实数据（不提交）
data/processed/hourly_power.csv            # 小时级数据（不提交）
analyze.py                                 # 数据准备 + 基线/候选对比主程序
tests/test_sequence.py                     # 单元测试
metrics.json                               # 同条件对比结果
forecast.png                               # 首个留出周曲线
largest_errors.csv                         # 最大误差真实案例
report.md                                  # 书面报告
presentation.pptx                          # 3 分钟答辩
submission.json                            # 提交清单
```
