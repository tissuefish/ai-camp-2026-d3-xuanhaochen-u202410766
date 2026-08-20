# 每日作业报告

> 本文档依据 `templates/submission/daily-report-template.md` 填写。除标注"（学生填写）"的个人反思部分外，命令、数字与输出均来自本机真实运行，可用 README 命令重新产生。

## 1. 本日问题

- 里程碑：day-03
- 学生或小组：（学生填写姓名或组名）
- 使用者：家庭能源研究小组——希望提前看到用电变化、但不能依赖昂贵系统的用户。
- 真实输入：一户家庭 2006-12 至 2007-03 的分钟级真实用电记录（取文件前 150,000 分钟，聚合为 2,501 小时）。
- 需要的输出：每个整点到来之前，估计下一小时平均有功功率（kW）。
- 与使用者最相关的错误：晚间高需求突增被漏报、低谷被误报；绝对误差最大的真实小时。
- 本日产品边界：单户历史数据实验，不控制电器，不承诺推广到其他家庭，不用于电网控制或安全告警。

## 2. 真实数据或真实课程输入

- 所有者/发布者：UCI Machine Learning Repository
- 标题：Individual Household Electric Power Consumption
- 原始 URL：https://archive.ics.uci.edu/dataset/235/individual+household+electric+power+consumption
- 下载入口（Kaggle）：https://www.kaggle.com/datasets/uciml/electric-power-consumption-data-set
- 许可标签或使用许可：Kaggle 页面需账号同意下载条款；原始 UCI 条目说明许可、变量与缺失值（详见 UCI 链接）。
- 下载/取得日期：（学生填写）
- 预期文件与结构：`data/raw/household_power_consumption.txt`，分号分隔，含 `Date`、`Time`、`Global_active_power` 等 9 列。
- 检查命令：`python analyze.py --prepare`
- 实际检查结果：`REAL DATA PREPARATION PASSED`；`source_rows_requested: 150000`；`hourly_rows: 2501`；`2006-12-16 17:00:00` → `2007-03-30 21:00:00`。
- 已知缺失、偏差或限制：窗口内分钟级缺失值被丢弃后再按小时聚合；只有一户、一个季节窗口，存在家庭与时间偏差。

## 3. 可复现运行

```powershell
# 当前目录
student-work\day-03-power

# 安装
python -m pip install -r requirements.txt

# 数据检查与准备
python analyze.py --prepare
# 预期：REAL DATA PREPARATION PASSED；hourly_rows: 2501

# 主程序（输出 metrics.json / forecast.png / largest_errors.csv）
python analyze.py

# 测试
python -m unittest discover -s tests -v
# 预期：Ran 2 tests ... OK
```

关键预期输出与实际输出文件位置：`metrics.json`（同条件对比）、`forecast.png`（首个留出周曲线）、`largest_errors.csv`（最大误差 12 小时）。

## 4. 基线与候选

### 简单基线

- 方法：持续性基线——用最近一小时功率（测试集的 `lag_1` 列）作为下一小时预测。
- 为什么足够简单：零训练、只用一个最近观测、完全可解释。
- 命令：`python analyze.py`（基线由程序自动计算）。
- 结果：MAE 0.794 kW，RMSE 1.112 kW。

### 候选方法

- 学生完成的核心改动：实现 `make_lagged`（滞后 1/2/3/24 小时 + `hour_of_day` + 下一小时目标）和 `build_candidate`（固定随机种子 `random_state=0` 的 `RandomForestRegressor`）。
- 保持不变的数据、划分、指标或参数：同一真实小时数据；按时间顺序 80/20 划分；MAE/RMSE；`LAGS=(1,2,3,24)`；高需求阈值 = 训练目标 90 分位（3.162 kW）。
- 命令：`python analyze.py`
- 结果：MAE 0.577 kW，RMSE 0.796 kW。

| 项目 | 基线 | 候选 | 含义 |
| --- | ---: | ---: | --- |
| 主指标 MAE（kW） | 0.794 | 0.577 | 平均绝对误差降低约 27% |
| RMSE（kW） | 1.112 | 0.796 | 大误差减少，约降 28% |
| 重要错误：高需求召回率（阈值 3.162 kW） | 0.184 | 0.132 | 候选漏报更多高需求小时 |
| 重要错误：误报 FP / 漏报 FN | 31 / 31 | 9 / 33 | 候选误报更少但漏报更多 |

## 5. 一个真实失败案例

- 样本位置/编号：`largest_errors.csv` 第 1 行（绝对误差最大的真实小时）。
- 真实结果：2007-03-28 17:00 真实 `Global_active_power` 小时平均 = 0.386 kW。
- 系统输出：候选模型预测 3.461 kW（绝对误差 3.076 kW）。
- 可以观察到什么：真实低谷时刻被模型误报为高需求；滞后特征不足以捕捉突然的行为变化。
- 说明的限制：平均指标（MAE/RMSE）掩盖了这类极端时刻；仅靠固定滞后值无法识别突发低谷。
- 不能证明什么：单个样本不能证明模型"普遍不可靠"，也不能推出该家庭某时段行为的原因。
- 下一项最小检查：查看该日前后小时序列上下文；加入滚动统计特征后在相同测试段重跑，观察该样本是否仍被误报。

## 6. 智能体与学生工作边界

- 智能体提出/生成/修改了什么：完成 `make_lagged` 与 `build_candidate` 两个 TODO；把真实数据移动到 `data/raw/`；清理重复测试文件；重写 `README.md`，依据模板填写 `report.md`，生成 `presentation.pptx` 与 `submission.json`。
- 学生怎样核对文件、来源、输出、测试和 diff：（学生填写）建议核对：数据文件来源与大小、`analyze.py` 改动、`python analyze.py` 输出数字、测试结果、`.gitignore` 排除项。
- 学生修改或拒绝了什么建议：（学生填写）
- 每名成员能独立解释的代码或证据：（学生填写）

## 7. 结论与限制

在同一个后段测试集（496 小时）上，固定种子的随机森林候选模型的平均绝对误差低于持续性基线（MAE 从 0.794 kW 降至 0.577 kW，RMSE 从 1.112 降至 0.796 kW）。但候选对高需求时段的召回率更低（0.132 对比 0.184），最大误差案例显示真实低谷会被误报为高需求。数据限制：只有一户、一个季节窗口。方法限制：固定滞后特征无法捕捉突发变化，平均指标掩盖极端时刻。边界：单户结果不能推广到其他家庭，也不能用于电网控制、安全告警或任何真实决策。下一步最小改进：在相同测试段加入滚动统计特征后重跑，比较 MAE、最大误差与高需求召回率。

## 8. 提交复核

- [x] README 从新环境可以开始运行（命令完整，见 `README.md`）
- [x] 数据检查、测试和主程序重新运行（本机通过：`--prepare`、2 测试、`analyze.py`）
- [x] 报告数字与保存输出一致（与 `metrics.json` / `largest_errors.csv` 一致）
- [x] `presentation.pptx` 在 3 分钟内讲完（4 页结构，含讲稿备注）
- [x] `submission.json` 路径正确
- [x] 无密钥、大数据、私人信息、虚拟环境或缓存（`data/raw`、`data/processed`、`__pycache__` 已由 `.gitignore` 排除）
- [ ] GitHub 网页复查并邮件发送 URL（按要求本次未执行任何 Git 配置或提交，留待学生完成）
