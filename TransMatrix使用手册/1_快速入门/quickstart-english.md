## 2. Quick Start

TransMatrix is a highly customized quantitative investment research framework that can help users perform tasks such as factor research, strategy research, and backtest analysis.

This chapter will briefly introduce how to use TransMatrix to conduct factor research and strategy research to help users get started quickly.

### 1.1 The Whole Process of Factor Research

In TransMatrix, factor research involves writing factor strategy logic, writing factor evaluation logic, data subscription, and configuration of other parameters.

The following is a simple factor study example showing how to use TransMatrix for factor calculation and analysis.

#### strategy.py

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
            NdarrayData.from_dataframes({'ret':ret})
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

In particular,

- First, in the `init` method, subscribe to the stock price data and set the backtest occurrence time;
- In the `pre_transform` method, calculate the rate of return and save it to `self.ret`;
- In the `on_clock` method, obtain the yield data with a window size of 3 based on the current time, and calculate and update the factors.

#### evaluator.py

```python
import pandas as pd
from transmatrix.evaluator.signal import SignalEvaluator

class MySignalEval(SignalEvaluator):
    def init(self):
        # Subscription data
        self.subscribe_data(
            'pv', ['default','stock_bar_1day',self.codes,'open,high,low,close', 1]
        )
    
    def critic(self):
        # Convert `pv` into a `Dict(field, dataframe)`
        critic_data = self.pv.to_dataframe()
        # Calculate `IC`
        price = critic_data['close']
        factor = self.strategy.signal.to_dataframe()
        ret_1d = price.shift(-1) / price - 1
        ic = vec_rankIC(factor, ret_1d)
        print(ic)
        
        return ic
    
def vec_rankIC(factor_panel: pd.DataFrame, ret_panel: pd.DataFrame):
    return factor_panel.T.corrwith(ret_panel.T, method='spearman').mean()
```

In the above code, we wrote a simple factor evaluation logic by inheriting from SignalEvaluator.

First, we also subscribed to data in `init`, and then calculated the IC value in `critic`.

- For details on how to use configuration files to connect and run the entire process of factors, see the next section [3. Conduct Factor Research] (TransMatrix User Manual/3_Carry out factor research/signal.md).

### 1.2 The Whole Process of Trading Strategy Research

Trading strategy research involves writing trading strategy logic, writing strategy evaluation logic, data subscription, and configuration of other parameters.

Below is a simple trading strategy example that demonstrates how to use TransMatrix for trade order placement and profit analysis.

#### strategy.py

```python
from transmatrix import Strategy
# Strategy logic component
class TestStra(Strategy):
    def init(self):
        # Subscribe to data
        self.subscribe_data(
            'macd', ['demo', 'factor_data__stock_cn__tech__1day__macd', self.codes, 'value', 10]
        )
        self.max_pos = 300
    
    # Callback execution logic: when market data updates
    def on_market_data_update(self, market_data):
        # Get and sort the macd values of the most recent three-day sample space
        macd = self.macd.query(time=self.time, periods=3)['value'].mean().sort_values() 
        # The two stocks with the smallest macd values are selected for buying
        buy_codes = macd.iloc[:2].index 

        for code in buy_codes:
            # Get the position of a certain stock
            pos = self.account.get_netpos(code)

            if pos < self.max_pos:
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
- In the `on_market_data_update` method, at the current time (i.e., when market_data is updated), we buy the two stocks with the smallest macd values.

#### evaluator.py

```python
from transmatrix.evaluator.simulation import SimulationEvaluator
import numpy as np

class MySimulationEval(SimulationEvaluator):
    
    def init(self):
        pass

    def critic(self):
        # Get daily profit and loss
        pnl = self.get_pnl()      
        print(pnl)
        
        return np.nansum(pnl)
```

In the above code, we wrote a simple strategy evaluation logic by inheriting from SimulationEvaluator.

First, in the critic method, we obtained the daily profit and loss of the strategy run through self.get_pnl() and summed it to get the total pnl.

- For details on how to use configuration files to connect and run the factor full process, see the next section [2. Conducting Strategy Research](TransMatrix User Manual/2_Conducting_Strategy_Research/simulation.md).

## Summary and Evaluation

The translated section from the TransMatrix User Manual offers a detailed guide on using the framework for quantitative investment research, specifically focusing on factor research and trading strategy research.

### Factor Research

The example provided outlines the process of creating a factor research strategy. It involves:

1. **Data Subscription:** Subscribing to stock price data.
2. **Return Calculation:** Calculating the rate of return.
3. **Signal Update:** Updating factor values based on historical returns.

The evaluation logic involves calculating the Information Coefficient (IC), which is a measure of the predictive power of the factor.

### Trading Strategy Research

The trading strategy example includes:

1. **Data Subscription:** Subscribing to MACD factor data.
2. **Trade Execution Logic:** Implementing logic to buy stocks with the lowest MACD values.

The evaluation logic involves calculating daily profit and loss (PnL) and summing it up to get the total PnL.

### Evaluation

The provided examples and explanations are clear and methodical, making it easy for users to follow and implement their strategies using TransMatrix. By breaking down the process into subscription, calculation, and evaluation phases, the manual ensures a comprehensive understanding of the framework's capabilities. The use of concrete code examples further enhances the learning experience, allowing users to see practical implementations of the theoretical concepts described.

Overall, the translated section is effective in guiding users through the initial setup and execution of factor and trading strategy research using TransMatrix. It provides a solid foundation for further exploration and utilization of the framework's features.
