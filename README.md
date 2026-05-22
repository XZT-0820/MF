> Agent：先深入读取与当前任务相关的文件，在读取过程中若出现与该文件紧密相关的其他文件，则加入读取过程，以此反复交互前进，直至了解、通透任务至项目。

# 量化项目`D:\Aquant project\MF`
> 基于 veighna_studio 4.3.0 

## 1. 项目概要、目标:

- 因子库多维度、多跨度、多节点;
- 行业、市值作为因子加入到特征中;
- 策略采用周频或日频调仓，股票池沪深300，因子计算以分线、日线、财务等为元数据;
- 采用机器学习、深度学习方法优化因子权重，捕捉市场信号;
- vnpy框架回测:`C:\veighna_studio\Lib\site-packages\vnpy`;  未进展到拟真与实盘...
- 数据接口为vnpy_rqdata:`C:\veighna_studio\Lib\site-packages\vnpy_rqdata`

## 2. 项目进展:

### 2.1 主代码库: `D:\Aquant project\MF\MF_code`

- `D:\Aquant project\MF\MF_code\download_300_template.ipynb`
  - 沪深300指数成分股分线、日线数据下载模板
  - 更新：
    - `extended_days`必须;

- `D:\Aquant project\MF\MF_code\download_industry.ipynb`
  - 下载行业数据

- `D:\Aquant project\MF\MF_code\download_all.ipynb`
  - 用米筐全A市场加权指数`"866011.RI"`下载A股全市场股票

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

- `D:\Aquant project\MF\MF_code\data_prepare.ipynb`
  - 数据准备，将数据整合到`dataset`里后存储
  - 更新：
    - 将加载因子的函数和计算缺失率的函数移到`lab.py`里;
    - 2026-05-22: 使用新架构, `FactorRequest`加载因子, `add_lag_features`生成滞后, `prepare_data`后释放中间状态,` Parquet+JSON`保存;

- `D:\Aquant project\MF\MF_code\model_train - rank.ipynb`
  - `Lightgbm`模型训练，`objective:rank`，预测排名
  - 使用`C:\veighna_studio\Lib\site-packages\lightgbm`

- `D:\Aquant project\MF\MF_code\model_train.ipynb`
  - `Lightgbm`模型训练，`objective:regression`

- `D:\Aquant project\MF\MF_code\backtest.ipynb`
  - 回测；
  - 策略采用`C:\veighna_studio\Lib\site-packages\vnpy\alpha\strategy\strategies\equity_demo_strategy2.py`

### 2.2 引用代码库: `C:\veighna_studio\Lib\site-packages\vnpy`  `C:\veighna_studio\Lib\site-packages\vnpy_rqdata`

- `C:\veighna_studio\Lib\site-packages\vnpy\alpha\strategy\backtesting2.py`
  - 新增回测引擎;
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

- `C:\veighna_studio\Lib\site-packages\vnpy\alpha\strategy\strategies\equity_demo_strategy2.py`
  - 新增回测策略 strategy;
  - 收盘计算信号，明日开盘执行交易;
  - 更新：
    - `self._buy_symbols`: 新增买单顺序, 供`cross_order`使用;

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
    - 2026-05-22: `save_dataset`/`load_dataset`/`remove_dataset` 重构为` Parquet+JSON` 目录格式, 适配重构的`Alphadataset`, 不再全部一次性加载, 用什么加载什么;
    - `save_dataset`/`load_dataset`/`remove_dataset`: 2026-05-22 重构为 `Parquet+JSON` 目录格式, 消除`pickle`内存峰值, 支持懒加载, 兼容旧`.pkl`;
    - `list_all_datasets`: 兼容新旧格式;
    - 需要一个计算行业特征的函数:lab.cal_industry_feature(self, `start`: `datetime` | `date`, `end`:`datetime` | `date`, `source`:`str`, `market`: `str`, `code`: `str`)
      比如 cal_industry_feature(`datetime`(2018,1,5), `datetime`(2018,1,10), "citics_2019", "cn","60") 函数会去D:\Aquant project\MF\MF_lab\industry\cn\citics_2019\`component`
      查找在start-end之间的目录，比如2018，1，5 根据对后缀的筛选找到D:\Aquant project\MF\MF_lab\industry\cn\citics_2019\`component`\2018-01-05\电子-60
      加载里面的60.json文件,这是该行业成分股列表，然后加载这个股票池的k线（可能要包含extended）那么什么特征呢，比如行业5日动量，就是该行业成分股5日收益率市值加权平均, 那么需要新的因子定义了，或许可以在`factor_define`里
      设定一个注册表，专门记录行业因子，比如这个5日动量， momentum(df, `start`)，参数也不一定是只有k线df，这里还需要市值数据，那就得再接受一个marketcap数据，这意味着加载该股票池的什么数据需要看该因子函数
      有什么参数。另外，cal_industry_feature还需要 feature_name参数，这样才知道计算哪个特征，加载哪些表等等，这些都应该记录在参数表里。而且，这意味着每当定义的因子函数需要一个
      新的表时，都需要在C:\veighna_studio\Lib\site-packages\vnpy_rqdata\rqdata_datafeed.py里写一个获取这个数据的新的接口，再在lab里写一个保存和加载的函数。 总之,这个行业函数 返回 pl.datafram "`datetime`","`code`", "`data`"
      所以还需要保存和加载的函数 将行业特征保存到D:\Aquant project\MF\MF_lab\industry\cn\citics_2019\feature\momentum_5d.parquet 可以不按月分区。

- `C:\veighna_studio\Lib\site-packages\vnpy_rqdata\rqdata_datafeed.py`
  - 数据获取接口,目前可获取k线 `query_bar_history`;
  - 未来要添加各种数据接口，例如资产表等;
  - 更新：
    - `_query_bar_history`: 添加了`adjust_type`, 不再默认前复权;
    - `query_industry_data`: 新增函数,获取行业数据;
    - 在`_query_bar_history`里新增可以查询"`RI`"后缀的指数;
    - 修改`to_rq_symbol`,兼容`'VTRI'`的转换;
    - `query_factor`: 新增函数, 获取`rqdata`的股票因子, 一次只请求一个因子;

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

- `C:\veighna_studio\Lib\site-packages\vnpy\trader\constant.py`
  - 定义交易常数;
  - 更新：
    - `Interval`: 添加`WEEKLY`,`MONTHLY`;
    - `Exchange`: 添加`'VTRI'`;
    - `FactorType`: 新增因子类型;

- `C:\veighna_studio\Lib\site-packages\vnpy\alpha\dataset\template.py`
  - 合并训练数据、准备数据、处理数据;
  - 更新：
    - `prepare_data`: 修改, 加入的特征dataframe中的因子列名不必要是`"data"`, 可以是因子名;
    - 2026-05-22: `raw_df`/`infer_df`/`learn_df`/`result_df` 改为`@property`懒加载, `prepare_data`后释放`self.df`和`feature_results`降低内存峰值, 新增`add_lag_features()`向量化滞后特征方法;

- `C:\veighna_studio\Lib\site-packages\vnpy\alpha\strategy\__init__.py`
  - 更新：
    - 将`BacktestingEngine2`加入初始化; 

- `C:\veighna_studio\Lib\site-packages\vnpy\alpha\strategy\template.py`
  - 更新：
    - `update_order2`: 新增函数，部分成交不算活跃;

- `C:\veighna_studio\Lib\site-packages\vnpy\alpha\factor_define.py`
  - 见主代码库

### 2.3 数据库:
            D:\Aquant project\MF\MF_lab
            ├── `component`
            │   └── 000300.`SSE`
            ├── dataset
            │   └── v5.pkl
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
            │   └── v7.pkl
            ├── signal
            │   └── v7.parquet
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

- `D:\Aquant project\MF\MF_lab\signal`
  - 信号 `.parquet`

- `D:\Aquant project\MF\MF_lab\contract.json`
  - 合约设置

- `D:\Aquant project\MF\MF_lab\trading_days.json`
  - 交易日列表 

- `D:\Aquant project\MF\MF_lab\trading_calendar.pkl`
  - 周线、月线合成时生成的日历

- `D:\Aquant project\MF\MF_lab\industry`
  - 存储行业分类及特征

- 使用`lab.load_bar_df()`加载的数据类型
  - `pl.dataframe columns: ['datetime', 'vt_symbol', 'open', 'close' , 'high', 'low' , 'volume', 'turnover', 'open_interest', 'vwap']`
  - 周线、月线没有 `'vwap'`
  - 分线里 `9:30:00` 表示的是第一根k线，也就是9:30:00-9:31:00之间的数据 `14:59:00`是最后一根k线
  - `'turnover'` 字段表示的是总成交金额 不是直接的换手率 ，计算换手率需要用 `'turnover'/ 'volume'`

- 每日需要下载的数据
  - 交易日, 股票池和benchmark的k线, contract_setting, 行业数据, rqdata获取的因子, factor_define计算的因子,
