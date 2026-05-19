# VNPY Alpha 回测框架完整流程解析

> 文件位置：`C:\Users\86188\Desktop\MF\vnpy_alpha_backtesting_flow.md`
> 生成时间：2026-03-30

---

## 目录

1. [初始化阶段](#一初始化阶段)
2. [回测运行](#二回测运行)
3. [每根K线处理](#三每根k线处理-new_bars)
4. [策略决策层](#四策略决策层-on_bars)
5. [交易执行](#五交易执行-execute_trading)
6. [委托链路](#六委托链路)
7. [撮合引擎](#七撮合引擎-cross_order)
8. [持仓更新](#八持仓更新)
9. [统计计算](#九统计计算)
10. [完整调用链](#十完整调用链图示)
11. [设计自己的策略](#十一设计自己的策略)

---

## 一、初始化阶段

在用户代码中完成，准备工作包括：

```python
from vnpy.alpha import BacktestingEngine, AlphaStrategy
from vnpy.trader.constant import Interval

# 1. 创建回测引擎
engine = BacktestingEngine(lab)

# 2. 设置回测参数
engine.set_parameters(
    vt_symbols=["600000.SSE", "000001.SZSE", "600519.SSE"],  # 股票池
    interval=Interval.DAILY,                                    # 数据周期（日线/分钟线）
    start=datetime(2020, 1, 1),                                # 开始日期
    end=datetime(2024, 12, 31),                                # 结束日期
    capital=1_000_000,                                         # 初始资金（默认100万）
    risk_free=0.02,                                            # 无风险利率
    annual_days=240                                            # 年化交易日数
)

# 3. 加载历史数据
engine.load_data()
# 内部逻辑：遍历 vt_symbols，调用 lab.load_bar_data() 加载每只股票的历史K线
# 数据存入：self.history_data[(datetime, vt_symbol)] = BarData
# 时间点存入：self.dts（所有出现过的时间点集合）

# 4. 添加策略
engine.add_strategy(
    strategy_class=EquityDemoStrategy,      # 策略类（需继承AlphaStrategy）
    setting={                               # 策略参数
        "top_k": 50,
        "n_drop": 5,
        "price_add": 0.05,
        "cash_ratio": 0.95
    },
    signal_df=signal_df                     # 模型预测信号（Polars DataFrame）
)
```

---

## 二、回测运行

主入口：`run_backtesting()`

```python
def run_backtesting(self) -> None:
    """开始回测"""
    self.strategy.on_init()           # 调用策略初始化回调
    logger.info("策略初始化完成")
    
    # 按时间顺序遍历所有时间点
    dts = sorted(self.dts)
    
    logger.info("开始回放历史数据")
    for dt in dts:
        try:
            self.new_bars(dt)         # 处理每根K线
        except Exception:
            logger.info("触发异常，回测终止")
            logger.info(traceback.format_exc())
            return
    
    logger.info("历史数据回放结束")
```

---

## 三、每根K线处理 `new_bars()`

这是**每步循环的核心枢纽**，位于 `backtesting.py` 第556行左右。

```python
def new_bars(self, dt: datetime) -> None:
    """推送历史数据"""
    self.datetime = dt
    
    bars: dict[str, BarData] = {}
    
    # 1. 准备当前K线数据
    for vt_symbol in self.vt_symbols:
        # 保存前一根K线的收盘价
        last_bar = self.bars.get(vt_symbol)
        if last_bar and last_bar.close_price:
            self.pre_closes[vt_symbol] = last_bar.close_price
        
        # 获取当前时间的K线数据
        bar = self.history_data.get((dt, vt_symbol))
        
        if bar:
            # 有数据：更新该合约当前K线
            self.bars[vt_symbol] = bar
            bars[vt_symbol] = bar
        elif vt_symbol in self.bars:
            # 无数据但之前有数据：用前一根K线填充
            old_bar = self.bars[vt_symbol]
            fill_bar = BarData(
                symbol=old_bar.symbol,
                exchange=old_bar.exchange,
                datetime=dt,
                open_price=old_bar.close_price,
                high_price=old_bar.close_price,
                low_price=old_bar.close_price,
                close_price=old_bar.close_price,
                gateway_name=old_bar.gateway_name
            )
            self.bars[vt_symbol] = fill_bar
    
    # 2. 【撮合】处理上一周期留下的委托单
    # ⚠️ 注意：先撮合，再决策！
    self.cross_order()
    
    # 3. 【决策】调用策略，生成新的目标仓位
    self.strategy.on_bars(bars)
    
    # 4. 更新每日收盘价（用于盈亏计算）
    self.update_daily_close(self.bars, dt)
```

**关键顺序**：
- 先 `cross_order()`：成交上一周期的委托
- 再 `on_bars()`：根据最新持仓生成新的委托

这意味着策略看到的 `pos_data` 是**上一批委托成交后的实际持仓**。

---

## 四、策略决策层 `on_bars()`

这是**用户写策略时必须实现的核心方法**。

### 4.1 基类定义（template.py）

```python
class AlphaStrategy(metaclass=ABCMeta):
    """Alpha策略模板类"""
    
    def __init__(self, strategy_engine, strategy_name, vt_symbols, setting):
        self.strategy_engine = strategy_engine
        self.strategy_name = strategy_name
        self.vt_symbols = vt_symbols
        
        # 关键数据结构
        self.pos_data: dict[str, float] = defaultdict(float)      # 实际持仓（成交后更新）
        self.target_data: dict[str, float] = defaultdict(float)   # 目标持仓（你设置的）
        
        self.orders: dict[str, OrderData] = {}
        self.active_orderids: set[str] = set()
        
        # 从setting设置参数
        for k, v in setting.items():
            if hasattr(self, k):
                setattr(self, k, v)
    
    @abstractmethod
    def on_init(self) -> None:
        """初始化回调（必须实现）"""
        pass
    
    @abstractmethod
    def on_bars(self, bars: dict[str, BarData]) -> None:
        """Bar切片回调（必须实现）"""
        pass
    
    @abstractmethod
    def on_trade(self, trade: TradeData) -> None:
        """成交回调（必须实现）"""
        pass
    
    def set_target(self, vt_symbol: str, target: float) -> None:
        """设置目标持仓"""
        self.target_data[vt_symbol] = target
    
    def get_target(self, vt_symbol: str) -> float:
        """查询目标持仓"""
        return self.target_data[vt_symbol]
    
    def get_pos(self, vt_symbol: str) -> float:
        """查询实际持仓"""
        return self.pos_data[vt_symbol]
    
    def get_signal(self) -> pl.DataFrame:
        """获取当前时间的模型信号"""
        return self.strategy_engine.get_signal()
```

### 4.2 示例策略逻辑（equity_demo_strategy.py）

```python
class EquityDemoStrategy(AlphaStrategy):
    """股票多头演示策略"""
    
    # 策略参数（可在setting中覆盖）
    top_k: int = 50                 # 最大持股数量
    n_drop: int = 5                 # 每次卖出信号最弱的n只
    min_days: int = 3               # 最小持仓天数
    cash_ratio: float = 0.95        # 资金使用率
    min_volume: int = 100           # 最小交易单位（手）
    price_add: float = 0.05         # 委托价调整比例（滑点）
    
    def on_init(self) -> None:
        """初始化"""
        self.holding_days: defaultdict = defaultdict(int)
        self.write_log("策略初始化完成")
    
    def on_trade(self, trade: TradeData) -> None:
        """成交回调：卖出时清除持仓天数记录"""
        if trade.direction == Direction.SHORT:
            self.holding_days.pop(trade.vt_symbol, None)
    
    def on_bars(self, bars: dict[str, BarData]) -> None:
        """核心策略逻辑"""
        # 1. 获取并排序信号
        signal_df = self.get_signal()
        signal_df = signal_df.sort("signal", descending=True)
        
        # 2. 获取当前持仓并更新持仓天数
        pos_symbols = [s for s, p in self.pos_data.items() if p != 0]
        for s in pos_symbols:
            self.holding_days[s] += 1
        
        # 3. 生成卖出列表
        # 3.1 取信号最强的top_k作为潜在持仓
        active_symbols = set(signal_df["vt_symbol"][:self.top_k])
        # 3.2 合并当前持仓（确保持仓股票能被处理）
        active_symbols.update(pos_symbols)
        # 3.3 过滤出这些股票的信号
        active_df = signal_df.filter(pl.col("vt_symbol").is_in(active_symbols))
        
        # 3.4 卖出不在成分股中的持仓
        component_symbols = set(signal_df["vt_symbol"])
        sell_symbols = set(pos_symbols).difference(component_symbols)
        
        # 3.5 卖出信号最弱的n_drop只持仓股
        for vt_symbol in active_df["vt_symbol"][-self.n_drop:]:
            if vt_symbol in pos_symbols:
                sell_symbols.add(vt_symbol)
        
        # 4. 生成买入列表
        # 需要买入的数量 = 卖出数量 + top_k - 当前持仓数
        buyable_df = signal_df.filter(~pl.col("vt_symbol").is_in(pos_symbols))
        buy_quantity = len(sell_symbols) + self.top_k - len(pos_symbols)
        buy_symbols = list(buyable_df[:buy_quantity]["vt_symbol"])
        
        # 5. 卖出再平衡
        cash = self.get_cash_available()  # 昨日结算后的可用资金
        
        for vt_symbol in sell_symbols:
            # 检查最小持仓天数
            if self.holding_days[vt_symbol] < self.min_days:
                continue
            
            bar = bars.get(vt_symbol)
            if not bar:
                continue
            
            sell_price = bar.close_price
            sell_volume = self.get_pos(vt_symbol)
            
            # 设置目标持仓为0
            self.set_target(vt_symbol, target=0)
            
            # 估算卖出后的资金
            turnover = sell_price * sell_volume
            cost = max(turnover * self.close_rate, self.min_commission)
            cash += turnover - cost
        
        # 6. 买入再平衡
        if buy_symbols:
            buy_value = cash * self.cash_ratio / len(buy_symbols)
            
            for vt_symbol in buy_symbols:
                buy_price = bars[vt_symbol].close_price
                if not buy_price:
                    continue
                
                # 计算买入数量（取整到最小交易单位）
                buy_volume = round_to(buy_value / buy_price, self.min_volume)
                self.set_target(vt_symbol, buy_volume)
        
        # 7. 执行交易（调用基类方法）
        self.execute_trading(bars, price_add=self.price_add)
```

---

## 五、交易执行 `execute_trading()`

位于 `template.py`，根据 target 和 pos 的差异生成委托。

```python
def execute_trading(self, bars: dict[str, BarData], price_add: float) -> None:
    """
    根据目标持仓和实际持仓的差异，自动发送委托
    
    Parameters:
        bars: 当前K线数据
        price_add: 价格调整比例（滑点）
                 买入时: order_price = close * (1 + price_add)
                 卖出时: order_price = close * (1 - price_add)
    """
    # 先撤销所有未成交委托
    self.cancel_all()
    
    # 遍历有数据的合约
    for vt_symbol, bar in bars.items():
        target = self.get_target(vt_symbol)
        pos = self.get_pos(vt_symbol)
        diff = target - pos   # 仓位差
        
        if diff > 0:
            # ========== 需要买入 ==========
            # 计算委托价：收盘价加价（确保成交）
            order_price = bar.close_price * (1 + price_add)
            
            cover_volume = 0
            buy_volume = 0
            
            if pos < 0:
                # 当前有空仓，先平空
                cover_volume = min(diff, abs(pos))
                self.cover(vt_symbol, order_price, cover_volume)
                diff -= cover_volume
            
            # 剩余部分开多仓
            buy_volume = diff
            if buy_volume > 0:
                self.buy(vt_symbol, order_price, buy_volume)
        
        elif diff < 0:
            # ========== 需要卖出 ==========
            # 计算委托价：收盘价降价（确保成交）
            order_price = bar.close_price * (1 - price_add)
            
            sell_volume = 0
            short_volume = 0
            
            if pos > 0:
                # 当前有多仓，先平仓
                sell_volume = min(abs(diff), pos)
                self.sell(vt_symbol, order_price, sell_volume)
                diff += sell_volume  # diff是负数
            
            # 剩余部分开空仓
            short_volume = abs(diff)
            if short_volume > 0:
                self.short(vt_symbol, order_price, short_volume)
```

---

## 六、委托链路

从策略到引擎的完整调用链：

```
strategy.buy(vt_symbol, price, volume)          # template.py 第85行
    ↓
strategy.send_order(
    vt_symbol, Direction.LONG, Offset.OPEN, price, volume
)  # template.py 第106行
    ↓
engine.send_order(
    strategy, vt_symbol, direction, offset, price, volume
)  # backtesting.py 第665行
    ↓
# 1. 价格取整到最小变动单位
price = round_to(price, self.priceticks[vt_symbol])

# 2. 创建订单ID
self.limit_order_count += 1
orderid = str(self.limit_order_count)

# 3. 创建OrderData对象
order = OrderData(
    symbol=symbol,
    exchange=exchange,
    orderid=orderid,
    direction=direction,        # LONG/SHORT
    offset=offset,              # OPEN/CLOSE
    price=price,                # 委托价
    volume=volume,              # 委托量
    status=Status.SUBMITTING,   # 初始状态：已提交
    datetime=self.datetime,     # 委托时间
    gateway_name=self.gateway_name,
)

# 4. 存入订单簿
self.active_limit_orders[order.vt_orderid] = order  # 活跃委托（待撮合）
self.limit_orders[order.vt_orderid] = order          # 所有委托（记录）

# 5. 返回订单ID
return [order.vt_orderid]
```

---

## 七、撮合引擎 `cross_order()`

位于 `backtesting.py` 第568行左右。

```python
def cross_order(self) -> None:
    """撮合限价单"""
    
    # 遍历所有活跃委托（必须用list复制，因为会修改原字典）
    for order in list(self.active_limit_orders.values()):
        bar = self.bars[order.vt_symbol]
        
        # 定义撮合价格边界
        long_cross_price = bar.low_price      # 多头：最低价能买到
        short_cross_price = bar.high_price    # 空头：最高价能卖出
        long_best_price = bar.open_price      # 最优买价
        short_best_price = bar.open_price     # 最优卖价
        
        # 更新委托状态（从SUBMITTING变为NOTTRADED）
        if order.status == Status.SUBMITTING:
            order.status = Status.NOTTRADED
            self.strategy.update_order(order)
        
        # 计算涨跌停价
        pricetick = self.priceticks[order.vt_symbol]
        pre_close = self.pre_closes.get(order.vt_symbol, 0)
        limit_up = round_to(pre_close * 1.1, pricetick)    # 涨停价
        limit_down = round_to(pre_close * 0.9, pricetick)  # 跌停价
        
        # 判断能否成交
        long_cross = (
            order.direction == Direction.LONG
            and order.price >= long_cross_price      # 委托价 ≥ 最低价
            and long_cross_price > 0
            and bar.low_price < limit_up             # 非全天涨停
        )
        
        short_cross = (
            order.direction == Direction.SHORT
            and order.price <= short_cross_price     # 委托价 ≤ 最高价
            and short_cross_price > 0
            and bar.high_price > limit_down          # 非全天跌停
        )
        
        if not long_cross and not short_cross:
            continue  # 不成交，保留到下一周期
        
        # ========== 成交处理 ==========
        
        # 1. 更新委托状态为全部成交
        order.traded = order.volume
        order.status = Status.ALLTRADED
        self.strategy.update_order(order)
        
        # 2. 从活跃委托列表移除
        self.active_limit_orders.pop(order.vt_orderid)
        
        # 3. 计算成交价（保护性价格）
        if long_cross:
            # 多头：取委托价和开盘价中较低者（保护买方）
            trade_price = min(order.price, long_best_price)
        else:
            # 空头：取委托价和开盘价中较高者（保护卖方）
            trade_price = max(order.price, short_best_price)
        
        # 4. 生成成交ID
        self.trade_count += 1
        
        # 5. 创建成交记录
        trade = TradeData(
            symbol=order.symbol,
            exchange=order.exchange,
            orderid=order.orderid,
            tradeid=str(self.trade_count),
            direction=order.direction,
            offset=order.offset,
            price=trade_price,
            volume=order.volume,
            datetime=self.datetime,
            gateway_name=self.gateway_name,
        )
        
        # 6. 更新资金
        size = self.sizes[trade.vt_symbol]  # 合约乘数
        turnover = trade.price * trade.volume * size
        
        # 计算手续费
        if trade.direction == Direction.LONG:
            commission = turnover * self.long_rates[trade.vt_symbol]
            self.cash -= turnover  # 买入扣减资金
        else:
            commission = turnover * self.short_rates[trade.vt_symbol]
            self.cash += turnover  # 卖出增加资金
        
        self.cash -= commission  # 扣减手续费
        
        # 7. 推送成交到策略
        self.strategy.update_trade(trade)
        
        # 8. 保存成交记录
        self.trades[trade.vt_tradeid] = trade
```

---

## 八、持仓更新

### 8.1 策略层 `update_trade()`（template.py）

```python
def update_trade(self, trade: TradeData) -> None:
    """成交后更新实际持仓"""
    if trade.direction == Direction.LONG:
        self.pos_data[trade.vt_symbol] += trade.volume
    else:
        self.pos_data[trade.vt_symbol] -= trade.volume
    
    # 调用策略自定义的成交处理
    self.on_trade(trade)
```

### 8.2 关键数据结构总结

| 变量 | 类型 | 含义 | 更新时机 |
|------|------|------|---------|
| `pos_data` | dict[str, float] | 实际持仓（已成交） | 成交后 update_trade() |
| `target_data` | dict[str, float] | 目标持仓（想持有） | 策略 set_target() |
| `active_limit_orders` | dict[str, OrderData] | 未成交委托 | 发单时加入，成交/撤销时移除 |
| `limit_orders` | dict[str, OrderData] | 所有委托记录 | 发单时加入，永不移除 |
| `trades` | dict[str, TradeData] | 所有成交记录 | 成交时加入 |

---

## 九、统计计算

回测结束后调用：

```python
# 1. 计算每日盈亏
engine.calculate_result()
# 遍历所有成交，按日期汇总，计算：
# - turnover（成交额）
# - commission（手续费）
# - trading_pnl（交易盈亏）
# - holding_pnl（持仓盈亏）

# 2. 计算统计指标
engine.calculate_statistics()
# 输出：
# - 总收益率、年化收益
# - 最大回撤、回撤天数
# - 夏普比率
# - 日均盈亏、交易次数等
```

---

## 十、完整调用链图示

```
用户代码
    ↓
【初始化阶段】
engine = BacktestingEngine(lab)
engine.set_parameters(...)
engine.load_data()           # 加载历史数据到 history_data
engine.add_strategy(...)     # 创建策略实例，传入signal_df
    ↓
【回测运行】
engine.run_backtesting()
    ↓
strategy.on_init()           # 策略初始化
    ↓
for dt in sorted(dts):       # 按时间顺序遍历
    ↓
    engine.new_bars(dt)
        │
        ├─► 准备 bars 数据
        │   ├─► 从 history_data 取当前时间K线
        │   └─► 无数据时用前一根K线填充
        │
        ├─► 【撮合】engine.cross_order()
        │   │
        │   ├─► 遍历 active_limit_orders
        │   ├─► 用当前 bars 的高低价格判断是否成交
        │   │       ├─► 多头：委托价 ≥ 最低价
        │   │       └─► 空头：委托价 ≤ 最高价
        │   ├─► 计算成交价（保护性价格）
        │   ├─► 生成 TradeData
        │   ├─► 更新 self.cash（资金变动）
        │   └─► 调用 strategy.update_trade(trade)
        │           │
        │           └─► 更新 pos_data（实际持仓）
        │
        ├─► 【决策】strategy.on_bars(bars)
        │   │
        │   ├─► signal_df = self.get_signal()
        │   │       └─► 从 engine.signal_df 过滤当前时间
        │   │
        │   ├─► 选股/计算目标仓位
        │   │   └─► self.set_target(vt_symbol, target_volume)
        │   │           └─► 更新 target_data
        │   │
        │   └─► self.execute_trading(bars, price_add)
        │       │
        │       ├─► 计算 diff = target - pos
        │       ├─► 计算委托价 = close * (1 ± price_add)
        │       ├─► 调用 buy/sell/short/cover()
        │       │       │
        │       │       └─► 调用 send_order()
        │       │               │
        │       │               └─► 存入 active_limit_orders
        │       │                   （等待下一周期撮合）
        │       │
        │       └─► 返回 vt_orderid 列表
        │
        └─► engine.update_daily_close()
            └─► 更新当日收盘价（用于盈亏计算）
    ↓
【回测结束】
engine.calculate_result()       # 计算每日盈亏
engine.calculate_statistics()   # 计算统计指标
```

---

## 十一、设计自己的策略

### 11.1 最小策略模板

```python
from vnpy.alpha import AlphaStrategy
from vnpy.trader.object import BarData, TradeData
import polars as pl

class MyStrategy(AlphaStrategy):
    """自定义策略模板"""
    
    # 1. 定义参数（可在 add_strategy 时覆盖）
    param1: int = 10
    param2: float = 0.5
    
    def on_init(self) -> None:
        """初始化：加载数据、设置初始状态"""
        self.write_log("策略初始化")
    
    def on_bars(self, bars: dict[str, BarData]) -> None:
        """
        核心逻辑：每个时间点调用一次
        
        输入：
            bars: {vt_symbol: BarData} 当前时间的K线数据
        
        输出（通过方法调用）：
            设置 target_data，然后调用 execute_trading()
        """
        # 1. 获取模型信号
        signal_df = self.get_signal()
        
        # 2. 根据信号计算目标仓位
        # （你的选股/择时逻辑在这里）
        for vt_symbol in self.vt_symbols:
            # 示例：等权重分配
            target = 100  # 每只股票买100股
            self.set_target(vt_symbol, target)
        
        # 3. 执行交易（基类方法，比较target和pos，自动发单）
        self.execute_trading(bars, price_add=0.001)
    
    def on_trade(self, trade: TradeData) -> None:
        """成交回调：处理成交后的逻辑"""
        self.write_log(f"成交: {trade.vt_symbol} {trade.direction} {trade.volume}")
```

### 11.2 关键设计原则

| 原则 | 说明 |
|------|------|
| **分离关注点** | 你只管 `target`，框架管 `price` 和撮合 |
| **避免未来函数** | 信号只能用**当前及之前**的数据计算 |
| **逐K线决策** | `on_bars` 每根K线调用一次，不能跨周期 |
| **滑点处理** | 通过 `price_add` 模拟交易成本 |
| **资金检查** | 用 `get_cash_available()` 获取可用资金 |

### 11.3 常见策略模式

**模式1：日频调仓（示例策略）**
- 每天收盘前根据信号调整持仓
- 次日开盘价附近成交
- 适用：机器学习因子策略

**模式2：分钟级交易**
- 用分钟线数据
- 更精确的成交时间控制
- 适用：高频/日内策略

**模式3：事件驱动**
- 在 `on_trade` 中触发二次交易
- 适用：止盈止损策略

---

## 附录：关键类和方法速查

### BacktestingEngine 关键方法

| 方法 | 功能 |
|------|------|
| `set_parameters()` | 设置回测参数 |
| `load_data()` | 加载历史K线数据 |
| `add_strategy()` | 添加策略实例 |
| `run_backtesting()` | 运行回测 |
| `cross_order()` | 撮合委托 |
| `calculate_result()` | 计算每日盈亏 |
| `calculate_statistics()` | 计算统计指标 |
| `send_order()` | 接收策略委托 |

### AlphaStrategy 关键方法

| 方法 | 功能 |
|------|------|
| `on_init()` | 初始化（必须实现） |
| `on_bars()` | K线回调（必须实现） |
| `on_trade()` | 成交回调（必须实现） |
| `set_target()` | 设置目标持仓 |
| `get_target()` | 获取目标持仓 |
| `get_pos()` | 获取实际持仓 |
| `execute_trading()` | 执行交易（基类提供） |
| `get_signal()` | 获取模型信号 |
| `get_cash_available()` | 获取可用资金 |

### 数据结构

| 结构 | 内容 |
|------|------|
| `BarData` | symbol, exchange, datetime, open, high, low, close, volume |
| `OrderData` | symbol, direction, offset, price, volume, status |
| `TradeData` | symbol, direction, offset, price, volume, tradeid |

---

*文档结束*
