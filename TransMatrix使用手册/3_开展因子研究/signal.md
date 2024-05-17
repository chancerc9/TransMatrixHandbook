# Factor Research (因子研究) - Important !!

Similar to strategy research, factor research comprises *three* main components:
- **matrix**
  - Configures the *information* required by the *Matrix component*, which serves as the *system's calculation engine*
    (配置了 Matrix 组件所需要的信息，它是系统的计算引擎).
- **strategy**
  - Configures the *factor calculation* component, where the factor calculation logic code is written.
    (配置因子计算组件，写因子计算逻辑代码的地方).
- **evaluator**
  - Configures the *evaluation component*, detailing how to perform *single-factor analysis* on *factor data* and *display* the results.
    (配置评价组件，因子数据如何作单因子分析，并展示结果).

In this chapter, we will use a simple reversal factor as an example to demonstrate how to use the TransMatrix system to accomplish various functionalities of factor research, such as factor calculation, factor evaluation, factor storage, and scheduled factor updates.
  (本章，我们以一个简单的反转因子为示例，介绍如何使用 TransMatrix 系统来实现诸如：因子计算、因子评价、因子入库、因子定时更新等因子研究的各个功能。)

### 3.1 How to Calculate a Factor (如何计算因子)

The logic behind the reversal factor is relatively simple: (1) **first**, calculate the *daily returns* of *individual stocks*, (2) **then** take the *5-day moving average* and [negate] it, and (3) **finally** compute the *z-score in the cross-section*, using the **z-score** as the **factor value**.
(反转因子的逻辑较为简单，首先计算个股的日度收益率，再取5日均值并取负值，最后在截面上计算 zscore，zscore的值作为因子值。)

#### 3.1.1 Configuring Relevant Information (配置相关信息)
To calculate this factor, we need to configure relevant information to clarify the following questions:

1. What time period do I intend to generate the factor for?
2. What stocks are included in the factor's cross-section?
3. Where is the factor data calculation logic written?
4. Where is the factor data saved?
5. After obtaining the factor data, how do I perform single-factor analysis and visualize the results?

Similar to strategy research, these configurations can either be written in a dictionary (dict) object or configured in a YAML file. We will create a new YAML file named `config.yaml` and enter the following content:
```text
matrix:
    mode: signal  # Research mode is factor research
    span: [2018-01-01, 2021-12-30]  # Time period for generating the factor
    codes: '*' 
    universe: [meta_data, critic_data, is_zz500]  # Stocks included in the factor's cross-section
    save_signal: False  # Where to save the factor data

strategy:  # Where to write the factor data calculation logic
    reverse_factor:
        class: 
            - strategy.py 
            - ReverseSignal

```

Since we only need to calculate the factor data and not evaluate it, there is no need to configure the evaluation component.
(由于我们只需要计算因子数据，而不作因子评价，故不需要配置评价组件。)

In the matrix configuration, we set the research mode to `signal`, indicating factor research mode, with a calculation period from January 1, 2018, to December 30, 2021. The sample space for factor calculation is the CSI 500, a dynamic stock pool defined by the `is_zz500` field in the `critic_data` table from the `meta_data` library. The value of this field is either 0 or 1; if it is 1, the stock belongs to the CSI 500 at the current cross-section. For more details on configuring `code` and `universe`, refer to [here](3_接口说明/Matrix/matrix.md).

(在 matrix 部分的配置，我们将研究模式设为 signal，代表是因子研究模式，计算区间是2018年1月1日到2021年12月30日。因子计算的样本空间是中证500，它是一个动态的股票池，由 meta_data 库的 critic_data 表的 is_zz500字段给定，它的取值为0或1，若为1则股票在当前截面属于中证500。关于code 和 universe 的配置的更多说明，参见[这里](3_接口说明/Matrix/matrix.md)。)

In the strategy configuration, the factor calculation logic is written in the `ReverseSignal` class in the `strategy.py` file.
(在 strategy 部分的配置，我们的因子计算逻辑，写在 strategy.py 文件的 ReverseSignal 类中。)

#### 3.1.2 Implementing the Factor Calculation Component (实现因子计算组件)
We will create a new `strategy.py` file and enter the following code:
```python
from transmatrix.strategy import SignalStrategy
from transmatrix.data_api import create_data_view, NdarrayData, DataView3d
from scipy.stats import zscore

class ReverseSignal(SignalStrategy):
    def init(self):
        self.add_clock(milestones='08:30:00')
        self.subscribe_data(
            'pv', ['demo','stock_bar_1day',self.codes,'open,high,low,close', 10]
        )
        self.pv: DataView3d


    def pre_transform(self):
        pv = self.pv.to_dataframe()
        ret = (pv['close'] / pv['close'].shift(1) - 1).fillna(0) # (1) Calculate daily returns 
        reverse = -ret.rolling(window = 5, min_periods = 5).mean().fillna(0) # (2) Take 5-day moving average and negate
        reverse = zscore(reverse, axis = 1, nan_policy='omit') # (3) Calculate z-score in the cross-section [and use the calculated z-score] as the factor value
        
        self.reverse: DataView3d = create_data_view(
            NdarrayData.from_dataframes({'reverse':reverse})
        )
        
        self.reverse.align_with(self.pv)
            
    def on_clock(self):
        self.update_signal(self.reverse.get('reverse'))
```

In the above code, we define a **`ReverseSignal`** class that inherits from **`SignalStrategy`**. **`SignalStrategy`** is the base class for factor calculation in the system, and all factor calculation classes must inherit from it. For more details on **`SignalStrategy`**, refer to [here](5_定制化模块_截面因子开发/signal.md).

The **`ReverseSignal`** class overrides the following three methods:
- **`init`** method: This method is called when initializing the factor calculation component. Here, we add a factor update scheduler set to 8:30, meaning it will trigger the **`on_clock`** method at 8:30 every trading day. We also subscribe to daily stock market data (from the **`demo`** library's **`stock_bar_1day`** table, including fields for open, high, low, and close prices, and the stocks filtered to match the sample space configured in the Matrix engine, i.e., the CSI 500).
- **`pre_transform`** method: This method is called before factor calculation, typically used for vectorized calculations on subscribed data to improve efficiency. Here, we implement the reversal factor calculation logic mentioned at the beginning of this section.
- **`on_clock`** method: This method is called at 8:30 every trading day. Since the **`reverse`** data is already calculated in **`pre_transform`**, we only need to retrieve it and update it in the **`signal`** attribute, which is a container for storing single-factor data.

(上述代码中，我们定义了一个 ReverseSignal 类，它继承自 SignalStrategy。SignalStrategy 是系统因子计算的基类，因子计算类都要继承它。关于 SignalStrategy 的详细说明，参见[这里](5_定制化模块_截面因子开发/signal.md)。)

3.1.3 Executing Factor Calculation (执行因子计算)

We will create a new **`.ipynb`** file named **`run.ipynb`**. In a new code cell, enter the following code:
```python
from transmatrix.workflow.run_yaml import run_matrix
mat = run_matrix('config.yaml')
```
After the system completes the factor data calculation, it will output the following content:
```text
Out:
    loading data meta_data__critic_data__is_zz500 from datacache.

    Updating private cache meta_data__critic_data__is_zz500: dataset_ca21686c-80ce-4e96-bd95-ceb53f03895c ....
    loading data demo__stock_bar_1day__open,high,low,close from datacache.
```
We can check the calculated factor data: (**notice** for ["how to check the calculated factor data"])
```python
strategy = mat.strategies['reverse_factor']
strategy.signal.to_dataframe()
```
<div align=center>
<img width="1000" src="TransMatrix使用手册/pics/signal_factor.png"/>
</div>
<div align=center style="font-size:12px">Factor Data</div>
<br />

Note that the **`signal`** attribute of the **`strategy`** object is a **`DataView2d`**, while the subscribed market data (**`pv`**) of the **`strategy`** object is a **`DataView3d`** type. **`DataView3d`** is a 3D data view providing various data query interfaces for `**Data3d`**; **`DataView2d`** is a 2D data view providing data query interfaces for **`Data2d`**. For more details on **`Data3d`**, **`Data2d`**, **`DataView2d`**, and **`DataView3d`**, refer to [here](3_接口说明/数据模型/set_model_view.md).

(注意，这里的 strategy 对象的 signal 属性，是一个 DataView2d，而 strategy 对象订阅的行情数据 pv，它是一个 DataView3d 类型。DataView3d 是一个 3d 数据视图，针对 Data3d 提供了诸多的数据查询接口；DataView2d 是一个 2d 数据视图，针对 Data2d 提供了数据查询接口。关于 Data3d, Data2d, DataView2d, DataView3d 等类的详细说明，参见[这里](3_接口说明/数据模型/set_model_view.md)。)
```python
print(strategy.signal)
print(strategy.pv)
```
```text
Out:
    <transmatrix.data_api.view.data_view.DataView2d object at 0x7efe40b9a940>
    <transmatrix.data_api.view.data_view.DataView3d object at 0x7efe40b9a940>
```

At this point, we have completed the calculation of a *reversal factor*.

### 3.2 How to Evaluate a Factor (如何评价因子)
Once the factor calculation is complete, a natural question arises: how to perform single-factor analysis to evaluate the factor's effectiveness? *Similar to strategy research, factor research also uses the evaluation component* for single-factor analysis and displays the results. Using the reversal factor from the previous section as an example, we only need to add an *evaluation component* and implement the evaluation logic, and the system will automatically call this component to evaluate the factor data.

We will reuse the `config.yaml` file from the previous section and add a section to specify the evaluation component.

```text
matrix:
    mode: signal  # Research mode is factor research
    span: [2018-01-01, 2021-12-30]  # Time period for generating the factor
    codes: '*'
    universe: [meta_data, critic_data, is_zz500]  # Stocks included in the factor's cross-section
    save_signal: False  # Where to save the factor data

strategy:  # Where to write the factor data calculation logic
    reverse_factor:
        class: 
            - strategy.py 
            - ReverseSignal

evaluator:  # How to perform single-factor analysis and visualize the results
    SimpleEval:
        class:
            - evaluator.py
            - SimpleEvaluator

```

[How-to:] As shown above, the first two parts of the YAML file are the same as in the previous section, with the addition of an evaluation component. The evaluation component is written in the `SimpleEvaluator` class in the `evaluator.py` file. [To-do:] We will create a new Python file named `evaluator.py` and enter the following code:
```python
import numpy as np
from transmatrix.evaluator import SignalEvaluator

class SimpleEvaluator(SignalEvaluator):
    def init(self):
        self.subscribe_data(
            'pv', ['demo','stock_bar_1day',self.codes,'close', 10]
        )
        
    def critic(self):
        close = self.pv.to_dataframe()['close']
        ret_panel = close.shift(-1) / close - 1
        ret_panel.replace([np.inf, -np.inf], np.nan, inplace=True)
        
        factor_panel = self.strategy.signal.to_dataframe()
        ic_ser = factor_panel.corrwith(ret_panel, axis=1, method = 'spearman') # Calculate IC
        self.ic_ser = ic_ser.iloc[:-1]
    
    def show(self):
        self.plot(self.ic_ser, title = 'Rank IC', name='rank_ic')
        
    def regist(self):
        pass
```

`SimpleEvaluator` inherits from the base class `SignalEvaluator`. All factor evaluation components must inherit from this class. Similar to [strategy research], it is also [necessary to override four base class methods to implement factor evaluation].
- `init` method: Mainly used to subscribe to the data required for evaluation. Here, we subscribe to daily stock closing price data.
- `critic` method: Used to implement the factor evaluation calculation logic. Here, we calculate the daily IC value of the factor, which is the rank correlation coefficient between the current factor data and the next trading day's stock returns.
- `show` method: Used to display the factor evaluation results. Here, we plot the daily IC values as a line chart.
- `regist` method: If you want to register the factor to the TransQuant strategy panel, this method needs to be overridden. Here, it is skipped.

For more information on the `SignalEvaluator` base class, refer to [here](5_定制化模块_截面因子开发/signal.md).

We run the cell in run.ipynb again, obtaining the following output:
<div align=center>
<img width="1000" src="TransMatrix使用手册/pics/signal_eval.png"/>
</div>
<div align=center style="font-size:12px">Factor Evaluation</div>
<br />

This completes the evaluation of the reversal factor from section 3.1. For demonstration purposes, we only calculate and display the IC value of the factor. Users can add more logic to implement content such as group analysis, regression analysis, turnover rate analysis, and correlation analysis for single-factor analysis. The system also provides multiple sets of factor evaluation templates, see [Factor Service - Evaluation Templates](8_测例代码/因子服务-评价模板.md).

(以上便完成了 3.1 节的反转因子的评价，出于演示目的，这里仅计算并展示了因子的 IC 值，用户可以添加更多的逻辑，以实现诸如分组分析、回归分析、换手率分析、相关性分析等单因子分析的内容。系统也附带提供了多套因子评价模板，参见[因子服务-评价模板](8_测例代码/因子服务-评价模板.md)。)

### 3.3 Saving Factor Data (保存因子数据)

[If] the factor calculation and evaluation are completed and the [conclusion] is that the factor is an [alpha] factor, we may need to **save the factor data** for *further analysis* or *strategy* construction. In the TransMatrix system, [factor data] is stored in a [personal space], which is a private database *accessible only by the user*. (if eval-res is indeed an alpha factor, then [**to-do**:]) To save factor data, we need to perform the following two steps:
-  [Create] a *new factor data table* to *store the factor data* (Note: This *step* can be *skipped if saving to an existing factor data table*).
```python
from transmatrix.data_api import create_factor_table
temp_table_name = 'temp_factor_table'
create_factor_table(temp_table_name) # Create a new factor data table named temp_factor_table
```
-   Call the `save_signal` method of the `SignalStrategy` to save the factor data to the specified data table.
```python
strategy = mat.strategies['reverse_factor']
strategy.save_signal(temp_table_name) # called (my own comment)
```
```text
Out:
    reverse_factor: saving signal reverse_factor to temp_factor_table1....
    更新完毕: xwq1_private temp_factor_table1 : ['reverse_factor']
```

This saves the reversal factor data calculated in section 3.1 to the `temp_factor_table1` table in the personal space.

[**Another way to save factor data**] is to (if) *[specify the target table name in the configuration information]*, (then) and the *Matrix engine will [automatically] save the calculated factor data*. As shown in the figure below:

<div align=center>
<img width="800" src="TransMatrix使用手册/pics/save_signal.png"/>
</div>
<div align=center style="font-size:12px">Factor Saving</div>
<br />

In the *`matrix`* part of the *YAML file*, [the `save_signal` field is configured with the table name `temp_factor_table1`], (means) *indicating* that we *want to save the factor data to this table*. When we [call] the `run_matrix` method, the *Matrix engine* will perform the *factor data saving operation* (!! Yay). Note that the ***table name** configured here **must already exist** in the personal space database*. [If it does not exist], it can be [created in advance] *using* the *`create_factor_table` method* (see above).

### 3.4 Scheduled Factor Data Updates (定时更新因子数据)

Suppose the calculated factor data passes the factor evaluation test and needs to be incrementally updated daily. [How should this be achieved]? Simply *open the scheduled task module* provided by the system and *configure the scheduling task for updating factor data to achieve this. For more information, see TransQuant's [Scheduled Tasks](http://transquant.gitee.io/transquantproductdoc.github.io/#/?id=_252-%e5%ae%9a%e6%97%b6%e4%bb%bb%e5%8a%a1).

(假设计算得到的因子数据，通过了因子评价检验，需要每日增量更新因子数据，应该如何实现？只需要打开系统提供的定时任务模块，通过配置定时更新因子数据的调度任务，即可实现。更多介绍，参见 TransQuant 的[定时任务](http://transquant.gitee.io/transquantproductdoc.github.io/#/?id=_252-%e5%ae%9a%e6%97%b6%e4%bb%bb%e5%8a%a1)。)

### 3.5 A Complete Stock Multi-Factor Strategy Example (一个完整的股票多因子策略示例) [Finalize, Entire Example, Follow this Implementation !!]

A complete stock multi-factor strategy typically includes the following process:
<div align=center>
<img width="1000" src="TransMatrix使用手册/pics/sig_all.png"/>
</div>
<div align=center style="font-size:12px">Multi-Factor Strategy Research Process(多因子策略研究流程)</div>
<br />

Here, we provide a complete stock multi-factor strategy (多因子策略) example based on this process.

#### 3.5.1 Factor Calculation and Storage (因子计算与入库)

Factor calculation and storage are the first part of the entire example. To distinguish it from the remaining two parts, we will create a new folder named "Factor Calculation and Storage" to store the files for the first part.
(因子计算与入库是整个示例的第 1 部分，为了与剩余的 2 部分区别开，我们这里新建一个名为“因子计算与入库”文件夹，该文件夹用来存放第 1 部分的文件。)

First, we select the following 10 factors and assume that they are all valid alpha factors without additional single-factor tests.
<div align=center>
<img width="500" src="TransMatrix使用手册/pics/10factor.png"/>
</div>
<div align=center style="font-size:12px">10 Factors and Descriptions (10个因子及说明)</div>
<br />

We will create a new `config.yaml` file to store the relevant configuration information.
```text
matrix:

    mode: signal
    span: [2014-1-1, 2020-1-4]
    codes: &universe ../data/codes.pkl # Stock sample space
    save_factors: factor_table # Save factor data to the table factor_table

strategy:
    factors:
        class:
            - strategy.py 
            - factors
```
Here, the sample space used for factor calculation is a custom sample space. Users can use custom sample spaces or dynamic stock pools such as the CSI 300, CSI 500, or the entire market as the sample space. For more information on configuring dynamic stock pools, refer to section 3.1.1.

We will create a new `strategy.py` file and enter the following content:
```python
import numpy as np
from transmatrix.strategy import SignalStrategy
from transmatrix.data_api import create_data_view, NdarrayData, DataView3d

class factors(SignalStrategy):

    def init(self):
        self.subscribe_data(
            'pv', ['common2', 'stock_bar_1day', self.codes, 'close,open,volume,turnover,vwap', 100]
        )
        self.subscribe_data(
            'capital', ['common2', 'capital', self.codes, 'market_cap,turnover_ratio,pb_ratio', 1]
        )
        self.subscribe_data(
            'income', ['common2', 'income', self.codes, 'np_parent_company_owners', '4Q', 'finance-report']
        )
        self.subscribe_data(
            'balance', ['common2', 'balance', self.codes, 'total_assets', '2Q', 'finance-report']
        )
        
        self.add_clock(milestones=['08:30:00'])
        self.create_factor_table([f'factor{i}' for i in range(1,11)] + ['y'])

    def pre_transform(self):
        pv = self.pv.to_dataframe()
        y = pv['vwap'].shift(-2) / pv['vwap'].shift(-1) - 1
        self.y: DataView3d = create_data_view(
            NdarrayData.from_dataframes({'y':y})
        )
        self.y.align_with(self.pv)
        
        # Generate ret
        retdata = pv['close'] / pv['close'].shift(1) - 1
        self.retdata: DataView3d = create_data_view(
            NdarrayData.from_dataframes({'ret': retdata})
        )
        self.retdata.align_with(self.pv)

    def on_clock(self):
        self.update_factor('y', self.y.get('y'))
        
        # factor1: ret20
        close = self.pv.get_window('vwap', 20)
        factor1 = close[-1] / close[0] - 1
        self.update_factor('factor1', factor1)

        # factor2: ret60
        close = self.pv.get_window('vwap', 60)
        factor2 = close[-1] / close[0] - 1
        self.update_factor('factor2', factor2)

        # factor3: std20
        ret = self.retdata.get_window('ret', 20)
        factor3 =  np.nanstd(ret, axis=0)
        self.update_factor('factor3', factor3)
        
        # factor4: turnover20
        turnover_ratio = self.capital.get_window('turnover_ratio', 20)
        factor4 =  np.nanmean(turnover_ratio, axis=0)
        self.update_factor('factor4', factor4)
        
        # factor5: tvstd20        
        turnover = self.pv.get_window('turnover', 20)
        factor5 = np.nanstd(turnover, axis=0)
        self.update_factor('factor5', factor5)
        
        # factor6: maxret20 过去20交易日最大3个日收益率均值
        ret = self.retdata.get_window('ret', 20)
        factor6 = np.nanmean(np.sort(ret, axis=0)[-3:], axis=0)
        self.update_factor('factor6', factor6)
        
        # factor7: bp
        pb = self.capital.get('pb_ratio')
        pb = np.where(pb == 0, np.nan, pb)
        factor7 = 1/pb
        self.update_factor('factor7', factor7)

        # factor8: ep_ttm
        ep = self.income.get_fields([f'np_parent_company_owners_lag{i}Q' for i in range(0,4)])
        ep = np.nansum(ep, axis=0)
        cap = self.capital.get('market_cap') * 1e8
        factor8 = np.divide(ep, cap)
        self.update_factor('factor8', factor8)
        
        # factor9: roe_qfa
        ep = self.income.get('np_parent_company_owners_lag0Q')
        assets0,assets1 = self.balance.get_fields(['total_assets_lag0Q', 'total_assets_lag1Q'])
        assets = (assets0 + assets1) / 2
        factor9 = np.divide(ep, assets)
        self.update_factor('factor9', factor9)
        
        # factor10: yoy_profit_qfa 单季度归属母公司股东的净利润增长率
        ep0,ep1 = self.income.get_fields(['np_parent_company_owners_lag0Q', 'np_parent_company_owners_lag1Q'])
        factor10 = np.divide(ep0 - ep1, ep1)
        self.update_factor('factor10', factor10)
        
        
    def post_transform(self):
        # Remove inf
        factor_data = {}
        for k,v in self.factor_data.items():
            v = v.to_dataframe().replace([np.inf,-np.inf], np.nan)
            factor_data[k] = v
            
        self.factor_data = factor_data
```

In the above code, we define a class `factors`, which inherits from the `SignalStrategy` base class. Here, since we need to generate 10 factors, in the `init` method, apart from subscribing to the required data, we also call the `create_factor_table` method to register a dictionary for storing the data of these 10 factors.
(上述代码里，我们定义了一个 factors 类，它继承自 SignalStrategy 基类。注意到这里，由于需要生成 10 个因子，因此在 init 方法中，我们除了订阅所需要的数据之外，还要调用 create_factor_table 方法，用于注册一个字典，用于存放这 10 个因子的数据。)
``` python
        self.create_factor_table([f'factor{i}' for i in range(1,11)] + ['y'])
```
This code registers the data of the 10 factors from `factor1`, `factor2`,..., to `factor10`, and for the *ease of subsequent model training*, we *include the next trading day's returns as the label `y`*, also *stored* in the dictionary container.

We can access this factor data dictionary object through the `factor_data` attribute.

[Comparing with section 3.1] *and* [this section], [**we can conclude**]:
- A factor calculation component (inherited from `SignalStrategy`) can generate **one** factor:
  - Factor data is obtained through the **signal** attribute (section 3.1);
  - When saving factor data, the **save_signal** method is called.
- A factor calculation component can generate **multiple** factors:
  - In the `init` method, the **create_factor_table** method is called to register these factors, and factor data is obtained through the **factor_data** attribute ([*this* section]);
- When saving factor data, the **save_factors** method is called.

The configuration for saving factor data in this example is specified in the `config.yaml` file:
```text
    save_factors: factor_table # Save factor data to the table factor_table
```
The logic of factor calculation is in the `on_clock` method, which can be understood by referring to the table **10 Factors and Descriptions** while reading this part of the code; no additional explanation is provided here.

We create a new `ipynb` file named `run.ipynb` and input the following content in a code cell:
```python
from transmatrix.data_api import create_factor_table
from transmatrix.workflow import run_matrix
create_factor_table('factor_table')
mat = run_matrix('config.yaml')
```
Running this cell will create a factor table named `factor_table` in the personal space, generate data for 10 factors, and save it to this table, completing the first part of factor generation.
运行本单元格，系统将在个人空间新建一张名为 factor_table 的因子表，生成 10 个因子数据，并保存到该表，第一部分的因子生成结束。

#### 3.5.2 Model Training and Signal Evaluation (模型训练与信号评价)

With the data of these 10 factors, we move on to the second part, model training and signal evaluation.
- Read factor data, split samples into training and testing sets, and train the model using the lightGBM algorithm. 
  (读取因子数据，分割样本为训练集和测试集，采用 lightGBM 算法训练模型)

We create a new `ipynb` file named `a_model_training.ipynb`. To read the factor data, we input the following code in a code cell:
```python
from sklearn.linear_model import Lasso, Ridge
import lightgbm as lgb
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import MinMaxScaler

import pandas as pd
import numpy as np
import pickle,os

from transmatrix.data_api import Database
from transmatrix.setting import PRIVATE_DB_NAME
db = Database(PRIVATE_DB_NAME)
factor_data = db.query('factor_table', 
                start='2014-1-1', end='2020-1-1').drop(['seqnum'], axis=1).sort_values('datetime')

factor_data['date'] = pd.to_datetime(factor_data['datetime']).dt.date
factor_data['date'] = pd.to_datetime(factor_data['date'])
factor_data.set_index(['date'], inplace=True)
factor_data.drop(['datetime'], axis=1, inplace=True)

print('Data loaded successfully - 数据完成加载')
display(factor_data)
```
<div align=center>
<img width="1000" src="TransMatrix使用手册/pics/signal_whole_1.png"/>
</div>
<div align=center style="font-size:12px">Reading Factor Data (读取因子数据)</div>
<br />

Next, we proceed to split the samples into training and testing sets. We create a code cell and input the following content:
```
# Define training and testing periods

train_start_year = '2014-1-1'
test_start_year = '2017-1-1'
test_end_year = '2020-1-1'

print(train_start_year, test_start_year, test_end_year) # Split samples into training and testing sets

# Select factors
factor_columns = [f'factor{i}' for i in range(1,11)]
with open('../data/factor_columns.pkl', 'wb') as f:
    pickle.dump(factor_columns, f)
print(factor_columns)
```
```text
Out:
    2014-1-1 2017-1-1 2020-1-1
    ['factor1', 'factor2', 'factor3', 'factor4', 'factor5', 'factor6', 'factor7', 'factor8', 'factor9', 'factor10']
```

Finally, we proceed with model training.
```python
from tools import get_train_valid_test_data, process_nan
models = {}
longshort_df = pd.DataFrame()

# Define the training and testing process
train_x, train_y, test_x, test_y = get_train_valid_test_data(train_start_year, test_start_year, test_end_year, factor_data, factor_columns)

# Handle missing values
_,train_x1,train_y1 = process_nan(train_x, train_y)
mask_test,test_x1,test_y1 = process_nan(test_x, test_y)

train_x1 = train_x1.drop(['code'],axis=1).values
train_y1 = train_y1.values
test_x1 = test_x1.drop(['code'],axis=1).values
test_y1 = test_y1.values

# Build pipeline
scaler = MinMaxScaler(feature_range=(-1, 1))
model = lgb.LGBMRegressor(n_estimators=50, 
                        learning_rate=0.01, 
                        max_depth=5, 
                        num_leaves=10, 
                        min_child_samples=20, 
                        subsample=0.8, 
                        colsample_bytree=0.8, 
                        reg_alpha=0.1, 
                        reg_lambda=0.1,
                        random_state=42)

pipeline = Pipeline([
    ('scaler',scaler),  # Data scaling
    ('model',model)  # Model training
    ])
pipeline.fit(train_x1, train_y1)

with open('model.pkl', 'wb') as f:
    pickle.dump(pipeline, f)

test_y_pred = pipeline.predict(test_x1)
test_loss = np.mean((test_y_pred - test_y1)**2)
print('test_loss:', test_loss)
```
```text
Out:
    test_loss: 0.0006311274019344903
```

At this point, we have completed the model training and saved the model locally. For demonstration purposes, the model training is relatively simple and does not include parameter optimization. Users can perform more complex optimization adjustments during model training.

-  Using the model to generate signals and evaluate the effectiveness of the signals (使用模型生成信号，评价信号的有效性)

This part includes two steps: the first is signal generation, similar to factor calculation, where we load factor data, use the model to predict signals, and save them to the database; the second is signal evaluation, where we add an evaluation component and implement the evaluation logic to evaluate the signals. Considering the length of the evaluation component code, it is not introduced here. You can refer to the system's sample template "Research Full Process" for more details. Below is the result of the signal evaluation:
(这部分包含 2 个步骤，1 是信号生成，这与因子计算类似，我们加载因子数据，使用模型预测得到信号并保存到数据库里；2 是信号评价，添加评价组件并实现评价逻辑，便可以对信号进行评价。考虑到评价组件的代码篇幅较长，这里不作介绍，具体可以参照系统附带的测例模板“研究全流程”。下图是信号评价的结果：)
<div align=center>
<img width="1000" src="TransMatrix使用手册/pics/signal_whole_2.png"/>
</div>
<div align=center style="font-size:12px">Signal Evaluation(信号评价)</div>
<br />

#### 3.5.1 Strategy Backtesting and Evaluation (策略回测与评价)

After the signal evaluation, we believe the signal is effective and can be further developed into a stock strategy. Here, we rank the stocks by signal size in each period, select the top 50 stocks with the highest signal values, and buy them equally at 2:30 PM. The strategy code and evaluation component code can be referred to in the system template sample. The backtesting results are as follows:
(在经过信号评价后，我们认为信号是有效的，可以进一步加工成一个股票策略。这里，我们每期按信号大小，对股票作排序，选取前 50 只信号值最大的股票，14点30分等权买入。策略代码和评价组件代码，可参考系统模板测例。运行完成输出结果如下：)
<div align=center>
<img width="1000" src="TransMatrix使用手册/pics/signal_whole_3.png"/>
</div>
<div align=center style="font-size:12px">Backtesting Results(回测结果)</div>
<br />

This completes the translation of the provided text while retaining the original meaning, accuracy, correctness, and semantics.

[**Evaluation of the Code**]

**Python Code Quality**:

- Syntax and Structure: The code is syntactically correct and follows a clear structure. It uses classes and methods appropriately, which helps in organizing the code logically.
- Modularity: The code is modular, with distinct methods for different tasks such as initializing, transforming data, updating factors, and saving results. This modularity makes the code easier to understand, maintain, and extend.
- Use of Libraries: The code utilizes well-known Python libraries such as pandas, numpy, scipy, and lightgbm, which are appropriate for the tasks being performed. The use of these libraries suggests that the code is leveraging efficient and tested implementations for data manipulation and machine learning.
- Data Handling: The code efficiently handles missing values and data standardization, which are critical steps in preparing data for machine learning models. The use of DataView3d and DataView2d classes indicates that the code is designed to work with multi-dimensional data structures effectively.
- Model Training: The machine learning model (lightGBM) is integrated within a pipeline along with data scaling. This ensures that the model training process includes necessary preprocessing steps, which is a good practice.

**Logic and Implementation**:

- Factor Calculation: The logic for calculating various factors is clearly defined. Each factor has a specific calculation method, which is implemented in the on_clock method. The factors are updated and stored appropriately.
- Model Training and Evaluation: The process for splitting data into training and testing sets, handling missing values, training the model, and evaluating its performance is well-structured. The use of MinMaxScaler for scaling and lightgbm for regression is appropriate and follows standard practices.
- Signal Generation and Backtesting: The code outlines steps for generating signals using the trained model and evaluating these signals through backtesting. While the detailed implementation of signal evaluation is not provided, the outline suggests a systematic approach to testing the effectiveness of the generated signals.

**Style:**

- Readability: The code is well-commented, which helps in understanding the purpose of each section. The naming conventions for variables and methods are clear and descriptive, enhancing readability.
- Documentation: The inclusion of documentation for the YAML configuration and explanations for each step in the process helps in understanding the overall workflow. This is particularly useful for users who may want to replicate or modify the process.


Overall, the code is well-written, logically sound, and follows good practices in Python programming. It demonstrates a clear workflow for factor calculation, model training, signal generation, and backtesting, making it a robust implementation for financial data analysis and strategy development.

