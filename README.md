# Verilog HDL 设计 64bits 算术乘法器

## 方案一

### 01. 设计方案

- 利用一个FPGA内部 ip 16\*16的无符号整数乘法器，设计状态机分时复用，实现64\*64的无符号整数乘法器。

- 假设被乘数是 a，乘数是 b，将 a 和 b 按16位各分成4个部分，按从低位到高位的顺序分别为 a1, a2, a3, a4; b1, b2, b3, b4; 每个数据16位。

- 将 a 类数据和 b 类数据分别互乘得到16个32位数据：

  |      |  b1  |  b2  |  b3  |  b4  |
  | :--: | :--: | :--: | :--: | :--: |
  |  a1  | p11  | p12  | p13  | p14  |
  |  a2  | p21  | p22  | p23  | p24  |
  |  a3  | p31  | p32  | p33  | p34  |
  |  a4  | p41  | p42  | p43  | p44  |

- 将上一步得到的16个数据，4个一组移位相加，每个数据间移16位，得到4个80位数据：
  
  ```verilog
  row1 = p11 + {p12, 16'b0} + {p13, 32'b0} + {p14, 48'b0};
  row2 = p21 + {p22, 16'b0} + {p23, 32'b0} + {p24, 48'b0};
  row3 = p31 + {p32, 16'b0} + {p33, 32'b0} + {p34, 48'b0};
  row4 = p41 + {p42, 16'b0} + {p43, 32'b0} + {p44, 48'b0};
  ```
  
- 最后把得到的4个80位数据，移位相加，每个数据间移16位，得到两数相乘的结果：

  ```verilog
  final_p = row1 + {row2, 16'b0} + {row3, 32'b0} + {row4, 48'b0};
  ```

### 02. 提交代码

```
src1/
----ip_tb.v				// 底层ip的测试文件
----mult64x64_check.v	// testbench中用到的模块
----mult64x64_top.v		// 工程的设计文件
----testbench.v			// 工程的测试文件
```

### 03. 仿真结果

> 写了一个参考模块，直接用\*号实现，测试100次两数相乘，比对设计模块与参考模块的输出是否一致。若一致，计数器不变，计数器初始为0；若不一致，计数器加一。结果以文字形式打印和波形图呈现。

#### 3.1 前仿（功能仿真）

文字形式打印，下图为一部分截图：

<img src=".\img1\word_rtl_sim.jpg" />

波形图，可以看到计数器cnt一直为0，证明设计没有错误：

<img src=".\img1\wave_rtl_sim.jpg" />

下图是局部放大图，时钟周期设定是12ns，输入与输出间延迟192ns，即16个周期，与设计相符，因为只用到了一个16\*16乘法器，需要计算16次，不考虑时序，理论延迟16个周期：

<img src=".\img1\wave_delay_rtl_sim.jpg" />

#### 3.2 后仿（时序仿真）

##### 3.2.1 时序约束

我只对时钟进行了约束：`create_clock -name "clock" -period 12.000ns [get_ports {clk}] -waveform {0.000 6.000}`

约束结果如下图，可以看出三个模型都满足时序要求：

<img src=".\img1\timing_analyzer.jpg" />

把`Slow 1200mV 85C Model`展开如下图：

<img src=".\img1\Fmax_slow_1200_85_model.jpg" />

<img src=".\img1\set_up_slow_1200_85_model.jpg" />

<img src=".\img1\hold_slow_1200_85_model.jpg" />

把`Slow 1200mV 0C Model`展开如下图： 

<img src=".\img1\Fmax_slow_1200_0_model.jpg" />

<img src=".\img1\set_up_slow_1200_0_model.jpg" />

<img src=".\img1\hold_slow_1200_0_model.jpg" />

##### 3.2.2 门级仿真

> 以`Slow 1200mV 85C Model`为模型进行门级仿真

文字形式打印，一部分截图如下，由于加了时序，存在输入输出延迟，设计输出一开始为不定态，而参考模块正常输出结果0，所以第一次初始化时结果不一致，后续结果都正确：

<img src=".\img1\word_gate_sim.jpg" />

波形图，计数器cnt一直为1，原因如上：

<img src=".\img1\wave_gate_sim.jpg" />

下图为局部放大图，输入输出延迟223.594ns，相较理论延迟增加了31.594ns，约为2.63个周期：

<img src=".\img1\wave_delay_gate_sim.jpg" />

<img src=".\img1\wave_delay1_gate_sim.jpg" />

### 04. 硬件资源

<img src=".\img1\fitter_summary.jpg" />

面积要求和速度性能是矛盾的，设计时应根据需求采取折衷措施，以面积换速度，或以速度换面积。

## 方案二

**方案一是方案二的改进，以方案一为准。**

方案一中的设计代码部分，我增加了一个输入端口作为使能信号`en`。我认为这样做的好处在于，在使用时会更方便，当使能信号有效时，才进行乘法运算；在后仿真时会更加方便。

在这个方案里进行的工作，只是把使能信号输入端去掉，微改方案一的代码。

### 提交代码

```
src2/
----mult64x64_check.v	// testbench中用到的模块
----mult64x64_top.v		// 工程的设计文件
----testbench.v			// 工程的测试文件
```

### 仿真结果

前仿与方案一大同小异，不再赘述，波形图如下：

<img src=".\img2\wave_rtl_sim.jpg">

后仿真时，时序约束与方案一相同，得到的综合结果与方案一略有不同：

`Slow 1200mV 85C Model`

<img src=".\img2\Fmax_slow_1200_85_model.jpg" />

<img src=".\img2\set_up_slow_1200_85_model.jpg" />

<img src=".\img2\hold_slow_1200_85_model.jpg" />

`Slow 1200mV 0C Model`

<img src=".\img2\Fmax_slow_1200_0_model.jpg" />

<img src=".\img2\set_up_slow_1200_0_model.jpg" />

<img src=".\img2\hold_slow_1200_0_model.jpg" />

以`Slow 1200mV 85C Model`为模型进行门级仿真结果如下：

<img src=".\img2\word_gate_sim.jpg" />

<img src=".\img2\wave_delay_gate_sim.jpg" />

可以看到前仿真输入输出延迟为336ns，周期设定12ns，即延迟28个周期；后仿真输入输出延迟364.282ns，即延迟30.36个周期。对比方案一，前仿和后仿的相对时间差是接近的，都为两个多周期，但是绝对延迟周期莫名多了12个周期。对此分析原因可能是测试代码写得有问题，或者是设计的某方面时序处理有问题。

总结：方案一是方案二的改进，以方案一为准。

## 方案三

> 参考资料：https://zhuanlan.zhihu.com/p/127164011

### 设计方案

先对乘数基于基4的Booth编码产生33个部分积，再经过基于4-2压缩的加法树处理，最后加上流水线设计，得到64\*64乘法器的最终结果，并且该设计支持无符号和有符号数两种输入形式。

1. 设`[63:0]A`和`[63:0]B`是乘法器的两个输入，A为被乘数，B为乘数。

2. 对乘数B以基4的方式进行Booth编码，由于Booth编码是一种有符号数的编码方式，所以要在乘数B的最高位扩展一位`B[64]`兼容无符号数，另外在最低位的右边要扩展一个附加位`B[-1]`，其值为零；从最低位开始，3bit为一组，相邻组的最高位和最低位重合，即`B[1:-1]`为第一组，`B[3:1]`为第二组，依此类推，最后一组只有2bit，要在最高位再加一位`B[65]`，使之为3bit一组，总共分为33组：

   |     1      |     2      |     3      |     4      |     5      |     6      |
   | :--------: | :--------: | :--------: | :--------: | :--------: | :--------: |
   | `B[1:-1]`  |  `B[3:1]`  |  `B[5:3]`  |  `B[7:5]`  |  `B[9:7]`  | `B[11:9]`  |
   | `B[13:11]` | `B[15:13]` | `B[17:15]` | `B[19:17]` | `B[21:19]` | `B[23:21]` |
   | `B[25:23]` | `B[27:25]` | `B[29:27]` | `B[31:29]` | `B[33:31]` | `B[35:33]` |
| `B[37:35]` | `B[39:37]` | `B[41:39]` | `B[43:41]` | `B[45:43]` | `B[47:45]` |
   | `B[49:45]` | `B[51:49]` | `B[53:51]` | `B[55:53]` | `B[57:55]` | `B[59:57]` |
   | `B[61:59]` | `B[63:61]` | `B[65:63]` |            |            |            |
   
3. 以上述的每组值对应基4Booth编码查找表（如下表所示），可以得到33个部分积PP1~PP33：

   | $x_{2i+1}$ | $x_{2i}$ | $x_{2i-1}$ | $PP_i$ |
   | :--------: | :------: | :--------: | :----: |
   |     0      |    0     |     0      |   0    |
   |     0      |    0     |     1      |   Y    |
   |     0      |    1     |     0      |   Y    |
   |     0      |    1     |     1      |   2Y   |
   |     1      |    0     |     0      |  -2Y   |
   |     1      |    0     |     1      |   -Y   |
   |     1      |    1     |     0      |   -Y   |
   |     1      |    1     |     1      |   0    |

4. 4-2压缩器有4个数据输入端和1个进位输入端，数据输入端接4个部分积相应的bit位，每个bit对应的进位输入端接其前1位的co输出，第0位的进位输入端接零值，无法加入到4-2压缩器输入的部份积保留到下一级，直到可以参与运算为止。注意两点：一是最终结果为128位，中间运算中凡是超出128位的数不进行计算，直接舍弃；二是自始至终必须保证数位对齐不能改变。对33个部分积基于4-2压缩器构建加法树，拓扑结构如下：

   <img src=".\img3\wtree拓扑图.svg" />

5. 具体的华莱士树分析见文件夹`.\doc3\w_tree_analyse.xlsx`。

6. 在每一级加入流水线，提高工作频率。

### 提交代码

```
src3/
----mult64x64_top.v			// 工程顶层模块
----booth_r4_64x64.v		// 基4Booth编码
----wtree_4to2_64x64.v		// 基于4-2压缩器的华莱士树
----counter_5to3.v			// 4-2压缩器
----full_adder.v			// 全加器（CSA进位保留加法器）
----half_adder.v			// 半加器
----testbench.v				// 测试模块
----mult64x64_check_s.v		// testbench中有符号数参考模块
----mult64x64_check_us.v	// testbench中无符号数参考模块
```

### 仿真结果

#### 前仿真

波形图如下，计数器cnt一直为0，说明设计输出与参考输出一致，功能正确：

<img src=".\img3\wave_rtl_sim.jpg" />

放大图如下，输入输出延迟为80ns，设定周期为16ns，即输入输出延迟5个周期，由于这是纯组合逻辑，我在设计中加入了5级流水线，所以，理论上延迟5个周期，没有问题：

<img src=".\img3\wave_delay_rtl_sim.jpg" />

#### 后仿真

1. 时序约束：只加了时钟约束`create_clock -name "clock" -period 10.000ns [get_ports {i_clk}] -waveform {0.000 5.000}`

2. 综合结果如下，可以看出三个模型都符合时序要求：

   <img src=".\img3\timing_analyzer.jpg" />

   `Slow 1200mV 100C Model`下最大工作频率为：

   <img src=".\img3\Fmax_slow_1200_100_model.jpg" />

   `Slow 1200mV -40C Model`下最大工作频率为：

   <img src=".\img3\Fmax_slow_1200_-40_model.jpg" />

3. 以`Slow 1200mV 100C Model`模型进行门级仿真，波形图如下，cnt一直为0，说明功能正确，`tc=0`表示无符号数计算，`tc=1`表示有符号数计算：

   <img src=".\img3\wave_gate_sim.jpg" />

   局部放大图如下，输入输出延迟为115.394ns，设定周期16ns，即延迟7.21个周期；相比理论延迟增加了35.394ns，即2.21个时钟周期：

   <img src=".\img3\wave_delay_gate_sim.jpg" />

### 硬件资源

<img src=".\img3\fitter_summary.jpg" />

|        | Total logic elements | Total registers |   Fmax    |
| :----: | :------------------: | :-------------: | :-------: |
| 方案一 |         1019         |       530       | 86.45MHz  |
| 方案三 |        12286         |      4790       | 188.01MHz |

以上对比说明，面积小了，功耗低了，速度慢了，性能降了；性能升了，速度快了，功耗高了，面积大了；面积和速度需要根据需求进行折衷，达到平衡。