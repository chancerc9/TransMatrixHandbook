# Parameter Optimization

This module is used for parameter optimization of factors or trading strategies.

- What types of parameters and optimization algorithms does this module support?

    The module supports parameter types including **Sequence**, **Box**, **Discrete**, **Category**, and **Bool**, with different optimization algorithms supporting different parameters.

    Implemented optimization algorithms include **Grid Search**, **Random Search**, **Bayesian Search**, **Augmented Random Search (ARS)**, and **Genetic Algorithm (GA)**. Additionally, for more flexible use of optimization algorithms, the module supports **Early Stopping**.

> Tips:
>
> For more detailed descriptions of parameter types and optimization algorithms, please refer to [Parameter Optimization Description (API Documentation)](9_workflow/optim.md).

The following content is explained with two examples:

- Parameter optimization of trading strategies using Grid Search
- Parameter optimization of factor research using Genetic Algorithm

### 6.1 How to Set Up the Objective Function for Parameter Optimization

Like standard factor research and trading strategy modes, you need `strategy.py` and `evaluator.py` to specify the strategy logic and evaluation logic.

Specifically:

- `strategy.py` uses `self.param` to get the parameters to be optimized.
- The `critic` method in `evaluator.py` is the objective function for optimization, which **must** return a numerical value (the direction of parameter optimization is **the larger the objective function value, the better**).

> Tips:
>
> **`self.param` is a dictionary** with the parameter name as the key and the parameter value as the value.
>
> Note that numerical parameters are stored as float types in `self.param`, which can be converted to the correct parameter type using `init()`.

**Trading Strategy:**

Suppose we want to optimize a strategy that uses `mean(macd, window_size)` as the trading signal.

The parameter we need to optimize is `window_size`, and the objective function is the strategy's total PnL. That is, we want to find the optimal `window_size` value to maximize the strategy's returns. In this case, in `strategy.py`, `self.param` is `{'window_size': some_value}`, and in `evaluator.py`, the `critic` method calculates and returns the total strategy return.

Here's the specific implementation:

- strategy.py
  Use `self.param['window_size']` to get the window size for calculating buy signals.

```python
from transmatrix import Strategy

class TestStra(Strategy):
    def init(self):
        self.subscribe_data(
            'macd', ['demo', 'factor_data__stock_cn__tech__1day__macd', self.codes, 'value', 10]
        )
        self.max_pos = 30
    
    def on_market_data_update(self, market):
        window_size = int(self.param['window_size'])
        macd = self.macd.query(self.time, window_size)['value'].mean().sort_values()
        buy_codes = macd.iloc[:2].index

        for code in buy_codes:
            pos = self.account.get_netpos(code)

            if pos < self.max_pos:
                price = market.get('close', code)
                self.buy(
                    price, 
                    volume=10, 
                    offset='open', 
                    code=code, 
                    market='stock'
                )
```

- evaluator.py
    Use the strategy's total PnL as the value of the optimization objective function.

```python
from transmatrix.evaluator.simulation import SimulationEvaluator
import numpy as np

class TestEval(SimulationEvaluator):
    
    def init(self):
        pass

    def critic(self):
        pnl = self.get_pnl()
        return np.nansum(pnl)
```

**Factor Research:**

Suppose we want to optimize the parameter calculation process of a reverse factor that uses two parameters `ret` and `roll`, which represent the time lag size for return calculation and the window size for the rolling mean, respectively.

We need to optimize the parameters `ret` and `roll`, with the objective function being the factor's IC value. That is, we want to find the optimal values for `ret` and `roll` to maximize the factor's IC value. In this case, in `strategy.py`, `self.param` is `{'ret': some_value, 'roll': some_value}`, and in `evaluator.py`, the `critic` method calculates and returns the factor's IC value.

Here's the specific implementation:

- strategy.py
    Use `self.param['ret']` and `self.param['roll']` to get the corresponding parameter values.

```python
from scipy.stats import zscore
from transmatrix.strategy import SignalStrategy
from transmatrix.data_api import create_data_view, NdarrayData, DataView3d

class TestStra(SignalStrategy):
    
    def init(self):
        self.subscribe_data(
            'pv', ['default','stock_bar_1day',self.codes,'open,high,low,close', 10]
        )
        self.add_clock(milestones='09:35:00')

    def pre_transform(self):
        ret_window = int(self.param['ret'])
        roll_window = int(self.param['roll'])

        pv = self.pv.to_dataframe()
        ret = (pv['close'] / pv['close'].shift(ret_window) - 1).fillna(0)
        reverse = -ret.rolling(window=roll_window, min_periods=roll_window).mean().fillna(0)
        reverse = zscore(reverse, axis=1, nan_policy='omit')
        
        self.reverse: DataView3d = create_data_view(NdarrayData.from_dataframes({'reverse':reverse}))
        self.reverse.align_with(self.pv)
            
    def on_clock(self):
        self.update_signal(self.reverse.get('reverse'))
```

- evaluator.py
    Use the factor's IC value as the value of the optimization objective function.

```python
import pandas as pd
from transmatrix.evaluator.signal import SignalEvaluator

class TestEval(SignalEvaluator):
    def init(self):
        self.subscribe_data(
            'pv', ['default','stock_bar_1day',self.codes,'open,high,low,close', 1]
        )
    
    def critic(self):
        critic_data = self.pv.to_dataframe()
        price = critic_data['close']
        factor = self.strategy.signal.to_dataframe()
        ret_1d = price.shift(-1) / price - 1
        ic = vec_rankIC(factor, ret_1d)
        
        return np.abs(ic)
    
def vec_rankIC(factor_panel: pd.DataFrame, ret_panel: pd.DataFrame):
    return factor_panel.T.corrwith(ret_panel.T, method='spearman').mean()
```

### 6.2 How to Configure the YAML File

Like standard factor research and strategy research, you need to configure the `matrix`, `strategy`, and `evaluator` fields in `config.yaml`.

Additionally, you need to configure an **OptimMatrix** field.

> Tips:
>
> For more detailed configuration of OptimMatrix (including the configuration methods and parameter types for other optimization algorithms), please refer to [Parameter Optimization Description (API Documentation)](9_workflow/optim.md).

- Parameter optimization of trading strategies using Grid Search

    The meaning of `window_size:  [[1,5,1], 'Sequence']` is:
    - The name of the parameter to be optimized is `window_size`.
    - The parameter space is set to `[1,5,1]`, and the type of the parameter space is 'Sequence', meaning the parameter space is `np.arange(1, 5, 1)`.
    - Therefore, the candidate values for the parameter are 1, 2, 3, 4.

```yaml
OptimMatrix:
    max_workers: 10
    policy: gridsearch

    params:
        window_size:  [[1,5,1], 'Sequence']
        
matrix:
    mode: simulation
    span: [2021-01-01, 2021-12-31]
    codes: custom_universe.pkl
    ini_cash: 10000000
    market:
        stock:
            data: [default, stock_bar_1day]
            matcher: daily
            account: detail

strategy:
    Trade_stra:
        class: [strategy.py, TestStra]
                
evaluator:
    Trade_eval:
        class: [evaluator.py, TestEval]
```

- Parameter optimization of factor research using Genetic Algorithm

    The meaning of `ret:  [[1,5,1], 'Sequence']` and `roll: [[5,15,1], 'Sequence']` is:
    - The names of the parameters to be optimized are `ret` and `roll`.
    - The parameter space for `ret` is set to `[1,5,1]`, and the type of the parameter space is 'Sequence', meaning the parameter space is `np.arange(1, 5, 1)`.
    - The parameter space for `roll` is set to `[5,15,1]`, and the type of the parameter space is 'Sequence', meaning the parameter space is `np.arange(5, 15, 1)`.
    - Therefore, the candidate values for `ret` are 1, 2, 3, 4, and the candidate values for `roll` are 5, 6, 7, 8, 9, 10, 11, 12, 13, 14.

    > Tips:
    >
    > Except for Grid Search, other parameter optimization algorithms can set algorithm parameters through the `policy_params` field. These values can also be left blank, as there are default values.

```yaml
OptimMatrix:
    max_workers: 10
    policy: GA


    policy_params: 
        seed: 81
        max_iter: 30
        population_size: 10
        tournament_size: 2
        pc: 0.3
        pm: 0.5
        
    params:
        ret:  [[1,5,1], 'Sequence']
        roll: [[5,15,1], 'Sequence']
        
matrix:
    mode: signal
    span: [2021-01-01, 2021-12-30]
    codes: custom_universe.pkl

strategy:
    Factor_stra:
        class: [strategy.py, TestStra]
                
evaluator:
    Factor_eval:
        class: [evaluator.py, TestEval]
```

### 6.3 How to Run

You can run the parameter optimization with just two lines of code:

```python
from transmatrix.workflow.run_optim import run_optim_matrix
result_df = run_optim_matrix('config.yaml')
```

**Result Display:**

- Parameter optimization of trading strategies using Grid Search

![Grid Search Result](TransMatrix使用手册/pics/optim_1.png)

- Parameter optimization of factor research using Genetic Algorithm

![Genetic Algorithm Result](TransMatrix使用手册/pics/optim_2.png)

### 6.4 How to Use Early Stopping

Using Early Stopping is straightforward, just add the Early Stopping configuration to OptimMatrix.

```yaml
OptimMatrix:
    max_workers: 10
    policy: GA
    policy_params: 
        seed: 81
        max_iter: 30
        population_size: 10
        tournament_size: 2
        pc: 0.3
        pm: 0.5
        
    params:
        ret:  [[1,5,1], 'Sequence']
        roll: [[5,15,1], 'Sequence']
        
    earlystopping:
        patience: 3
        delta: 0.0001
        
matrix:
    ...(省略)

strategy:
    ...(省略)
                
evaluator:
    ...(省略)
```

## Summary and Evaluation

This document provides a comprehensive guide to using the parameter optimization module for trading strategies and factor research. It details the types of parameters and optimization algorithms supported, how to set up the objective function, configure the YAML file, and run the optimization process. The module supports various optimization algorithms like Grid Search, Random Search, Bayesian Search, ARS, and Genetic Algorithm, with an option for Early Stopping to enhance flexibility.

The examples illustrate practical implementations for optimizing trading strategies and factor research parameters, making the process clear and accessible. The inclusion of code snippets and YAML configurations ensures that users can follow along and apply the methods to their own projects.

Overall, this guide is well-structured and informative, providing a valuable resource for users looking to optimize their trading strategies and factor research using the parameter optimization module.
