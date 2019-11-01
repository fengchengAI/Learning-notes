## <center>C++ 源码基于softmax回归的MNIST分类<center> ##



有关定义如下:

$X\in \lbrace n,m\rbrace,Y\in \lbrace o,m\rbrace,W\in \lbrace o,n\rbrace$

n为数据维度,m为样本数量,o为输出类别数.

$X$为输入矩阵,$Y$为输出矩阵(one-hot),数据按列排列,W为权重信息(此处省略b)



### 单个样本时:

$softmax$定义如下:
$$
\begin{equation*} p_i = \frac {e^{z_i}}{\sum_{k=1}^{C} e^{z_k}} \end{equation*} \tag{1}
$$
假设网络最后一层的输出为**z**，经过softmax后输出为**p**，真实标签为**y**（one hot编码），则损失函数为：
$$
\begin{equation*}
L = - \sum_{i=1}^{C} y_i \log p_i
\end{equation*}  \tag{2}
$$

其中*C*表示共有*C*个类.

 对softmax loss层求导，即求$\frac{\partial L}{\partial \mathbf{z}}$，可以通过求$\frac{\partial L}{\partial z_j}$进行说明$j\in \lbrace1 ...  C\rbrace$。

$$
\begin{equation*}
\begin{aligned}
\frac {\partial L}{\partial z_j} &= - \sum_{i=1}^{C} y_i \frac{\partial \log p_i}{\partial z_j} \\
&= - \sum_{i=1}^{C} \frac{y_i}{p_i} \frac{\partial p_i}{\partial z_j}
\end{aligned}
\end{equation*}   \tag{3}
$$


$\frac{\partial p_i}{\partial z_j}$的求解分为两种情况，即$i = j$和$i \neq j$，分别进行推导，如下：
$$
\begin{equation*}
\begin{aligned}
i = j 时：\\
\frac{\partial p_i}{\partial z_j} &=  \frac{\partial p_j}{\partial z_j} \\
&= \frac{\partial \frac {e^{z_j}}{\sum_{k=1}^{C} e^{z_k}}}{\partial z_j} \\
&= \frac {e^{z_j}}{\sum_{k=1}^{C} e^{z_k}} +  e^{z_j} \times (-1) \times {(\frac{1}{\sum_{k=1}^{C} e^{z_k}})}^2 \times e^{z_j}\\
&= p_j - p_j^2 \\
&= p_j(1-p_j)
\end{aligned}
\end{equation*}
$$

$$
\begin{equation*} \begin{aligned} i \neq j 时：\\ \frac{\partial p_i}{\partial z_j} &=  e^{z_i} \times (-1) \times {(\frac{1}{\sum_{k=1}^{C} e^{z_k}})}^2 \times e^{z_j}\\ &= -p_ip_j \end{aligned} \end{equation*}
$$

故有， 

$$
\begin{equation*}
\begin{aligned}
\frac{\partial L}{\partial z_j} &= -\frac{y_j}{p_j}p_j(1-p_j) - \sum_{i\neq j} \frac{y_i}{p_i}(-p_ip_j) \\
&= -y_i + p_j \sum_{i=1}^{C} y_i \\
&= p_j - y_j
\end{aligned}
\end{equation*}   \tag{4}
$$



### 多个样本

在下面例子中,设定$m=4,n=3,o=2$

计$X,Z,P$如下

$$
X =\begin{bmatrix}
 	x_{1} & x_{2}& x_{3} & x_{4} 
 	\end{bmatrix} ,
 	Z =\begin{bmatrix}
 	z_{1} & z_{2}& z_{3} & z_{4} 
 	\end{bmatrix},
    P =\begin{bmatrix}
p_{1} & p_{2}& p_{3} & p_{4} 
\end{bmatrix}
$$

$$
W =\begin{bmatrix}
 	w_{11} & w_{12}& w_{13}  \\
 	w_{21} & w_{22}& w_{23}
 	\end{bmatrix},
  X =\begin{bmatrix}
 	x_{11} & x_{12}& x_{13} & x_{14} \\
 	x_{21} & x_{22}& x_{23} & x_{24} \\  
 	x_{31} & x_{32}& x_{33} & x_{34}
 	\end{bmatrix},
$$


$$
Z =\begin{bmatrix}
 	z_{11} & z_{12}& z_{13} & z_{14} \\
 	z_{21} & z_{22}& z_{23} & z_{24} 
 	\end{bmatrix} =\begin{bmatrix}
 	w_{11} & w_{12}& w_{13}  \\
 	w_{21} & w_{22}& w_{23}
 	\end{bmatrix}\begin{bmatrix}
 	x_{11} & x_{12}& x_{13} & x_{14} \\
 	x_{21} & x_{22}& x_{23} & x_{24} \\  
 	x_{31} & x_{32}& x_{33} & x_{34}
 	\end{bmatrix}    \tag{5}
$$

$$
P =\begin{bmatrix}
p_{11} & p_{12}& p_{13} & p_{14} \\
p_{21} & p_{22}& p_{23} & p_{24} 
\end{bmatrix}
 =softmax\lbrace \begin{bmatrix}
z_{11} & z_{12}& z_{13} & z_{14} \\
z_{21} & z_{22}& z_{23} & z_{24} 
\end{bmatrix}\rbrace
$$


其中:			
$$
\begin{equation*} p_{ij} = \frac {e^{z_{i,j}}}{\sum_{i=1}^{C} e^{z_{i,j}}} \end{equation*} (i为行,j为列)   \tag{6}
$$

有公式(4)得:
$$
\begin{equation*}\begin{aligned}\frac{\partial L}{\partial Z_{ij}} = P_{ij} - Y_{ij}\end{aligned}\end{equation*}  (i为行即类别数,j为列即样本数)   \tag{7}
$$

有公式(5)得:

$$
\frac{\partial Z}{\partial W} = X^T
$$

由公式6,7得,表示成向量形式有 
$$
\begin{equation*}
\frac{\partial L}{\partial {W}} = (P -Y)X^T
\end{equation*}
$$




本文主要参考文章:

1:[softmax loss层的求导反向传播](https://blog.csdn.net/b876144622/article/details/80958092). 