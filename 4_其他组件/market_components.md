
<b> 订单(Order) </b> 与 <b> 撮合算法（Matcher）共同构成了 TransMatrix 系统中策略与市场间的信息交互机制</b> 

系统默认支持 日频 / k线 / 快照 与 Orderflow 撮合成交。

此外，开发者可根据特定场景开发定制化的撮合算法。

---

#### Market

市场组件的抽象基类。继承该基类，市场组件分为 StockMarket / FuturesMarket / BatchTradingStockMarket / BatchTradingFuturesMarket。市场组件会订阅市场数据、调用账户组件和撮合器、回放数据，并处理策略组件中的交易逻辑和订单。

**\__init__**

- 参数：
  - data (List): 市场数据的描述
  - matcher (Union[BaseMatcher, str]): 撮合器
  - name (str): 市场组件的名称
  - account (str): 账户

##### Market的数据回放

Market 会更新市场数据并执行回调函数，在配置不同 matcher 时，对 Strategy 执行的回调函数不同：

- matcher 设为 bar 时, Strategy 对应的回调函数为 [on_market_data_update](3_接口说明/策略/strategy.md#回调函数)，其中参数 data 的数据结构是 [DataView3d](3_接口说明/数据模型/set_model_view.md#DataView3d)。

  ```python
  class MyStrategy(Strategy):
      
      def on_market_data_update(self, data: DataView3d):
          close_price = data.get(field='close', codes='000001.SZ')
  ```

- matcher 设为 tick / order 时, Strategy 对应的回调函数为 [on_tick](3_接口说明/策略/strategy.md#回调函数) 和 [on_market_data_update](3_接口说明/策略/strategy.md#回调函数)，其中参数 data 的数据结构：

  - on_market_data_update: Dict[code, [DataViewStruct](3_接口说明/数据模型/set_model_view.md#DataViewStruct)]
  - on_tick: 结构体数组 numpy.void，即 `DataViewStruct.get()` 得到的数据切片

  > 注意，高频回测只支持单标的场景，即即codes中只包含一个标的。

  ```python
  class MyStrategy(Strategy):
      
      def on_market_data_update(self, data: dict):
          ask_price_1 = data.get()['ask_price_1']
      
      def on_tick(self, tick: numpy.void):
          ask_price_1 = tick['ask_price_1']
  ```

---

#### Order

order 对象有以下主要属性：
- price  : float, 委托价格
- volume : int, 委托数量
- direction : ORDER_DIRECTION 买卖方向
- offset : ORDER_OFFSET 开平方向
- code   : str，标的代码
- market : str，市场代码
- strategy : str，策略id
- status: ORDER_STATUS，订单状态，初始化为 SUBMITTING
- id: str 订单id
- insert_time: datetime, 委托指令发出时间
- cancel_time: datetime, 撤单指令发出时间
- latency: 策略与市场的通信延迟
- insert_receive_time: 市场收到委托指令的时间
- cancel_receive_time: 市场收到撤单指令的时间
- pending_volume: int，未成交数量
  

其中 ORDER_DIRECTION / ORDER_OFFSET / ORDER_STATUS 为:

```python
class ORDER_DIRECTION(Enum):
    BUY = 'buy'
    SELL = 'sell'

class ORDER_OFFSET(Enum):
    OPEN = 'open'
    CLOSE = 'close'

class ORDER_STATUS(Enum):
    SUBMITTING = '提交中'
    PENDING = '未成交'
    PARTIAL_TRADE = '部分成交'
    ALL_TRADE = '全部成交'
    CANCELLING = '待撤'
    CANCELED = '已撤'
    REJECTED = '拒单'
```

---

#### Matcher

撮合器的抽象基类。

其本质上是订单在状态空间ORDER_STATUS中的转化函数。

子类必须实现 match_submitting、match_pending 、match_cancelling 与 match_partial_traded 方法。

```python
class Matcher(ABC):

    @abstractmethod
    def match_submitting(self, order:Order) -> MatchResult:
        pass
    
    @abstractmethod
    def match_pending(self, order:Order) -> MatchResult:
        pass
    
    @abstractmethod
    def match_cancelling(self, order:Order) -> MatchResult:
        pass
    
    @abstractmethod
    def match_partial_traded(self, order: Order) -> MatchResult:
        pass

# 其中：
@dataclass
class MatchResult:
    order: Orde
    event: MATCH_EVENT
    price: float = None
    volume: int = None

class MATCH_EVENT(Enum):
    NOTHING = 0        # 状态未变
    INSERT2PEND = 1    # 未成交
    PARTIAL_TRADE = 2  # 部分成交
    PARTIAL_CANCEL = 3 # 撤单时余量部分成交
    ALL_TRADE = 4      # 全部成交
    CANCEL = 5         # 余量全撤
    REJECT = 6         # 拒单

```

##### Bar撮合器

通过配置market的matcher字段为'bar'来使用。

bar撮合逻辑：

- match_submitting: 对于买单，价格超过涨停价时拒单；不触发拒单时，价格大于等于当前bar的close价格时全部成交；若未成交，调整为挂单。对于卖单，价格低于跌停价时拒单；不触发拒单时，价格小于等于当前bar的close价格时全部成交；若未成交，调整为挂单。
- match_pending: 对于买单，价格高于当前bar的low价格时全部成交；否则状态不变。对于卖单，价格低于当前bar的high价格时全部成交；否则状态不变。
- match_cancelling: 直接全部撤单。

```python
class BarMatcher(BaseMatcher):
    
    data_type = 'bar'

    def match_submitting(self, order:Order) -> MatchResult: 
        lim_up = self.limit_up_prices.get(order.code, None)
        lim_down = self.limit_down_prices.get(order.code, None)
        close = self.data.get('close', order.code) # 当前bar的收盘价         
        
        if order.direction == ORDER_DIRECTION.BUY: # 买单
            if order.price > lim_up: # 买单价格超过涨停价
                order.status = ORDER_STATUS.REJECTED # 拒单
                return MatchResult(order = order, event = MATCH_EVENT.REJECT) # 返回拒单事件
            elif order.price >= close: # 买单价格大于等于收盘价
                order.status = ORDER_STATUS.ALL_TRADE # 全部成交
                order.pending_volume = 0 # 挂单量调整为0
                return MatchResult(order = order, event = MATCH_EVENT.ALL_TRADE, price = close, volume = order.volume) # 返回成交事件:全部成交
            
        elif order.direction == ORDER_DIRECTION.SELL: # 卖单
            if order.price < lim_down: # 卖单价格低于跌停价
                order.status = ORDER_STATUS.REJECTED # 拒单
                return MatchResult(order = order, event = MATCH_EVENT.REJECT) # 返回拒单事件
            elif order.price <= close: # 卖单价格小于等于收盘价
                order.status = ORDER_STATUS.ALL_TRADE # 全部成交
                order.pending_volume = 0 # 挂单量调整为0
                return MatchResult(order = order, event = MATCH_EVENT.ALL_TRADE, price = close, volume = order.volume) # 返回成交事件:全部成交

        order.status = ORDER_STATUS.PENDING  # 若未成交，则调整为挂单      
        return MatchResult(order = order, event = MATCH_EVENT.INSERT2PEND, volume = 0)

    def match_pending(self, order:Order) -> MatchResult:  
        high = self.data.get('high', order.code) # 当前bar的最高价
        low = self.data.get('low', order.code) # 当前bar的最低价
        if (order.direction == ORDER_DIRECTION.BUY and order.price > low) \
        or (order.direction == ORDER_DIRECTION.SELL and order.price < high): # 如果买单价格高于最低价，或卖单价格低于最高价
            order.status = ORDER_STATUS.ALL_TRADE # 全部成交
            order.pending_volume = 0 # 挂单量调整为0
            return MatchResult(order = order, event = MATCH_EVENT.ALL_TRADE, price = order.price, volume = order.volume) # 返回成交事件:全部成交
        else:
            return MatchResult(order = order, event = MATCH_EVENT.NOTHING) # 返回成交事件: 状态未变

    def match_cancelling(self, order:Order) -> MatchResult: 
        order.status = ORDER_STATUS.CANCELED # 状态改为撤单
        return MatchResult(order = order, event = MATCH_EVENT.CANCEL) # 返回事件：全部撤单
```

##### Tick撮合器

通过配置market的matcher字段为'tick'来使用，默认使用10档深度。

tick撮合逻辑：

- match_submitting: 
  - 当策略发单并经过延迟（参数可设置）后，市场触发match_submitting，在经过延迟后，策略收到回报。
  - 对于买单：
    - 如果订单价格高于最高限价，或者订单价格低于ask_price_10，拒单。
    - 遍历下一个 tick 数据的对手盘口进行撮合操作（订单价大于等于ask_price_1）：
      - 如果订单委托价格大于等于当前对手价格，则根据该档量价数据进行撮合；
      - 如果订单的待成交数量为零，订单状态改为全部成交，成交价格记录为不同档位价格的成交量加权平均值；
      - 如果对手盘口完毕遍历完毕后订单的待成交数量仍大于0，则作为本方一档插入订单，并将状态设置为`ORDER_STATUS.PARTIAL_TRADE`。
    - 遍历下一个 tick 数据的本方盘口，插入订单（订单价小于ask_price_1）：
      - 如果订单价与该档位价格相等，将订单状态设置为 `ORDER_STATUS.PENDING`，前部挂单量设为该档位的挂单量；
      - 如果订单价大于该档位价格，则将该订单作为new level插入，将订单状态设置为 `ORDER_STATUS.PARTIAL_TRADE`，前部挂单量设为0。
  - 对于卖单：同理。
- match_pending: 
  - 当市场 tick 数据更新时，对于状态为 `ORDER_STATUS.PENDING`的订单触发match_pending，在经过延迟（参数可设置）后，策略收到回报。
  - 对于买单：
    - 撮合市场主卖量。
    - 遍历本tick的本方盘口，根据order的价格档位和前部挂单量进行撮合：
      - 如果订单价低于该档位价格，主卖量消耗掉该档位挂单量；
      - 否则，主卖量消耗掉order的前部挂单量后，再对order的剩余挂单量进行成交；
      - 假如order的剩余挂单量能被全部消耗，则状态改为全部成交；假如仅消耗掉部分，则状态改为 `ORDER_STATUS.PARTIAL_TRADE`，成交价格记录为订单价。
  - 对于卖单，撮合市场主买量，具体逻辑同理。
- match_partial_traded: 
  - 当市场 tick 数据更新时，对于状态为 `ORDER_STATUS.PARTIAL_TRADE`的订单触发match_pending，在经过延迟（参数可设置）后，策略收到回报。
  - 对于买单，撮合市场主卖量，具体逻辑同 match_pending。
  - 对于卖单，撮合市场主买量，具体逻辑同 match_pending。
- match_cancelling: 
  - 当策略撤单并经过延迟（参数可设置）后，市场触发match_cancelling，在经过延迟后，策略收到回报。
  - 对于买单，撮合市场主卖量，具体逻辑同 match_pending，唯一区别是对于未成交的部分直接撤单。
  - 对于卖单，撮合市场主买量，具体逻辑同 match_pending，唯一区别是对于未成交的部分直接撤单。



##### Order撮合器

通过配置market的matcher字段为'order'来使用，默认使用10档深度。

order撮合逻辑：

- match_submitting: 
  - 对于买单：
    - 如果订单价格高于最高限价，或者订单价格低于ask_price_10，拒单。
    - 遍历本tick 数据的对手盘口进行撮合操作（订单价大于等于ask_price_1）：
      - 如果订单委托价格大于等于当前对手价格，则根据该档量价数据进行撮合；
      - 如果订单的待成交数量为零，订单状态改为全部成交，成交价格记录为不同档位价格的成交量加权平均值；
      - 如果对手盘口完毕遍历完毕后订单的待成交数量仍大于0，则作为本方一档插入订单，并将状态设置为`ORDER_STATUS.PARTIAL_TRADE`。
    - 遍历本tick 数据的本方盘口，插入订单（订单价小于ask_price_1）：
      - 如果订单价与该档位价格相等，将订单状态设置为 `ORDER_STATUS.PENDING`，前部挂单量设为该档位的挂单量；
      - 如果订单价大于该档位价格，则将该订单作为new level插入，将订单状态设置为 `ORDER_STATUS.PARTIAL_TRADE`，前部挂单量设为0。
  - 对于卖单：同理。

- match_pending: 
  - 仅当trade和cancel类型的事件发生时，才会对本订单进行撮合。
  - 对于买单：
    - 当cancel事件发生时，若cancel的订单也是买单、订单号比本订单小、价格与本订单价格相等，则更新本订单的状态：
      - 前部挂单量减去cancel的订单量；
      - 若前部挂单量为0，则订单的状态变为`ORDER_STATUS.PARTIAL_TRADE`。
    - 当trade事件发生时，若为主卖：
      - 如果该成交只吃了本方一档，若本订单的价格大于等于成交价，则对本订单进行撮合，同时更新本订单的各种状态；
      - 如果该成交吃了本方多档，则遍历上一个tick的本方盘口，根据order的价格档位和前部挂单量进行撮合：
        - 如果订单价低于该档位价格，主卖量消耗掉该档位挂单量；
        - 否则，主卖量消耗掉order的前部挂单量后，再对order的剩余挂单量进行成交；
        - 假如order的剩余挂单量能被全部消耗，则状态改为全部成交；假如仅消耗掉部分，则状态改为 `ORDER_STATUS.PARTIAL_TRADE`，成交价格记录为订单价。
  - 对于卖单，具体逻辑同理。
- match_partial_traded: 
  - 仅当trade类型的事件发生时，才会对本订单进行撮合。
  - 对于买单：
    - 当trade事件发生时，若为主卖：
      - 如果该成交只吃了本方一档，若本订单的价格大于等于成交价，则对本订单进行撮合，同时更新本订单的各种状态；
      - 如果该成交吃了本方多档，则遍历上一个tick的本方盘口，根据order的价格档位和前部挂单量进行撮合：
        - 如果订单价低于该档位价格，主卖量消耗掉该档位挂单量；
        - 否则，主卖量消耗掉order的前部挂单量后，再对order的剩余挂单量进行成交；
        - 假如order的剩余挂单量能被全部消耗，则状态改为全部成交；假如仅消耗掉部分，则状态改为 `ORDER_STATUS.PARTIAL_TRADE`，成交价格记录为订单价。
  - 对于卖单，具体逻辑同理。
- match_cancelling: 
  - 当前事件不是trade类型，则直接撤单。
  - 当前事件是trade类型：
    - 对于买单，若当前事件为主卖，撮合逻辑同match_partial_traded，唯一区别的是对于未成交的部分，直接撤单。
    - 对于卖单，若当前事件为主买，撮合逻辑同match_partial_traded，唯一区别的是对于未成交的部分，直接撤单。



---