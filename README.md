> Agent：先深入读取与当前任务相关的文件，在读取过程中若出现与该文件紧密相关的其他文件，则加入读取过程，以此反复交互前进，直至了解、通透任务至项目。
---
# 量化项目`D:\Aquant project\MF`
> 基于 veighna_studio 4.3.0 



## 1. 项目概要:

- 特征 = 因子字段 + 因子中性化字段 + 市场风格字段
  - 因子字段 = 基本面因子 + 价量因子 等股票风格因子 多维度、多跨度、多节点
  - 因子中性化字段 = 行业 + 市值
  - 市场风格字段 = 牛熊 + 行业轮动
- 因子字段不能是原始数据字段，需要包含原始数据间的关系
- 模型学习的是高阶关系，需要挖掘特征间的非线性交互、条件依赖和时序演化模式，捕捉市场信号
  - XGBoost/LightGBM
  - Transformer
  - CNN
  - GNN
  - 全连接网络
  - RL DQN
- 策略采用周频或日频调仓，股票池沪深300
- vnpy框架回测:`C:\veighna_studio\Lib\site-packages\vnpy`;  未进展到拟真与实盘...
- 数据接口为vnpy_rqdata:`C:\veighna_studio\Lib\site-packages\vnpy_rqdata`

---

## 2. 项目进展:

### 2.1 主代码库: `D:\Aquant project\MF\MF_code`

- `D:\Aquant project\MF\MF_code\download_300_template.ipynb`
  - 沪深300指数成分股分线、日线数据下载模板
  - 更新：
    - `extended_days`必须;
    
---

- `D:\Aquant project\MF\MF_code\download_industry.ipynb`
  - 下载行业数据
---

- `D:\Aquant project\MF\MF_code\download_all.ipynb`
  - 用米筐全A市场加权指数`"866011.RI"`下载A股全市场股票
---

- `D:\Aquant project\MF\MF_code\factor_define.py`
  - 因子定义: 接受数据表 计算返回因子表 接受的数据表是何种数据,涵盖哪些股票, 哪些交易日在函数外确定, 函数只负责接收表格
  - 因子`return`格式统一:`pl.DataFrame` with `columns`: [`"vt_symbol"`, `"date"`, `"data"`];
  - `FACTOR_REGISTRY`存储因子对应的函数, `PARAMS_REGISTRY`存储因子函数参数;
  - 将文件移至`C:\veighna_studio\Lib\site-packages\vnpy\alpha\factor_define.py`, 之后导入的是该路径下的模块, 原路径的`factor_define`只是用来更新, 每次更新后记住要复制过去;
  - 将文件移至`C:\veighna_studio\Lib\site-packages\vnpy\factor_define.py`, 放在alpha里会有循环导入的bug, 之后导入的是该路径下的模块, 原路径的`factor_define`只是用来更新, 每次更新后记住要复制过去;
  - 更新：
    - 不需要`min_periods`, 将各个因子定义内的`window_size`视为形参处理, 并且增加形参 `max_window`, `max_window`在填入实参时填的是所有的`window_size`之间的最大值，这样保证在因子计算函数里加载除指定时间段的k线外还加载`extended_days` = `max_window`的k线，k线表有足够数据可以滚动, 最后返回的是将多余的数据截断的`dataframe`, 所以还需要形参`start`, 只保留`start`后的数据;
    - 增加参数注册字典: `PARAMS_REGISTRY1`, `PARAMS_REGISTRY2`, `PARAMS_REGISTRY3`, 分别对应因子注册表, 存入函数参数, 例如: `func(df, start, window1, window2)` 则存入: `{name: {"window1" = 21, "window2" = 5, "max_window" = 21}}`;
    - 在计算因子时, 先判断在哪个因子注册表里确定加载什么类型的k线, 再去对应的参数注册表中获得`"max_window"`, 传入`extended_days`(实际的`extended_days`要比`window`大,因为表内是交易日，回溯前21个交易日需要回溯更多天), 将除了`max_window`键的字典放入`params`, 最后调用`func(df, start, **params)`;
    - 注意, 对于分线因子的滚动要区分是在交易日滚动还是以分钟滚动，只有按交易日滚动才算做参数,才需要`extended_days`;
    - 对于不需要滚动的定义也不需要截断，故没有`start`参数;
    - 对于`PARAMS_REGISTRY3`: `"max_window_daily"`, `"max_window_minute"` 替代 `"max_window"`, 分别是日k表最大窗口和分k表最大窗口;
    - 除了rolling需要的`window`外, 在计算过程中还出现了 `shift` , 需要将`shift`的天数也看作参数, 因为其仍然需要之前交易日的数据, 所以参数字典中应该加入`"shift"`参数, 并且`"max_window"`要在所有`window`和`shift`之间比较了, `shift`是前移, `window`是窗口, `max_window`是为了保证数据非空所需的额外交易日天数;
    - 修正了`max_window`逻辑上的bug, 因子缺失率正常了(不超过0.3%): `max_window` 参数只考虑了单层 `rolling` 窗口，没有覆盖函数内部链式 `rolling`/`shift` 的累积效应,当多个 `rolling_mean` / `rolling_max` / `shift` 串联时,后面的操作需要前面操作的输出,实际所需历史天数是各层 (`window` - 1) 的叠加,按月分区计算时，`extended_days` = `max_window` + 5 不够长，导致月初大量 `null`;
    - 因子name和func不必要是1对1关系: 多个name可以对应1个func, 根据`params`的不同, 例如
      `'streverse_1m': calc_streverse,'streverse_1m': {"shift": 21, "max_window": 21}`,
      `'streverse_2m': calc_streverse, 'streverse_2m': {"shift": 42, "max_window": 42}`,
      这样可以根据参数的不同计算不同跨度的因子, 以不同的名字保存;
    - 重构因子注册表和参数表: 
      以`PARAMS_REGISTRY2 = { 'rstr': {"window_long": 252, "window_short": 21, "shift": 1, "max_window": 252} }`为例, 无论是`window`还是`shift`参数,都作用于加载的日线表, 所以可以写成
      `PARAMS_REGISTRY2 = {'rstr': {"df_daily": {"window_long": 252, "window_short": 21, "shift": 1, "max_window": 252}}}`,因为接受的数据表是何种数据,涵盖哪些股票, 哪些交易日在函数外确定, 函数只负责接收表格,
      所以`"df_daily"`对应的是因子函数`"cal_rstr"`中作用日线表的参数(需要把函数定义的形参名改成`df_daily`),这样结构的好处是不用再有REGISTRY1,2,3了,只需要一个注册表和参数表,并且不单单是`"df_daily"`,`"df_minute"`,
      可以是任意接受的表格,比如`"df_mcap"`,并且滚动的`"max_window"`也做了很好的区分,不同的表格回溯不同的天数.另外,对于不作用于表的参数,虽然现在还没出现这钟因子,只需要修改成: 
      `PARAMS_REGISTRY = { factor_name:{"constant_params":{},"df_params":{"df_daily":{} } } }` 也就是将其与作用于表的区别成constant和df就可以了. 
      为了方便填写参数,在因子定义时,各个表和其对应的参数放一起,例如: `func(df_daily,window,df_minute,window_m.... constant_params, start = None)`, `start`放在最后.据此, 需要修改`factor_define`和`lab.cal_factor_daliy`;
    - 在因子注册表里添加因子类型`FactorType`;
---

- `D:\Aquant project\MF\MF_code\calculate_factor.ipynb`
  - 因子计算，切分时间段分批计算, 因为计算因子时加载300只股票, 6年的分线内存占用直接爆满. 日频的不会, 日频只增加了0.2G, 那如果5000只, 10年, 日频要增加5.5G;
  - 只计算该月的沪深300成分股因子;
  - 更新：
    - `compute_factors_for_month()`: 给`load_bar_df`加上`extended_days`,不然滚动长时间窗口一大堆`null`;
    - 适配了因子定义模块里的滚动等需求，现在可以判断加载什么k线，是否需要extend，以及自由传入所需参数;
    - 更新`cal_factors_for_month()`至`cal_factor()`: 因子计算更灵活, 接受`start`,`end`而不是`int`类型了. 不再接受`index`作为参数, 而是`symbols`: `list[str]`, 因子存储更兼容, 不只是存储某一个股票池;
    - 将按月分区保存的逻辑放到计算函数外, 并且对于新计算的且已在`parquet`中的因子值可以选择是更新还是不更新, 对于新的日期新的股票 因子值不再覆盖原数据而是添加;
    - 将因子计算函数调到`lab`里;
    - 将按月分区的逻辑打包成函数放到`lab`里;
    - 2026-05-22: `logger`替代`print`, 使用`FactorRequest->cal_factor_daliy->save_factor`, 按月分批计算;
---

- `D:\Aquant project\MF\MF_code\data_prepare.ipynb`
  - 数据准备，将数据整合到`dataset`里后存储
  - 更新：
    - 将加载因子的函数和计算缺失率的函数移到`lab.py`里;
    - 2026-05-22: 使用新架构, `FactorRequest`加载因子, `add_lag_features`生成滞后, `prepare_data`后释放中间状态,` Parquet+JSON`保存;
---

- `D:\Aquant project\MF\MF_code\model_train - rank.ipynb`
  - `Lightgbm`模型训练，`objective:rank`，lambda rank预测排名
  - 使用`C:\veighna_studio\Lib\site-packages\lightgbm`
  - 2026-05-26: 奇怪的现象：以验证集NDCG为评估指标，一个最佳迭代(第8轮)和一个特意选择的指标没那么好的迭代(第3轮)（无论训练集、验证集还是测试集，3的NDCG都显著低于8，3样本内回测结果明显低于8，却在验证集和测试集上的回测明显优于8;
---

- `D:\Aquant project\MF\MF_code\model_train.ipynb`
  - `Lightgbm`模型训练，`objective:regression`
  - 不用看这个代码，现在不用它训练，用model_train - rank
---

- `D:\Aquant project\MF\MF_code\backtest.ipynb`
  - 回测；
  - 策略采用`C:\veighna_studio\Lib\site-packages\vnpy\alpha\strategy\strategies\equity_demo_strategy3.py`
---

- `D:\Aquant project\MF\MF_code\backtest_WFO.ipynb`
  - 2026-05-27: 新增滚动回测, 打包`model_train-rank.ipynb`和`backtest.ipynb`
  
  -
    ```
    WFO 旨在减少过拟合风险，通过模拟策略（模型策略、选股策略和回测策略）在历史数据上的动态适应过程来评估策略超参数效果。
    在当前框架下，策略包含：
    - 模型策略：使用什么模型，决定什么超参数 当前是lightgbm D:\Aquant project\MF\MF_code\model_train - rank.ipynb
    - 选股、挂单策略：C:\veighna_studio\Lib\site-packages\vnpy\alpha\strategy\strategies\equity_demo_strategy2.py，得到信号后如何买入和卖出，买多少，卖多少
    - 回测策略：C:\veighna_studio\Lib\site-packages\vnpy\alpha\strategy\backtesting2.py 如何更贴合实际交易，如何撮合交易等等
    参数包含：
    - 数据集切分参数：lambda-rank的n_quantiles
    - 模型超参数：如lightgbm的'num_leaves': 1024,'max_depth': -1,'min_data_in_leaf': 300等
    - 模型内参数
    - 选股策略超参数：如top_k，min_days
    - 回测策略参数：capital等
    
    固定下所有参数后，模型在训练集上训练，验证集上评估，选取最优模型内参数，在测试集上计算出信号，发送给选股策略，给出买单和卖单，回测策略撮合交易，执行回测，计算出指标
    这一轮流程下来可以知道该模型超参数下的model在测试时间段的选股效果如何，但是金融市场信噪比太低，过去不代表未来，至于模型实盘怎么样就不敢保证了，
    所以如何有信心让模型应用到实盘呢？ 必须用到滚动回测，只有将固定下超参数的模型放在多个时间段内检验才可以知道模型是否稳健。
    以18-26年为例，以3年训练，1年验证，1年测试，1年滚动，那么会有如下5个窗口：
            训练集         |        验证集          |        测试集
    2018-1-1 ---- 2021-1-1  2021-1-1 ---- 2022-1-1  2022-1-1 ---- 2023-1-1
    2019-1-1 ---- 2022-1-1  2022-1-1 ---- 2023-1-1  2023-1-1 ---- 2024-1-1
    2020-1-1 ---- 2023-1-1  2023-1-1 ---- 2024-1-1  2024-1-1 ---- 2025-1-1
    2021-1-1 ---- 2024-1-1  2024-1-1 ---- 2025-1-1  2025-1-1 ---- 2026-1-1
    2022-1-1 ---- 2025-1-1  2025-1-1 ---- 2026-1-1  2026-1-1 ---- 2026-5-8
    固定好模型的超参数后，让模型在每个窗口上训练、验证，得到最优参数下的模型并计算信号，将5个窗口的测试集拼接起来，让策略在这个连贯的测试数据上运行一次，
    但根据数据所在的时间组动态切换参数，也就是将信号拼接起来，这次完整回测得到的绩效指标（如收益率、夏普比率、最大回撤等），即作为该超参数下策略效果的最终评价依据。
    在滚动回测的基础上进行optuna超参数调优，调的是模型超参数，其余参数或超参数固定。
    ```

---
- `D:\Aquant project\MF\MF_code\Optuna_LGBMLR.ipynb`
  - 2026-05-28新增对LightGBM LambdaRank用`optuna`优化

---






### 2.2 引用代码库: `C:\veighna_studio\Lib\site-packages\vnpy`  `C:\veighna_studio\Lib\site-packages\vnpy_rqdata`

- `C:\veighna_studio\Lib\site-packages\vnpy\alpha\strategy\backtesting2.py`
  - 新增回测引擎;
  - `backtesting.py` 是模拟交易所和我向交易所执行的动作, 与选股策略模块耦合，最重要的是new_bars(), 这是每天用户执行交易的动作, 内部的的on_bars()执行选股及各种交易操作, 而交易操作又是引擎里的;
  - 关键函数调用:`run_backtesting(self)` -> `self.new_bars(dt)` -> `self.cross_order() -> process_order(self, order: OrderData)` -
    -> `self.strategy.on_bars(bars)` -
    -> `class EquityDemoStrategy2(AlphaStrategy): self.execute_trading(bars, price_add=self.price_add)` -
    -> `self.cancel_all() self.buy(vt_symbol, order_price, buy_volume)` -
    -> `self.send_order(vt_symbol, Direction.LONG, Offset.OPEN, price, volume)` ;
    `self.cross_order()`-> `process_order(self, order:OrderData)`: 先执行卖单， 再根据信号大小依次执行买单， 买入分配资金最大可买入volume;
  - 更新：
    - `self.slippage`: 新增滑点;
    - `self.long_trade_count`: 新增买入次数统计;
    - `self.short_trade_count`: 新增卖出次数统计;
    - `calculate_max_volume`: 新增函数，计算订单最大可交易volume;
    - `self.adjust_type`: 新增adjust_type;
---

- `C:\veighna_studio\Lib\site-packages\vnpy\alpha\strategy\backtesting3.py`
  - 基于`backtesting2.py`重构的新回测引擎, 新增同日撮合与信号逐日获取;
  - 关键调用:`new_bars(dt)` -> `strategy.on_bars(bars)` -> `cross_order()`(同日撮合) -> `cancel_all()`(清理未成交) -> `sync_targets()`(镜像target) -> `update_daily_close()`;
    `cross_order()`-> `process_order()`: 先执行卖单, 再按`buy_symbols`信号顺序逐个执行买单, 资金不足自动缩量;
    `get_previous_signal()`: 获取`< self.datetime`的最新一期信号, 天然跨周末/节假日;
  - 更新：
    - `new_bars`: 去掉前置`cross_order()`(不再撮合昨日遗留订单), `on_bars`后追加`cross_order`+`cancel_all`+`sync_targets`;
    - `send_order`: 修复`size`未定义bug, 恢复`LONG`委托资金检查;
    - `process_order`: 成交价直接使用`open*(1±slippage)`, 不再`min/max(order_price, slippage_price)`;
    - `get_previous_signal`: 新增函数, 从`signal_df`中取`<self.datetime`的最新日期信号;
    - `holding_value`: `PortfolioDailyResult`新增日末持仓市值, `calculate_statistics`输出日均资金利用率;
    - `calmar_ratio`: 新增Calmar Ratio = 年化收益 / |最大回撤|;
    - `sortino_ratio`: 新增Sortino Ratio, 以下行标准差替代总标准差;
    - `print_trading_log`: 新增函数, 按日分组打印委托/成交明细, 卖出显示成本基准与涨跌百分比;
---

- `C:\veighna_studio\Lib\site-packages\vnpy\alpha\strategy\strategies\equity_demo_strategy2.py`
  - 新增选股策略 strategy;
  - 此策略是通过信号计算出目标持仓， 该模块与回测引擎模块耦合;
  - 收盘计算信号，明日开盘执行交易;
  - 更新：
    - `self._buy_symbols`: 新增买单顺序, 供`cross_order`使用;
  - 2026-05-25: 当前的策略用无复权k线做回测会导致实际持仓数超过top_k, 因为除权时的跳空引擎会判断成跌停，不会卖出，这导致卖的股票数少于计划，
    而买入资金是平均分配，剩余资金少了，搭配买入最多能买入量机制的存在，买的股票数不变，于是持仓数增加，如果不控制`buy_quantity`, 就会慢慢计算成负数;
---

- `C:\veighna_studio\Lib\site-packages\vnpy\alpha\strategy\strategies\equity_demo_strategy3.py`
  - 基于`equity_demo_strategy2.py`重构的选股策略, 信号加权TopK分配, open_price定价, holding_days锁仓;
  - 选股逻辑: `get_previous_signal()`获取昨日收盘信号 → 排序取TopK → 锁定持仓(held且∉TopK且days<min_days不卖) → `available_slots = top_k - len(locked)`保证≤top_k → 仅正信号参与资金分配;
  - 不在TopK的持仓`set_target=0`(全部卖出), 停牌股(`open_price==0 or volume==0`)跳过;
  - 交易执行: `execute_trading_open()`三阶段 — Phase1全卖→Phase2撮合卖→Phase3按信号顺序逐个买(资金不够则`calc_affordable_buy`重算再下);
  - 成本追踪: `cost_basis`买入加权平均, 卖出`sell_cost`快照留存;
  - 更新：
    - `on_bars`: 完全重写, 新增`holding_days`锁仓机制(持仓<min_days不卖), `available_slots`保证持仓数≤top_k, 仅正信号参与资金分配;
    - `on_init`: 初始化`holding_days`/`cost_basis`/`sell_cost`;
    - `on_trade`: 卖出快照`cost_basis`+清仓归零, 买入更新加权平均成本;
---

- `C:\veighna_studio\Lib\site-packages\vnpy\alpha\lab.py`
  - 数据存储、加载、过滤等;  一个完整的数据获取,保存,加载,使用 流程: `rqdata_datafeed`下载数据(或许在`object`里定义新的数据类型和数据`req`),` lab`接收数据然后保存, 并且有加载的接口,在calcu时调用加载函数加载数据;
  - 更新：
    - `__init__`: 初始化新增`post`,`pre`目录, 交易日列表`trading_days.json`; 
    - `save_bar_data`: 需表明`adjust_type`;
    - `load_bar_data`: 需表明`adjust_type`;
    - `load_bar_df`: 需表明`adjust_type`, 不对价格做正则化, 如果是`WEEKLY`或`MONTHLY`, 不选取'`vwap`'列;
    - `load_component_data2`: 新增函数，`rqdata`不会更新最近的成分股列表导致`load_component_data`返回的字典不含最近的`datetime`，这使得最近几天没有信号，该新增函数对字典进行前向填充，对没有成分股列表的日期补充最新的列表;
    - `load_component_filters2`: 新增函数，调用`load_component_data2`对指数成分进行过滤，确保每天都有股票池产生信号;
    - `load_trading_days`: 新增函数，从`self.trading_days_path`中加载交易日数据，`list[date]`;
    - `save_industry_data`: 新增函数, 接收`list[IndustryData]`，保存行业数据;
    - `cal_factor_daily`: 新增函数, 日频计算因子, 加载lab里的k线去计算`factor_define`里的因子,返回`pl.dataframe["datetime", "vt_symbol", "data"]`;
    - `save_factor_by_month`: 新增函数, 按月分区保存因子, 对于新计算的且已在`parquet`中的因子值可以选择是更新还是不更新, 对于新的日期新的股票 因子值不再覆盖原数据而是添加;
    - 导入`factor_define`模块;
    - `load_factor`: 新增函数, 从按月分区的因子库中加载数据;
    - `missing_ratio`: 新增函数, 计算因子缺失率 `null`+`nan`;
    - `load_factor_from_month`: `load_factor`更名;
    - 更改目录结构, 见数据库;
    - `save_factor`: 保存`query_factor()`返回的`FactorData`, 一次只保存一个因子, 路径为`D:\Aquant project\MF\MF_lab\Factor\FactorType\factorname.parquet`;
    - `load_bar_df`: 改善加载时的内存占用和时间消耗. 原来直接`read_parquet`, 连续加载2018-2026的500只股票的分线内存不够. 所以在计算因子时需要切分时间段分批计算保存到因子库. 现在`load_bar_df`不一次加载全部数据, 而是`scan_parquet`, 先筛选后加载;
    - 对因子做统一的接口: `FactorRequest`, `FactorData`, 想给因子添加什么属性, 分类等就加进去;
    - `cal_factor_daily`: 修改, 现在接收`FactorRequest`, 返回 `list[FactorData]`, 一次只计算一个因子;
    - 移除`save_factor_by_month`, `load_factor_from_month`, 不再按月分区, 只是分批计算, 单个因子用一个`parquet`文件保存;
    - `load_factor`: 新增函数, 从`parquet`文件中加载因子;
    - 修改`missing_ratio`: 适配新的`load_factor`;
    - 2026-05-22: `save_dataset`/`load_dataset`/`remove_dataset` 重构为` Parquet+JSON` 目录格式, 适配重构的`Alphadataset`, 消除`pickle`内存峰值, 支持懒加载, 兼容旧`.pkl`, 不再全部一次性加载, 用什么加载什么;
    - 2026-05-25: 修改`save_signal`, `load_signal`函数, 信号分为样本内和样本外信号, 样本内观察模型是否学到了关系, 样本外观察关系能否延续;
    - 2026-06-12: 修改`load_bar_df`, 添加`skip_suspended`参数, `False`保留停牌日数据, `True`除去停牌日数据, `rqdata`返回的停牌数据使用前向填充, `volume` = 0;
---

- `C:\veighna_studio\Lib\site-packages\vnpy_rqdata\rqdata_datafeed.py`
  - 数据获取接口,目前可获取k线 `query_bar_history`;
  - 未来要添加各种数据接口，例如资产表等;
  - 更新：
    - `_query_bar_history`: 添加了`adjust_type`, 不再默认前复权;
    - `query_industry_data`: 新增函数,获取行业数据;
    - 在`_query_bar_history`里新增可以查询"`RI`"后缀的指数;
    - 修改`to_rq_symbol`,兼容`'VTRI'`的转换;
    - `query_factor`: 新增函数, 获取`rqdata`的股票因子, 一次只请求一个因子;
---

- `C:\veighna_studio\Lib\site-packages\vnpy\trader\object.py`
  - 定义数据类型;
  - 更新：
    - `HistoryRequest`: 添加了`adjust_type`,默认`post`;
    - `BarData`: 添加`adjust_type`,默认`post`;
    - `ACTIVE_STATUSES2`: 新增， 将 `PARTTRADED` 不看做活跃状态, 即在订单交易中 交易可以成交的部分 订单完成， 参考`backtesting2 process_order()`;
    - `is_active2`: 新增函数, 以`ACTIVE_STATUSES2` 为标准判断是否活跃;
    - `IndustryRequest`: 新增行业类;
    - `IndustryData`: 新增行业数据类;
    - `FactorRequest`: 新增因子请求, 调用`rq.get_factor()`;
    - `FactorData`: 新增因子类;
    - 修改`FactorRequest`: 区分请求`rqdat`a还是`factor_define`;
---

- `C:\veighna_studio\Lib\site-packages\vnpy\trader\constant.py`
  - 定义交易常数;
  - 更新：
    - `Interval`: 添加`WEEKLY`,`MONTHLY`;
    - `Exchange`: 添加`'VTRI'`;
    - `FactorType`: 新增因子类型;
---

- `C:\veighna_studio\Lib\site-packages\vnpy\alpha\dataset\template.py`
  - 合并训练数据、准备数据、处理数据;
  - 更新：
    - `prepare_data`: 修改, 加入的特征dataframe中的因子列名不必要是`"data"`, 可以是因子名;
    - 2026-05-22: `raw_df`/`infer_df`/`learn_df`/`result_df` 改为`@property`懒加载, `prepare_data`后释放`self.df`和`feature_results`降低内存峰值, 新增`add_lag_features()`向量化滞后特征方法;
    - 2026-05-23: 新增extract_lambdarank_data函数, 对数据集的label按等级分类, 即对每天的股票进行收益率评级;
---

- `C:\veighna_studio\Lib\site-packages\vnpy\alpha\strategy\__init__.py`
  - 更新：
    - 将`BacktestingEngine2`加入初始化; 
---

- `C:\veighna_studio\Lib\site-packages\vnpy\alpha\strategy\template.py`
  - 更新：
    - `update_order2`: 新增函数，部分成交不算活跃;
---

- `C:\veighna_studio\Lib\site-packages\vnpy\alpha\strategy\template3.py`
  - 基于`template.py`增强的策略模板类;
  - `execute_trading_open`: 三阶段执行(全卖→撮合卖→逐个买), 基于`open_price`定价, 用`process_order`单笔撮合;
  - `calc_affordable_buy`: 用户/券商用接口, 计算当前资金下最大可买股数(考虑佣金+`min_volume`);
  - `sync_targets`: 日末`target_data`同步到`pos_data`, 防止过期target延续至下一日;
---

- `C:\veighna_studio\Lib\site-packages\vnpy\alpha\factor_define.py`
  - 见主代码库
---
- `C:\veighna_studio\Lib\site-packages\vnpy\alpha\walkforward`
  - 2026-5-27：新增滚动回测模块
---
- `C:\veighna_studio\Lib\site-packages\vnpy\alpha\walkforward\window.py`
  - 滚动回测窗口类`WalkForwardWindow`, 训练集+验证集+测试集算作一个窗口
---

- `C:\veighna_studio\Lib\site-packages\vnpy\alpha\walkforward\runner.py`
  - `WalkForwardRunner`: 滚动回测运行器
  - `LGBMLR_Runner`: 继承自`WalkForwardRunner`的LightGBM LambdaRank 滚动回测运行器

---


### 2.3 数据库:
            D:\Aquant project\MF\MF_lab
            ├── component
            │   └── 000300.SSE
            ├── dataset
            │   ├── v100
            │   │     ├── infer.parquet
            │   │     ├── learn.parquet
            │   │     ├── meta.json
            │   │     ├── raw.parquet
            │   │     └── result.parquet
            ├── factor
            │   ├── Price and Volume
            │   │   └── late_skew_ret.parquet
            │   └── Fundamental
            │        └── market_cap_3.parquet
            ├── industry
            │   └── cn
            │       └── citics_2019
            │           ├── `component`
            │           │   └── 2018-01-02
            │           │       └── 电子-60
            │           │           ├── 元器件-6030
            │           │           │      ├── 其他元器件Ⅲ-603010
            │           │           │      │    └── 603010.json
            │           │           │      └── 6030.json
            │           │           └── 60.json
            │           └── mapping
            │               └── 2018-01-02
            │                   └── mapping.parquet
            ├── K
            │   ├── daily
            │   │   └── 000001.`SZSE`.parquet
            │   ├── daily_post
            │   ├── daily_pre
            │   ├── minute
            │   ├── minute_post
            │   ├── minute_pre
            │   ├── monthly
            │   ├── monthly_post
            │   ├── monthly_pre
            │   ├── weekly
            │   ├── weekly_post
            │   └── weekly_pre
            ├── model
            │   ├── v7.pkl
            │   └── walkforward
            │       └── lgb_baseline
            │           ├── 0.pkl
            │           └── 1.pkl
            ├── signal
            │   ├── v100
            │   │    ├── v100_ALL.parquet
            │   │    ├── v100_IS.parquet
            │   │    ├── v100_OS.parquet
            │   │    ├── v100_TV.parquet
            │   │    └── v100_VA.parquet
            │   └── walkforward
            │       └── lgb_baseline
            │           ├── 0.parquet
            │           └── all.parquet
            ├── contract.json
            ├── trading_calendar.pkl
            └── trading_days.json

- `D:\Aquant project\MF\MF_lab\component`
  - 成分股名单 `dict[str: list[str]]`  前一个str是时间 后面str列表是股票代码列表，后缀`'SSE'`、`'SZSE'`

- `D:\Aquant project\MF\MF_lab\K\daily`
  - 无复权日线数据 k线数据均是`parquet`文件

- `D:\Aquant project\MF\MF_lab\K\daily_post`
  - 后复权日线

- `D:\Aquant project\MF\MF_lab\K\daily_pre`
  - 前复权日线

- `D:\Aquant project\MF\MF_lab\K\minute`
  - 无复权分线

- `D:\Aquant project\MF\MF_lab\K\weekly`
  - 无复权周k 合成得到

- `D:\Aquant project\MF\MF_lab\K\monthly`
  - 无复权月k 合成得到

- `D:\Aquant project\MF\MF_lab\dataset`
  - 整合好的学习数据
  - 旧格式: `.pkl` (pickle, 单文件)
  - 新格式(2026-05-22): 目录 + `Parquet` + `JSON`, 如 `dataset/v5/meta.json + learn.parquet + raw.parquet + infer.parquet + result.parquet`

- `D:\Aquant project\MF\MF_lab\factor`
  - 因子

- `D:\Aquant project\MF\MF_lab\factor\Price and Volume`
  - 价量因子

- `D:\Aquant project\MF\MF_lab\factor\Fundamental`
  - 基本面因子
  - 因子数据的`"datetime"`对应的`"data"`是这一天收盘后根据已有信息计算的;

- `D:\Aquant project\MF\MF_lab\model`
  - 模型 `.pkl`
  - `\walkforward` 滚动回测的模型

- `D:\Aquant project\MF\MF_lab\signal`
  - 信号 `.parquet`
  - `\walkforward` 滚动回测的信号

- `D:\Aquant project\MF\MF_lab\contract.json`
  - 合约设置

- `D:\Aquant project\MF\MF_lab\trading_days.json`
  - 交易日列表 

- `D:\Aquant project\MF\MF_lab\trading_calendar.pkl`
  - 周线、月线合成时生成的日历

- `D:\Aquant project\MF\MF_lab\industry`
  - 存储行业分类及特征
---

- 使用`lab.load_bar_df()`加载的数据类型
  - `pl.dataframe columns: ['datetime', 'vt_symbol', 'open', 'close' , 'high', 'low' , 'volume', 'turnover', 'open_interest', 'vwap']`
  - 周线、月线没有 `'vwap'`
  - 分线里 `9:30:00` 表示的是第一根k线，也就是9:30:00-9:31:00之间的数据 `14:59:00`是最后一根k线
  - `'turnover'` 字段表示的是总成交金额 不是直接的换手率 ，计算换手率需要用 `'turnover'/ 'volume'`

- 每日需要下载的数据
  - 交易日, 股票池和benchmark的k线, contract_setting, 行业数据, rqdata获取的因子, factor_define计算的因子,






---
#### version: 2026-06-14




