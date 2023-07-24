# 因子研究

和策略研究一样，因子研究包含 3 种主要的组件。
- matrix
  - 配置了 Matrix 组件所需要的信息，它是系统的计算引擎
- strategy
  - 配置因子计算组件，写因子计算逻辑代码的地方
- evaluator
  - 配置评价组件，因子数据如何作单因子分析，并展示结果

本章，我们以一个简单的反转因子为示例，介绍如何使用 TransMatrix 系统来实现诸如：因子计算、因子评价、因子入库、因子定时更新等因子研究的各个功能。

### 3.1 如何计算因子

反转因子的逻辑较为简单，首先计算个股的日度收益率，再取5日均值并取负值，最后在截面上计算 zscore，zscore的值作为因子值。

#### 3.1.1 配置相关信息
为了计算该因子，我们需要配置相关信息，以明确以下几个问题：

1、我打算生成哪一段时间区间的因子？

2、因子在截面上包含哪些股票？

3、因子数据的计算逻辑写在哪里？

4、因子数据保存到哪里？

5、有了因子数据后，如何做单因子分析，分析结果怎么可视化？

与策略研究一样，这些配置信息既可以写在字典 (dict) 对象里，也可以配置在 yaml 文件中。我们新建一个 yaml 文件，命名为 config.yaml，并输入以下内容：
```text
matrix:
    mode: signal # 研究模式为因子研究
    span: [2018-01-01, 2021-12-30] # 因子计算区间
    codes: '*'
    universe: [meta_data, critic_data, is_zz500] # 因子在截面上包含哪些股票？
    save_signal: False # 是否保存因子到指定表

strategy: # 因子数据的计算逻辑写在哪里？
    reverse_factor:
        class: 
            - strategy.py 
            - ReverseSignal

```

由于我们只需要计算因子数据，而不作因子评价，故不需要配置评价组件。

在 matrix 部分的配置，我们将研究模式设为 signal，代表是因子研究模式，计算区间是2018年1月1日到2021年12月30日。因子计算的样本空间是中证500，它是一个动态的股票池，由 meta_data 库的 critic_data 表的 is_zz500字段给定，它的取值为0或1，若为1则股票在当前截面属于中证500。关于code 和 universe 的配置的更多说明，参见[这里]。

在 strategy 部分的配置，我们的因子计算逻辑，写在 strategy.py 文件的 ReverseSignal 类中。

#### 3.1.2 实现因子计算组件
我们新建一个 strategy.py 文件，输入以下代码：
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
        ret = (pv['close'] / pv['close'].shift(1) - 1).fillna(0) # 计算日度收益率
        reverse = -ret.rolling(window = 5, min_periods = 5).mean().fillna(0) # 取5日均值，并取负值
        reverse = zscore(reverse, axis = 1, nan_policy='omit') # 在截面上计算zscore作为因子值
        
        self.reverse: DataView3d = create_data_view(
            NdarrayData.from_dataframes({'reverse':reverse})
        )
        
        self.reverse.align_with(self.pv)
            
    def on_clock(self):
        self.update_signal(self.reverse.get('reverse'))
```

上述代码中，我们定义了一个 ReverseSignal 类，它继承自 SignalStrategy。SignalStrategy 是系统因子计算的基类，因子计算类都要继承它。关于 SignalStrategy 的详细说明，参见[这里]。

ReverseSignal 类重载了以下 3 个方法：
- init 方法，初始化因子计算组件时将调用本方法。这里添加了一个因子更新定时调度器，时间设置为 8:30，代表每个交易日的 8:30 会触发调度器调用 on_clock 方法。另外，还订阅了股票的日线行情数据（来源于demo库的stock_bar_1day表，字段是高开低收，筛选的股票与 Matrix 引擎配置的样本空间一样，即中证500）。
- pre_transform 方法，因子计算前将调用本方法，通常用于对订阅数据作向量化计算，以提高效率。这里，我们实现了本节开头提到的反转因子的计算逻辑。
- on_clock 方法，每个交易日的 8：30，将调用本方法。由于 reverse 数据已在 pre_transform 计算好，这里只需要取出，并更新到 signal 属性中，该属性是用于存放单个因子数据的容器。

#### 3.1.3 执行因子计算

我们新建一个 ipynb 文件，命名为 run.ipynb。新建一个代码单元格，输入以下代码：
```python
from transmatrix.workflow.run_yaml import run_matrix
mat = run_matrix('config.yaml')
```
系统计算完因子数据后，将输出以下内容：
```text
Out:
    loading data meta_data__critic_data__is_zz500 from datacache.

    Updating private cache meta_data__critic_data__is_zz500: dataset_ca21686c-80ce-4e96-bd95-ceb53f03895c ....
    loading data demo__stock_bar_1day__open,high,low,close from datacache.
```
我们可以查看计算的因子数据：
```python
strategy = mat.strategies['reverse_factor']
strategy.signal.to_dataframe()
```
<div align=center>
<img width="1000" src="pics/signal_factor.png"/>
</div>
<div align=center style="font-size:12px">因子数据</div>
<br />

注意，这里的 strategy 对象的 signal 属性，是一个 DataView2d，而 strategy 对象订阅的行情数据 pv，它是一个 DataView3d 类型。DataView3d 是一个 3d 数据视图，针对 Data3d 提供了诸多的数据查询接口；DataView2d 是一个 2d 数据视图，针对 Data2d 提供了数据查询接口。关于 Data3d, Data2d, DataView2d, DataView3d 等类的详细说明，参照[这里]。
```python
print(strategy.signal)
print(strategy.pv)
```
```text
Out:
    <transmatrix.data_api.view.data_view.DataView2d object at 0x7efe40b9a940>
    <transmatrix.data_api.view.data_view.DataView3d object at 0x7efe40b9a940>
```

到这里，我们完成了一个反转因子的计算。

### 3.2 如何评价因子
因子计算完成后，很自然的会有一个疑问，如何对该因子作单因子分析，评价因子的有效性？其实与策略研究类似，因子研究也是通过评价组件来进行单因子分析，并展示分析结果。以上节的反转因子为例，我们只需要添加一个评价组件，并实现评价逻辑，系统便会自动调用该组件对因子数据作评价。

我们复用上节的 config.yaml 文件，给它添加一节配置，用于指定评价组件。

```text
matrix:
    mode: signal # 研究模式为因子研究
    span: [2018-01-01, 2021-12-30] # 因子计算区间
    codes: '*'
    universe: [meta_data, critic_data, is_zz500] # 因子在截面上包含哪些股票？
    save_signal: False # 是否保存因子到指定表

strategy: # 因子数据的计算逻辑写在哪里？
    reverse_factor:
        class: 
            - strategy.py 
            - ReverseSignal

evaluator: # 如何评价因子
    SimpleEval:
        class:
            - evaluator.py
            - SimpleEvaluator
```

如上所示，yaml 文件的前 2 部分与上节一样，只是添加了一个评价组件。评价组件写在 evaluator.py 文件的 SimpleEvaluator 类里。我们新建一个 py 文件，命名为 evaluator.py，输入以下代码：
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
        ic_ser = factor_panel.corrwith(ret_panel, axis=1, method = 'spearman') # 计算 ic
        self.ic_ser = ic_ser.iloc[:-1]
    
    def show(self):
        self.plot(self.ic_ser, title = 'Rank IC', name='rank_ic')
        
    def regist(self):
        pass
```

SimpleEvaluator 继承自基类 SignalEvaluator。所有因子评价组件都要继承该类，与策略研究类似，同样需要重载 4 个基类方法，以实现因子评价。
- init 方法，主要用于订阅评价要用到的数据。这里我们订阅了股票日线收盘价数据。
- critic 方法，用于实现因子评价的计算逻辑。这里我们计算了因子的每日 IC 值，即当前因子数据与下一交易日的股票收益率的秩相关系数。
- show 方法，用于展示因子评价结果。这里我们将计算得到的每日 IC 值绘制成线图。
- regist 方法，若要将因子注册到 TransQuant 策略面板，则需要重载本方法。这里跳过。

关于 SignalEvaluator 基类的更多说明，参见[这里]。

我们再次运行 run.ipynb 里的单元格，得到以下输出：
<div align=center>
<img width="1000" src="pics/signal_eval.png"/>
</div>
<div align=center style="font-size:12px">因子评价</div>
<br />

以上便完成了 3.1 节的反转因子的评价，出于演示目的，这里仅计算并展示了因子的 IC 值，用户可以添加更多的逻辑，以实现诸如分组分析、回归分析、换手率分析、相关性分析等单因子分析的内容。系统也附带提供了多套因子评价模板，参见[因子服务-评价模板]。

### 3.3 保存因子数据

假如因子计算完成，评价也完成，得出结论，该因子是一个 alpha 因子，我们可能需要保存因子数据，以便进一步分析或构建策略。
在 TransMatrix 系统中，因子数据保存在个人空间里，个人空间是一个私有数据库，只有用户自己能够访问。为了保存因子数据，我们需要进行以下 2 步：
-   我们新建一张因子数据表，用于存放因子数据（注：若保存到现有因子数据表，此步骤可跳过）。
```python
from transmatrix.data_api import create_factor_table
temp_table_name = 'temp_factor_table'
create_factor_table(temp_table_name) # 新建一张名为 temp_factor_table 的因子数据表
```
-   调用 SignalStrategy 的 save_signal 方法，将因子数据保存到指定数据表。
```python
strategy = mat.strategies['reverse_factor']
strategy.save_signal(temp_table_name)
```
```text
Out:
    reverse_factor: saving signal reverse_factor to temp_factor_table1....
    更新完毕: xwq1_private temp_factor_table1 : ['reverse_factor']
```

这样，我们在 3.1 节计算得到的反转因子数据，便保存到了个人空间里的 temp_factor_table1 表里。

另一种保存因子数据的方式，是在配置信息中，指定要保存的目标表名，Matrix 引擎会自动保存计算得到的因子数据。如下图所示：

<div align=center>
<img width="1000" src="pics/save_signal.png"/>
</div>
<div align=center style="font-size:12px">因子保存</div>
<br />

我们在 yaml 文件的 matrix 部分，save_signal 配置了表名 temp_factor_table1，表示我们需要将因子数据保存在该表。当我们调用 run_matrix 方法时，Matrix 引擎会执行因子数据保存操作。注意，这里配置的表名必须在个人空间数据库中已存在，若不存在可提前用 create_factor_table 新建出来。

### 3.4 定时更新因子数据

假设计算得到的因子数据，通过了因子评价检验，需要每日增量更新因子数据，应该如何实现？只需要打开系统提供的定时任务模块，通过配置定时更新因子数据的调度任务，即可实现。更多介绍，参见 TransQuant 的[定时任务]。

### 3.5 一个完整的股票多因子策略示例

一个完整的股票多因子策略，通常包括以下流程：
<div align=center>
<img width="1000" src="pics/sig_all.png"/>
</div>
<div align=center style="font-size:12px">多因子策略研究流程</div>
<br />

这里，我们基于该流程，给出一个完整的股票多因子策略示例。

#### 3.5.1 因子计算与入库

因子计算与入库是整个示例的第 1 部分，为了与剩余的 2 部分区别开，我们这里新建一个名为“因子计算与入库”文件夹，该文件夹用来存放第 1 部分的文件。

首先，我们选择以下 10 个因子，并假设它们都是有效的 alpha 因子，不额外作单因子检验。
<div align=center>
<img width="1000" src="pics/10factor.png"/>
</div>
<div align=center style="font-size:12px">10个因子及说明</div>
<br />

我们新建一个 config.yaml 文件，用于存放相关配置信息。
```text
matrix:

    mode: signal
    span: [2014-1-1, 2020-1-4]
    codes: &universe ../data/codes.pkl # 股票样本空间
    save_factors: factor_table # 因子数据保存到表 factor_table 中

strategy:
    factors:
        class:
            - strategy.py 
            - factors
```
这里，我们因子计算所用的样本空间，是一个自定义样本空间。用户可以使用用自定义的样本空间，也可以使用用沪深300、中证500或全市场股票等动态股票池作为样本空间，关于动态股票池的配置，可以参考 3.1.1 节。

我们新建一个 strategy.py 文件，输入以下内容：
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
        
        # 生成ret
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
        # 去除inf
        factor_data = {}
        for k,v in self.factor_data.items():
            v = v.to_dataframe().replace([np.inf,-np.inf], np.nan)
            factor_data[k] = v
            
        self.factor_data = factor_data
```

上述代码里，我们定义了一个 factors 类，它继承自 SignalStrategy 基类。注意到这里，由于需要生成 10 个因子，因此在 init 方法中，我们除了订阅所需要的数据之外，还要调用 create_factor_table 方法，用于注册一个字典，用于存放这 10 个因子的数据。
``` python
        self.create_factor_table([f'factor{i}' for i in range(1,11)] + ['y'])
```
这一代码，注册了从 factor1, factor2 到 factor10，这 10 个因子数据，另外为了便于之后方便用模型训练，我们把下一交易日的收益率作为标签 y，同样存放到字典容器中。

我们可以通过属性 factor_data，来获取这一因子数据字典对象。

对比 3.1 节和本节，我们可以得出：
- 一个因子计算组件（继承自 SignalStrategy）可以生成**一个**因子：
  - 因子数据通过 **signal** 属性获取（3.1 节）；
  - 保存因子数据时 调用 **save_signal** 方法。
- 一个因子计算组件可以生成**多个**因子：
  - 在 init 方法中调用 **create_factor_table** 方法来注册这些因子，因子数据通过 **factor_data** 属性来获取（本节）；
  - 保存因子数据时 调用 **save_factors** 方法。

本示例把因子数据保存配置在了 config.yaml 文件里：
```text
    save_factors: factor_table # 因子数据保存到表 factor_table 中
```
因子计算的逻辑在 on_clock 方法中，可对照表格 **10个因子及说明** 阅读这部分代码，这里不作额外解释。

我们新建一个 ipynb 文件，命名为 run.ipynb，在单元代码格里输入以下内容：
```python
from transmatrix.data_api import create_factor_table
from transmatrix.workflow import run_matrix
create_factor_table('factor_table')
mat = run_matrix('config.yaml')
```
运行本单元格，系统将在个人空间新建一张名为 factor_table 的因子表，生成 10 个因子数据，并保存到该表，第一部分的因子生成结束。

#### 3.5.2 模型训练与信号评价

有了这 10 个因子的数据后，我们进入第 2 部分，模型训练与信号评价。
-   读取因子数据，分割样本为训练集和测试集，采用 lightGBM 算法训练模型

我们新建一个 ipynb 文件，命名为 a_模型训练.ipynb。为了读取因子数据，我们在代码单元格里输入
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

print('数据完成加载')
display(factor_data)
```
<div align=center>
<img width="1000" src="pics/signal_whole_1.png"/>
</div>
<div align=center style="font-size:12px">读取因子数据</div>
<br />

接下来，切割样本为训练集和测试集，我们新建一个代码单元格，输入以下内容：
```
# 定义训练和测试期间

train_start_year = '2014-1-1'
test_start_year = '2017-1-1'
test_end_year = '2020-1-1'

print(train_start_year, test_start_year, test_end_year) # 切割样本为训练集和测试集

# 选择因子
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

最后，着手训练模型
```python
from tools import get_train_valid_test_data, process_nan
models = {}
longshort_df = pd.DataFrame()

# 定义训练和测试过程
train_x, train_y, test_x, test_y = get_train_valid_test_data(train_start_year, test_start_year, test_end_year, factor_data, factor_columns)

# 处理缺失值
_,train_x1,train_y1 = process_nan(train_x, train_y)
mask_test,test_x1,test_y1 = process_nan(test_x, test_y)

train_x1 = train_x1.drop(['code'],axis=1).values
train_y1 = train_y1.values
test_x1 = test_x1.drop(['code'],axis=1).values
test_y1 = test_y1.values

# 建立pipline
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
    ('scaler',scaler),  # 数据缩放
    ('model',model)  # 模型训练
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

至此，我们便完成了模型的训练，并把模型保存在本地。基于演示的目的，模型训练比较简单，没有进行参数优化，用户在模型训练时，可以进行更复杂的优化调整。

-   使用模型生成信号，评价信号的有效性

这部分包含 2 个步骤，1 是信号生成，这与因子计算类似，我们加载因子数据，使用模型预测得到信号并保存到数据库里；2 是信号评价，添加评价组件并实现评价逻辑，便可以对信号进行评价。考虑到评价组件的代码篇幅较长，这里不作介绍，具体可以参照系统附带的测例模板“研究全流程”。下图是信号评价的结果：
<div align=center>
<img width="1000" src="pics/signal_whole_2.png"/>
</div>
<div align=center style="font-size:12px">信号评价</div>
<br />

#### 3.5.1 策略回测与评价

在经过信号评价后，我们认为信号是有效的，可以进一步加工成一个股票策略。这里，我们每期按信号大小，对股票作排序，选取前 50 只信号值最大的股票，14点30分等权买入。策略代码和评价组件代码，可参考系统模板测例。运行完成输出结果如下：
<div align=center>
<img width="1000" src="pics/signal_whole_3.png"/>
</div>
<div align=center style="font-size:12px">回测结果</div>
<br />



