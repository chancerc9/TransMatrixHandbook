## [Strategy](8_测例代码\策略服务-回测场景支持\股票日频.md) 


策略编写组件，用户通过继承该组件实现策略逻辑。

Strategy 是 [Generator](3_接口说明/策略/generator.md) 的子类，可实现 [订阅数据](3_接口说明/策略/generator.md#subscribe_data)、[订阅因子](3_接口说明/策略/generator.md#subscribe)、[添加定时器](3_接口说明/策略/generator.md#add_scheduler)、[注册信息流、发布信息、构建自定义回调。](3_接口说明/策略/generator.md#generator-间的信息传递)

此外，Strategy 包含以下接口:

---
### 交易相关

- <b>buy(`price`,`volume`,`offset`,`code`,`market`)</b> : 买入指令， 参数：价格，数量，开平方向，标的代码，市场代码 (在Matrix中配置)
- <b>sell(`price`,`volume`,`offset`,`code`,`market`)</b> : 卖出指令， 参数：价格，数量，开平方向，标的代码，市场代码 (在Matrix中配置)
- <b>cancel([order](4_其他组件/market_components.md#order))</b> : 撤单指令，参数 order 为Order 实例。
- <b> get_netpos(`code`)</b> : 查看净仓位
- **get_closeable_long(`code`)**: 获得当前持有的可平多头仓位
- **get_closeable_short(`code`)**: 获得当前持有的可平空头仓位
- **get_equity(`price_field`)**: 用于获取当前账户总的资产价值（持仓价值+现金），price_field 是用于计算权益的价格字段，默认为 None，将自动从订阅的市场行情数据中匹配。 
- **get_position_value(`price_field`)**: 用于获取当前账户总的持仓市值，price_field 是用于计算权益的价格字段，默认为 None，将自动从订阅的市场行情数据中匹配。 
- **get_equity_code(`code`,`price_field`)**: 用于获取指定标的的相关持仓信息。返回 cur_pnl (标的持仓的当前盈亏)、pnl (账户在该标的的累积盈亏)、hold (标的当前的持仓)、equity (标的当前持仓的权益)。
- <b> pending_orders</b> (属性):  查看当前未成交挂单 Dict[order_id, Order]


---
### 统计指标相关
- <b>get_pnl()</b>: 获取交易盈亏。返回字典: key: 交易日期, value: 账户盈亏。
- <b>get_trade_table()</b>: 获取交易记录，返回 dataframe。
- <b>get_daily_stats()</b>: 获取按 [交易日, 代码] 统计的指标(持仓，累计盈亏，当日盈亏，当日持仓盈亏，当日交易盈亏，结算价，手续费)


---
### 回调函数

用户编写策略逻辑的接口。回调函数可分为系统回调和策略回调两大类。

<b> 系统回调 </b>
- <b> init()</b>  完成数据订阅、因子订阅、自定义回调函数、信息流、定时器的注册。
- <b> on_init()</b>  回测开始前的用户操作
- <b> on_market_open(`market_name`)  </b>  每日开盘时的用户操作（对应 Matrix 配置信息中 market 下的字段名）
- <b> on_market_data_update([data](4_其他组件/market_components.md#Market的数据回放))</b>  市场数据更新时的用户操作
- <b> on_tick([tick](4_其他组件/market_components.md#Market的数据回放))</b>  当matcher配置为'tick'时，tick级市场数据更新时的用户操作

> 关于 on_market_data_update 和 on_tick 的具体说明，见[Market的数据回放](4_其他组件/market_components.md#Market的数据回放)。

- <b> on_market_close(`market_name`) </b>  每日收盘时的用户操作（对应 Matrix 配置信息中 market 下的字段名）
- **on_receive([order](4_其他组件/market_components.md))** 账户收到订单时的用户操作
- <b> on_trade([order](4_其他组件/market_components.md)) </b> 订单成交时的用户操作
- <b>on_order_response([order](4_其他组件/market_components.md))</b>  订单状态改变时(委托/成交/撤单等）触发的用户操作

```python
# def on_trade(self, order: Order):
# def on_receive(self, order: Order):
def on_order_response(self, order: Order):
	print('--'*20)
    print('当前时间', self.time)
    print('委托编号', order.id)
    print('委托时间', order.insert_time)
    print('委托价格', order.price)
    print('委托方向', order.direction)
    print('委托数量', order.volume)
    print('未成交量', order.pending_volume)
    print('委托状态', order.status)
    print('--'*20)
```

<b> 自定义回调 </b>

- 通过 [add_scheduler](3_接口说明/策略/generator.md#add_scheduler) 注册的回调
- 通过 [callback](3_接口说明/策略/generator.md#generator-间的信息传递) 注册的回调

---

## BasketStrategy

批量交易组件

BasketStrategy是Strategy的子类，实现了批量下单的功能，策略默认采用Vwap算法交易撮合成交。此外，Strategy支持的方法，它同样支持。

---
 ### 批量交易

- <b>basket_buy(`basket_orders`,`market`)</b> : 买入指令， 参数：所有股票的买入数量（一维nd.array，与codes属性一一对应），市场代码 (在Matrix中配置)

- <b>basket_sell(`basket_orders`,`market`)</b> : 卖出指令， 参数：所有股票的卖出数量（一维nd.array，与codes属性一一对应），市场代码 (在Matrix中配置)