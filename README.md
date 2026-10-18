📖 项目简介
本项目基于 Kaggle 平台完成，构建了一个由大语言模型驱动的奶茶店经营模拟智能体。
系统将「定价、采购、投放广告」等经营动作封装为 6 个可调用工具，交由 GLM-4-flash 通过 Function Calling 自主决策；
每日循环执行「查询状态 → 查询环境 → 查询需求预测 → 定价 + 采购（可选广告）→ 日终结算」的完整流程，
连续模拟 30 天经营，并输出逐日经营记录 CSV。
notebook 中包含模拟器实现、工具定义、Agent 主循环、兜底策略与完整运行结果。

📂 文件说明
ai-interviewer.ipynb：Jupyter Notebook，项目全部代码与 30 天模拟输出（文件名沿用仓库现有 notebook，建议重命名为 milk-tea-shop-agent.ipynb 等更贴切的名称）

shop_result.csv：运行后自动生成的逐日经营记录（day / money / stock / price / demand / sell / reputation / shortage / event）

📌 Notebook 在线查看链接
👉 milk-tea/ai-bubble-tea-shop-simulation .ipynb at main · zsy-yt/milk-tea

⚙️ 环境依赖
python
Python 3.13
openai          # 通过 OpenAI 兼容接口调用智谱 GLM
pandas          # 经营记录 DataFrame 与 CSV 导出
kaggle_secrets  # Kaggle 环境读取 API Key；本地运行可改为环境变量
需要智谱开放平台 API Key（ZHIPU_API_KEY），模型为 glm-4-flash。

✨ 项目亮点
完整的多轮 Function Calling Agent 循环：每天最多 8 轮工具调用，模型先查询状态再决策，逐步获取工具返回结果后继续推理，模拟真实经营者的「观察—判断—行动」闭环。

6 个经营工具覆盖核心决策：get_state / get_environment / get_forecast（查询类）、set_price / buy_ingredients / run_ad（行动类），工具以 JSON Schema 声明参数，可被模型自主组合调用。

环境驱动的需求模型：calc_demand 将天气（晴 / 阴 / 小雨 / 中雨）、高温（≥30℃）、低温（≤15℃）、活动客流系数、广告投放、口碑、价格弹性、随机因子统一纳入需求计算，形成可复现的经营环境。

兜底策略保障流程鲁棒性：若模型在 8 轮内未同时完成定价与采购，自动启用默认策略（定价 15 元、按预测需求 +5 采购），避免单日决策缺失导致模拟中断。

完整经营结果可导出：逐日记录（资金、库存、定价、需求、销量、口碑、是否缺货、当日活动）导出为 UTF-8-SIG 编码的 CSV，便于后续分析。

🧠 工作流程
text
初始化(资金10万/库存0/口碑50)
  └─ 每日循环(最多30天)
       ├─ LLM 多轮调用工具：get_state → get_environment → get_forecast
       ├─ LLM 决策：set_price + buy_ingredients（可选 run_ad）
       ├─ 兜底校验：未定价或未采购则启用默认策略
       ├─ daily_step：按需求模型结算销量/收入/库存/口碑
       └─ 记录当日结果 → 破产或满30天终止
  └─ export_csv 导出 shop_result.csv
📊 环境数据说明
本项目不依赖外部数据集，经营环境由 notebook 内嵌的 30 天数据（2024-11-01 至 11-30）构成，每日包含：

字段	含义
date	日期与星期
weather	天气（晴 / 多云 / 阴 / 小雨 / 中雨）
temp_high / temp_low	最高 / 最低气温
wind	风向风力
event	当日活动（潜水展、金鸡百花电影节、咖啡展 + ISPO 展等）
passenger_coeff	客流系数（无活动 1.0，重大活动最高 1.7）
random_factor	随机扰动因子（0.88–1.11）
经营参数：初始资金 100000 元、初始库存 0、初始口碑 50、原料成本 4 元/份、日需求基数为 50。

🖥️ 30 天模拟结果
指标	初始	第 30 天	变化
资金	100000	105937	+5937（+5.9%）
库存	0	487	积压 487 份（约 1948 元成本占用）
口碑	50	77	+27
缺货天数	—	1 天（第 1 天）	—
定价区间	—	6–12 元	多数为 8 元
广告投放	—	0 元	模型全程未调用 run_ad
模型决策中的有效行为：

在潜水展（第 7–9 天）、金鸡电影节（第 11–15 天）、咖啡展（第 21–23 天）等活动日主动提高采购量（最高 150 份），销量对应提升至 96–122 份；

活动结束后能下调采购量，但下调幅度不足，导致库存持续累积。

⚠️ 已知限制与改进方向
库存积压严重：30 天结束时库存达 487 份，占用约 1948 元成本资金，说明模型的采购决策偏保守于「足量供应」，缺乏对库存周转与资金效率的权衡。

定价偏低、毛利率受限：模型定价集中在 8 元（成本 4 元），毛利率约 50%，30 天仅盈利 5937 元；若结合需求弹性适度提价，收益空间更大。

广告工具从未被使用：run_ad 全程调用次数为 0，模型未探索「投放广告提升需求」的策略，可在 System Prompt 中增加「活动日前后可评估广告 ROI」的引导。

首日缺货导致口碑扣分：第 1 天预测需求 80、实际采购 70，缺货 10 份，口碑从 50 降至 48，后续才逐步回升。

需求模型与真实决策的耦合：当前 calc_demand 与 get_forecast 使用同一套公式，预测值即真实值，缺少预测误差的模拟，后续可加入独立的随机扰动以测试模型在不确定环境下的决策能力。

📝 使用说明
下载仓库代码

安装上述依赖，配置 ZHIPU_API_KEY（Kaggle 环境使用 Secrets；本地运行改为 os.getenv 读取环境变量）

打开 notebook，逐块运行：读取密钥 → 加载环境数据 → 定义 ShopSimulator → 定义工具 → 定义 agent_loop → 启动模拟

运行结束后自动打印逐日经营日志，并将结果保存至 /kaggle/working/shop_result.csv

