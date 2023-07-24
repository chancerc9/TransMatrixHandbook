# 参数寻优

本模块是用来对**因子或者交易策略的参数**进行参数寻优的。

- 本模块支持哪些参数类型和优化算法？

    本模块支持的参数类型包括**Sequence**，**Box**，**Discrete**，**Category**和**Bool**，其中不同的优化算法支持的参数有所不同。

    本模块实现的优化算法包括**网格搜索** **gridsearch**，**随机搜索** **randomsearch**，**贝叶斯搜索** **bayessearch**，**强化随机搜索** **ARS**和**遗传算法** **GA**。除此以外，为了使得优化算法的使用更灵活，本模块还支持**早停法 earlystopping**。

> Tips:
>
> 关于参数类型和优化算法更详细的说明，请参考[参数优化说明（API文档版）](。。。)。



接下来的内容以两个样例作为辅助说明：

- 使用网格搜索的交易策略参数优化
- 使用遗传算法的因子研究参数优化

### 6.1 如何设置参数优化的目标函数

与标准的因子研究和交易策略模式一样，同样需要 strategy.py 和 evaluator.py 来指定策略逻辑和评价逻辑。

不同的是，

- strategy.py 中通过 self.param 获得待优化的参数；

- evaluator.py 中的 critic 方法<u>必须</u>返回一个数值，此数值就是目标函数的值，参数优化的方向为<u>目标函数值越大越好</u>。

> Tips:
>
> **self.param 是一个字典**，key 为参数的名字，value 为参数的值。
>
> 需要注意的是，数值型的参数在 self.param 都会被储存为 float 类型，可以通过 int() 获得正确的参数类型。



**交易策略：**

- strategy.py
  通过 self.param['window_size'] 获得计算买入信号的窗口大小。


```python
from transmatrix import Strategy
# 策略逻辑组件
class TestStra(Strategy):
    def init(self):
        # 订阅策略所需数据
        self.subscribe_data(
            'macd', ['demo', 'factor_data__stock_cn__tech__1day__macd', self.codes, 'value', 10]
        )
        self.max_pos = 30
    
    #回调执行逻辑： 行情更新时
    def on_market_data_update(self, market):
        # 通过self.param获取获取参数
        window_size = int(self.param['window_size'])
        # 获取macd值
        macd = self.macd.query(self.time, window_size)['value'].mean().sort_values() 
        # macd值最小的两只股票作为买入股票
        buy_codes = macd.iloc[:2].index 

        for code in buy_codes:
            # 获取某只股票的仓位
            pos = self.account.get_netpos(code)

            if  pos < self.max_pos:
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

    简单地将策略的总pnl作为优化的目标函数值。

```python
from transmatrix.evaluator.simulation import SimulationEvaluator

import numpy as np

class TestEval(SimulationEvaluator):
    
    def init(self):
        pass

    def critic(self):
        # 获得每日损益
        pnl = self.get_pnl()
        
        return np.nansum(pnl)
```



**因子研究：**

- strategy.py

    通过 self.param['ret'] 和 self.param['roll'] 获得相应的窗口大小。

```python
from scipy.stats import zscore
from transmatrix.strategy import SignalStrategy
from transmatrix.data_api import create_data_view, NdarrayData, DataView3d

class TestStra(SignalStrategy):
    
    def init(self):
        # 订阅数据
        self.subscribe_data(
            'pv', ['default','stock_bar_1day',self.codes,'open,high,low,close', 10]
        )
        # 设置回测发生时间
        self.add_clock(milestones='09:35:00') 

    def pre_transform(self):
        # 通过self.param获取获取参数
        ret_window = int(self.param['ret'])
        roll_window = int(self.param['roll'])

        pv = self.pv.to_dataframe()
        ret = (pv['close'] / pv['close'].shift(ret_window) - 1).fillna(0) 
        reverse = -ret.rolling(window = roll_window, min_periods = roll_window).mean().fillna(0)
        reverse = zscore(reverse, axis = 1, nan_policy='omit') # 因子标准化
        
        self.reverse: DataView3d = create_data_view(NdarrayData.from_dataframes({'reverse':reverse})) 
        self.reverse.align_with(self.pv) # 与原始数据对齐
            
    def on_clock(self):
        self.update_signal(self.reverse.get('reverse')) # 定时更新因子数据

```

- evaluator.py

    简单地将因子的 IC 值作为优化的目标函数值。

```python
import pandas as pd
from transmatrix.evaluator.signal import SignalEvaluator

class TestEval(SignalEvaluator):
    def init(self):
        # 订阅数据
        self.subscribe_data(
            'pv', ['default','stock_bar_1day',self.codes,'open,high,low,close', 0]
        )
    
    def critic(self):
        # 将pv转成了一个Dict(field, dataframe)
        critic_data = self.pv.to_dataframe()
        # 计算IC
        price = critic_data['close']
        factor = self.strategy.signal.to_dataframe()
        ret_1d = price.shift(-1) / price - 1
        ic = vec_rankIC(factor, ret_1d)
        
        return np.abs(ic)
    
def vec_rankIC(factor_panel: pd.DataFrame, ret_panel: pd.DataFrame):
    return factor_panel.T.corrwith(ret_panel.T, method = 'spearman').mean()
```



### 6.2 如何配置yaml文件

和标准的因子研究和策略研究类似，同样需要在config.yaml中配置matrix、strategy和evaluator字段。

不同的是，额外需要配置一个**OptimMatrix**字段。

> Tips：
>
> 关于OptimMatrix更详细的配置说明（包括其他优化算法的配置方法和参数类型等），请参考[参数优化说明（API文档版）](path)。



- 使用网格搜索的交易策略参数优化

    下面的 `window_size:  [[1,5,1], 'Sequence']` 的含义是：

    - 待优化的参数的名称为 window_size；
    - 参数空间设置为 [1,5,1]，参数空间的类型是'Sequence'，意思是参数空间为  np.arange(1, 5, 1)；
    - 因此参数的备选值为 1、2、3、4。

```yaml
OptimMatrix:
    max_workers: 10  # 并行运算的worker数量
    policy: gridsearch  # 参数优化方法，可选择gridsearch, randomsearch, bayessearch, GA, ARS

	# 待优化的超参数，以"参数名称: [样本空间, 空间类型]"的形式配置:
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



- 使用遗传算法的因子研究参数优化

    下面的 `ret:  [[1,5,1], 'Sequence']` 和 `roll: [[5,15,1], 'Sequence']` 的含义是：

    - 待优化的参数的名称分别为 ret 和 roll；
    - ret 的参数空间设置为 [1,5,1]，参数空间的类型是 'Sequence'，意思是参数空间为  np.arange(1, 5, 1)；
    - roll 的参数空间设置为 [5,15,1]，参数空间的类型是 'Sequence'，意思是参数空间为  np.arange(5, 15, 1)；
    - 因此 ret 的备选值为 1、2、3、4，roll 的备选值为 5、6、7、8、9、10、11、12、13、14。

    > Tips:
    >
    > 除了网格搜索以外，其他参数优化算法都可以设置算法参数的值，通过 policy_params 字段实现。也可以不填算法参数的值，因为都有默认值。

```yaml
OptimMatrix:
    max_workers: 10  # 并行运算的worker数量
    policy: GA  # 参数优化方法，可选择gridsearch, randomsearch, bayessearch, GA, ARS
    # 优化算法参数设置
    policy_params: 
    	# 不同优化算法都有的公共参数
        seed: 81  # 随机种子
        max_iter: 30  # 最大迭代次数
		
		# 以下为不同优化算法特有的参数    
        ## GA
        population_size: 10  # 种群大小
        tournament_size: 2  # 每次迭代中选取多少个最优个体不经过交叉、变异，直接复制进入下一代
        pc: 0.3  # 交叉概率
        pm: 0.5  # 变异概率
        
	# 待优化的超参数，以"参数名称: [样本空间, 空间类型]"的形式配置:
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



### 6.3 如何运行

通过两行代码就可以将参数寻优运行起来：

```python
from transmatrix.workflow.run_optim import run_optim_matrix
result_df = run_optim_matrix('config.yaml')
```



**结果展示：**

- 使用网格搜索的策略研究参数优化

<div align=center>
<img width="300" src="pics\optim_1.png"/>
</div>
<div align=left style="font-size:12px"></div>

- 使用遗传算法的因子研究参数优化

<div align=center>
<img width="300" src="pics\optim_2.png"/>
</div>
<div align=left style="font-size:12px"></div>



### 6.4 如何使用早停法 Earlystopping

早停法的使用很简单，仅需在OptimMatrix里添加早停法的配置即可。

```yaml
OptimMatrix:
    max_workers: 10  # 并行运算的worker数量
    policy: GA  # 参数优化方法
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
        
	# 早停法，可选
    earlystopping:
        patience: 3  # 不再提升的容忍次数
        delta: 0.0001  # 提升的最小变化量
        
matrix:
    ...(省略)

strategy:
    ...(省略)
                
evaluator:
    ...(省略)
```

