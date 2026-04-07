# 风控方案介绍

Freqtrade 提供了多层次的风险控制体系，涵盖止损机制、保护机制、资金管理和杠杆管理等多个维度，帮助用户在自动化交易中管理风险、保护本金。

---

## 一、止损机制（Stop Loss）

止损是最基本的风险控制手段。Freqtrade 支持以下几种止损模式：

### 1.1 静态止损（Static Stop Loss）

以固定比例设置止损线。例如设置 `-0.10` 表示亏损超过 10% 时自动平仓。

```python
stoploss = -0.10
```

### 1.2 追踪止损（Trailing Stop Loss）

随价格上涨自动上移止损线，锁定利润。

```python
stoploss = -0.10
trailing_stop = True
```

示例：
- 以 100$ 买入，止损设为 -10%，初始止损线为 90$
- 价格上涨至 102$，止损线自动上移至 91.8$
- 价格回落时，止损线不会下移

### 1.3 正利润区间不同追踪止损

达到一定盈利后，切换到更紧的追踪止损比例，进一步保护利润。

```python
stoploss = -0.10
trailing_stop = True
trailing_stop_positive = 0.02
trailing_stop_positive_offset = 0.03
trailing_only_offset_is_reached = True
```

- 未达到 `trailing_stop_positive_offset` 前，使用 `stoploss`（-10%）
- 价格涨过 offset（+3%）后，切换为 -2% 的追踪止损

### 1.4 交易所端止损（Stoploss on Exchange）

将止损订单直接挂在交易所，避免网络延迟带来的滑点风险。

```python
order_types = {
    "stoploss": "market",
    "stoploss_on_exchange": True,
    "stoploss_on_exchange_interval": 60,
    "stoploss_on_exchange_limit_ratio": 0.99
}
```

!!! Note "注意"
    不要设置过小的止损值，否则可能导致交易所订单无法成交。

### 1.5 自定义止损（Custom Stoploss）

通过策略回调函数 `custom_stoploss`，可以根据市场状态、时间、技术指标等动态调整止损。详见[策略回调文档](strategy-callbacks.md#custom-stoploss)。

---

## 二、保护机制（Protections）

保护机制在策略层面提供更高级别的风控，可临时暂停某个交易对或全部交易，以应对异常行情。

!!! Note "回测支持"
    保护机制支持回测，但需要显式传入 `--enable-protections` 参数。

### 2.1 止损守卫（StoplossGuard）

在指定的回溯期内，若触发止损的次数达到阈值，则暂停交易。

**作用范围：** 全局或单交易对  
**适用场景：** 防止在趋势性下跌行情中连续亏损

```python
{
    "method": "StoplossGuard",
    "lookback_period_candles": 24,
    "trade_limit": 4,
    "stop_duration_candles": 4,
    "required_profit": 0.0,
    "only_per_pair": False,
    "only_per_side": False
}
```

| 参数 | 说明 |
|------|------|
| `trade_limit` | 触发止损的次数阈值 |
| `lookback_period_candles` | 回溯周期（K线数量） |
| `stop_duration_candles` | 暂停交易的时长（K线数量） |
| `only_per_pair` | 仅作用于单个交易对 |
| `only_per_side` | 仅作用于某一方向（多/空） |

### 2.2 最大回撤保护（MaxDrawdown）

当账户回撤超过设定阈值时，暂停全部交易。

**作用范围：** 全局  
**适用场景：** 防止在剧烈行情波动中发生大幅亏损

支持两种计算模式：

- `"ratios"`（默认）：基于累计交易盈亏比例计算，向后兼容
- `"equity"`（推荐）：基于账户净值曲线计算标准峰谷回撤，更准确反映实际账户风险

```python
{
    "method": "MaxDrawdown",
    "calculation_mode": "equity",
    "lookback_period_candles": 48,
    "trade_limit": 20,
    "stop_duration_candles": 12,
    "max_allowed_drawdown": 0.2
}
```

| 参数 | 说明 |
|------|------|
| `max_allowed_drawdown` | 最大允许回撤比例（如 0.2 = 20%） |
| `calculation_mode` | 计算模式：`"equity"` 或 `"ratios"` |
| `trade_limit` | 最少需要的交易笔数 |

### 2.3 低盈利交易对锁定（LowProfitPairs）

对在回溯期内盈利低于阈值的交易对进行锁定，防止持续在表现差的交易对上开仓。

**作用范围：** 单交易对  
**适用场景：** 自动排除表现不佳的交易对

```python
{
    "method": "LowProfitPairs",
    "lookback_period_candles": 6,
    "trade_limit": 2,
    "stop_duration": 60,
    "required_profit": 0.02,
    "only_per_side": False
}
```

| 参数 | 说明 |
|------|------|
| `required_profit` | 最低要求盈利比例（如 0.02 = 2%） |
| `trade_limit` | 最少需要的交易笔数 |

### 2.4 冷却期（CooldownPeriod）

在平仓后锁定该交易对一段时间，避免立刻重新开仓。

**作用范围：** 单交易对  
**适用场景：** 防止在短时间内对同一交易对频繁操作

```python
{
    "method": "CooldownPeriod",
    "stop_duration_candles": 2
}
```

### 2.5 综合使用示例

多个保护机制可以叠加使用，形成多层次的风控防线：

```python
@property
def protections(self):
    return [
        {
            "method": "CooldownPeriod",
            "stop_duration_candles": 5
        },
        {
            "method": "MaxDrawdown",
            "calculation_mode": "equity",
            "lookback_period_candles": 48,
            "trade_limit": 20,
            "stop_duration_candles": 4,
            "max_allowed_drawdown": 0.2
        },
        {
            "method": "StoplossGuard",
            "lookback_period_candles": 24,
            "trade_limit": 4,
            "stop_duration_candles": 2,
            "only_per_pair": False
        },
        {
            "method": "LowProfitPairs",
            "lookback_period_candles": 6,
            "trade_limit": 2,
            "stop_duration_candles": 60,
            "required_profit": 0.02
        }
    ]
```

---

## 三、资金管理（Capital Management）

合理的资金管理是风控的基础。Freqtrade 提供了以下配置项：

### 3.1 可用资金比例（tradable_balance_ratio）

设定机器人可动用的资金占总资产的比例，保留一定的安全缓冲。

```json
"tradable_balance_ratio": 0.99
```

### 3.2 单笔仓位大小（stake_amount）

控制每次开仓使用的资金量。可以设为固定金额，也可以设为 `"unlimited"` 让机器人根据可用资金平均分配。

```json
"stake_amount": 100
```

或

```json
"stake_amount": "unlimited",
"max_open_trades": 5
```

使用 `"unlimited"` 时，机器人会将可用资金均分到最多 `max_open_trades` 个仓位。

### 3.3 最大同时持仓数（max_open_trades）

限制同时持有的仓位数量，控制总敞口风险。

```json
"max_open_trades": 5
```

### 3.4 可用本金（available_capital）

在使用共享账户或多机器人场景下，可以指定本机器人可以使用的本金上限。

```json
"available_capital": 1000
```

---

## 四、杠杆风控（Leverage Risk Management）

使用杠杆交易时，风险显著放大，需要额外的保护措施。

!!! Danger "警告"
    杠杆交易风险极高！请确保充分了解杠杆机制后再使用。

### 4.1 清算缓冲（liquidation_buffer）

在强平价格与止损价格之间设置安全缓冲，防止仓位在达到止损前就被强平。

```json
"liquidation_buffer": 0.05
```

计算示例：
- 入场价格：10 USDT/coin
- 强平价格：8 USDT/coin  
- 设置 `liquidation_buffer = 0.05`
- 最低止损价格 = 8 + (10 - 8) × 0.05 = **8.1 USDT**

### 4.2 隔离保证金模式（Isolated Margin）

每个交易对单独使用独立保证金，单个仓位的亏损不会影响其他仓位。

```json
"trading_mode": "futures",
"margin_mode": "isolated"
```

### 4.3 全仓保证金模式（Cross Margin）

所有仓位共享账户余额作为保证金。灵活性高，但风险更大，一个仓位亏损可能影响其他仓位。

```json
"trading_mode": "futures",
"margin_mode": "cross"
```

!!! Warning "注意"
    全仓模式下，单一仓位的大幅亏损可能导致账户整体被强平，需谨慎使用。

### 4.4 动态杠杆倍数

通过策略回调函数 `leverage()` 可以根据交易对、市场状态动态调整杠杆，避免在高波动市场使用过高杠杆。详见[策略回调文档](strategy-callbacks.md#leverage-callback)。

---

## 五、风控方案总结

| 风控层级 | 机制 | 作用范围 | 触发条件 |
|----------|------|----------|----------|
| 单笔交易 | 静态/追踪止损 | 单个仓位 | 价格跌至止损线 |
| 单笔交易 | 交易所端止损 | 单个仓位 | 价格跌至止损线（交易所执行） |
| 交易对级别 | CooldownPeriod | 单个交易对 | 每次平仓后 |
| 交易对级别 | LowProfitPairs | 单个交易对 | 交易对盈利低于阈值 |
| 全局级别 | StoplossGuard | 全部交易对 | 频繁触发止损 |
| 全局级别 | MaxDrawdown | 全部交易对 | 账户回撤超限 |
| 资金管理 | stake_amount / tradable_balance_ratio | 账户 | 持续生效 |
| 杠杆保护 | liquidation_buffer | 单个仓位 | 持仓期间 |

**推荐配置思路：**

1. 根据策略设定合理的止损比例，避免过于宽松或过于紧绷
2. 启用 `StoplossGuard` 防止连续亏损场景
3. 启用 `MaxDrawdown`（推荐使用 `calculation_mode: "equity"`）监控账户整体风险
4. 使用 `CooldownPeriod` 避免在同一交易对上反复追单
5. 合理设置 `max_open_trades` 和 `stake_amount`，分散风险
6. 杠杆交易务必设置 `liquidation_buffer`，推荐使用隔离保证金模式
