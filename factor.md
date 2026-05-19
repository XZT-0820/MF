**27 个因子**（15 个分钟频 + 8 个逐笔成交 + 4 个逐笔委托）

---

# 因子复现说明书

## 一、分钟频因子（15个）

写在代码里的name依次是：
late_skew_ret
down_vol_perc
corr_ret_lastret
corr_close_nextopen
volume_perc2
volume_perc3
volume_perc4
volume_perc5
volume_perc6
volume_perc7
corr_volume_item
early_corr_volume_ret
bigorder_ret
down_single_amt_perc
corr_volume_amplitude
---

### 因子 1：尾盘收益率偏度

| 项目 | 内容 |
|------|------|
| **所需数据** | 分钟频收盘价 \( P_t \)；尾盘时段定义（如 14:30–15:00） |
| **定义** | 收盘前一段时间内分钟收益率的分布不对称性 |
| **公式** | \( r_t = \frac{P_t}{P_{t-1}} - 1 \) <br> \( \text{Skewness} = \frac{ \frac{1}{n} \sum (r_i - \bar{r})^3 }{ \left( \frac{1}{n} \sum (r_i - \bar{r})^2 \right)^{3/2} } \) |

---

### 因子 2：下行收益率波动占比

| 项目 | 内容 |
|------|------|
| **所需数据** | 分钟频收盘价 \( P_t \) |
| **定义** | 负收益率分钟的波动占全天总波动的比例 |
| **公式** | \( r_t = \frac{P_t}{P_{t-1}} - 1 \) <br> \( \sigma_{\text{total}}^2 = \frac{1}{n} \sum (r_i - \bar{r})^2 \) <br> \( \sigma_{\text{down}}^2 = \frac{1}{n_{\text{down}}} \sum_{r_i < 0} (r_i - \bar{r}_{\text{down}})^2 \) <br> \( \text{Factor} = \frac{\sigma_{\text{down}}^2}{\sigma_{\text{total}}^2} \) |

---

### 因子 3：前后两分钟收益率相关性

| 项目 | 内容 |
|------|------|
| **所需数据** | 分钟频收盘价 \( P_t \) |
| **定义** | 相邻分钟收益率的一阶自相关性 |
| **公式** | \( r_t = \frac{P_t}{P_{t-1}} - 1 \) <br> \( \rho = \frac{ \sum_{t=2}^n (r_{t-1} - \bar{r})(r_t - \bar{r}) }{ \sqrt{ \sum_{t=2}^n (r_{t-1} - \bar{r})^2 } \sqrt{ \sum_{t=2}^n (r_t - \bar{r})^2 } } \) |

---

### 因子 4：前一分钟收盘价与后一分钟开盘价相关性

| 项目 | 内容 |
|------|------|
| **所需数据** | 分钟频开盘价 \( O_t \)、收盘价 \( C_t \) |
| **定义** | 前一分钟收盘价与后一分钟开盘价的线性相关性 |
| **公式** | \( \rho = \frac{ \sum_{t=2}^n (C_{t-1} - \bar{C})(O_t - \bar{O}) }{ \sqrt{ \sum_{t=2}^n (C_{t-1} - \bar{C})^2 } \sqrt{ \sum_{t=2}^n (O_t - \bar{O})^2 } } \) |

---

### 因子 5：第2个半小时成交量占全天成交量比例

| 项目 | 内容 |
|------|------|
| **所需数据** | 分钟频成交量 \( V_t \)；时段定义（10:00–10:30） |
| **定义** | 第2个半小时（10:00–10:30）成交量占全天比例 |
| **公式** | \( \text{Factor} = \frac{\sum_{t \in T_2} V_t}{\sum_{t=1}^n V_t} \) |

---

### 因子 6-10：第3-7个半小时成交量占比

| 项目 | 内容 |
|------|------|
| **所需数据** | 分钟频成交量 \( V_t \)；各半小时时段定义 |
| **定义** | 各半小时时段成交量占全天比例 |
| **公式** | \( \text{Factor}_k = \frac{\sum_{t \in T_k} V_t}{\sum_{t=1}^n V_t} \)，\( k = 3,4,5,6,7 \) |

---

### 因子 11：成交量与成交笔数相关性

| 项目 | 内容 |
|------|------|
| **所需数据** | 分钟频成交量 \( V_t \)、成交笔数 \( N_t \) |
| **定义** | 每分钟成交量与每分钟成交笔数的相关性 |
| **公式** | \( \rho = \frac{ \sum_{t=1}^n (V_t - \bar{V})(N_t - \bar{N}) }{ \sqrt{ \sum_{t=1}^n (V_t - \bar{V})^2 } \sqrt{ \sum_{t=1}^n (N_t - \bar{N})^2 } } \) |

---

### 因子 12：早盘成交量与收益率相关性

| 项目 | 内容 |
|------|------|
| **所需数据** | 分钟频收盘价 \( P_t \)、成交量 \( V_t \)；早盘时段定义（如 9:30–10:30） |
| **定义** | 早盘时段内，每分钟收益率与每分钟成交量的相关性 |
| **公式** | \( r_t = \frac{P_t}{P_{t-1}} - 1 \) <br> \( \rho = \text{corr}(r_t, V_t) \)，\( t \in T_{\text{morning}} \) |

---

### 因子 13：大单推动涨幅（分钟频）

| 项目 | 内容 |
|------|------|
| **所需数据** | 分钟频数据 + 大单标识（由逐笔聚合得到）；基准价（如开盘价） |
| **定义** | 大单主动买入的成交量加权均价相对基准价的涨幅 |
| **公式** | \( P_{\text{big\_buy}} = \frac{\sum_{t \in \text{BigMinutes}} P_t^{\text{avg}} \times V_t}{\sum_{t \in \text{BigMinutes}} V_t} \) <br> \( \text{Factor} = \frac{P_{\text{big\_buy}} - P_{\text{base}}}{P_{\text{base}}} \) |

---

### 因子 14：下行单笔成交金额占比

| 项目 | 内容 |
|------|------|
| **所需数据** | 分钟频收益率 \( r_t \)、逐笔成交金额及方向 |
| **定义** | 下跌时段内主动卖出成交金额占全天总成交金额的比例 |
| **公式** | \( \text{Amt}_{\text{sell\_down}} = \sum_{i \in \text{SellDown}} \text{Amount}_i \) <br> \( \text{Factor} = \frac{\text{Amt}_{\text{sell\_down}}}{\text{Amt}_{\text{total}}} \) |

---

### 因子 15：成交量与振幅相关性

| 项目 | 内容 |
|------|------|
| **所需数据** | 分钟频最高价 \( H_t \)、最低价 \( L_t \)、成交量 \( V_t \) |
| **定义** | 每分钟成交量与每分钟振幅的相关性 |
| **公式** | \( A_t = \frac{H_t - L_t}{L_t} \) <br> \( \rho = \text{corr}(A_t, V_t) \) |

---

## 二、逐笔成交因子（8个）

---

### 因子 1：大单成交金额占比

| 项目 | 内容 |
|------|------|
| **所需数据** | 逐笔成交金额 \( \text{Amount}_i \)、大单阈值 |
| **定义** | 大单成交金额占全天总成交金额的比例 |
| **公式** | \( \text{Amt}_{\text{big}} = \sum_{i} \text{Amount}_i \times \mathbb{I}(\text{Amount}_i \geq \text{threshold}) \) <br> \( \text{Factor} = \frac{\text{Amt}_{\text{big}}}{\text{Amt}_{\text{total}}} \) |

---

### 因子 2：早盘大单买入占比

| 项目 | 内容 |
|------|------|
| **所需数据** | 逐笔成交金额 \( \text{Amount}_i \)、买卖方向、早盘时段、大单阈值 |
| **定义** | 早盘时段内，大单主动买入金额占早盘总成交金额的比例 |
| **公式** | \( \text{Amt}_{\text{buy\_big\_morning}} = \sum_{i} \text{Amount}_i \times \mathbb{I}(\text{side}_i = \text{buy}) \times \mathbb{I}(\text{Amount}_i \geq \text{threshold}) \times \mathbb{I}(t_i \in T_{\text{morning}}) \) <br> \( \text{Factor} = \frac{\text{Amt}_{\text{buy\_big\_morning}}}{\text{Amt}_{\text{morning}}} \) |

---

### 因子 3：大单推动涨幅（逐笔）

| 项目 | 内容 |
|------|------|
| **所需数据** | 逐笔成交价 \( P_i \)、成交量 \( V_i \)、买卖方向、大单阈值、基准价 |
| **定义** | 全天主动买入大单的成交量加权均价相对基准价的涨幅 |
| **公式** | \( P_{\text{big\_buy}} = \frac{\sum_{i \in \text{BuyBig}} P_i \times V_i}{\sum_{i \in \text{BuyBig}} V_i} \) <br> \( \text{Factor} = \frac{P_{\text{big\_buy}} - P_{\text{base}}}{P_{\text{base}}} \) |

---

### 因子 4：早盘主动大单推动涨幅

| 项目 | 内容 |
|------|------|
| **所需数据** | 逐笔成交价 \( P_i \)、成交量 \( V_i \)、买卖方向、早盘时段、大单阈值、开盘价 |
| **定义** | 早盘主动买入大单的成交量加权均价相对开盘价的涨幅 |
| **公式** | \( P_{\text{morning\_big\_buy}} = \frac{\sum_{i \in \text{BuyBigMorning}} P_i \times V_i}{\sum_{i \in \text{BuyBigMorning}} V_i} \) <br> \( \text{Factor} = \frac{P_{\text{morning\_big\_buy}} - P_{\text{open}}}{P_{\text{open}}} \) |

---

### 因子 5：买卖单集中度之差

| 项目 | 内容 |
|------|------|
| **所需数据** | 逐笔成交金额 \( \text{Amount}_i \)、买卖方向、大单阈值 |
| **定义** | 买入大单集中度与卖出大单集中度之差 |
| **公式** | \( \text{Concentration}_{\text{buy}} = \frac{\text{Amt}_{\text{big\_buy}}}{\text{Amt}_{\text{buy}}} \) <br> \( \text{Concentration}_{\text{sell}} = \frac{\text{Amt}_{\text{big\_sell}}}{\text{Amt}_{\text{sell}}} \) <br> \( \text{Factor} = \text{Concentration}_{\text{buy}} - \text{Concentration}_{\text{sell}} \) |

---

### 因子 6：早盘主动买入占比

| 项目 | 内容 |
|------|------|
| **所需数据** | 逐笔成交金额 \( \text{Amount}_i \)、买卖方向、早盘时段 |
| **定义** | 早盘主动买入金额占早盘总成交金额的比例 |
| **公式** | \( \text{Amt}_{\text{buy\_morning}} = \sum_{i \in \text{MorningTrades}} \text{Amount}_i \times \mathbb{I}(\text{side}_i = \text{buy}) \) <br> \( \text{Factor} = \frac{\text{Amt}_{\text{buy\_morning}}}{\text{Amt}_{\text{morning}}} \) |

---

### 因子 7：后一分钟净主动买入与前一分钟收益率相关性

| 项目 | 内容 |
|------|------|
| **所需数据** | 分钟频收盘价 \( P_t \)、逐笔成交金额及方向（用于聚合分钟净主动买入） |
| **定义** | 第 \( t+1 \) 分钟净主动买入与第 \( t \) 分钟收益率的相关系数 |
| **公式** | \( r_t = \frac{P_t}{P_{t-1}} - 1 \) <br> \( \text{NetBuy}_t = \text{BuyAmt}_t - \text{SellAmt}_t \) <br> \( \rho = \text{corr}(r_t, \text{NetBuy}_{t+1}) \) |

---

### 因子 8：前一分钟净主动买入与后一分钟收益率相关性

| 项目 | 内容 |
|------|------|
| **所需数据** | 分钟频收盘价 \( P_t \)、逐笔成交金额及方向（用于聚合分钟净主动买入） |
| **定义** | 第 \( t+1 \) 分钟收益率与第 \( t \) 分钟净主动买入的相关系数 |
| **公式** | \( r_t = \frac{P_t}{P_{t-1}} - 1 \) <br> \( \text{NetBuy}_t = \text{BuyAmt}_t - \text{SellAmt}_t \) <br> \( \rho = \text{corr}(\text{NetBuy}_t, r_{t+1}) \) |

---

## 三、逐笔委托因子（4个）

---

### 因子 1：买卖单委托量偏度之差

| 项目 | 内容 |
|------|------|
| **所需数据** | 逐笔委托量 \( V_i \)（或委托金额）、买卖方向 |
| **定义** | 买入委托量偏度与卖出委托量偏度之差 |
| **公式** | \( \text{Skew}_{\text{buy}} = \frac{ \frac{1}{N_{\text{buy}}} \sum (V_i - \bar{V}_{\text{buy}})^3 }{ \left( \frac{1}{N_{\text{buy}}} \sum (V_i - \bar{V}_{\text{buy}})^2 \right)^{3/2} } \) <br> \( \text{Skew}_{\text{sell}} \) 同理 <br> \( \text{Factor} = \text{Skew}_{\text{buy}} - \text{Skew}_{\text{sell}} \) |

---

### 因子 2：买卖单委托量峰度之差

| 项目 | 内容 |
|------|------|
| **所需数据** | 逐笔委托量 \( V_i \)（或委托金额）、买卖方向 |
| **定义** | 买入委托量峰度与卖出委托量峰度之差 |
| **公式** | \( \text{Kurtosis}_{\text{buy}} = \frac{ \frac{1}{N_{\text{buy}}} \sum (V_i - \bar{V}_{\text{buy}})^4 }{ \left( \frac{1}{N_{\text{buy}}} \sum (V_i - \bar{V}_{\text{buy}})^2 \right)^{2} } - 3 \) <br> \( \text{Kurtosis}_{\text{sell}} \) 同理 <br> \( \text{Factor} = \text{Kurtosis}_{\text{buy}} - \text{Kurtosis}_{\text{sell}} \) |

---

### 因子 3：早盘委托量峰度

| 项目 | 内容 |
|------|------|
| **所需数据** | 逐笔委托量 \( V_i \)（或委托金额）、委托时间、早盘时段定义 |
| **定义** | 早盘时段内委托量分布的峰度 |
| **公式** | \( \text{Kurtosis}_{\text{morning}} = \frac{ \frac{1}{N} \sum_{i=1}^{N} (V_i - \bar{V})^4 }{ \left( \frac{1}{N} \sum_{i=1}^{N} (V_i - \bar{V})^2 \right)^{2} } - 3 \)，\( i \in T_{\text{morning}} \) |

---

### 因子 4：买单委托量与委托价格相关性

| 项目 | 内容 |
|------|------|
| **所需数据** | 逐笔委托量 \( V_i \)、委托价格 \( P_i \)、买卖方向 |
| **定义** | 买入委托中，委托量与委托价格的相关系数 |
| **公式** | \( \rho = \frac{ \sum_{i=1}^{N_{\text{buy}}} (V_i - \bar{V})(P_i - \bar{P}) }{ \sqrt{ \sum_{i=1}^{N_{\text{buy}}} (V_i - \bar{V})^2 } \sqrt{ \sum_{i=1}^{N_{\text{buy}}} (P_i - \bar{P})^2 } } \) |

---

## 四、数据需求汇总

| 数据类型 | 所需字段 |
|----------|----------|
| **分钟频数据** | 开盘价、收盘价、最高价、最低价、成交量、成交笔数 |
| **逐笔成交数据** | 成交价、成交量、成交金额、买卖方向、成交时间 |
| **逐笔委托数据** | 委托量、委托价格、买卖方向、委托时间 |

---

## 五、通用处理说明

| 处理项 | 说明 |
|--------|------|
| **大单阈值** | 建议使用成交金额分位数（如前 10%）动态定义 |
| **时段定义** | 早盘：9:30–10:30；尾盘：14:30–15:00；半小时时段按标准划分 |
| **午休处理** | 剔除 11:30–13:00 的跨分钟配对 |
| **平滑处理** | 单日因子建议取过去 5-20 日均值 |
| **中性化** | 建议对市值和行业进行截面中性化 |
| **异常值** | 建议 Winsorize（1%-99%）处理 |

---

以上是所有 27 个因子的完整复现说明。如需进一步补充（如代码示例、测试方法等），请告知。


