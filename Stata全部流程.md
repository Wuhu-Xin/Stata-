## https://www.bilibili.com/video/BV1qvwszpE3w/?spm_id_from=333.1007.top_right_bar_window_dynamic.content.click&vd_source=996b116eefec956d8dd6d598b97d87e2

## Stata实证流程

数据导入→变量构建→数据清洗→描述性统计→相关性分析→回归分析→进一步研究【稳健性检验、机制方面】

- 数据处理调整方向：样本筛选条件、变量计算方式、指标构建方法

- 稳健性检验：更换固定效应结构、更换变量的度量方式、调整变量的范围、使用其他处理内生性的方法

- 机制方面：分组检验、中介机制、调节效应

如果遇到结果不显著，那处理的核心思路就是：**理解数据处理流程，在此基础上扩展研究**。

## 一、数据导入与检查（use/describe/count）

### 核心命令与实操

**数据导入**：使用`use`命令导入dta格式数据集，命令格式为`use "数据文件存储路径/文件名.dta", clear`，其中`clear`表示清空Stata原有数据，避免数据冲突。

**数据检查**：使用`describe`（可简写为`desc`）命令，一键输出数据核心信息，包括**观测值数量**、**变量数量**、**每个变量的名称/类型/格式/标签**等。使用`count`命令统计有效观测值数量，确认筛选结果。

### 关键要点

导入后需重点核查核心变量是否存在类型错误（如数值型变量被识别为字符型），若有需及时用`destring`命令转换，例如`destring Year, replace force`将字符型年份转换为数值型。

## 二、样本筛选与变量构建（drop/gen/label)

### 操作目的

根据研究设计剔除不符合要求的样本，保证样本的有效性；同时基于原始数据构建**核心解释变量、因变量、控制变量**，匹配研究主题的变量定义。

### 核心命令与实操

#### （一）样本筛选：`drop/keep`命令

通过条件筛选剔除异常/无效样本：

剔除ST/*ST企业：`keep if LISTINGSTATE == "正常上市"`（保留经营状态正常的企业）；

剔除财务异常样本：`drop if TotalAssets <= 0`、`drop if TotalLiabilities < 0`（排除资产/负债为0或负数的异常观测）；

限定研究时间范围：`keep if Year >= 2010 & Year <= 2024`（聚焦特定时间面板，避免样本跨度不合理）；

筛选后用`count`命令统计有效观测值数量，确认筛选结果。

![](C:\Users\huawei\AppData\Roaming\marktext\images\2026-03-15-08-55-08-image.png)

#### （二）变量构建与标签添加：`gen`命令/`label var`命令

因变量：企业绩效（ROA，总资产净利润率），若原始数据无直接指标，可通过公式生成`gen ROA = 净利润/资产总计`，标签`label var ROA "总资产净利润率"`；

核心解释变量：`gen CEO_Age_Num = CEO_Age`，将原始CEO年龄转换为数值型核心变量，标签`label var CEO_Age_Num "CEO年龄"`；

控制变量：

- 企业规模：`gen Size = ln(TotalAssets)`（总资产取对数，缓解异方差），标签`label var Size "企业规模"`；
- 财务杠杆：`gen Leverage = TotalLiabilities / TotalAssets`（资产负债率），标签`label var Leverage "资产负债率"`；
- 企业年龄：先提取成立年份`gen establishyear_temp = substr(EstablishDate,1,4)`，再计算`gen FirmAge = Year - establishyear_temp`，最后删除临时变量`drop establishyear_temp`。

#### （三）函数补充egen命令

除了简单的取log，还有绝对值···

https://www.stata.com/manuals/fnfunctionsbycategory.pdf#fnFunctionsbycategory

### 关键要点

样本筛选的条件需基于**研究领域通用标准**，保证研究的可复制性；

构建变量时注意**公式的经济意义**，如对数处理、比率计算需符合实证研究惯例；

为所有变量添加清晰标签，避免后续操作中混淆变量含义。

## 三、缺失值与样本统计（egen rmiss/unique）

### 操作目的

精准统计变量缺失值情况，剔除存在缺失值的观测行，保证样本的完整性；同时统计样本的唯一性特征（如企业数量、年份数量），明确研究的面板数据范围，这是数据质量控制的关键步骤。

### 核心命令与实操

#### （一）缺失值处理：`egen rmiss`命令

- 因变量：`global y ROA`

- 自变量：`global x CEO_Age_Num`

- 控制变量：`global controls 控制变量的名字，可多个`

先定义核心变量组：`global all_vars $y $x $controls`（将因变量、核心解释变量、控制变量整合为一个全局宏，方便批量处理）；

![](C:\Users\huawei\AppData\Roaming\marktext\images\2026-03-15-09-43-46-image.png)

统计每行缺失值数量：`egen miss_count = rmiss($all_vars)`，`rmiss`函数会统计指定变量组（即all_vars变量组）中每一行的缺失值个数，生成新变量`miss_count`；

![](C:\Users\huawei\AppData\Roaming\marktext\images\2026-03-15-09-46-30-image.png)

剔除存在缺失值的样本：`drop if miss_count > 0`，删除任意核心变量存在缺失的观测行，即将上表中第1行保留，第2-4行删除；

删除临时变量：`drop miss_count`，清理冗余变量。

#### （二）样本唯一性统计：`unique`命令

用于统计变量的**不重复取值数量**，以此来确定样板中个体的数目：

`unique Stkcd`：统计股票代码的不重复数量，即**研究中的企业数量**；

- 例如stkcd的取值是：12、21、21、31、31，`unique Stkcd`→3

`unique Year`：统计年份的不重复数量，即**研究的时间跨度**；

`unique CEO_Age_Num`：统计CEO年龄的不重复取值，辅助分析变量分布特征。

### 关键要点

用`misstable summarize 变量名1 变量名2`可提前核查单个变量的缺失值数量、有效观测值数量，例如`misstable summarize ROA CEO_Age_Num Size`，精准定位缺失值严重的变量；

![](C:\Users\huawei\AppData\Roaming\marktext\images\2026-03-15-09-40-47-image.png)

`global`全局宏的使用可简化批量变量操作，避免重复输入变量名，提升操作效率；

缺失值处理优先选择**删除法**（本研究采用），若缺失值比例较高，可考虑插补法，需结合研究实际选择。

## 四、极端值处理（winsor2）

### 操作目的

剔除连续变量的极端值（异常值），避免极端值对回归结果的干扰，保证实证结果的稳健性，这是实证研究中**必须进行**的步骤，尤其是上市公司财务数据，易存在极端值问题。

### 核心命令与实操

#### （一）极端值判断

简单描述统计：`summarize ROA`

![](C:\Users\huawei\AppData\Roaming\marktext\images\2026-03-15-10-17-17-image.png)

详细描述统计：`summarize ROA,detail`

![](C:\Users\huawei\AppData\Roaming\marktext\images\2026-03-15-10-47-37-image.png)

通过这个描述性统计，可以看出以下内容：

- 标准差：数据离散程度较高

- 偏度：高度左偏→数据分布向左拖尾，存在较多极端低值

- 峰度：远高于正态分布的峰度3，属于尖峰厚尾分布→极端低值占比高

判断是否需要缩尾处理：

**描述性统计直观判断**

- 极值与分位数差距过大：最小值`-1.089764`远低于1%分位数`-0.260655`最大值远高于99%分位数→存在原理主体分布的极端值

- 偏度/峰度异常：偏度绝对值＞2→分布严重偏斜；峰度＞3→存在大量厚尾极端值→拉高方差、干扰回归系数

- 标准差/均值比值过大：0.0821/0.0340=2.41→离散程度是均值的2.4倍→数据波动剧烈，极端值影响明显

**实操检验法**：对比回归

- 不处理极端值→回归1

- 1%/5%缩尾→回归2

- 对比核心变量系数显著性、符号、大小变化：如果回归1结果不稳定、系数符号反常或显著性忽高忽低，说明极端值干扰大→必须缩尾

**缩尾处理选择**

一般样本量N＜300倾向于使用1%缩尾

#### （二）缩尾处理

![](C:\Users\huawei\AppData\Roaming\marktext\images\2026-03-15-11-31-29-image.png)

安装外部命令：`winsor2`是Stata外部命令，首次使用需安装`ssc install winsor2, replace`；

核心操作：对**所有连续变量**进行**1%和99%分位数缩尾处理**，命令格式为`winsor2 连续变量1 连续变量2, replace cuts(1 99)`；

- `replace`：表示用缩尾后的值**替换原变量**，无需生成新变量；

- `cuts(1 99)`：表示在1%和99%分位数处缩尾，即低于1%分位数的数值替换为1%分位值，高于99%分位数的数值替换为99%分位值。【有些时候可能存在`cuts（5 95）`】

### 关键要点

缩尾处理仅针对**连续变量**（如ROA、CEO年龄、企业规模、资产负债率等），分类变量（如企业性质SOE）无需处理；

缩尾比例通常选择**1%和99%**，这是实证研究的通用标准，也可根据数据分布调整为5%和95%；

处理后可用`summarize 变量名, detail`命令核查变量分布，确认极端值已被有效控制。

## 五、面板设定与分组变量（xtset/egen group）

### 操作目的

将普通数据设定为**面板数据结构**，适配后续的固定效应回归；同时生成分组编号变量（如企业编号、省份编号、行业编号），为后续控制不同维度的固定效应奠定基础。

### 核心命令与实操

#### （一）分组变量生成：`egen group`命令

在进行面板数据设定前，需要确定个体标识和时间标识均为数值型。

通过`egen group`将字符型/分类变量转换为**数值型分组编号**，方便后续控制固定效应：

企业编号：`egen firm_id = group(Stkcd)`，根据股票代码生成唯一的企业数值编号；

省份编号：`egen province_id = group(PROVINCE)`，根据省份名称生成省份数值编号；

行业编号：`egen indId_3 = group(IndustryCode3)`、`egen indId_1 = group(IndustryCode1)`，分别生成行业细类、行业大类编号。

#### （二）面板数据设定：`xtset`命令

Stata中进行面板回归前，必须先通过`xtset`定义**个体标识**和**时间标识**，命令如下：

基础设定：`xtset firm_id Year`，`firm_id`为**个体标识**，`Year`（年份）为**时间标识**；

基于企业编号设定：若已生成企业编号，可使用`xtset firm_id Year`，结果一致。

![](C:\Users\huawei\AppData\Roaming\marktext\images\2026-03-15-11-43-50-image.png)

设定后Stata会输出面板类型：本研究为**非平衡面板数据**（Panel variable: firm_id (unbalanced)），即部分企业并非在研究期内每年都有观测值，这是上市公司数据的常见特征。

![](C:\Users\huawei\AppData\Roaming\marktext\images\2026-03-15-11-44-51-image.png)

### 关键要点

`xtset`是面板回归的**前提操作**，未设定面板结构直接进行固定效应回归会导致结果错误；

`egen group`生成的分组编号为连续数值型，可直接作为`reghdfe`命令的固定效应控制变量；

设定面板后可用`xtsum`命令查看面板变量的描述性统计，进一步了解数据结构。

## 六、描述统计与相关分析（sum2docx/corr2docx）

### 操作目的

完成数据的描述性统计和Pearson相关性分析，初步分析变量的分布特征和变量间的线性关系，为后续回归分析提供初步证据；同时通过专用命令将结果**直接导出为Word规范表格**，无需手动整理，贴合论文写作需求。

### 核心命令与实操

#### （一）描述性统计：`sum2docx`命令

核心命令：将因变量、核心解释变量、控制变量的描述性统计结果导出为Word：

```stata
sum2docx $y $x $controls ///
using "01_描述性统计.docx", ///
replace ///
stats(N mean(%9.3f) sd(%9.3f) min(%9.3f) median(%9.3f) max(%9.3f))///
title("表2.描述性统计")///
font("Times New Roman",11)///
note("注:所有连续变量已经过1%和99%缩尾处理")
```

![](C:\Users\huawei\AppData\Roaming\marktext\images\2026-03-15-12-02-01-image.png)

关键参数说明：

- `using`：指定Word文件的存储路径和文件名；
- `replace`：覆盖同名文件，避免文件重复；
- `stats`：指定输出的统计指标，包括样本量（N）、均值（mean）、标准差（sd）、最小值（min）、中位数（median）、最大值（max），`%9.3f`表示字段宽度9位、保留3位小数；
- `title/font/note`：设置表格标题、字体（Times New Roman为学术论文通用字体）、表格注释。

![](C:\Users\huawei\AppData\Roaming\marktext\images\2026-03-15-12-16-30-image.png)

#### （二）相关性分析：`corr2docx`命令

核心命令：导出Pearson相关系数矩阵为Word，视频中命令为：

```stata
corr2docx $y $x $controls using "02_相关系数.docx",///
replace///
star(*0.1 **0.05 ***0.01)///
fmt(%6.3f)///
title("表3.Pearson相关系数矩阵")///
font("Times New Roman",10)///
landscape
```

关键参数说明：

- `star`：自动标注相关性的显著性水平，*p<0.1、**p<0.05、***p<0.01，是学术论文的通用标注标准；
- `fmt(%6.3f)`：相关系数保留3位小数；
- `landscape`：表格**横向排版**，适配多变量的相关系数矩阵展示，避免表格拥挤。而`Portrait`是竖向排版，一般默认就是竖向排版

![](C:\Users\huawei\AppData\Roaming\marktext\images\2026-03-15-12-21-02-image.png)

### 关键要点

`sum2docx`和`corr2docx`均为Stata外部命令，首次使用需安装，安装命令分别为`ssc install sum2docx`、`ssc install corr2docx`；

描述性统计需重点关注变量的均值、标准差和取值范围，判断是否存在异常；

相关性分析需关注**核心变量间的相关系数**（如CEO年龄与ROA的相关性）和**多重共线性风险**（若两个控制变量相关系数绝对值大于0.7，需考虑剔除其中一个）。

## 七、基准回归与结果导出（reghdfe/reg2docx)

### 操作目的

构建不同维度的固定效应回归模型，检验核心解释变量对因变量的影响（本研究为CEO年龄对企业绩效的影响）；同时将多个回归模型的结果整合到一个Word表格中，规范呈现回归系数、显著性、模型拟合度等核心信息，满足论文写作的结果展示要求。

### 核心命令与实操

#### （一）基准回归：`reghdfe`命令

`reghdfe`是Stata中用于**高维固定效应回归**【Hight-Dimensional Fixed Effects】的核心命令，可同时控制多个维度的固定效应，且支持聚类标准误，视频中构建了**四大固定效应模型**，逐步验证结果的稳健性，核心命令如下（先定义全局宏`global y ROA`、`global x CEO_Age_Num`、`global controls 控制变量列表`）：

**模型1：行业固定效应+年份固定效应**

```stata
reghdfe $y $x $controls, absorb(indId_3 Year) cluster(indId_3)
estimates store m1
```

**模型2：行业大类固定效应+年份固定效应**

```stata
reghdfe $y $x $controls, absorb(indId_1 Year) cluster(indId_3)
estimates store m2
```

**模型3：企业固定效应+年份固定效应**

```stata
reghdfe $y $x $controls, absorb(firm_id Year) cluster(firm_id)
estimates store m3
```

**模型4：省份固定效应+年份固定效应**

```stata
reghdfe $y $x $controls, absorb(province_id Year) cluster(firm_id)
estimates store m4
```

![](C:\Users\huawei\AppData\Roaming\marktext\images\2026-03-15-12-24-11-image.png)

关键参数说明：

- `absorb`：指定需要控制的**固定效应变量**，可同时控制多个（如`indId_3 Year`控制行业细类和年份固定效应）；
- `cluster`：指定**聚类标准误的维度**（如`indId_3`为行业层面聚类，`firm_id`为企业层面聚类），缓解异方差和序列相关问题，是实证研究的通用要求；
- `estimates store`：将每个模型的回归结果保存为指定名称（如m1/m2），方便后续批量导出。

#### （二）结果导出：`reg2docx`命令

将保存的4个回归模型结果整合导出为一个Word规范表格，视频中核心命令为：

```stata
reg2docx m1 m2 m3 m4 using "03_基准回归.docx",///
replace///
b(%9.3f) t(%9.3f)///
scalars(N r2(%9.3f) r2_a(%9.3f))///
title("表4.CEO年龄与企业绩效:基准回归")///
star(*0.1 **0.05 ***0.01)///
font("Times New Roman",11)关键参数说明：
```

- `m1 m2 m3 m4`：按顺序导入已保存的回归模型结果；
- `b(%9.3f) t(%9.3f)`：回归系数（b）和t统计量均保留3位小数，论文中也可选择展示标准误（se），将`t`替换为`se`即可；
- `scalars`：指定输出的模型拟合度指标，包括样本量（N）、拟合优度（r2）、调整后拟合优度（r2_a）；
- `star`：自动为回归系数标注显著性水平，*p<0.1、**p<0.05、***p<0.01。

![](C:\Users\huawei\AppData\Roaming\marktext\images\2026-03-15-12-27-30-image.png)

### 关键要点

`reghdfe`为外部命令，首次使用需安装`ssc install reghdfe`；

固定效应的选择需基于**研究问题和数据特征**，企业固定效应能更好地控制企业层面的不随时间变化的异质性，是上市公司研究的常用选择；

回归结果解读需重点关注**核心解释变量的系数符号、大小和显著性**，同时关注模型的拟合度（R2/调整后R2）和样本量变化；

聚类标准误的维度需与固定效应匹配，避免统计推断偏误。

## 总结

这七大模块的核心逻辑是**先保证数据质量，再进行统计分析，最后开展回归检验**，每一步都有明确的操作目的和通用标准，只要掌握核心命令和参数设置，就能高效、规范地完成Stata实证分析。后续可在此基础上，进一步学习稳健性检验、内生性处理、机制分析、异质性分析等进阶内容，深化实证研究结论。
