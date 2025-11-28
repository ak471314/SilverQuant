# SilverQuant 开发者指南

面向有 Python 基础的开发者，帮助快速理解 SilverQuant 的整体架构、核心模块和扩展方式。阅读顺序建议为：`README.md` → 本文档 → 相关代码。

---

## 1. 框架总览

SilverQuant 是围绕「行情订阅 → 策略扫描 → 委托撮合 → 盘后复盘」构建的组件化框架。可视化架构参考 `_imgs/architecture.png`，核心链路如下：

1. **行情源**：QMT、掘金（GM）或离线历史数据。
2. **订阅器**：`delegate/xt_subscriber.py` 负责 Tick 订阅、盘前/盘后调度、心跳监测和报表。
3. **策略入口**：`run_*.py` 组合股票池、买入器、卖出器与委托代理，定义具体策略。
4. **交易执行**：`delegate/xt_delegate.py` / `delegate/gm_delegate.py` 将策略信号翻译成真实下单。
5. **缓存与工具**：`tools/` 提供本地缓存、远程拉取、通知、风控等通用能力。

---

## 2. 目录速览

| 目录 | 说明 |
| --- | --- |
| `_doc/` | 文档集合（CHANGELOG、CONFIGURATION、本文） |
| `_cache/` | 运行时缓存（资产、成交、持仓、行情历史、调试数据） |
| `delegate/` | 对接行情和交易端的适配层（QMT、掘金、订阅器、回调、盘后报告） |
| `selector/` | 选股模块及 Prompt 配置，支持问财、DeepSeek 等 |
| `trader/` | 买入器、卖出组件、股票池定义 |
| `tools/` | 基础设施：缓存、常量、远端请求、钉钉/飞书推送、券商 API 包装 |
| `reader/` | Tushare 等数据源封装 |
| `public/` | 对外示例和派生脚本（如银翼策略合集） |
| `tests/` | Pytest 用例示例 |
| `run_*.py` | 策略入口脚本，组合上述模块形成具体策略 |

---

## 3. 运行时生命周期

`XtSubscriber` 是默认的运行时管家，负责将策略函数注入 QMT tick 回调，并串起整天流程：

1. **盘前（`before_trade_day`）**  
   - 更新本地持仓缓存、股票池、历史行情等重任务。  
   - 示例：`run_wencai_qmt.py` 中 `before_trade_day()` 会刷新池子并同步持仓。

2. **临近开盘（`near_trade_begin`，可选）**  
   - 确保开盘前的数据全部准备完毕（如除权信息）。

3. **盘中循环**  
   - `callback_sub_whole()` 每秒聚合 tick → 触发 `execute_strategy()`。  
   - 策略根据 `time_ranges` 和 `interval` 控制买卖扫描频率，实现「分钟级调度 + 秒级去抖」(`run_wencai_qmt.py` 中 `execute_strategy()` 逻辑)。

4. **盘后**  
   - 自动取消订阅、清算缓存、生成日报 (`delegate/daily_reporter.py`)。

5. **心跳与容灾**  
   - `callback_monitor()` 每 10 分钟检测数据源，必要时自动重订阅并推送告警。

APSheduler / schedule 双模式运行：`use_ap_scheduler=True` 时默认采用 APScheduler，支持更细粒度控制和开盘自动补偿。

---

## 4. 关键函数流程

下表梳理常见模块的函数级调用路径，方便定位扩展点。

### 4.1 `run_wencai_qmt.py`

1. **入口配置**
   - `PoolConf/BuyConf/SellConf`：静态类存放参数，后续通过 `hasattr` 读取。
2. **盘前逻辑 `before_trade_day()`**
   - `update_position_held()` → 同步 QMT 实际持仓到 `positions.json`。
   - `all_held_inc()` → 全部持仓天数 +1（用于卖出策略）。
   - `my_pool.refresh()` → 根据股票池实现更新候选集合。
   - `my_delegate.check_positions()` → 取实时持仓，传给 `XtSubscriber.update_code_list()` 订阅行情。
3. **买点链路**
   - `execute_strategy()` 在买入时间窗调用 `scan_buy()`。
   - `scan_buy()`：
     1. `pull_stock_codes()` 从问财获取候选（`get_wencai_codes()`）→ 过滤黑名单。
     2. `xt_get_ticks()` 拉取最新 Tick。
     3. `check_stock_codes()` 按价格/涨幅过滤并生成 `{code: {'price','lastClose'}}`。
     4. `BaseBuyer.buy_selections()` → 根据仓位/现金/限额决定下单。
     5. `order_buy()` → `delegate.order_market_open/limit_open()` 真正下单，并通过 `callback.record_order()` 记录。
4. **卖点链路**
   - `execute_strategy()` 在卖出时间窗调用 `scan_sell()`。
   - `scan_sell()`：
     1. `update_max_prices()`：基于实时行情更新建仓后的最高/最低价。
     2. `my_seller.execute_sell()`（在 `ClassicGroupSeller` 中实现）→ 调用 `group_check_sell()`。
     3. `group_check_sell()` 依次执行混入的卖出组件 `check_sell()`，一旦返回 True 即触发 `delegate.order_*`。
5. **委托与回调**
   - 生产模式：`XtDelegate` 下单 → `XtCustomCallback` 接收成交 → 更新 `deal_hist.csv`、`max_price.json` 等。
   - 模拟模式：`GmDelegate` + `GmCallback` 流程相同。

### 4.2 `trader/buyer.py`

```python
buy_selections() -> order_buy() -> delegate.order_*() -> callback.record_order()
```

- `buy_selections()` 核心步骤：
  1. 计算 `available_cash`（来自 `delegate.check_asset()`）。
  2. 推导可用仓位数 `available_slot`、剩余仓位 `slot_count`、单次限额 `once_buy_limit`。
  3. 遍历候选股票，跳过已买、持仓、量不足等情况。
  4. 调用 `order_buy()` 下单并在 `today_buy[curr_date]` 中记录，防止重复。
- `order_buy()`：
  1. 依据 `order_premium` 生成委托价，并校验涨停价 `get_limit_up_price()`。
  2. 决定使用市价（优先）还是限价。
  3. 触发 `delegate.order_market_open/limit_open()`，并记录日志 & 回调。

### 4.3 `trader/seller_groups.py` & `trader/seller_components.py`

```python
execute_sell() -> group_check_sell() -> component.check_sell() -> delegate.order_*()
```

- `GroupSellers.group_check_sell()`：
  1. 迭代多继承父类（如 `HardSeller`, `FallSeller`, `ReturnSeller`）。
  2. 按顺序传入 `quote/curr_time/position/held_day/max_price/history/ticks`。
  3. 任一父类返回 True 即打断循环。
- 典型组件逻辑（以 `HardSeller.check_sell()` 为例）：
  1. 对比当前价格与建仓价乘以 `risk_limit/earn_limit`。
  2. 触发 `delegate.order_market_close()` 并调用 `cache.del_held_day()` 重置持仓天数。
  3. 通过 `callback.record_order()` 记录卖出行为。

### 4.4 `delegate/xt_subscriber.py`

主要函数关系：

| 函数 | 功能 |
| --- | --- |
| `start_scheduler()` | 根据 `use_ap_scheduler` 选择 APScheduler/schedule，注册 cron 任务 |
| `subscribe_tick()` | 调用 `xtdata.subscribe_whole_quote()` 绑定 `callback_sub_whole` |
| `callback_sub_whole()` | 每秒聚合 quotes → 更新 `cache_quotes` → 调用 `execute_strategy()` |
| `callback_monitor()` | 检测回调超时 → 自动重订阅 & 发出 Ding 消息 |
| `download_cache_history()` | 盘前批量加载历史行情（AKShare / TDX / Tushare） |
| `record_tick_to_memory()` | 将 tick 逐条写入内存 DataFrame/列表，盘后保存 |
| `daily_summary()` | 盘后生成成交/持仓/资产报告 |

`callback_sub_whole()` 的细节：
1. 记录当前时间并格式化为 `curr_date/curr_time/curr_seconds`。
2. 维护每分钟心跳（打印 `[\*]`）和每秒执行锁。
3. `self.cache_quotes.update(quotes)` → `execute_strategy()` → 若返回 True 则清空缓存（避免重复扫描）。
4. 根据 `open_tick` 配置选择执行前/后记录 tick。

### 4.5 `tools/utils_cache.py`

核心辅助函数：

- `load_json/save_json`：所有缓存文件的基础 IO。
- `update_position_held()`：扫描 `delegate.check_positions()` 并同步到 `positions.json`。
- `all_held_inc()`：每日只执行一次的持仓天数自增，使用 `_cache[InfoItem.IncDate]` 防重。
- `update_max_prices()`：遍历 positions → 读取 `quotes` 中 `high/low` → 更新 `max_price.json/min_price.json`。
- `record_deal()` / `record_assets()`：线程安全地写入交易和资产 CSV。
- `download_cache_history()` 调用的 `get_daily_history()` / `get_tdxzip_history()` 位于 `tools/utils_remote.py` 与 `tools/utils_mootdx.py`，负责实际抓取。

---

## 4. 核心策略组件

### 4.1 Credentials 与缓存

- `credentials_sample.py` 提供默认配置，复制为 `credentials.py` 后填写账号、密钥、通知 token。  
- 缓存根目录由 `CACHE_BASE_PATH` 控制，常见文件：  
  - `assets.csv`：资金曲线  
  - `deal_hist.csv`：委托/成交  
  - `positions.json`：持仓天数  
  - `max_price.json / min_price.json`：建仓后的极值轨迹  
- `tools/utils_cache.py` 内提供全部缓存读写、持仓日计数、历史价更新、交易日判断等辅助函数。

### 4.2 行情与委托代理（Delegates）

- `delegate/xt_delegate.py`：QMT 实盘接口，包装 xtquant API，提供市价/限价下单、撤单、资产/持仓查询。
- `delegate/gm_delegate.py`：掘金模拟盘接口 (`gmtrade`)，封装订单、持仓、资产结构，并融合钉钉推送。
- `delegate/gm_callback.py`、`delegate/xt_callback.py`：和券商 API 绑定的回调适配层，负责回写成交、持仓等信息到缓存。

### 4.3 订阅器（Subscriber）

`delegate/xt_subscriber.py` 核心职责：

- Tick 订阅管理：`subscribe_tick` / `unsubscribe_tick` / `resubscribe_tick`
- Tick 缓存与供策略读取 (`cache_quotes`)
- 历史行情缓存：可从 AKShare、TDX ZIP、Tushare 等来源批量下载 (`download_cache_history`)
- 盘前/盘中/盘后定时调度（APSheduler 或 schedule）
- 报表与健康检查（钉钉告警）

### 4.4 股票池（Pools）

位于 `trader/pools.py`（未在此文展开）。股票池负责维持白名单、黑名单、指数过滤、问财黑词等。`run_wencai_qmt.py` 中的 `PoolConf` 示范了如何配置问财黑名单。

### 4.5 买入器（Buyer）

`trader/buyer.py` 定义了通用买入流程：

- 依据配置 (`slot_count`、`slot_capacity`、`daily_buy_max`、`inc_limit` 等) 控制仓位和风险。
- `buy_selections()` 接收待买列表，自动排除重复/持仓/涨停等不满足条件的标的。
- `order_buy()` 将信号转换为市价或限价委托，并在回调中记录下单 (`delegate.callback.record_order`)。
- `LimitedBuyer` 示例展示如何按比例缩放下单量。

### 4.6 卖出组件 & 组合（Seller Groups）

`trader/seller_components.py` 提供原子卖出策略（硬止损、回落止盈、均线跌破等）。  
`trader/seller_groups.py` 使用多继承组合这些组件，实现可复用的卖出策略簇（例如 `ClassicGroupSeller`、`WencaiGroupSeller`、`LTT2GroupSeller`），对策略而言只需实例化合适的组即可。  
组合通过 `group_check_sell()` 串联父类的 `check_sell`，一旦有组件给出卖出信号即停止后续组件执行。

### 4.7 选股器（Selector）

- `selector/select_wencai.py`：基于问财的选股器，支持多 Prompt、价格字段自动识别及限速提示。  
- Prompt 由 `selector/select_prompts.py` 提供，`get_prompt()` 支持 list / dict 两种结构。  
- 其他选股脚本（DeepSeek、银翼策略）位于同目录，可自由扩展。

### 4.8 工具层（Tools）

- `tools/utils_basic.py`：日志、涨跌停价、代码转换等常规工具。  
- `tools/utils_remote.py`：封装远程推票 / 历史行情 HTTP 拉取、Tick 转换。  
- `tools/utils_ding.py` / `tools/utils_feishu.py`：统一的机器人推送接口，策略可选是否开启。  
- `tools/utils_cache.py`：详见 4.1，是缓存与数据准备的中心。

---

## 5. 策略入口脚本

| 脚本 | 用途 |
| --- | --- |
| `run_wencai_qmt.py` | 问财 + QMT 模式，可在 `IS_PROD=False` 时连掘金模拟盘测试 |
| `run_wencai_tdx.py` | 问财 + 通达信本地数据，脱离 QMT 运行 |
| `run_shield.py` | 半自动防守（人工买入 + 程序化卖出） |
| `run_swords.py` / `run_swords_tdx.py` | 打板策略模板，分别读取线上池或通达信自选 |
| `run_remote.py` | 「推票服务」样板，可在 Linux/分布式环境执行远程买入 |
| `run_ai_gen.py` | 公式选股示例，偏教学用途 |
| 其他 `run_*` | 针对不同数据源/券商的样板，可参照 `README` 的入口说明章节 |

所有脚本遵循同一结构：定义 `PoolConf/BuyConf/SellConf` → 初始化 `Pool/Buyer/Seller` → 创建 `Delegate` & `Callback` → 启动 `XtSubscriber`。

---

## 6. 数据与缓存

### 6.1 盘前下载

- `XtSubscriber.download_cache_history()` 支持 AKShare、TDX ZIP、Tushare、MooTDX。  
- 优先尝试读取 `_cache/*.pkl`，不存在时批量下载并写入；TDX ZIP 模式需提前准备除权数据。

### 6.2 盘中持仓追踪

- `update_position_held()` 自动发现 QMT 中的真实持仓并同步到 `positions.json`。  
- `all_held_inc()` 在每日盘前执行，所有持仓天数 +1，用于卖出策略中「持仓天数」相关条件。  
- `update_max_prices()` 根据实时 tick 更新历史高/低价，供回撤类卖出组件使用。

### 6.3 资产 / 成交 / 报表

- 所有委托、成交和资产变化都会写入 CSV，并可通过 `delegate/daily_reporter.py` 在盘后自动生成日报。  
- `tools/utils_cache.record_deal()`、`record_assets()` 等函数负责落地数据，确保断电可恢复。

---

## 7. 扩展指南

### 7.1 自定义策略

1. 复制现有 `run_*.py`，根据需求替换 `Pool/Buyer/Seller/Delegate`。
2. 在 `PoolConf/BuyConf/SellConf` 中添加策略参数，买卖逻辑可通过继承 Buyer/Seller 进一步扩展。
3. 若需要额外数据，可在 `before_trade_day` 中调用 `XtSubscriber.download_cache_history()` 或自定义读取。
4. 通过 `DingMessager` / `FeishuMessager` 及时推送策略状态，避免静默出错。

### 7.2 新的卖出组件

1. 在 `trader/seller_components.py` 中实现 `check_sell()`，遵循返回 `True` 即停止后续执行的约定。
2. 将组件混入新的 GroupSeller（`trader/seller_groups.py`），策略入口只需替换组名即可复用。

### 7.3 新数据源 / 远程模式

- `tools/utils_remote.py` 提供 `DataSource` 枚举，可拓展 `get_daily_history()` 适配新的 API。  
- Linux / 分布式场景可用 `run_remote.py`，通过 HTTP 下发信号，本地仅负责执行买卖。

### 7.4 测试与调试

- 单元测试示例在 `tests/`，运行 `pytest` 即可。  
- 运行前可设置 `IS_DEBUG=True`，配合 `tools.utils_basic.debug()` 输出策略筛选细节。  
- Tick 历史可通过 `XtSubscriber` 打开 `open_tick_memory_cache` 并在盘后 `save_tick_history()` 保存到 `_cache/debug/`。

---

## 8. 常见排错 Checklist

- **无 Tick 输出**：确认 QMT 行情源状态、`xtdata` 是否连接；观察 `callback_monitor()` 是否推送中断告警。  
- **持仓卖不掉**：检查是否按「每日开市前启动」要求运行，否则 `positions.json` 无法正确累计持仓日。  
- **问财无返回**：确保 Prompt 能在网页端搜索到，且 `selector/select_wencai.py` 的 `interval` 不小于 30 秒避免封 IP。  
- **历史数据缺失**：检查 `_cache/_daily*` 是否命中缓存，如未命中需确认 AKShare/TDX 是否可用或适当延长盘前时间窗口。

---

通过以上信息，开发者可以快速定位到需要修改的模块，并在保持框架约定（缓存、订阅、通知、风控）的前提下扩展自定义策略。更多细节可结合源代码与 `README.md`、`CONFIGURATION.md` 交叉参考。


