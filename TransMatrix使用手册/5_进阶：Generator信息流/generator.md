# Generator 信息流
Generator 是 TransMatrix 系统的主要开发接口。

它是第二章介绍的策略组件基类 Strategy 和 第三章介绍的因子研究基类 SignalStrategy 的父类，用户也可以直接继承 Generator 类，用来实现因子计算或交易逻辑。关于 Generate 类的详细介绍，参见[这里](3_接口说明/策略/generator.md)。

### 5.1 Generator 使用示例

使用 f1_gen 创造一个非定频发生的交易逻辑：当全市场交易量大于10000时，买入交易量最大的股票，否则不执行任何交易。

```python
# strategy.py
class f1_gen(Generator):
    def init(self):
        self.subscribe_data(
            'pv', ['common2','stock_bar_1day', self.codes,'volume', 1]
        )
        # 添加回测发生时间：随着数据pv的时间戳发生
        self.add_scheduler(with_data='pv', handler=self.compute_f1)
        # 添加消息流的名称
        self.add_message('f1_value')
        
    def compute_f1(self):
        volume = self.pv.get('volume')
        # 传出消息：当全市场交易量大于10000时，传出volume数据
        if np.nansum(volume) >= 10000:
        	self.public('f1_value', volume)
        

class trade(SimulationStrategy):
    def init(self):
        self.subscribe_data(
            'pv', ['common2','stock_bar_1day',self.codes,'close', 0]
        )
        # 订阅f1_gen
        f1 = self.subscribe(f1_gen())
        # 在接收f1_gen的消息f1_value时，执行self.on_f1
        self.callback(f1['f1_value'], self.on_f1)
        
    def on_f1(self, f1_value):
		# 买入交易量最大的股票
       	f1_value = np.where(np.isnan(f1_value), -np.inf, f1_value)
        buy_code = self.codes[np.argmax(f1_value)]
        price = self.pv.get('close', buy_code)
        self.buy(
            price, 
            volume=100, 
            offset='open', 
            code=buy_code, 
            market='stock'
        )
```

```yaml
# config.yaml
matrix:

    mode: simulation
    span: [2022-01-04, 2022-01-10]
    codes: custom_universe.pkl
    market:
        stock:
            data: [meta_data, stock_bar_1day] # 挂载的行情数据
            matcher: daily # 订单撮合模式为daily
            account: detail # 账户类型为detail

strategy:
    trade:
        class:
            - strategy.py
            - trade
```