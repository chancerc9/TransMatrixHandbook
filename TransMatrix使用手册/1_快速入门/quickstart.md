# 二，快速入门 - original
# The following will be translated to English. (From TransQuant User Manuel.)

# 2. Quick Start 

TransMatrix is ​​a highly customized quantitative investment research framework that can help users perform tasks such as factor research, strategy research, and backtest analysis.

This chapter will briefly introduce how to use TransMatrix to conduct factor research and strategy research to help users get started quickly.



### 1.1 The whole process of factor research (因子研究全流程)

In TransMatrix, factor research involves writing factor strategy logic, writing factor evaluation logic, data subscription, and configuration of other parameters.

The following is a simple factor study example showing how to use TransMatrix for factor calculation and analysis.

- strategy.py
    ```python
    # Import in `TransMatrix Strategy` module 
    from transmatrix.strategy import SignalStrategy
    from transmatrix.data_api import create_data_view, NdarrayData, DataView3d, DataView2d
    from scipy.stats import zscore
    
    class Factor(SignalStrategy):
        def init(self):
            # Subscription data test
            self.subscribe_data(
                'pv', ['default','stock_bar_1day',self.codes,'open,high,low,close', 5]
            )
            # Set the time when backtesting occurs
            self.add_clock(milestones='09:25:00')
    
        def pre_transform(self):
            # Convert `pv` into a `Dict(field, dataframe)`
            pv = self.pv.to_dataframe()
    
            # Calculate the rate of return `ret`
            ret = (pv['close'] / pv['close'].shift(1) - 1).fillna(0)
    
            # Save to `self.ret`
            self.ret: DataView3d = create_data_view(
                NdarrayData.from_dataframes({'ret':reverse})
            )
            # Align timestamp with `self.pv`
            self.ret.align_with(self.pv)
    
        def on_clock(self):
            # Calculate factor values
            ret_arr = self.ret.get_window(field='ret', length=3)
            mysignal = np.mean(ret_arr, axis=0)
            # Update factor value
            self.update_signal(mysignal)
    ```
    
    In the above code, we calculate a simple factor and write logic by inheriting SignalStrategy.
    
    in particular,
    
    - First, in the `init` method, subscribe to the stock price data and set the backtest occurrence time;
    - In the `pre_transform` method, calculate the rate of return and save it to `self.ret`;
    - In the `on_clock` method, obtain the yield data with a window size of 3 based on the current time, and calculate and update the factors.
    
      [在上面代码中，我们计算了一个简单的因子，通过继承(继承 - jicheng)SignalStrategy来编写逻辑。
    
    具体而言，
    
    - 首先在`init`方法中，订阅了股价数据，并设置了回测发生时间；
    - 在`pre_transform`方法中，计算收益率，并将其保存到`self.ret`中；
    - 在`on_clock`方法中，根据当前时刻，获取窗口大小为3的收益率数据，并计算和更新因子。] - original
    
- evaluator.py

  ```python
  import pandas as pd
  from transmatrix.evaluator.signal import SignalEvaluator
  
  class MySignalEval(SignalEvaluator):
      def init(self):
          # Subscription data [订阅数据]
          self.subscribe_data(
              'pv', ['default','stock_bar_1day',self.codes,'open,high,low,close', 1]
          )
      
      def critic(self):
          # Convert `pv` into a `Dict(field, dataframe)` [将pv转成了一个Dict(field, dataframe)]
          critic_data = self.pv.to_dataframe()
          # Calculate `IC`
          price = critic_data['close']
          factor = self.strategy.signal.to_dataframe()
          ret_1d = price.shift(-1) / price - 1
          ic = vec_rankIC(factor, ret_1d)
          print(ic)
          
          return ic
      
  def vec_rankIC(factor_panel: pd.DataFrame, ret_panel: pd.DataFrame):
      return factor_panel.T.corrwith(ret_panel.T, method = 'spearman').mean()
  ```


In the above code, we wrote a simple factor evaluation logic by inheriting from SignalEvaluator.

  First, we also subscribed to data in `init`, and then calculated the IC value in `critic`.

- For details on how to use configuration files to connect and run the entire process of factors, see the next section [3. Conduct factor research] (TransMatrix User Manual/3_Carry out factor research/signal.md).
  
在上面代码中，我们编写了一个简单的因子评价逻辑，通过继承SignalEvaluator来实现。

  首先，我们同样在int中订阅数据，然后在critic中计算IC值。

- 关于如何使用配置文件将因子全流程串联和运行起来，详见接下来的[三、开展因子研究](TransMatrix使用手册/3_开展因子研究/signal.md)部分。

Zài shàngmiàn dàimǎ zhōng, wǒmen biānxiěle yīgè jiǎndān de yīnzǐ píngjià luójí, tōngguò jìchéng SignalEvaluator lái shíxiàn.

  Shǒuxiān, wǒmen tóngyàng zài int zhōng dìngyuè shùjù, ránhòu zài critic zhòng jìsuàn IC zhí.

- Guānyú rúhé shǐyòng pèizhì wénjiàn jiāng yīnzǐ quán liúchéng chuànlián hé yùnxíng qǐlái, xiáng jiàn jiē xiàlái de [sān, kāizhǎn yīnzǐ yánjiū](TransMatrix shǐyòng shǒucè/3_kāizhǎn yīnzǐ yánjiū/signal.Md) bùfèn.


### 1.2 The whole process of trading strategy research [交易策略研究全流程]
### 1.2 Trading Strategy Research Full Process

Trading strategy research involves the development of trading strategy logic, strategy evaluation logic, data subscription, and other parameter configurations.

Trading strategy research involves writing trading strategy logic, writing strategy evaluation logic, data subscription, and configuration of other parameters.

交易策略研究涉及交易策略逻辑的编写、策略评价逻辑的编写、数据的订阅，以及其他参数的配置。

Below is a simple trading strategy example that demonstrates how to use TransMatrix for order execution and return analysis.

Below is a simple trading strategy example that shows how to use TransMatrix for trade order placement and profit analysis.

以下是一个简单的交易策略例子，展示了如何使用 TransMatrix 进行交易下单和收益分析。

- strategy.py

  ```python
  from transmatrix import Strategy
  # Strategy logic component (策略逻辑组件)
  class TestStra(Strategy):
      def init(self):
          # Subscribe to data
          self.subscribe_data(
              'macd', ['demo', 'factor_data__stock_cn__tech__1day__macd', self.codes, 'value', 10]
          )
          self.max_pos = 300
      
      # Callback execution logic: when market data updates (回调执行逻辑： 行情更新时)
      def on_market_data_update(self, market_data):
          # Get and sort the macd values of the most recent three-day sample space
          macd = self.macd.query(time=self.time, periods=3)['value'].mean().sort_values() 
          # The two stocks with the smallest macd values are selected for buying (macd值最小的两只股票作为买入股票)
          buy_codes = macd.iloc[:2].index 
  
          for code in buy_codes:
              # Get the position of a certain stock
              pos = self.account.get_netpos(code)
  
              if  pos < self.max_pos:
                  price = market_data.get('close', code)
                  self.buy(
                      price, 
                      volume=100, 
                      offset='open', 
                      code=code, 
                      market='stock'
                  )
  ```
  
  In the above code, we implemented a simple trading strategy by inheriting from Strategy.
  
  Specifically,
  
  - First, in the `init` method, we subscribed to the macd factor data;
  - In the `on_market_data_update method`, at the current time (i.e., when market_data is updated), we buy the two stocks with the smallest macd values;

  在上面代码中，我们实现了一个简单的交易策略，通过继承Strategy来编写逻辑。
  
  具体而言，
  
  - 首先在`init`方法中，订阅了因子macd数据；
  - 在`on_market_data_update`方法中，根据当前时刻（即market_data更新的时刻），对macd值最小的两只股票进行买入操作；
  
- evaluator.py

  ```python
  from transmatrix.evaluator.simulation import SimulationEvaluator
  
  import numpy as np
  
  class MySimulationEval(SimulationEvaluator):
      
      def init(self):
          pass
  
      def critic(self):
          # Get daily profit and loss (获得每日损益)
          pnl = self.get_pnl()      
          print(pnl)
          
          return np.nansum(pnl)
  ```

  In the above code, we wrote a simple strategy evaluation logic by inheriting from SimulationEvaluator.
  
  First, in the critic method, we obtained the daily profit and loss of the strategy run through self.get_pnl() and summed it to get the total pnl.

- For details on how to use configuration files to connect and run the factor full process, see the next section [2. Conducting Strategy Research](TransMatrix User Manual/2_Conducting_Strategy_Research/simulation.md).

  在上面代码中，我们编写了一个简单的策略评价逻辑，通过继承SimulationEvaluator来实现。

  首先，我们在critic方法中通过self.get_pnl()获得策略运行结果的每日损益，并通过加和获得总pnl。

- 关于如何使用配置文件将因子全流程串联和运行起来，详见接下来的[二、开展策略研究](TransMatrix使用手册/2_开展策略研究/simulation.md)。
