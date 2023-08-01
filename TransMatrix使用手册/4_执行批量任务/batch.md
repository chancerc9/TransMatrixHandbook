# 批量任务

系统支持通过 yaml 文件配置任务流，一个任务流包含多个顺序执行的批任务，每个批任务中包含多个可并行子任务。

基于任务流，我们可以轻松实现**批量执行因子研究**或者**批量执行策略研究**的需求。

### 4.1 通过yaml文件配置任务流

我们假设以下场景：

我们要生成 4 个因子，factor_A, factor_B, factor_C 和 factor_D。它们之间的依赖关系如下：
-   factor_A 和 factor_B 相关独立，factor_C 和 factor_D 相关独立
-   factor_C 和 factor_D 依赖于 factor_A 和 factor_B 因子，即需要先生成 factor_A 和 factor_B 因子，才能生成 factor_C 和 factor_D 因子

上述场景的工程文件结构示意：

```text
taskflow
├── evaluator.py
├── factor_A
│   ├── config.yaml
│   └── strategy.py
├── factor_B
│   ├── config.yaml
│   └── strategy.py
├── factor_C
│   ├── config.yaml
│   └── strategy.py
├── factor_D
│   ├── config.yaml
│   └── strategy.py
└── taskflow.yaml
```

通过配置系统批量任务流，我们可以轻松实现上述场景。我们在新建的 taskflow.yaml 文件中，输入以下内容：
```text

# 模式设置
workflow: BatchMatrix
max_child_level: 1 # task 最大穿透目录层数 (defalult 为 1)

# 公有信息
matrix:
    span: [2021-01-01,2021-12-31]
    logging: True

# 任务信息
task:
    - regex factor_[A|B] # 以'regex '开头。利用正则表达式筛选符合条件的因子文件夹。
    - 
        - factor_C
        - factor_D
```

这一文件中包含 3 部分：模式设置，公有信息，任务信息。我们分别来看：
-   模式设置
```text
workflow: BatchMatrix
max_child_level: 1 # task 最大穿透目录层数 (defalult 为 1)
```

这里，我们把 workflow 模式设置为 BatchMatrix，代表当前模式为任务流模式。

max_child_level 设为1，代表系统将搜索当前目录及子目录的文件。即搜索 taskflow 目录，和子目录（例如 factor_A 文件夹），子目录之下的目录，系统不做搜索。

-   公有信息
```text
matrix:
    span: [2021-01-01,2021-12-31]
    logging: True
```

公有信息可以是标准配置文件中 matrix / strategy / evaluator 中的任何信息， 其将覆盖子任务相应为之的配置信息。这里，我们matrix 部分的 span 和 logging 设置是公有信息，则 factor_A 文件夹下的 config.yaml 文件中在 matrix 部分的 span 和 logging 设置将会被公有的设置所覆盖。同理，其他因子文件夹下的对应部分也会被覆盖。

-   任务信息
```
# 任务信息
task:
    - regex factor_[A|B] # 以'regex '开头。利用正则表达式筛选符合条件的因子文件夹。
    - 
        - factor_C
        - factor_D
```

这里，我们配置了 2 个批任务，第 1 个批任务：
```
    - regex factor_[A|B] # 以'regex '开头。利用正则表达式筛选符合条件的因子文件夹。
```

第 1 个批任务，以 regex 开头，系统将使用正则表达式筛选符合条件的因子文件夹。具体而言，从当前入口文件所在目录递归地扫描子文件夹(os.walk)，若某一级文件夹名称调用 re.compile('factor_[A|B]').search()返回True时，该文件夹下的config.yaml（若有）将被加入任务列表。

第 2 个批任务，包含 2 个子任务：
```
    - 
        - factor_C
        - factor_D
```

在执行时，系统的执行顺序如下：
-   首先执行第 1 个批任务，它包含 2 个子任务: factor_A 和 factor_B，子任务将并行执行
-   接着执行第 2 个批任务，它包含 2 个子任务：factor_C 和 factor_D，子任务将并行执行

注意，每个因子文件夹下面都有各自的因子计算逻辑代码和对应的配置文件，参照 4.1 节的工程文件结构示意。出于篇幅考虑，这里不作介绍，具体内容可参考对应的测例。

### 4.2 运行批量任务
我们新建一个 ipynb 文件，命名为 run.ipynb，新建一代码单元格，输入以下代码：
```python
from transmatrix.data_api import create_factor_table
temp_table_name = 'batch_insert'
create_factor_table(temp_table_name) # 创建因子表

from transmatrix.workflow.run_batch import run_matrices
run_matrices('task_flow.yaml') # 运行批量任务
```
代码里，我们在个人空间新建了 1 张因子数据表，然后调用 run_matrices 方法执行批量任务。关于 run_matrices 方法的详细说明，参见[这里](9_workflow/batch.md)。运行该单元格，输出以下内容：
```text
Out:
    2023-06-19 15:38:20.326217: 批量任务开始:
    loading data demo__stock_bar_1day__open,high,low,close from database.

    loading data demo__critic_data__is_hs300,is_zz500,ind from database.

    loading data demo__stock_index__close from database.

    Cashing demo__critic_data__is_hs300,is_zz500,ind to pickle file: dataset_151b3bab-c21a-46a7-8154-6c88e1827f16 ....Cashing demo__stock_index__close to pickle file: dataset_34766158-a39b-4060-a3dc-61e8b21b83b5 ....
    Cashing demo__stock_bar_1day__open,high,low,close to pickle file: dataset_5a7c6838-3b01-44b3-bda0-59cfaa2f8018 ....

    loading data demo__stock_bar_1day__open,high,low,close from datacache.
    loading data demo__critic_data__is_hs300,is_zz500,ind from datacache.
    loading data demo__stock_index__close from datacache.



    loading data demo__stock_bar_1day__open,high,low,close from datacache.
    loading data demo__critic_data__is_hs300,is_zz500,ind from datacache.
    loading data demo__stock_index__close from datacache.



    saving report to /root/workspace/我的项目/功能演示230619/transmatrix特色功能/批量任务-workflow/批量任务/factor_A/report
    saving report to /root/workspace/我的项目/功能演示230619/transmatrix特色功能/批量任务-workflow/批量任务/factor_B/report
    factorA: saving signal factora to batch_insert....
    factorB: saving signal factorb to batch_insert....
    因子库数据更新: xwq1_private batch_insert : ['factora']
    因子库数据更新: xwq1_private batch_insert : ['factorb']
    更新完毕: xwq1_private batch_insert : ['factora']
    更新完毕: xwq1_private batch_insert : ['factorb']
    loading data demo__stock_bar_1day__open,high,low,close from datacache.
    loading data demo__critic_data__is_hs300,is_zz500,ind from datacache.
    loading data demo__stock_index__close from datacache.



    loading data demo__stock_bar_1day__open,high,low,close from datacache.
    loading data demo__stock_index__close from datacache.

    loading data demo__critic_data__is_hs300,is_zz500,ind from datacache.


    saving report to /root/workspace/我的项目/功能演示230619/transmatrix特色功能/批量任务-workflow/批量任务/factor_C/report
    saving report to /root/workspace/我的项目/功能演示230619/transmatrix特色功能/批量任务-workflow/批量任务/factor_D/report
    factorC: saving signal factorc to batch_insert....
    factorD: saving signal factord to batch_insert....
    因子库数据更新: xwq1_private batch_insert : ['factorc']
    因子库数据更新: xwq1_private batch_insert : ['factord']
    更新完毕: xwq1_private batch_insert : ['factorc']
    更新完毕: xwq1_private batch_insert : ['factord']
    2023-06-19 15:38:41.160341: 批量任务完成。
```

由输出结果可知，系统顺序执行了 2 个批任务，每个批任务内的子任务是并行执行的。

> Tips:
> 
> 系统支持 **2 种运行方式**，执行批量任务：
> 
> - **run_matrices** 方法: 在 py 文件或 jupyter 环境中调用本方法
> - **BatchMatrix** 命令：在命令行窗口中输入 BatchMatrix -p [yaml_path] -s [startdate]-[enddate]
>       我们在命令行窗口中输入以下命令： 
> 
>       BatchMatrix -p /root/workspace/我的项目/批量任务-workflow/3.workflow.yaml -s 20210101-20211230 
>
>       系统将运行配置信息文件： "/root/workspace/我的项目/批量任务-workflow/3.workflow.yaml"，并且回测时间段为 2021年1月1日到2021年12月30日

