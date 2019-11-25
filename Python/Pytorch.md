##  <center> Pytorch</center>

`nSamples x nChannels x Height x Width`.

`backward()`

```python
# 对于y=f(x), 如y,x都是向量, 则backward需要传递一个同y一样纬度的矩阵,作为加权信息
# 如 x = [2,3,4] , y = x*x 则:
import torch
a = torch.tensor([2,3,4],requires_grad=True, dtype=float)
b = a*a
print(b)
b.backward(torch.tensor([1.,3.,4.]))
print(a.grad)

>> tensor([ 4.,  9., 16.], dtype=torch.float64, grad_fn=<MulBackward0>)
>> tensor([ 4., 18., 32.], dtype=torch.float64)
```

如果`b.backward(torch.tensor([1.,1.,1.]))`,则`print(a.grad)`的结果应该为  `[4,6,8]`



