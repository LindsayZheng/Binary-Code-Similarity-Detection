| **Title** | Improving Binary Code Similarity Transformer Models by Semantics-Driven Instruction Deemphasis |
|----------|-------------|
| **Author** | Xiangzhe Xu |
| **Institution** | Purdue University |
| **Venu** | ISSTA |
| **Year** | 2023 |

# ABSTRACT
1. 现有方法的缺陷
   - 在特定编译器的惯例下，容易出现不良的指令分布**偏差**
   - i.e.,
     - 不同编译器或编译设置会在生成二进制代码时，插入某些“无关“指令，这些指令可能会被模型**错误的判别为重要特征**，进而影响模型的学习和性能
2. 本文方法
   - 对于这种无关指令提出一种新的方法，能够在数据集中**删除相应的无关指令**，从而修正这种偏差


# CONCLUSION
本文提出了一种判断某条指令重要性分析的方法，并实现从数据集中删除语义无关的指令


# 1. INTRODUCTION
1. 编译器会有特定行为产生偏差指令，进而影响模型的学习和性能
   - i.e., 编译器会在函数前生成特定的指令序列
2. 本文方法
   - 1. 确定对分类具有**重要影响**的指令
   - 2. 确定这些重要影响的指令的**语义是否具有重要性**
   - 3. 删除没有语义重要性，且具有重大影响的指令
3. 本文贡献点
   - 1. 指令减重技术：对程序功能或语义没有实际影响的指令
   - 2. 识别对 BCSD 模型具有严重影响的指令
   - 3. 衡量语义重要性的指标
   - 4. DiEmph


# 2. MOTIVATION
## 2.1. Motivating Example
![alt text](<images/Improving Binary Code Similarity Transformer Models by Semantics-Driven Instruction Deemphasis/img.png>)
1. 函数解释 `xrealloc`
   1. 作用：调整已分配内存区域的大小
   2. 参数：指向内存区域的`指针`，调整后的`空间的大小`
   3. 内部结构
      1. 第一个 `if` 表示新空间为 `0` 时，释放空间，并返回空指针
      2. 第 `11` 行表示重新 `n` 大小的空间 
      3. 第二个 `if` 表示如果返回的 `p` 时空指针，则调用 `xalloc_die` 函数处理错误信息
2. CFG 解释
   1. 第一个基本块包含函数前序
      - 保存寄存器等等
   2. 第 2,3 个基本块包含函数**一般情况下**的程序逻辑
      - 调用 `realloc` 并返回结果
   3. 第 4 个基本块表示重新分配空间为 0 时，返回空指针
   4. 第 5 个基本块用于处理异常情况


## 2.2. Limitations in State-of-the-Art Models
1. 从 `xrecalloc` 函数看现有模型 `jTrans` 的局限性
   - 1. 面对 (b) 和 (c) 时，`jTrans` 应该判定为相似函数，但它给出的相似性分数很低
     - 同一源函数，但是使用了不同的编译器和优化等级
   - 2. 原因：(b) 有 `endbr64` 指令，而 (c) 中没有，因此，`jTrans` 认为两者不相似
     - `endbr64` 是编译器 `GCC` 插入的额外指令，并不改变函数语义
   - 3. 手动 `endbr64` 指令后，相似性得分得到显著提升，验证了这个原因的猜想
2. 测试带有 `endbr64` 指令的函数的相似性
![alt text](<images/Improving Binary Code Similarity Transformer Models by Semantics-Driven Instruction Deemphasis/img-1.png>)
   -  两个函数中只有一个函数带有 `endbr64`，不相似函数对远多于相似函数对（493k 对 18k，即 27:1），而两者被设定的应该约等于（2:1）
     - 这种编译器添加语义无关指令的行为严重影响了模型的性能
3. **部分偏差指令**带有特定功能，并不能简单删除，需要进行筛选


## 2.3. Our Technique
1. 核心思想
   - 将不表示关键程序语义的指令弱化处理，例如 `endbr64`
2. 步骤
   1. 确定对整个训练集的**分类结果**产生**重要影响**的指令
      - 通过**移除第 $i$ 条指令及其后续指令**所产生的**嵌入向量变化**判断
   2. 确定**重要指令**的**语义重要性**
      - 如果指令 $i$ 出现在稳定变量 $S$ 的反向切片中，则视为**语义重要的指令**
   3. 从训练数据集中删除**具有分类重要性，但不具有语义重要性**的指令
3. i.e.,
   - 1. (b) 中的 `endbr64` 就是具有分类重要性，但不具有语义重要性的指令；其分类重要性为 `0.1`，甚至大于 `call realloc` （0.05），这明显是不合理的
   - 2. (b) 的第九行是 `ret` 指令，将 `rax` 的值返回，估 `rax@9` 被认为是稳定变量；往前追溯（反向切片）找到所有会影响 `rax@9` 的指令


# 3. DESIGN
![alt text](<images/Improving Binary Code Similarity Transformer Models by Semantics-Driven Instruction Deemphasis/img-2.png>)
1. 总体步骤（图中最上方
   1. 使用大规模的原始数据集对模型进行预训练，得到 `Pretrained Model`
   2. 应用 `DiEmph` 模型对原始数据集进行处理，得到精简数据集，再对 `Pretrained Model` 进行 FuneTuning
2. DiEmph 对数据集进行处理（图中绿框
   1. 分类重要性分析：从训练数据集中采样 $N$ 个函数，计算这些函数中每条指令的**分类重要性**
   2. 筛选高分类重要性的指令：从所有指令中筛选出分类重要性**异常高**的指令
   3. 语义重要性分析：利用程序分析方法判断这些高分类重要性的指令是否对具有**语义重要性**（是否直接或者间接影响稳定变量
   4. 移除无语义的重要指令：对于那些具有高分类重要性但无语义重要性的指令
   5. 重新微调模型：使用去除了无关指令的训练数据集重新微调模型（FineTuning
   6. 推理阶段移除问题指令：在推理阶段，从模型输入中**同样移除这些问题指令**，以确保微调和测试阶段的输入空间一致性，从而保证模型的稳定性和一致性


## 3.1. Classification Importance Analysis
1. 核心思想
   - **如果移除一条指令后函数嵌入发生显著变化，那么相似性查询结果也可能会发生显著变化**；因此，视该指令对分类结果非常重要
   - 显著变化的计算基于**原指令嵌入和移除指令后的余弦相似度的差值**
![alt text](<images/Improving Binary Code Similarity Transformer Models by Semantics-Driven Instruction Deemphasis/img-3.png>)
2. 高分类重要性筛选算法
   1. 输入：从原始数据集中采样的 $N$ ge hanshu 
   2. 输出：指令重要性的 $top@K$
   3. `3-10` 循环处理每个函数
      1. `allInstrs` 存储当前函数中的每条指令及其分类重要性
      2. 对于函数 $f$ 中的每条指令 $i$
         - 得到函数 $f$ 的嵌入向量 $emb(f)$
         - 得到 $f$ 去除 $i$ 后的嵌入向量 $emb(f\setminus \{i\})$
         - 根据余弦相似度得到 $i$ 的分类重要性，并添加到 `allInstrs` 中
      3. 使用 `KDE` 识别出异常值（分类重要性显著高）
      4. 对于检测出的异常指令，将它们在 `importantInstr` 中的计数加 1
3. 异常值检测 `KDE`
   1. 作用：在每个函数内部，算法使用核密度估计（KDE）方法来检测分类重要性值的异常指令。KDE 通过拟合一个概率分布来识别分类重要性值显著高于其他指令的异常值
   2. 方法：给定指令 $i$，算法计算其分类重要性值的概率 $P(X<I_c(i))$，其中 $X$ 表示拟合分布的随机变量；如果这个概率接近 1，则表示该指令的重要性显著高于其他指令


## 3.2. Semantics Importance Analysis
- 核心思想
    - 如果一条指令在稳定变量的反向切片中频繁出现，则认为它具有语义重要性
### 3.2.1. 稳定变量的检测
1. 稳定变量的特点
   - 编译器和编译选项的更改不敏感
2. 稳定变量的类别
   - 全局变量、堆变量、返回变量和作为实际参数传递的变量
3. 稳定变量的识别
   1. **全局变量和堆变量**： 通过**对内存访问的地址操作数**进行反向切片来判断是否访问了全局或堆地址
      1. i.e., 全局变量
         - 识别 `mov [rdi], 0` 是对内存地址操作
         - **反向切片**找到 `rdi@10` 的定义位置，即 `lea rdi, GLOBAL_VAR`
         - 判断 `rdi@10` 为全局变量
      2. i.e., 堆变量
         - 识别 `mov [rbx+0x10],0` 对内存地址操作
         - **反向切片**找到 `rbx@10` 的定义位置，即 `mov rbx, rax`，即 `rbx@10` 源自 `rbx@2`
         - **反向切片**找到 `rbx@2` 的定义位置 >> `rbx@2` 是 `call malloc` 的返回值 >> `malloc` 用于分配堆内存
         - 判断 `rbx10` 为堆变量
   2. **返回变量**：DiEmph 通过检查调用该函数的其他位置，**观察是否在调用后直接使用 `rax` 寄存器（返回值寄存器）**，来判断该函数是否具有返回变量
      - i.e. `ret` 如果没有返回值，它就不作为稳定变量
      - `ret@1` 是 `f@10` 的返回值
      - 找到 `f@10` 的调用处 >> `mov rbx, rax` 中使用了 `rax` >> 认为 `f@10` 即 `ret@1` 有返回值 >> 稳定变量
      - **如果有的 `ret` 找不到调用点，则保守的视为有返回值**，即稳定变量
   3. **函数实际参数**：由于前六个寄存器用于参数传递，这些寄存器中的值在被调用函数内是相对稳定的，不易受到编译器优化的影响。因此，利用这些寄存器数据流，可以识别被传递给函数的实际参数
      - i.e., 
      - 1. 进入被调用函数后未定义即使用的寄存器：如果某个寄存器在被调用函数中使用，而未在函数内重新定义，则可以认为这个寄存器是传递的参数
        - `rdi@10` 直接使用 `rdi` 寄存器 >> `rdi@2` 存的是 `gee` 的参数 >> `rdi@2` 为稳定变量
      - 2. 传递参数的隐式调用
        - `gee@3` 调用 `foo@11` >> `foo@11` 中直接使用未定义的 `rsi@20` >> `rdi@10` 为稳定变量


### 3.2.2. Semantics Importance via Binary Slicing
1. 反向切片的挑战
   - 二进制代码大多**采用地址访问变量**，在进行反向切片的时候，需要判断**某个内存的读取是否依赖某个内存的写入**，但现有方法难以准确识别
   - `DiEmph` 采用保守方案
     - 1. **全局和堆内存**：默认任何堆这些区域的读取**都依赖写入**
     - 2. **对于栈内存**：实现**准确判断**
2. 排除栈地址操作相关的原因
   - 尽管这些栈地址计算对栈布局有影响，但它们并不反映源代码中的实际语义，只是编译器在栈上的实现细节
3. 方法
   在程序的某个点 `p` 上，分析是否变量 `v` 含有栈地址。算法判断 `v` 是否包含栈地址的依据是：`v` 在所有到达 `p` 的路径上都必须包含栈地址


# 4. EVALUATION
## 4.2. RQ1: Performance Improvement on the Out-of-Distribution Dataset
1. 研究问题：**DiEmph 是否能够提升模型性能**
2. 实验方法：
   1. 输入：DiEmph 接收模型和训练数据集，并输出一组需要降权的指令
   2. 模型改进过程：从训练数据集中**移除高频出现的前四个问题指令**，然后对模型进行重新微调，生成改进模型
   3. 测试设置：在 **Out-of-Distribution 数据集**上测试基线模型和改进模型的性能，并保持输入空间的一致性
      - Out-of-Distribution 数据集和训练数据集没有重叠
![alt text](<images/Improving Binary Code Similarity Transformer Models by Semantics-Driven Instruction Deemphasis/img-4.png>)
![alt text](<images/Improving Binary Code Similarity Transformer Models by Semantics-Driven Instruction Deemphasis/img-5.png>)
3. 实验结果 
   1. DiEmph 对模型在 Out-of-Distribution 数据集上的性能有**显著提升**
   2. 最高 14.4% 
      - jTrans 在 BinaryCrop 数据集上
   3. 最低 3.7%
      - 原因可能是数据集过小，即多样性较低；或者模型在特定数据集中并没有相应的特殊结构捕捉特征
   4. 图 6 表示 `DiEmph` 模型对训练集进行删减后，可以显著提高模型的泛化能力 


## 4.3. RQ2: Effectiveness with Different Pool Sizes
![alt text](<images/Improving Binary Code Similarity Transformer Models by Semantics-Driven Instruction Deemphasis/img-6.png>)
1. 研究问题：**候选池大小对 `DiEmph` 的影响**
2. 实验结果：**DiEmph 在所有不同池大小下都能有效提升模型性能**，尤其在池大小大于 100 时


## 4.4. RQ3: Effects on In-Distribution Data
1. 研究问题：**`DiEmph` 是否会影响模型在分布内数据集上的性能**
2. 实验结果：性能平均提升 `3%`
3. 分析：分布内数据集中，现有模型和原始数据集已经足够让模型拥有非常好的性能；但 `DiEmph` 还是能通过删除没有语义重要性的重要指令的方式提升模型性能，这样可以让模型聚焦于那些真正重要的指令


## 4.5. RQ4: Run Time Efficiency
DiEmph 分析一个模型耗时 29 分钟（从训练数据中随机抽取 200 个函数作为样本），且是一次性工作


## 4.6. RQ5: Ablation Study
![alt text](<images/Improving Binary Code Similarity Transformer Models by Semantics-Driven Instruction Deemphasis/img-7.png>)
1. 研究问题：`DiEmph` 中每个组件的作用
2. 各组件效果分析
   1. **No-Stack**：不跟踪栈指针且不修剪由栈操作引入的依赖，**性能略有提升，但不如完全体**
      1. 在优化后的数据集中，依然可能保有部分不具有语义重要性的指令
         - 删除指令时，模型的策略更加保守
      2. 在 Out-Of-Distribution 数据集中的性能更差，即泛化能力下降
   2. **No-Sem**：不分析语义重要性，**性能略有提升，但有限**
      - 可能会删除**具有语义重要性的指令**
   3. **No-Class**：不分析分类重要性，**几乎没有提升**
3. 随机种子
   - 不同随机种子下，`DiEmph` 的性能具有一致性
4. 采样数量
   - 在采样数量大于 `50` 后，模型性能基本不变（所以 50 也是可以的？
5. 移除指令数量
   - 移除过多会导致模型无法准确学习语义


# Related Knowledge
## 1. 稳定变量（2.3）
1. 定义
   - 指在**不同编译选项下**仍保持一致的变量，在编译优化过程中不易发生变化
2. 语义重要性
   - 如果某条指令影响了稳定变量的值，则在语义上被视为重要，**这意味着该指令对程序的核心功能或逻辑有直接或者间接的影响**
3. i.e.,
   - 局部变量 `int`
   - 可能会放在不同的寄存器中，位置发生变化，即在不同编译条件下是不稳定的


## 2. 反向切片分析（2.3）
1. 定义
   - 一种程序分析技术，目的是追溯某个变量的计算过程，找出所有对该变量产生影响的指令
2. i.e.,
   -  稳定变量 $S$
   -  在程序的某个断点（例如，稳定变量 $S$ 的值被使用或返回的地方）之前，追溯并找到所有对 $S$ 有影响的指令或代码行


## 3. KDE（3.1）
1. 定义
   - 一种用于估计随机变量概率密度函数的非参数方法。它通过平滑数据点，来生成一个连续的概率密度曲线，从而近似数据的分布
2. 步骤
   1. KDE 生成一条平滑的概率密度曲线，这条曲线表示每个分类重要性值的“出现概率”
   2. 通过 KDE，可以计算出某些分类重要性值的概率。如果某个值的出现概率很低（即靠近尾部），则该值被认为是异常值