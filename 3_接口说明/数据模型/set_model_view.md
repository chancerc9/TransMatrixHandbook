### [数据管理](8_测例代码\transmatrix特色功能\数据获取-data_api\量价数据)
TransMatrix-python 框架使用 numpy 作为默认的数据后端。

在回测引擎中，数据管理分为[数据订阅](3_接口说明/策略/generator.md/#subscribe_data)和数据回放两个阶段:

数据订阅：用户的一次数据订阅将产生一个指向目标[数据集合](#Dataset)的[视图](#DataView)。

数据回放：数据视图的时间状态随着回测进行更新。

数据视图对外提供丰富的查询接口，保证用户获取最新数据。

---
---

### Dataset

描述一个数据集合

---
<b> \__init__ </b>

- 参数: 
  - data_model (str): 'ndarray' / 'finance-report'，分别对应普通数据和财报数据。
  - describe (dict):
      - source: 'db' / 'file', (传入db 表示从数据库中读取数据 / 传入 file 表示从文件读取数据)
      - db_name (str): 数据库名 (source 为 db 时)。
      - table_name (str): 数据表名 (source 为 db 时)。
      - file_path (str) : 数据文件路径 (source 为 file 时)。
      - start: 开始时间
      - end: 结束时间
      - codes (List[str]): 标的代码列表
      - fields: (List[str]) : 字段列表
  - use_cache: bool 是否采用数据缓存。Defaults to True.

---
<b> load_data </b>： 读取数据

- 参数：meta_info (dict), 缓存信息。 Defaults to None（将读取系统默认的缓存信息文件）。
- 返回值： [Data3d](#NdarrayData) / [FinanceReportData](#FinanceReportData)

---

<b> cache </b>： 缓存数据

- 参数：meta_info (dict), 缓存信息。 Defaults to None（将读取系统默认的缓存信息文件）。
- 返回值： 无
---

---
### DataModel

数据模型基类，用于存放一个数据集。

目前系统提供了 Data3d, Data2d 和 FinanceReportData 三个数据模型。 

---

#### Data3d

一个三维数据集合，后端为 numpy.ndarray。

系统默认使用 （时间，字段，标的代码）的三维数据结构。

---

<b> \__init__ </b>

- 参数:
  - data (numpy.ndarray): 数据集（将根据数据集的形状推断数据维度，目前支持 2维 和 3维数据）
  - describe (List): 数据集的描述信息（长度为3）
    - 时间戳 (Iterable[datetime])
    - 字段列表 (Iterable[str] 或 str)
    - 代码列表 (Iterable[str] 或 str)
  
---

<b> from_dataframes </b>

将多个 dataframe 拼接成一个 3d 数据集

- 参数: 
  - dataframes: dict
    - key: 字段名
    - value: dataframe 数据 (列为标的代码)
- 返回值：Data3d

---

<b> to_dataframes </b>

将 Data3d 转化为 dataframe

- <b> 参数 </b>: 
  - col (str): 传入 code 或 field, 分别表示以标的代码或字段名作为列名。
- <b> 返回值 </b> : dataframes: dict
  - key: 当 col = 'code' 时为字段名 , 当 col = 'field' 时列为股票代码
  - value: dataframe 数据 (当 col = 'code' 时列为股票代码 , 当 col = 'field' 时列为字段明)

---
<b> from_csv </b>

读取csv数据并转化成 Ndarray 数据模型

- <b> 参数 </b> :
  - file_path (str): 数据路径
  - start, end: 起止时间.
  - codes (List[str]): 标的代码列表
  - fields: (List[str]) : 字段列表
  
- <b> 返回值: </b> Data3d
  
- <b> csv格式要求 </b> :
  - df.index 必须为时间戳格式 (pd.DatetimeIndex)。
  - df 中必须存在名为 code 的列 
  - df.index * code 维度下数据必须唯一

- <b>  start, end 处理逻辑 </b> :
  - start, end 接受  str / datetime / date 类型的输入
  - 若输入的 end 为日期，则将时间修改为当日 16:00。

---

<b> from_db </b>

读取 db 数据并转化成 Data3d 数据模型

- <b> 参数 </b>
  - table_name (str): 数据表名
  - db_name (str): 数据库名
  - start, end: 起止时间.
  - codes (List[str]): 标的代码列表
  - fields: (List[str]) : 字段列表
  - padding_codes (bool): 是否按照 codes填充确实标的的数据(整列数据为 nan), 默认为 True。
  - datetime_checker: 按照 checker 剔除无效数据, 默认为 None (不剔除)。
  
- <b> 返回值 </b>: Data3d

- <b> start, end 处理逻辑 </b> :
  - start, end 接受  str / datetime / date 类型的输入
  - 若输入的 end 为日期，则将时间修改为当日 16:00


---
<b> query </b>

对给定的时间点，返回该时间（之前）的数据 (窗口)

- <b> 参数 </b>
  - time (datetime): 查询时间点
  - periods (int): 查询的最近 N 条数据
  - start_time (datetime): 查询 start_time 到 time 之间的数据
  - window (timedelta): 查询 time - window 到 time 之间的数据

- <b> 返回值 </b>
  - Dict[str, pd.DataFrame] : 
    - key: 字段名
    - value: dataframe 数据 (列为标的代码)


---
---

#### Data2d

存储一个二维数据集合，底层为 numpy.ndarray。

---

<b> from_csv </b>

读取csv数据并转化成 Data2d 数据模型

- <b> 参数 </b> : 
  - file_path (str): 数据路径
  - start, end: 起止时间.
  - cols: (List[str]): 字段列表
  
- <b> 返回值 </b> : Data2d

- <b> csv格式要求 </b> 
  - 必须存在名为 datetime 的时间戳列
  - 返回的结果数据只包含一个标的 (2维面板数据)。 此时要求 datetime 列中不包含重复时间戳。

- <b> start, end 处理逻辑 </b> :
  - start, end 接受  str / datetime / date 类型的输入
  - 若输入的 end 为日期，则将时间修改为当日 16:00

---

<b> _from_dataframe </b>

将dataframe数据转化成 Data2d 数据模型

- <b> 参数 </b> : 
  - df (Dataframe): 数据路径
  - start, end: 起止时间.
  - cols: (List[str]): 字段列表
  
- <b> 返回值 </b> : Data2d

- <b> csv格式要求 </b> 
  - 必须存在名为 datetime 的时间戳列
  - 返回的结果数据只包含一个标的 (2维面板数据)。 此时要求 datetime 列中不包含重复时间戳。

- <b> start, end 处理逻辑 </b> :
  - start, end 接受  str / datetime / date 类型的输入
  - 若输入的 end 为日期，则将时间修改为当日 16:00

---

<b> from_db </b>

读取 db 数据并转化成 Data2d 数据模型

- <b> 参数 </b>:
  - table_name (str): 数据表名
  - db_name (str): 数据库名
  - src (str): 数据路径
  - start, end: 起止时间.
  - code (str): 标的代码
  - fields: (List[str]) : 字段列表

- <b> start, end 处理逻辑 </b> :
  - start, end 接受  str / datetime / date 类型的输入
  - 若输入的 end 为日期，则将时间修改为当日 16:00

---

<b> query </b>

对给定的时间点，返回该时间（之前）的数据 (窗口)

- <b> 参数 </b>
  - time (datetime): 查询时间点
  - periods (int): 查询的最近 N 条数据
  - start_time (datetime): 查询 start_time 到 time 之间的数据
  - window (timedelta): 查询 time - window 到 time 之间的数据

- <b> 返回值 </b>
  - DataFrame
    - index: 目标时间段
    - columns: 字段名

---

<b> to_dataframe </b>

将 Data2d 数据转化成 dataframe

- <b> 参数 </b>: 无
- <b> 返回值 </b>: 
  - Dataframe
    - index: 时间戳
    - columns: 字段集合

---

---

#### StructArray

存储一个二维结构体数组，底层为 numpy.ndarray（结构体数据）。

---

**\__init__** 

- 参数:
  - data (numpy.ndarray): 数据集（将根据数据集的形状推断数据维度，目前支持 2维 和 3维数据）
  - describe (List): 数据集的描述信息（长度为2）
    - 时间戳 (Iterable[datetime])
    - 字段列表 (Iterable[str] 或 str)

---

**from_dataframe**

将dataframe数据转化成 StructArray 数据模型

- <b> 参数 </b> : 
  - data (Dataframe): 数据
  - start, end: 起止时间.
  - cols: (List[str]): 字段列表
- <b> 返回值 </b> : StructArray

---

**from_ndarray**

将numpy.ndarray数据转化成 StructArray 数据模型

- <b> 参数 </b> : 
  - data (ndarray): 数据
  - describe (List): 数据集的描述信息（长度为2）
    - 时间戳 (Iterable[datetime])
    - 字段列表 (Iterable[str] 或 str)
- <b> 返回值 </b> : StructArray

---

**from_data2d**

将Data2d数据转化成 StructArray 数据模型

- <b> 参数 </b> : 
  - data (Data2d): Data2d数据
- <b> 返回值 </b> : StructArray

---

**from_csv**

读取csv数据并转化成 StructArray 数据模型

- <b> 参数 </b> : 
  - file_path (str): 数据路径
  - start, end: 起止时间.
  - cols: (List[str]): 字段列表

- <b> 返回值 </b> : StructArray 

- <b> csv格式要求 </b> 
  - 必须存在名为 datetime 的时间戳列
  - 返回的结果数据只包含一个标的 (2维面板数据)。 此时要求 datetime 列中不包含重复时间戳。

- <b> start, end 处理逻辑 </b> :
  - start, end 接受  str / datetime / date 类型的输入
  - 若输入的 end 为日期，则将时间修改为当日 16:00

---

**from_parquet**

读取parquet数据并转化成 StructArray 数据模型

- <b> 参数 </b> : 
  - file_path (str): 数据路径
  - start, end: 起止时间.
  - cols: (List[str]): 字段列表

- <b> 返回值 </b> : StructArray 

- <b> parquet格式要求 </b> 
  - 必须存在名为 datetime 的时间戳列
  - 返回的结果数据只包含一个标的 (2维面板数据)。 此时要求 datetime 列中不包含重复时间戳。

- <b> start, end 处理逻辑 </b> :
  - start, end 接受  str / datetime / date 类型的输入
  - 若输入的 end 为日期，则将时间修改为当日 16:00

---

**from_db**

读取 db 数据并转化成 StructArray 数据模型

- <b> 参数 </b>:
  - table_name (str): 数据表名
  - db_name (str): 数据库名
  - start, end: 起止时间.
  - codes (str): 标的代码
  - fields: (Union[List[str],str]) : 字段列表

- <b> start, end 处理逻辑 </b> :
  - start, end 接受  str / datetime / date 类型的输入
  - 若 start,end 为字符串, 则将其转换为 datetime 格式
  - 若输入的 end 为日期，则将时间修改为当日 16:00

---

**to_dataframe**

将 StructArray 数据转化成 dataframe

- <b> 参数 </b>: 无
- <b> 返回值 </b>: 
  - Dataframe
    - index: 时间戳
    - columns: 字段集合

---
---

#### FinanceReportData
财务数据模型基类，用于存放财务数据

---
---

#### FinanceReportPanelData
- 财务面板数据，*通过Dataset构造*
- 底层为 multi-index dataframe

<b> \__init__ </b>

- 参数:
  - data (pd.DataFrame): 财务面板数据（multi-index dataframe)
  - report_period_dict: 报告期字典
  
---

<b> copy </b>

复制当前的财务面板数据

- 参数: 无
- FinanceReportPanelData

---

<b> query </b>
- 查询规则: 
    - 返回按照财报期对齐的数据
    - 给定查询时间点，返回该时间之前
- args: 
    - time: *datetime* 
    - 1 of：
        - periods (int) 返回N条数据
        - start_time (detetime) 返回某时刻**对应的报告期**及其之后的数据
        - window (timedelta) 返回一段时间的数据
        - shift (int) 返回前数第N条数据 
---
---

#### FinanceReportSectionData
- 财报截面数据，通过Dataset构造
- 指定data_model为'finance-report'，lag参数格式为'nQ'或'nY'的格式。例如当lag为'8Q'时，表示取当前最新可用的财务数据，并取过去连续8个季度的财务数据；当lag为'8Y'时，表示取当前最新可用的财务数据，并取过去8年同一季度的财务数据
- 继承自NdarrayData，**NdarrayData支持的属性方法，它都支持**。同时也支持创建数据视图。

<b> \__init__ </b>

- 参数:
  - data (numpy.ndarray): 财务截面数据
  - describe: : 数据集的描述信息（长度为3）
    - 时间戳 (Iterable[datetime])
    - 字段列表 (Iterable[str] 或 str)
    - 代码列表 (Iterable[str] 或 str)
  
---
---

### DataView

数据视图基类，系统默认使用 3d 数据视图。

数据视图的时间状态随着回测进行更新。

数据视图对外提供丰富的查询接口，保证用户获取最新数据。

多个数据视图可以共享一份数据。

时间轴相同的数据视图之间可以实现状态同步。


---
---

#### DataView2d

2d数据视图，对外提供数据查询接口，用于获取最新数据。

---

<b> \__init__ </b>
- 参数: data (Data3d): 数据集


---

<b> get </b>

- <b>功能</b>: 获取最新一条数据
- <b>参数</b>: 无
- <b>返回值</b>: np.array, shape = (len(cols),) 

---

<b> get_dict </b>

- <b>功能</b>: 获取最新一条数据, 返回字典
- <b>参数</b>: 无
- <b> 返回值 </b>: Dict[str, np.float]

---

<b> get_window </b>

- <b>功能</b>: 获取指定时间窗口的数据
- <b>参数</b>: length (int): 窗口长度
- <b>返回值></b>: np.array, shape = (length, len(cols))


---
<b> to_dataframe </b>

- <b>功能</b>: 将数据转化为 dataframe
- <b>参数</b>: 无


---
---
#### DataView3d

3d数据视图，对外提供数据查询接口，用于获取最新数据。

---
<b> \__init__ </b>

- 参数: data (Data3d): 数据集

---

```python
from transmatrix.data_api import Dataset
from transmatrix.data_api import create_data_view

desc = {"db_name": 'demo', "table_name": 'stock_bar_1day', "start": '20210101', "end": '20210120', "fields": ['open','high','low','close','volume'], "codes": ['000001.SZ','000002.SZ']}
# 查询股票000001.SZ和000002.SZ，在时间段20210101到20210120的行情数据，包括：开盘价、最高价、最低价、收盘价、成交量
dataset = Dataset(data_model = 'ndarray', describe=desc).load_data()
dv3d = create_data_view(dataset)
dv3d
```
```text
<transmatrix.data_api.view.data_view.DataView3d at 0x7fcca5ee1fa0>
```

```python
# 展示收盘价数据
dv3d.to_dataframe()['close']
```
```text
	000001.SZ	000002.SZ
datetime		
2021-01-04 15:00:00	18.60	27.78
2021-01-05 15:00:00	18.17	27.91
2021-01-06 15:00:00	19.56	28.75
2021-01-07 15:00:00	19.90	28.79
2021-01-08 15:00:00	19.85	29.34
2021-01-11 15:00:00	20.38	29.78
2021-01-12 15:00:00	21.00	29.70
2021-01-13 15:00:00	20.70	29.90
2021-01-14 15:00:00	20.17	29.99
2021-01-15 15:00:00	21.00	29.95
2021-01-18 15:00:00	22.70	31.26
2021-01-19 15:00:00	22.34	31.28
2021-01-20 15:00:00	22.47	30.50
```
<b> get </b>

  - <b>功能</b>: 获取指定字段的最新一条数据
  - <b>参数</b>:
    - field (str): 字段名
    - codes (Union[list, str], optional): 标的代码集合. Defaults to '*' (返回所有标的数据)。
  - <b>返回值</b>: 
    - object 或 np.array (shape = (len(codes), )): 返回指定字段的数据 

```python
dv3d.get('close', '000001.SZ') # 返回当前游标对应的 000001.SZ 数据
```
```text
22.47
```

```python
dv3d.get('close', '*') # 返回当前游标对应的所有股票的数据
```
```text
array([22.47, 30.5 ])
```

---

<b> get_dict </b>
  - <b>功能</b>: 获取指定字段的最新一条数据, 返回字典
  - <b>参数</b>:
    - field (str): 字段名
    - codes (Union[list, str], optional): 标的代码集合. Defaults to '*' (返回所有标的数据).
  - <b>返回值</b>: 
    - dict: key: 标的代码, value: 指定字段的数据

```python
dv3d.get_dict('close', '000001.SZ')
```
```text
22.47
```

```python
dv3d.get_dict('close', ['000001.SZ', '000002.SZ'])
```
```text
{'000001.SZ': 22.47, '000002.SZ': 30.5}
```

---

<b> get_code </b>
  - <b>功能</b>: 获取指定标的最新一条数据
  - <b>参数</b>:
    - code (str): 标的代码
    - fields (Union[list, str], optional): 字段集合. Defaults to '*' (返回所有字段数据).
  - <b>返回值</b>: 
    - np.array (shape = (len(fields), )): 返回指定标的数据

```python
dv3d.get_code('000001.SZ', fields = 'close')
```
```text
22.47
```

```python
dv3d.get_code('000001.SZ', fields = '*')
```
```text
array([2.24700000e+01, 2.29700000e+01, 2.21200000e+01, 2.21500000e+01,
       1.28079316e+08])
```

---

<b> get_code_dict </b>
  - <b>功能</b>: 获取指定标的最新一条数据, 返回字典
  - <b>参数</b>:
    - code (str): 标的代码
    - fields (Union[list, str], optional): 字段集合. Defaults to '*' (返回所有字段数据).
  - <b>返回值</b>: 
    - dict, key: 字段名, value: 指定标的数据。

```python
dv3d.get_code_dict('000001.SZ', fields = '*') # 获取 000001.SZ 当前所有字段对应的数据
```
```text
{'close': 22.47,
 'high': 22.97,
 'low': 22.12,
 'open': 22.15,
 'volume': 128079316.0}
```

```python
dv3d.get_code_dict('000001.SZ', fields = 'close')
```
```text
22.47
```

---

<b> get_window </b>
  - <b>功能</b>: 获取指定字段的最新 N 条数据
  - <b>参数</b>:
    - field (str): 字段名
    - length (int): 数据长度
    - codes (Union[list, str], optional): 标的代码集合. Defaults to '*' (返回所有标的数据).
  - <b>返回值</b>: 
    - np.array (shape = (length, len(codes))): 指定字段的数据

```python
dv3d.cursor.value = 3 # 把游标调到3，对应的时间是20210107 15点
dv3d.get_window('close', 3, codes = '*')
```
```text
array([[18.17, 27.91],
       [19.56, 28.75],
       [19.9 , 28.79]])
```

```python
dv3d.get_window('close', 3, codes = '000001.SZ')
```
```text
array([18.17, 19.56, 19.9 ])
```

---

<b> get_window_df </b>
  - <b>功能</b>:获取指定字段的最新 N 条数据, 返回 DataFrame
  - <b>参数</b>:
    - field (str): 字段名
    - length (int): 数据长度
    - codes (Union[list, str], optional): 标的代码集合. Defaults to '*' (返回所有标的数据).
  - <b>返回值</b>: 
    - pd.DataFrame: 指定字段的数据

```python
dv3d.get_window_df('close', 3, codes = '*')
```
```text
	000001.SZ	000002.SZ
0	18.17	27.91
1	19.56	28.75
2	19.90	28.79
```

```python
dv3d.get_window_df('close', 3, codes = '000001.SZ')
```
```text
0    18.17
1    19.56
2    19.90
Name: 000001.SZ, dtype: float64
```

---

<b> get_window_code </b>
  - <b>功能</b>:获取指定字段的最新 N 条数据
  - <b>参数</b>:
    - code (str): 标的代码
    - length (int): 数据长度
    - fields (Union[list, str], optional):  字段集合. Defaults to '*' (返回所有字段数据).
  - <b>返回值</b>: 
    - np.array (shape = (length, len(fields))): 返回指定标的数据

```python
dv3d.get_window_code('000001.SZ', 3, fields = '*')
```
```text
array([[1.81700000e+01, 1.84800000e+01, 1.78000000e+01, 1.84000000e+01,
        1.82135210e+08],
       [1.95600000e+01, 1.95600000e+01, 1.80000000e+01, 1.80800000e+01,
        1.93494512e+08],
       [1.99000000e+01, 1.99800000e+01, 1.92300000e+01, 1.95200000e+01,
        1.58418530e+08]])
```

```python
dv3d.get_window_code('000001.SZ', 3, fields = 'close')
```
```text
array([18.17, 19.56, 19.9 ])
```

---

<b> get_window_code_df </b>
  - <b>功能</b>: 获取指定标的最新 N 条数据, 返回 DataFrame
  - <b>参数</b>:
    - code (str): 标的代码
    - length (int): 数据长度
    - fields (Union[list, str], optional):  字段集合. Defaults to '*' (返回所有字段数据).
  - <b>返回值</b>: 
    - pd.DataFrame: 指定标的数据

```python
dv3d.get_window_code_df('000001.SZ', 3, fields = '*')
```
```text
close	high	low	open	volume
0	18.17	18.48	17.80	18.40	182135210.0
1	19.56	19.56	18.00	18.08	193494512.0
2	19.90	19.98	19.23	19.52	158418530.0
```

```python
dv3d.get_window_code_df('000001.SZ', 3, fields = 'close')
```
```text
0    18.17
1    19.56
2    19.90
Name: close, dtype: float64
```

---

<b> iloc </b>
  - <b>功能</b>: 返回指定位置的数据
  - <b>参数</b>:
    - i (int): 指定的位置
    - field (str): 指定的字段名称
    - codes (Union[list, str], optional): 标的代码合集. Defaults to '*'.
  - <b>返回值</b>: 
    - Union[np.array, object]: 对应的数据，若指定单个标的则直接返回对应值，若指定多个标的则返回 np.array

```python
dv3d.iloc(3, 'close', codes = '*')
```
```text
array([19.9 , 28.79])
```

```python
dv3d.iloc(3, 'close', codes = '000001.SZ')
```
```text
19.9
```
---


<b> loc </b>
  - <b>功能</b>: 返回指定时间的数据，若给定时间不在数据时间索引中，则返回在该时间之前并且最接近的数据
  - <b>参数</b>:
    - date_index (Union[datetime, pd.Timestamp, List[datetime], List[pd.Timestamp]]): 指定时间
    - field (str): 指定的字段名称
    - codes (Union[list, str], optional): 标的代码合集. Defaults to '*'.
  - <b>返回值</b>: 
    - Union[np.array, object]: 对应的数据，若指定单个标的则直接返回对应值，若指定多个标的则返回 np.array

```python
from datetime import datetime
dv3d.loc(datetime(2021,1,8,15), 'close', codes = '*') # 返回2021年1月8号15点的数据
```
```text
array([19.85, 29.34])
```
```python
dv3d.loc(datetime(2021,1,8,14), 'close', codes = '*') # dv3d中无1月8号14点的数据，返回它之前的那条数据，即1月7号15点的数据
```
```text
UserWarning: 给定的 date_index 不全在数据时间索引中，请注意。
array([19.9 , 28.79])
```

---

---

<b> iloc_code </b>
  - <b>功能</b>: 返回指定位置的数据
  - <b>参数</b>:
    - i (int): 指定位置
    - code (str): 标的代码
    - fields (Union[list, str], optional): 指定的字段合集. Defaults to '*'.
  - <b>返回值</b>: 
    - Union[np.array, object]: 对应的数据，若指定单个字段则直接返回对应值，若指定多个字段则返回 np.array

```python
dv.iloc_code(1, code='000001.SZ', fields='*')
```
```text
array([1.8170000e+01, 1.8480000e+01, 1.7800000e+01, 1.8400000e+01,
       1.8213521e+08])
```
```python
dv.iloc_code(1, code='000001.SZ', fields='close')的数据
```
```text
18.17
```

---

---

<b> loc_code </b>
  - <b>功能</b>: 返回指定时间、指定标的代码的数据，若给定时间不在数据时间索引中，则返回在该时间之前并且最接近的数据
  - <b>参数</b>:
    - date_index (Union[datetime, pd.Timestamp, List[datetime], List[pd.Timestamp]]): 指定时间
    - code (str): 标的代码
    - fields (Union[list, str], optional): 指定的字段合集. Defaults to '*'.
  - <b>返回值</b>: 
    - Union[np.array, object]: 对应的数据，若指定单个标的则直接返回对应值，若指定多个标的则返回 np.array

```python
dv3d.loc_code(datetime(2021,1,8,15), code='000001.SZ', fields='*')
```
```text
array([1.98500000e+01, 2.01000000e+01, 1.93100000e+01, 1.99000000e+01,
       1.19547322e+08])
```
```python
dv3d.loc_code(datetime(2021,1,8,14), code='000001.SZ', fields='*')) # dv3d中无1月8号14点的数据，返回它之前的那条数据，即1月7号15点的数据
```
```text
UserWarning: 给定的 date_index 不全在数据时间索引中，请注意。
array([1.9900000e+01, 1.9980000e+01, 1.9230000e+01, 1.9520000e+01,
       1.5841853e+08])
```

```python
dv3d.loc_code(datetime(2021,1,8,15), code='000001.SZ', fields='close')
```
```text
19.85
```

---

<b> query </b>
  - <b>功能</b>: 根据指定时间查询数据
  - <b>参数</b>:
    - time (datetime): 指定时间
    - periods (int, optional): 返回N条数据. Defaults to None.
    - start_time (datetime, optional): 返回从指定时间开始的数据. Defaults to None.
    - window (timedelta, optional): 返回指定时间窗口内的数据. Defaults to None.
  - <b>返回值</b>: 
    - dict: 
      - key: 字段名
      - value: pd.dataframe, index 为时间, columns 为标的代码

```python
dv3d.query(datetime(2021,1,12), periods = 3)
```
```text
{'close':                      000001.SZ  000002.SZ
 datetime                                 
 2021-01-07 15:00:00      19.90      28.79
 2021-01-08 15:00:00      19.85      29.34
 2021-01-11 15:00:00      20.38      29.78,
 'high':                      000001.SZ  000002.SZ
 datetime                                 
 2021-01-07 15:00:00      19.98      29.50
 2021-01-08 15:00:00      20.10      29.45
 2021-01-11 15:00:00      20.64      30.35,
 'low':                      000001.SZ  000002.SZ
 datetime                                 
 2021-01-07 15:00:00      19.23      28.39
 2021-01-08 15:00:00      19.31      28.81
 2021-01-11 15:00:00      20.00      29.27,
 'open':                      000001.SZ  000002.SZ
 datetime                                 
 2021-01-07 15:00:00      19.52      29.00
 2021-01-08 15:00:00      19.90      28.98
 2021-01-11 15:00:00      20.00      29.50,
 'volume':                        000001.SZ    000002.SZ
 datetime                                     
 2021-01-07 15:00:00  158418530.0  122675574.0
 2021-01-08 15:00:00  119547322.0  102856329.0
 2021-01-11 15:00:00  179045714.0  138812124.0}
```
```python
dic_data = dv3d.query(datetime(2021,1,12), start_time=datetime(2021,1,8))
dic_data['close']
```
```text
                  000001.SZ	000002.SZ
datetime		
2021-01-08 15:00:00	19.85	29.34
2021-01-11 15:00:00	20.38	29.78
```

```python
from datetime import timedelta
dic_data = dv3d.query(datetime(2021,1,12), window=timedelta(days=5))
dic_data['close']
```
```text
                  000001.SZ	000002.SZ
datetime		
2021-01-07 15:00:00	19.90	28.79
2021-01-08 15:00:00	19.85	29.34
2021-01-11 15:00:00	20.38	29.78
```

---

<<<<<<< HEAD
<b> query_code </b>
  - <b>功能</b>: 根据指定时间和标的代码查询数据
  - <b>参数</b>:
    - time (datetime): 指定时间
    - code (str): 指定的股票代码
    - fields (Union[list, str], optional): 指定的字段合集
    - periods (int, optional): 返回N条数据. Defaults to None.
    - start_time (datetime, optional): 返回从指定时间开始的数据. Defaults to None.
    - window (timedelta, optional): 返回指定时间窗口内的数据. Defaults to None.
  - <b>返回值</b>: 
    - dict: 
      - key: 字段名
      - value: pd.dataframe, index 为时间, columns 为标的代码

```python
dv3d.query_code(datetime(2021,1,12), code='000001.SZ', fields='close', periods=3)
```
```text
	                  close
datetime	
2021-01-07 15:00:00	19.90
2021-01-08 15:00:00	19.85
2021-01-11 15:00:00	20.38
```

```python
dv3d.query_code(datetime(2021,1,12), code='000001.SZ', fields='*', periods=3)
```
```text
                    close	high	low	open	volume
datetime					
2021-01-07 15:00:00	19.90	19.98	19.23	19.52	158418530.0
2021-01-08 15:00:00	19.85	20.10	19.31	19.90	119547322.0
2021-01-11 15:00:00	20.38	20.64	20.00	20.00	179045714.0
```

```python
dv3d.query_code(datetime(2021,1,12), code='000001.SZ', fields='*', start_time=datetime(2021,1,8))
```
```text
                    close	high	low	open	volume
datetime					
2021-01-08 15:00:00	19.85	20.10	19.31	19.9	119547322.0
2021-01-11 15:00:00	20.38	20.64	20.00	20.0	179045714.0
```

---
=======
---

#### DataViewStruct

StructArray的数据视图，对外提供数据查询接口，用于获取最新数据。

---

<b> \__init__ </b>

- 参数: 
  - data (StructArray): StructArray数据
  - cursor (BaseCursor): 控制数据回放的游标

---

**get**

获取所有字段的最新一条数据

  - <b>参数</b>:
    - 无
  - <b>返回值</b>: 
    - numpy.void：结构体数据

> 示例：要想获得某个字段的数据
>
> ```python
> tick_data  = data.get()
> ask_price_1 = tick_data['ask_price_1']
> ```

---

<b> get_dict </b>

获取最新一条数据, 返回字典

  - <b>参数</b>:
    - 无
  - <b>返回值</b>: 
    - dict: key: 字段, value: 对应字段的数据值

> 示例：要想获得某个字段的数据
>
> ```python
> tick_data  = data.get_dict()
> ask_price_1 = tick_data['ask_price_1']
> ```

---

<b> get_window </b>

获取最新n条数据

  - <b>参数</b>:
    - length (int): window长度
  - <b>返回值</b>: 
    - np.ndarray : 指定长度的数据

> 示例：要想获得某个字段的数据
>
> ```python
> tick_data  = data.get_window(5)
> ask_price_1 = tick_data['ask_price_1']
> # ask_price_1.shape: (5,)
> ```

---

<b> get_future </b>

获取所有字段的未来的一条数据

  - <b>参数</b>:
    - shift (int): 读取未来第shift个时间戳的数据
  - <b>返回值</b>: 
    - dict: key: 字段, value: 对应字段的数据值

> 示例：要想获得下一个时间戳某个字段的数据
>
> ```python
> tick_data  = data.get_future(shift=1)
> ask_price_1 = tick_data['ask_price_1']
> ```

---

<b> get_future_dict</b>

获取所有字段的未来的一条数据, 返回字典

  - <b>参数</b>:
    - shift (int): 读取未来第shift个时间戳的数据
  - <b>返回值</b>: 
    - dict: key: 字段, value: 对应字段的数据值

> 示例：要想获得下一个时间戳某个字段的数据
>
> ```python
> tick_data  = data.get_future_dict(shift=1)
> ask_price_1 = tick_data['ask_price_1']
> ```

---

**get_future_window**

获取从当前时间开始，未来的最近n条数据

  - <b>参数</b>:
    - length (int): 读取未来最近length个时间戳的数据
  - <b>返回值</b>: 
    - np.ndarray : 指定长度的数据

> 示例：要想获得某个字段的数据
>
> ```python
> tick_data  = data.get_future_window(5)
> ask_price_1 = tick_data['ask_price_1']
> # ask_price_1.shape: (5,)
> ```
>>>>>>> e6a1227fe21e477c74e5bcc5e43f0660aa51c3fc
