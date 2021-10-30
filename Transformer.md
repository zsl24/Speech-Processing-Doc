# Transformer 变形金刚
![image](https://user-images.githubusercontent.com/40049927/138542043-dac8cc56-7230-4709-b0e4-7fec4db276f5.png)  
模型的左侧为`Encoder`，将输入序列x=(x<sub>1</sub>,x<sub>2</sub>,...,x<sub>n</sub>)映射到z=(z<sub>1</sub>,z<sub>2</sub>,...,z<sub>n</sub>)。z作为`Decoder`的`Multi-Head Attention`输入。给定z,`Decoder`产生输出序列y=(y<sub>1</sub>,y<sub>2</sub>,...,y<sub>m</sub>)，每次产生一个点。`Decoder`是一个自回归模型，即每次`Decoder`的输入是上一个step的输出。  
举例：  
> 利用Transformer完成中文到英文的翻译，例如“机器学习”→“Machine Learning”  
> `Encoder`的输入是“机器学习”对应的Embedding，然后输出隐变量z  
> 第一步，我们向`Decoder`输入BOS，代表语句开头，`Decoder`通过输入的字符以及`Encoder`输出的隐变量z，输出“Machine”；  
> 第二步，"Machine"作为`Decoder`的输入，`Decoder`输出“Learning”；  
> 第三步，翻译结束  


## 自注意力机制 Self-attention

自注意力机制是一种能够通过关联一个序列当中不同位置的点而形成这个序列的表征的注意力机制。

## Layer Normalization vs. Batch Normalization

Batch Norm: 计算一个批内样本的均值和方差，以及训练两个参数λ和β。  
Layer Norm: 计算一个样本的feature求均值和方差。

### 为什么Layer Norm使用的更多？  
一般来说，现在样本点数据很多情况下都是一个序列，序列的每个点都有特征向量，样本与样本之间的序列长度不尽相同。在这种情况下，batch norm均值和方差的抖动会非常大，尤其是在计算全局的均值和方差，可能会跟一个批的均值方差差距很大。Layer Norm则不会出现这种情况。
