# 四、开展策略研究 - original (sidebar)
# Four: Conducting Strategy Research

# (We will translate the following to English.)

# Strategy Research (策略研究)

To quickly get started with the TransMatrix system, let's take a classic futures dual moving average CTA strategy as an example and use this system to implement the entire backtesting process step by step.

The dual moving average strategy is a classic CTA strategy with relatively simple logic:

- If there is no current position, go long when the moving averages cross upwards (golden cross) and go short when the moving averages cross downwards (death cross).
- If there is a short position and the moving averages cross upwards, close the short position first and then go long.
- If there is a long position and the moving averages cross downwards, close the long position first and then go short.

为了快速的上手使用 TransMatrix 系统，我们先以一个经典的期货双均线 CTA 策略为示例，使用本系统，一步步地实现整个回测程序。

双均线策略是一个经典的 CTA 策略，它的逻辑较为简单：
- 当前无仓位，则均线金叉时做多，均线死叉时做空
- 若当前有空头，并且均线金叉，则先平掉空头，再做多
- 若当前有多头，并且均线死叉，则先平掉多头，再做空

The following diagram illustrates the logic (下图是逻辑示例):
<div align=center>
<img width="1000" src="TransMatrix使用手册/pics/2ma.png"/>
</div>
<div align=center style="font-size:12px">双均线策略</div>
<br />

### 2.1 Configuring Backtest Information (配置回测信息)

Before starting to write this strategy, we need to clarify a few basic questions (在着手编写本策略前，我们首先要明确几个基本问题):

1、What time period do I intend to backtest? (我打算回测哪一段时间区间？)

2、What instruments will I backtest? (我要对哪些品种做回测？)

3、What is the trading commission rate? How will my orders be executed? (交易时的手续费率是多少？下单时我的订单会怎么成交？)

4、Where will I write my strategy logic code? (我的策略逻辑代码，写在哪里？)

5、After running the backtest, how will I visualize the backtest results? (运行完回测，怎么对回测结果做可视化？)

The answers to these questions need to be configured as information provided to the TransMatrix system. The system supports two configuration methods (这些问题的答案，需要做成配置信息，提供给 TransMatrix 系统。系统支持2种配置方式):
- Write the configuration information in a YAML file. (将配置信息写在 yaml 文件)
- Write the configuration information in a dictionary (dict) object. (将配置信息写到字典(dict)对象 )

Here, we use a YAML file as an example. We create a new file named `config.yaml` and input the following content:

这里，我们以 yaml 文件为例子，进行说明。我们新建一个文件，命名为 config.yaml，并输入以下内容：

```text
# Configure Matrix component backtest information
matrix:

    mode: simulation # Indicate the current research mode is backtesting
    span: [2021-01-01, 2022-12-31] # What time period do I intend to backtest?
    codes: &universe [I8888.DCE,J8888.DCE,JM8888.DCE,AP8888.ZCE,MA8888.ZCE,FG8888.ZCE] # What instruments will I backtest?
    ini_cash: 10000000 # Initial capital, modify as needed
    fee_rate: 0.000023 # What is the trading commission rate?

    market:
        future: # Market component name
            data: [meta_data, future_bar_10min] # Mounted market data
            matcher: bar # How will my orders be executed?
            account: detail # Account type is detail

# Where will I write my strategy logic code?
strategy:
    CTA: # Component name, modify as needed
        class:
            - strategy.py # Strategy class file name, strategy logic component is in this file
            - DoubleMaStrategy # Strategy class name, strategy logic is written in this class
        kwargs: # Parameters needed by the strategy
            config:
                fast_window: 6
                slow_window: 66
    

# After running the backtest, how will I visualize the backtest results?
evaluator:
    EVAL1: # Component name, modify as needed
        class:
            - evaluator.py # Strategy evaluation class file name, evaluation component is in this file
            - TestEval # Strategy evaluation class name, evaluation logic is written in this class
```

```text
# 配置Matrix组件回测信息
matrix:

    mode: simulation # 标识当前研究模式为回测
    span: [2021-01-01, 2022-12-31] # 我打算回测哪一段时间区间？
    codes: &universe [I8888.DCE,J8888.DCE,JM8888.DCE,AP8888.ZCE,MA8888.ZCE,FG8888.ZCE] # 我要对哪些品种做回测？
    ini_cash: 10000000 # 初始资金，根据需要修改
    fee_rate: 0.000023 # 交易时的手续费率是多少？

    market:
        future: # Market组件名称
            data: [meta_data, future_bar_10min] # 挂载的行情数据
            matcher: bar # 下单时我的订单会怎么成交？
            account: detail # 账户类型为detail

# 我的策略逻辑代码，写在哪里？
strategy:
    CTA: # 组件名称，根据需要修改
        class:
            - strategy.py # 策略类文件名，策略逻辑组件在这个文件
            - DoubleMaStrategy # 策略类名，策略逻辑写在了这个类里
        kwargs: # 策略需要用到的参数
            config:
                fast_window: 6
                slow_window: 66
    

# 运行完回测，怎么对回测结果做可视化？
evaluator:
    EVAL1: # 组件名称，根据需要修改
        class:
            - evaluator.py # 策略评价类文件名，评价组件在这个文件
            - TestEval # 策略评价类名，评价逻辑写在了这个类里
```

Based on the content of this YAML file, we can see that it contains three major categories of configuration information, each corresponding to three types of system components.

- matrix
    - Configures the information needed by the Matrix component, which is the system's backtesting engine.
- strategy
    - Configures the strategy component, where the strategy logic code is written.
- evaluator
    - Configures the evaluation component, which visualizes the backtest results.

As shown in the diagram below, by comparing the directory structure of the current strategy and the contents of the `config.yaml` file, we can clearly see how we configure the backtest information.
<div align=center>
<img width="1000" src="TransMatrix使用手册/pics/sim_dir.png"/>
</div>
<div align=center style="font-size:12px">目录结构</div>
<br />

Now, let's look at the specific contents of these configurations (现在，让我们来分别看这些配置的具体内容).

[根据这个 yaml 文件的内容，我们可以知道，里面有3大类配置信息，它们分别对应系统的3种组件。
- matrix
  - 配置了 Matrix 组件所需要的信息，它是系统的回测引擎
- strategy
  - 配置策略组件，写策略逻辑代码的地方
- evaluator
  - 配置评价组件，回测完怎么对回测结果做可视化

如下图所示，我们把当前策略的目录结构和 config.yaml 文件的内部放在一起作对比，便能很直观地看到我们是怎么配置回测信息的。] - original

#### 2.1.1 Configuring the Matrix Engine (配置 Matrix 引擎)

The Matrix engine is the core component of the system, and its functions include:

- Loading backtest information.
- Loading data subscribed by each component.
- Managing the event flow in the backtesting process.
- Running the backtest.
- Calling the evaluation component to run the evaluation (optional).

For more detailed explanations about the Matrix engine, see [here](3_接口说明/Matrix/matrix.md). To make the strategy program run as we expect, we need to configure the Matrix engine as follows:

```text
matrix:

    mode: simulation # Indicate the current research mode is backtesting
    span: [2021-01-01, 2022-12-31] # What time period do I intend to backtest?
    codes: &universe [I8888.DCE,J8888.DCE,JM8888.DCE,AP8888.ZCE,MA8888.ZCE,FG8888.ZCE] # What instruments will I backtest?
    ini_cash: 10000000 # Initial capital, modify as needed
    fee_rate: 0.000023 # What is the trading commission rate?

    market:
        future: # Market component name
            data: [meta_data, future_bar_10min] # Mounted market data
            matcher: bar # How will my orders be executed?
            account: detail # Account type is detail
```

As shown above, here we set the mode to simulation, indicating the current mode is strategy research. TransMatrix supports two research modes: strategy research and factor research. For the factor research mode, see [Chapter Three](3_接口说明/Matrix/matrix.md). The strategy research mode requires setting the mode to simulation.

Then, configure the backtest period. Here, we choose the backtest start time as January 1, 2021, and the end time as the close of December 31, 2022.

Next is configuring which instruments to backtest. Here, we provide a list that includes the main continuous contracts for iron ore, coke, coal, and other commodities.
```text
    codes: &universe [I8888.DCE,J8888.DCE,JM8888.DCE,AP8888.ZCE,MA8888.ZCE,FG8888.ZCE] # What instruments will I backtest?
```
In addition to using a list as input for the backtest instruments, the system also supports using a pkl file as input, which contains a list object. The configuration in this case is as follows:
```text
    codes: &universe custom_universe.pkl
```

Here, we define a variable `universe`, which holds the list of instruments, allowing us to use the `universe` variable elsewhere in the YAML file to replace the entire list.

Next, we configure the initial capital of the account, set here to 10 million.
```text
    ini_cash: 10000000 # Initial capital, modify as needed
```

Then, configure the trading fee rate, set here to 0.0023%.
```text
    fee_rate: 0.000023 # What is the trading commission rate?
```

Next, nest the market information configuration, which includes the following three parts:

- Subscribed market data
```text
            data: [meta_data, future_bar_10min] # Mounted market data
```

The market data needed by the strategy will trigger the `on_market_update` method of the strategy component when updated. The frequency of the market data determines the execution frequency of `on_market_update`. If the subscribed market data is daily, `on_market_update` will be called every trading day. If the frequency is 10 minutes, the method will be called every 10 minutes, and so on.

Here, we subscribe to the `future_bar_10min` table in the `meta_data` database, which stores the 10-minute Bar market data for futures. For detailed explanations about the database API, see [the Database section](3_接口说明/数据库/DatabaseAPI.md). For detailed explanations about the DataApi, see [the DataApi section](3_接口说明/数据模型/set_model_view.md).

- The Matrix engine's order matcher
```text
            matcher: bar # How will my orders be executed?
```

The matcher informs the Matrix engine how the strategy's trading orders should be matched in the market. Here, the configured matcher is the k-line matcher, with the following matching mechanism:

If the order price for buying (selling) exceeds the upper (lower) price limit, the order is rejected and not matched.
If the order price for buying (selling) exceeds the closing price of the current Bar, the order is fully matched; otherwise, it is adjusted to a pending order and continues to be matched in the next Bar.
If the pending order remains unmatched by the close, it is canceled.
The k-line matcher is a commonly used matcher supported by the system. Additionally, the system supports daily/snapshot and Orderflow matching. Developers can develop their own matchers for specific scenarios by overloading a few methods of the matcher base class `Matcher`. For detailed explanations of matchers, see [here](4_其他组件/market_components.md).

- The account type of the strategy (策略的账户类型)
```text
            account: detail # Account type is detail
```
    Here, the account mode configured is `detail`. The system supports two types of account modes:
    
    - **base** account, does not check for cash or securities, **cash attribute is not maintained.**
    - **detail** account, checks for cash and securities according to market properties, specifically:
      -   股票账户 StockAccount
          -   Trading rules (交易规则):
              -   T+1 trading
              -   Daily settlement (每日结算)
          -   Cash and securities check rules (验资验券规则):
              -   Short selling is not allowed (forbidden to sell short or buy to cover)
              -   **Pending orders do not occupy funds or positions (adjust cash after transaction) **
              -   Same-day sell orders are not allowed
      -   期货账户 FuturesAccount
          -   Trading rules:
              -   Updates margin occupation when a contract is traded or at the end of each day
              - Daily settlement
          -   Cash and securities check rules:
              -   **Pending orders do not occupy funds or positions**
              -   Updates the position and margin occupation when a contract is traded

The default account mode configured by the system is the detail mode. Based on a comprehensive consideration of system performance and business accuracy, we have established the above cash and securities check rules. **Please be aware when using.**

系统的默认配置的账户模式是 detail 模式，基于系统性能和业务准确性的综合考虑，我们制定了上述验资验券规则，**使用时请知悉**。

#### 2.1.2 配置策略组件
配置完 Matrix 引擎所需要的信息后，我们还需要指定我们的策略组件在哪里。
```text
strategy:
    CTA: # 组件名称，根据需要修改
        class:
            - strategy.py # 策略类文件名，策略逻辑组件在这个文件
            - DoubleMaStrategy # 策略类名，策略逻辑写在了这个类里
        kwargs: # 策略需要用到的参数
            config:
                fast_window: 6
                slow_window: 66
```

如上所示，这里相当于告诉系统，我的策略代码写在了同目录下的 strategy.py 文件中，并且策略的类名叫做 DoubleMaStrategy。该策略还需要指定2个参数，一个是快线窗口6，另一个是慢线窗口66。

#### 2.1.3 配置评价组件
接下来，我们需要配置评价组件。
```text
evaluator:
    EVAL1: # 组件名称，根据需要修改
        class:
            - evaluator.py # 策略评价类文件名，评价组件在这个文件
            - TestEval # 策略评价类名，评价逻辑写在了这个类里
```

如上所示，我们告诉系统，我们的评价组件写在当前目录下的 evaluator.py 文件里，并且其类名是 TestEval。当 Matrix 引擎完成回测时，它将调用 TestEval，对回测结果进行评价，并绘图显示。

### 2.2 实现策略组件

如2.1.2节所述，我们在配置 yaml 时，告诉系统，策略写在 strategy.py 文件中，实现策略的类是 DoubleMaStrategy。这里给出 strategy.py 的代码：
```python
import pandas as pd
from transmatrix import Strategy

class DoubleMaStrategy(Strategy):
    
    def __init__(self, config):
        super().__init__(config)
        
    def init(self):
        self.fast_window = self.config.get('fast_window', 20) # 快线窗口
        self.slow_window = self.config.get('slow_window', 60) # 慢线窗口
        
    #回调执行逻辑：行情更新时
    def on_market_data_update(self, data):
        for code in self.codes:
            close = data.get_window(length = self.slow_window + 5, field = 'close', codes = code).flatten()
            
            if len(close) < self.slow_window + 1:
                return
            
            close = pd.Series(close)
            slow_ma = close.rolling(window = self.slow_window).mean().values
            fast_ma = close.rolling(window = self.fast_window).mean().values
            close = close.values

            cross_over = (fast_ma[-1] > slow_ma[-1]) and (fast_ma[-2] < slow_ma[-2]) # 金叉
            corss_down = (fast_ma[-1] < slow_ma[-1]) and (fast_ma[-2] > slow_ma[-2]) # 死叉
            
            pos = self.account.get_netpos(code)
            if pos == 0: # 无仓位
                if cross_over: # 金叉做多
                    self.buy(close[-1], 1, 'open', code, 'stock')
                elif corss_down: # 死叉做空
                    self.sell(close[-1], 1, 'open', code, 'stock')
            elif pos < 0: # 有空头
                if cross_over: # 金叉平空并做多
                    self.buy(close[-1], 1, 'close', code, 'stock')
                    self.buy(close[-1], 1, 'open', code, 'stock')
            elif pos > 0: # 有多头
                if corss_down: # 死叉平多并做空
                    self.sell(close[-1], 1, 'close', code, 'stock')
                    self.sell(close[-1], 1, 'open', code, 'stock')
```

策略继承自 Strategy 类，所有的回测策略，都要继承该类，以实现自身的策略逻辑。关于策略组件基类 Strategy 的详细说明，参见[这里](3_接口说明/策略/strategy.md)。

DoubleMaStrategy 类的 init 方法，会在组件初始化时调用。这里，它从 config 属性中获取了 2 个参数，用于设定快慢均线的计算窗口。这 2 个外部参数配置在 yaml 文件的 strategy 部分，参见 2.1.2 节。

我们重载了 on_market_data_update 方法，该方法在 Matrix 订阅的行情数据有更新时会被执行。这里，由于我们订阅的是日线数据，故每个交易日收盘时，都会执行一次本方法。我们在该方法里实现了 2.1 节所给出的双均线策略逻辑。

> Tips:
>
> 策略组件内置支持了许多回调方法，用户可以重载这些方法，以实现自己的逻辑：
> -   on_market_open: 每日开盘时将回调本方法
> -   on_market_close: 每日收盘时将回调本方法
> -   on_market_data_update: 行情数据有更新时，将回调本方法。若行情数据频率是日频的，则每个交易日会回调本方法；若行情数据是 5 分钟频率的，则每 5 分钟会回调本方法，依此类推
> -   on_trade: 有订单成交时将回调本方法
> -   on_receive: 账户收到订单时将回调本方法
> -   on_order_response: 订单状态改变时(委托/成交/撤单等）将回调本方法
>
> 用户也可以添加调度器，实现自定义的回调需求，调度器的更多说明，参见[这里](TransMatrix使用手册/2_开展策略研究/simulation.md?id=_25-一个股票策略示例)。


### 2.3 实现评价组件

在配置评价组件时，我们告诉系统，评价组件写在了 evaluator.py 文件里。这里给出 evaluator.py 的所有代码：

```python
from transmatrix import Evaluator

import pandas as pd
import numpy as np

class TestEval(Evaluator):
    
    def init(self):
        # 订阅基准数据，评价时需要用到。这里基准是沪深300
        self.subscribe_data(
            'benchmark', ['demo', 'stock_index', '000300.SH', 'close', 0]
        )

    def critic(self):
        ini_cash = self.strategy.ini_cash
        daily_stats = self.get_daily_stats()
        pnl = self.get_pnl()
        
        dt_index = daily_stats.index.get_level_values('datetime').unique().sort_values()
        dt_index = pd.to_datetime(dt_index).date
        perf = pd.DataFrame(index=dt_index)
        
        # 策略收益
        perf['每日损益'] = pnl
        stra_netvalue = (perf['每日损益'].cumsum() + ini_cash) / ini_cash
        perf['策略净值'] = stra_netvalue
        perf['策略收益率'] = (stra_netvalue / stra_netvalue.shift(1) - 1).fillna(0)
        perf['策略累计收益率'] = (1 + perf['策略收益率']).cumprod() - 1

        # benchmark收益
        bench = self.benchmark.to_dataframe()['close']
        bench.index = pd.to_datetime(bench.index).date
        bench = bench.loc[perf.index]
        bench_netvalue = bench / bench.iloc[0]
        perf['基准净值'] = bench_netvalue
        bench_return = (bench / bench.shift(1) - 1).fillna(0)
        perf['基准收益率'] = bench_return
        perf['基准累计收益率'] = (1 + perf['基准收益率']).cumprod() - 1
        
        # 超额收益
        perf['超额收益率'] = perf['策略收益率'] - perf['基准收益率']
        perf['累计超额收益率'] = perf['策略累计收益率'] - perf['基准累计收益率']
        
        self.perf = perf      
        
    def show(self):
        perf = self.perf
        
        self.plot(perf[['策略净值','基准净值']], kind='line', title='策略和基准净值')
        self.plot(perf[['策略收益率','基准收益率']], kind='line', title='策略和基准收益率')
        self.plot(perf[['超额收益率']], kind='line', title='超额收益率')
        self.plot(perf[['策略累计收益率','基准累计收益率','累计超额收益率']], kind='line', title='策略和基准累计收益率', areacol=0)
        self.plot(perf[['每日损益']], kind='bar', title='每日损益', name='损益')
    
        # -----------------绩效指标表格可视化-----------------
        dates = perf.index.astype(str)
        kpis = {}
        kpis['回测时间区间'] = f"{dates[0]} ~ {dates[-1]}"
        # ---------绝对/相对收益---------
        kpis['年化收益'] = np.round(perf['策略收益率'].mean() * 250, 4)
        kpis['基准'] = '沪深300指数'
        kpis['年化超额收益率'] = np.round(perf['超额收益率'].mean() * 250, 4)
        kpis['基准年化收益'] = np.round(perf['基准收益率'].mean() * 250, 4)
        # ---------风险---------
        kpis['年化波动率'] = np.round(perf['策略收益率'].std() * np.sqrt(250), 4)
        
        drawdrown = (perf['策略净值'].cummax() - perf['策略净值']) / perf['策略净值'].cummax()
        kpis['最大回撤'] = np.round(drawdrown.max(), 4)
        
        mask = perf['策略净值'].cummax() > perf['策略净值'].cummax().shift(1).fillna(-np.inf)
        dd_span = pd.Series(np.nan, index=mask.index)
        dd_span[mask.values] = mask.index[mask.values]
        dd_span.ffill(inplace=True)
        loc = drawdrown.argmax()
        d0,d1 = dd_span.iloc[loc],drawdrown.index[loc]
        kpis['最大回撤时间区间'] = f"{d0} ~ {d1}"
        # ---------风险调整收益---------
        stratret = perf['策略收益率'].to_numpy()
        std_d = np.std(stratret[stratret<0]) * np.sqrt(250)
        kpis['夏普率'] = np.round(kpis['年化收益'] / kpis['年化波动率'], 4)
        kpis['索提诺比率'] = np.round(kpis['年化收益'] / std_d, 4)
        kpis['卡玛比率'] = np.round(kpis['年化收益'] / kpis['最大回撤'], 4)
        # ---------策略稳定性---------
        tmp = perf['策略收益率'].rolling(21).apply(lambda x: np.nanprod(1+x) - 1).values
        kpis['持有1个月盈利概率'] = np.round(np.nanmean(np.where(tmp>0,1,0)), 4)
        tmp = perf['策略收益率'].rolling(63).apply(lambda x: np.nanprod(1+x) - 1).values
        kpis['持有3个月盈利概率'] = np.round(np.nanmean(np.where(tmp>0,1,0)), 4)
        tmp = perf['策略收益率'].rolling(126).apply(lambda x: np.nanprod(1+x) - 1).values
        kpis['持有6个月盈利概率'] = np.round(np.nanmean(np.where(tmp>0,1,0)), 4)
        tmp = perf['策略收益率'].rolling(252).apply(lambda x: np.nanprod(1+x) - 1).values
        kpis['持有12个月盈利概率'] = np.round(np.nanmean(np.where(tmp>0,1,0)), 4)
        # ---------结构特征---------
        kpis['Beta'] = np.round(np.cov(perf['策略收益率'],perf['基准收益率'])[0,1] / np.var(perf['基准收益率']), 4)
        kpis['Alpha'] = np.round(kpis['年化收益'] - kpis['Beta'] * kpis['基准年化收益'], 4)
        kpis['收益峰度'] = np.round(perf['策略收益率'].kurt(), 4)
        kpis['收益偏度'] = np.round(perf['策略收益率'].skew(), 4)
        # ---------近端表现---------
        timelength = len(perf['策略收益率'])
        kpis['近1个月收益'] = np.round(perf['策略收益率'].iloc[-min(21,timelength):].mean(), 4)
        kpis['近3个月收益'] = np.round(perf['策略收益率'].iloc[-min(63,timelength):].mean(), 4)

        table = pd.DataFrame()
        nrow = np.ceil(len(kpis) / 2).astype(int)
        kpi_names = list(kpis.keys())
        kpi_values = list(kpis.values())
        table['绩效指标1'] = pd.Series(kpi_names[:nrow])
        table['值1'] = pd.Series(kpi_values[:nrow])
        table['绩效指标2'] = pd.Series(kpi_names[nrow:])
        table['值2'] = pd.Series(kpi_values[nrow:])
        table.replace([np.nan, np.inf, -np.inf], None, inplace=True)

        self.plot(table, kind='table', title='策略绩效指标')
        display(table.round(4))

    def regist(self):
        pass
```

代码文件里，我们定义了一个 TestEval 类，它继承自 Evalutor 基类。所有的回测评价组件，都要继承该基类。TestEval 类重载了基类的4个方法：
- init 方法，该方法在组件初始化时调用，通常用于订阅评价时所需要用到的数据
- critic 方法，用于实现评价的具体逻辑，Matrix引擎 的 eval() 会调用本方法
- show 方法，用于展示评价结果，Matrix 引擎的 show() 会调用本方法
- regist 方法，用于将评价结果注册到 TransQuant 平台策略面板

这里，我们在 init 方法中订阅了demo数据库里的stock_index表的品种，000300.SH，并且取用了表的字段 close，实际对应的数据是沪深300指数的收盘价。我们以它作为基准，在 critic 方法中获取策略每日的pnl，计算了策略收益、基准收益和超额收益等指标，通过 show 方法将这些计算结果绘制展示出来。

关于 评价组件基类 Evaluator 的详细说明，参见[这里](3_接口说明/评价/evaluator.md)。

### 2.4 运行策略程序

我们新建一个 ipynb 文件，命名为 run.ipynb，作为整个回测程序的入口文件。

在该文件中，新建一个代码单元格，输入以下内容：

```python
from transmatrix.workflow import run_matrix
mat = run_matrix('config.yaml')
```

这段代码的意思是告诉系统，运行以 config.yaml 中的配置信息的回测程序。run_matrix 方法背后实现了以下5个步骤：
- 从 config.yaml 中加载回测配置信息
- 参照配置信息，初始化策略组件：DoubleMaStrategy，并添加到 Matrix 引擎
- 参照配置信息，初始化评价组件：TestEval，并添加到 Matrix 引擎
- Matrix 引擎收集各个组件的订阅数据信息，从数据库中读取数据
- Matrix 引擎运行回测，并调用评价组件进行回测结果评价

我们运行该代码单元格，输出以下结果：

<div align=center>
<img width="1000" src="TransMatrix使用手册/pics/sim_future.png"/>
</div>
<div align=center style="font-size:12px">回测结果</div>
<br />

对照 run_matrix 方法背后实现的5个步骤，结合上面的运行结果，便能完整理解整个策略的流程。 

> Tips:
>
> 系统支持 **2 种运行方式**，执行策略回测：
> -   **run_matrix** 方法: 在 py 文件或 jupyter 环境中调用本方法
> -   **Matrix** 命令： 在命令行窗口中输入 Matrix -p [yaml_path] -s [startdate]-[enddate]
>       我们在命令行窗口中输入以下命令： 
> 
>       Matrix -p /root/workspace/我的项目/研究全流程/1.因子计算与入库/config.yaml -s 20210101-20211230 
>
>       系统将运行配置信息文件： "/root/workspace/我的项目/研究全流程/1.因子计算与入库/config.yaml"，并且回测时间段为 2021年1月1日到2021年12月30日

​    


我们再新建一个代码单元格，查看策略所有的历史交易记录。

```python
strategy = mat.strategies['CTA']
strategy.get_trade_table()
```
<div align=center>
<img width="1000" src="TransMatrix使用手册/pics/sim_future_trade.png"/>
</div>
<div align=center style="font-size:12px">交易记录</div>
<br />

这段代码输出了策略历史记录，至此，整个期货 CTA 策略完成。

### 2.5 一个股票策略示例

2.1 节到 2.4 节，我们完整介绍了如何实现一个期货CTA策略。在该期货策略示例中，我们以 config.yaml 作为回测配置信息的存储文件。

2.1 节提到，TransMatrix 系统可支持2种配置方式，一是将配置信息写在 yaml 文件里，二是将配置信息写到字典 (dict) 对象 。这里，我们采用第二种配置方式，一步步地实现一个简单的股票策略。

#### 2.5.1 策略实现
策略逻辑很简单：从自定义的股票样本空间中，每日选出 macd 值最小的2只股票，若股票已有持仓不超过300股，则进行买入，否则跳过。

首先，与前面的期货策略示例一样，我们要先配置回测信息。这次，配置信息写在字典对象里。我们新建一个 ipynb 文件，命名为 stock_strategy.ipynb。

打开该文件，新建一个代码单元格，输入以下代码：

```python
from transmatrix import Matrix
import pandas as pd

# 获取可交易的股票池
CODES = pd.read_pickle('custom_universe.pkl') # 从pkl文件中加载自定义样本空间
config = {   
    'span': ['2021-01-01','2021-12-31'],
    'codes': CODES,
    'ini_cash':10000000,
    'fee_rate': 0.0015 # 手续费率

    'market': {
        'stock':{
            'data': ['demo','stock_bar_1day'],
            'matcher': 'daily', # 日线撮合器
            'account': 'detail',
        },
    },
    
    }

# 实例化 Matrix 引擎
matrix = Matrix(config)
```

如上所示，我们从 pkl 文件中读取了一个股票代码 list，它是本策略的股票样本空间。与上一个期货策略类似，这里同样配置了回测区间，回测品种、账户初始资金、手续费率，以及市场组件信息。其中，撮合器选用了日线撮合器，交易订单都会撮合成交。这些配置信息，存放在字典对象 config 变量中。

我们传入 config 对象，实例化了一个 Matrix 引擎对象。

接下来，我们需要初始化策略组件和评价组件，并添加到 Matrix 引擎。
```python
from strategy import MacdStrategy
from evaluator import TestEval

strategy = MacdStrategy() # 初始化策略组件
analyzer = TestEval() # 初始化评价组件

matrix.add_component(strategy) # 添加到 Matrix 引擎
matrix.add_component(analyzer) # 添加到 Matrix 引擎

```
MacdStrategy 类写在 strategy.py 文件中，它实现了策略逻辑。strategy.py 的代码如下：

```python
from transmatrix import Strategy

# 策略逻辑组件
class MacdStrategy(Strategy):
    def init(self):
        # 订阅策略所需数据
        self.subscribe_data(
            'macd', ['demo', 'factor_data__stock_cn__tech__1day__macd', self.codes, 'value', 10]
        )
        self.subscribe_data(
            'pv60min', ['meta_data', 'stock_bar_60min', self.codes, 'close', 10]
        )
        self.add_scheduler(milestones = ['14:00:00'], handler = self.callback_on_14)
        self.max_pos = 300
    
    # 每天下午14点执行
    def callback_on_14(self):
        macd = self.macd.query(self.time, 3)['value'].mean().sort_values() # 获取最近三天样本空间内的macd值，并排序
        buy_codes = macd.iloc[:2].index # macd值最小的两只股票作为买入股票

        for code in buy_codes:
            # 获取某只股票的仓位
            pos = self.account.get_netpos(code)

            if  pos < self.max_pos:
                price = self.pv60min.get('close', code) # 获取当前时间点的收盘价
                self.buy(
                    price, 
                    volume=100, 
                    offset='open', 
                    code=code, 
                    market='stock'
                )

```
上述代码里，定义了一个MacdStrategy类，它继承自 Strategy 类，并重载了 init 方法。该方法订阅了一个日频的股票 macd 数据，以及60分钟频率的股票行情数据。

另外，init 方法还添加了一个定时调度器 (FixTimeScheduler)，它在每个交易日的下午14点，都会调用 callback_on_14 方法，callback_on_14 实现了策略的具体交易逻辑。

#### 2.5.2 调度器说明
系统内置支持以下调度器，这些调度器可以满足策略各种不同的回调需求：
- 定时调度器：**FixTimeScheduler**，指定一个时间，定时触发回调方法。
```python
    # 指定每个交易日的 14 点，触发用户自定义的 callback_on_14 方法
    self.add_scheduler(milestones = ['14:00:00'], handler = self.callback_on_14)
```
- 定频调度器：**FixFreqScheduler**
  -   需要提供 freq 参数，该参数的支持的格式，可以参照 pandas 的 to_timedelta 方法。
  -   freq 不能大于一小时。

```python
    # 指定每隔 1 小时，触发回调方法 callback
    from transmatrix.common.scheduler import FixFreqScheduler
    scheduler = FixFreqScheduler(freq = '60m') # 回调频率为 60 分钟一次
    scheduler.add_span([self.matrix.start, self.matrix.end])
    self.add_scheduler(scheduler = scheduler, handler = self.callback)
```
- 外部导入调度器：**ExternalTimeScheduler**
```python
    # 外部导入回调时间，触发回调方法 callback
    from transmatrix.common.scheduler import ExternalTimeScheduler
    date_list = [datetime(2023,1, 3, 10), datetime(2023,1, 4, 10), datetime(2023,1, 5, 10)]
    scheduler = ExternalTimeScheduler(external_steps = date_list)
    self.add_scheduler(scheduler = scheduler, handler = self.callback)
```

- 跟随数据调度器：**DataClock**
```python
    # 添加跟随订阅数据 pv_5min 的调度器，调度时间列表与订阅时间一致。这里，每 5 分钟会调用 callback 方法
    self.subscribe_data(
        'pv_5min', ['demo','stock_bar_5min', self.codes, 'open,high,low,close', 0]
    )
    self.add_scheduler(with_data='pv_5min', handler = self.callback)
```

- 周期调度器：**PeriodScheduler**

  按照一定周期触发，支持频率：W: 每周结束， WS: 每周开始， W-MON:周一，W-TUE:周二，W-WED:周三，W-THU:周四，W-FRI:周五，M:每月末，SM:15号或月末，MS:每月初，Q:每季度末，QS:每季度初
```python
    # 每周一的10点回调 callback 方法
    from transmatrix.common.scheduler import PeriodScheduler
    scheduler = PeriodScheduler('W-MON','10:00:00')
    scheduler.add_span(['2023-01-01','2023-05-18'])
    self.add_scheduler(scheduler = scheduler, handler = self.callback)
```

此外，用户也可以重载 调度器基类 BaseScheduler, 以实现自定义的回调规则。此时，需要重载实现 gen_steps 方法，以生成实例的 steps 属性，系统将依照该 steps 列表来执行回调操作。

#### 2.5.3 运行策略
评价组件写在了 evaluator.py 文件中，它的代码如下：
```python
from transmatrix import Evaluator
import pandas as pd
import numpy as np

class TestEval(Evaluator):
    
    def init(self):
        
        self.subscribe_data(
            'benchmark', ['demo', 'stock_index', '000300.SH', 'close', 0]
        )

    def critic(self):
        ini_cash = self.strategy.account.ini_cash
        daily_stats = self.get_daily_stats()
        pnl = self.get_pnl()
        
        dt_index = daily_stats.index.get_level_values('datetime').unique().sort_values()
        dt_index = pd.to_datetime(dt_index).date
        perf = pd.DataFrame(index=dt_index)
        
        # 策略收益
        perf['每日损益'] = pnl
        stra_netvalue = (perf['每日损益'].cumsum() + ini_cash) / ini_cash
        perf['策略净值'] = stra_netvalue
        perf['策略收益率'] = (stra_netvalue / stra_netvalue.shift(1) - 1).fillna(0)
        perf['策略累计收益率'] = (1 + perf['策略收益率']).cumprod() - 1

        # benchmark收益
        bench = self.benchmark.to_dataframe()['close']
        bench.index = pd.to_datetime(bench.index).date
        bench = bench.loc[perf.index]
        bench_netvalue = bench / bench.iloc[0]
        perf['基准净值'] = bench_netvalue
        bench_return = (bench / bench.shift(1) - 1).fillna(0)
        perf['基准收益率'] = bench_return
        perf['基准累计收益率'] = (1 + perf['基准收益率']).cumprod() - 1
        
        # 超额收益
        perf['超额收益率'] = perf['策略收益率'] - perf['基准收益率']
        perf['累计超额收益率'] = perf['策略累计收益率'] - perf['基准累计收益率']
        
        self.perf = perf      
        
    
    def show(self):
        perf = self.perf
        
        self.plot(perf[['策略净值','基准净值']], kind='line', title='策略和基准净值')
        self.plot(perf[['策略收益率','基准收益率','超额收益率']], kind='line', title='策略和基准收益率')
        self.plot(perf[['策略累计收益率','基准累计收益率','累计超额收益率']], kind='line', title='策略和基准累计收益率', areacol=0)
        self.plot(perf[['每日损益']], kind='line', title='每日损益', name='损益')

    def regist(self):
        pass

```
与前面的期货策略的评价组件类似，它也重载了基类的4个方法。其中，init 方法订阅了基准数据，critic 方法实现了回测结果评价逻辑，show 方法用于展示评价结果。

回到 stock_strategy.ipynb 文件，我们新建一个单元格，输入以下内容：
```python
matrix.init() # 初始化 Matrix 引擎
matrix.run() # 运行回测
matrix.eval() # 对回测结果作评价
```

运行本单元格，输出以下内容：
<div align=center>
<img width="1000" src="TransMatrixHandbook/TransMatrix使用手册/pics/sim_stock.png"/>
</div>
<div align=center style="font-size:12px">回测结果</div>
<br />

至此，我们完成了本股票策略示例。

### 2.6 其他策略支持

前文我们以一个期货 CTA 策略和一个股票 CTA 策略为示例，详细介绍了如何使用 TransMatrix 系统开展策略研究。事实上，TransMatrix 支持各种类型的策略回测，包括但不限于：股票 T0、期货做市、ETF 套利、可转债高频等等。系统也提供了不同类型的策略示例，参见[这里](8_测例代码/index.md)。
<div align=center>
<img width="500" src="TransMatrix使用手册/pics/sim_demo.png"/>
</div>
<div align=center style="font-size:12px">回测场景支持</div>
<br />
