# Generator 信息流
Generator 是 TransMatrix 系统的主要开发接口。

它是第二章介绍的策略组件基类 Strategy 和 第三章介绍的因子研究基类 SignalStrategy 的父类，用户也可以直接继承 Generator 类，用来实现因子计算或交易逻辑。关于 Generate 类的详细介绍，参见[这里](3_接口说明/策略/generator.md)。

### 5.1 Generator 使用示例

通常我们交易时所用的 K 线，都是基于某一时间频率切分得到的，即时间采样序列。例如日线是每个交易日对应一个 Bar，10分钟线是每 10 分钟对应一个 Bar。这种按时间采样的方式，往往会在成交低活跃期采样信息过度，在成交高活跃期采样信息不足，并且通常表现出较差的统计特性，如序列相关性、异方差和回报的非正态性。为了解决这一问题，我们可以引入其他采样方式：
-   按成交量进行采样，当阶段成交量达到指定条件时，生成一根 Bar，即 Volume Bar
-   按价格进行采样，每当创新低或新高时，生成一根 Bar，即 Price Bar

使用 Generator，我们可以很轻松实现这一需求。

我们新建一个名为 strategy.py 文件，输入以下代码：


```python
from transmatrix import Generator,Strategy
import numpy as np
  
# 生成 Volume Bar      
class GenVolumeBar(Generator):
    def init(self):
        self.subscribe_data(
            'tick_pv', ['common2','stock_snapshot', self.codes,'volume,ask_price_1', 0]
        )
        self.add_scheduler(with_data='tick_pv', handler=self.on_tick) # 添加调度器：随着数据pv的时间戳调，用 on_tick 方法，这里是每个快照回调一次。
        self.add_message('volume_bar') # 添加信息流的名称
        
    def on_tick(self):
        total_volume = self.tick_pv.get_window('volume', 2)
        if total_volume.shape[0] == 2:
            volume_bar = total_volume[1] - total_volume[0]
            # 广播：当该股票交易量大于1000时，发布数据
            if volume_bar[0] >= 1000:
                self.public('volume_bar', self.tick_pv)
   
# 生成 Price Bar
class GenPriceBar(Generator):
    def init(self):
        self.subscribe_data(
            'tick_pv', ['common2','stock_snapshot', self.codes,'high,low,volume,ask_price_1,bid_price_1', 0]
        )
        self.add_scheduler(with_data='tick_pv', handler=self.on_tick) # 添加调度器：随着数据pv的时间戳调，用 on_tick 方法，这里是每个快照回调一次。
        self.add_message('price_bar')  # 添加信息流的名称
        
    def on_tick(self):
        high = self.tick_pv.get_window('high',2)[0,0]
        low = self.tick_pv.get_window('low',2)[0,0]
        ap = self.tick_pv.get('ask_price_1')[0]
        bp = self.tick_pv.get('bid_price_1')[0]
        if ap < low: # 广播：当该股票的卖一价小于之前的最低价时，发布数据
            self.public('price_bar', [1, ap])
        elif bp > high: # 广播：当该股票的买一价大于之前的最高价时，发布数据
            self.public('price_bar', [-1, bp])
                    
# 交易策略
class Trade(Strategy):
    def init(self):
        f1 = self.subscribe(GenVolumeBar()) # 订阅 GenVolumeBar
        f2 = self.subscribe(GenPriceBar()) # 订阅 GenPriceBar
        
        self.callback(f1['volume_bar'], self.on_volume_bar) # 在接收f1的消息volume_bar时，执行self.on_volume_bar方法
        self.callback(f2['price_bar'], self.on_price_bar) # 在接收f2的消息price_bar时，执行self.on_price_bar方法
        
    def on_volume_bar(self, tick_pv):
   		# 买入股票
        buy_code = self.codes[0]
        price = tick_pv.get('ask_price_1', buy_code)
        if np.isnan(price): return
        self.buy(
            price, 
            volume=100, 
            offset='open', 
            code=buy_code, 
            market='stock'
        )
            
    def on_price_bar(self, info):
        signal, price = info
        if np.isnan(price): return
        buy_code = self.codes[0]
        # 买入股票
        if signal==1:
            self.buy(
                price, 
                volume=100, 
                offset='open', 
                code=buy_code, 
                market='stock'
            )
        # 卖出股票
        elif signal==-1:
            pos = self.get_netpos(buy_code)
            if pos > 0:
                self.sell(
                    price, 
                    volume=pos, 
                    offset='close', 
                    code=buy_code, 
                    market='stock'
                )
```

上述代码里，我们定义了 2 个继承自 Generator 的类：GenVolumeBar 类用于生成 Volume Bar，GenPriceBar 类用于生成 Price Bar。Trade 类用于接收前 2 个类的消息，便据此构建买卖逻辑。针对 GenVolumeBar 类：
-   init 方法中注册了一个名为 volume_bar 的信息流
-   on_tick 方法每个快照都会被回调一次，当前快照的成交量大于 1000 时，广播数据

针对 Trade 类：
-   init 方法订阅了 GenVolumeBar 和 GenPriceBar：
```python
        f1 = self.subscribe(GenVolumeBar()) # 订阅 GenVolumeBar
        f2 = self.subscribe(GenPriceBar()) # 订阅 GenPriceBar
```
    并注册了 2 个回调方法，用于响应广播：
```python
        self.callback(f1['volume_bar'], self.on_volume_bar) # 在接收f1的消息volume_bar时，执行self.on_volume_bar方法
        self.callback(f2['price_bar'], self.on_price_bar) # 在接收f2的消息price_bar时，执行self.on_price_bar方法
```

-  on_volume_bar 和 on_price_bar 方法，获取收到的数据，并构建交易逻辑

我们新建一个 config.yaml 文件，存放本示例的配置信息：

```text
matrix:

    mode: simulation
    span: [2023-01-04, 2023-01-04]
    codes: [000001.SZ]
    market:
        stock:
            data: [meta_data, stock_bar_1day] # 挂载的行情数据
            matcher: daily # 订单撮合模式为 daily
            account: base # 账户类型为 base

strategy:
    trade:
        class:
            - strategy.py
            - Trade
```

新建 run.ipynb 文件，运行代码单元格

```python
from transmatrix.workflow.run_yaml import run_matrix
mat = run_matrix('config.yaml')
```
得到以下输出：
```text
loading data common2__stock_snapshot__volume,ask_price_1 from datacache.
loading data common2__stock_snapshot__high,low,volume,ask_price_1,bid_price_1 from datacache.

loading data meta_data__stock_bar_1day__* from datacache.

loading data meta_data__critic_data__is_halt from datacache.


loading data meta_data__stock_bar_1day__close from datacache.
```

输出交易记录：
```python
strategy = mat.strategies['trade']
strategy.get_trade_table()
```

<div align=center>
<img width="800" src="TransMatrix使用手册/pics/generator.png"/>
</div>
<div align=center style="font-size:12px">交易记录</div>
<br />