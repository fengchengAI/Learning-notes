## Numpy

当要保存的数据比较复杂时,如string,ndarray等都存在时,可以使用np.savez()进行保存,然后使用np.load()进行加载

`代码清单1`

```python
import numpy as np 
# 保存
a = {}
a['file_name'] = "cfeng.txt"
a['data'] = np.ones((4,3))
a['mask'] = np.ones((3,2))
np.savez('data.npz',a)

#加载
data = np.load('data.npz',allow_pickle=True) # 如果保存的数据中有字典,则必须有这个参数
#此时加载的是一个ndarray字典,需要将其转化为普通字典.
data = data['arr_0'].item() # 为什么要加'arr_0',因为在savez的时候,默认将a作为'arr_0'的值了,或者需要制定一个键值,如`代码清单2`所示
#此时就可以使用
b = data['file_name']
c = data['data']
```



或者

`代码清单2`

```python
import numpy as np 

# 保存
np.save('data.npz',file_name="cfeng.txt", data=np.ones((4,3)), mask=np.ones((3,2)))

#加载
data = np.load('data.npz',allow_pickle=True) # 如果保存的数据中有字典,则必须有这个参数

#此时可以直接使用
b = data['file_name']
c = data['data']
```



`代码清单2`  有关numpy.str_

```Python
import numpy as np
import os
root = "/media/feng/PC/data/tianchi/fusai_train_new/json"
path = os.listdir(root)
path = [root+"/"+i for i in path]
path.sort()
np.savez('train.npz',path)
dat = np.load('train.npz')['arr_0'][0] # 这里的dat是numpy.str_格式的,并不是python內建的str
```

