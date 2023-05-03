<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [autograd](#autograd)
- [training pipeline](#training-pipeline)
- [softmax & corss-entropy & binary_cross_entropy_with_logits](#softmax-corss-entropy-binary_cross_entropy_with_logits)
- [常见函数](#常见函数)

<!-- /code_chunk_output -->

## autograd

正向传播时，自动生成动态图；反向传播时，利用动态图自动计算梯度。

```python {.line-numbers}
import torch

# forward pass
x = torch.randn(3, requires_grad=True)
y = x+2
z = y*y*2
z = z.mean()

# backward pass
z.backward()  # dz/dx
print(x.grad)

# zero gradients
x.grad.zero_()
```

## training pipeline

```python {.line-numbers}
import torch
import torch.nn as nn

# 自定义的网络结构与网络参数。效果等价于：
# model = nn.Linear(input_size, output_size)
w = torch.tensor(0.0, dtype=torch.float32, requires_grad=True)
def forward(x):
    return w * x

# 本质上与 forward() 相同，只是不含需要训练的网络参数。
loss = nn.MSELoss()

# optimizer = torch.optim.SGD(model.parameters(), learning_rate=0.01)
optimizer = torch.optim.SGD([w], learning_rate=0.01)

# 自定义 dataset
# 假设：number_sample = 50k
DATASET = DetectionVotesDataset()
# 令：batch_size = 100
# 则：iteration = len(DataLoader) = 50k/100 = 500
DATALOADER = DataLoader(DATASET, batch_size=100)

# 每个 epoch 会训练所有的 sample
# number_epoch = 200
for epoch in Range(200):
    # iteration = 500，每次 load 100 个样本
    for i, (input, labels) in  enumerate(DATALOADER):
        # y_pred = model(input)
        y_pred = forwad(input)
    
        # 计算 100 个样本的总 loss
        l = loss(labels, y_pred)
    
        l.backward()

        # update weights: [w]
        optimizer.step()

        # zero gradients
        optimizer.zero_grad()
```

## softmax & corss-entropy & binary_cross_entropy_with_logits

```python {.line-numbers}
import numpy as np
import torch
import torch.nn as nn
from torch.nn import functional as F

# softmax: [2.0,1.0,0.1]->[0.7,0.2,0.1]
# 自己写的版本
def softmax(x):
    return np.exp(x) / np.sum(np.exp(x), axis=0)
# torch 版本
torch.softmax(x, dim=0)

# cross_entropy: [1,0,0], [0.7,0.2,0.1] -> 0.36
# 自己写的版本
def cross_entropy(actual, predicted):
    loss = -np.sum(actual * np.log(predicted))
    return loss # / float(predicted.shape[0])
# torch 版本
# (nsamples)
actual = torch.tensor([0])  # 对应 [1,0,0]
# (nsamples, nclasses)
# nn.CrossEntropyLoss() 会先做一个 softmax
predicted = torch.tensor([[2.0,1.0,0.1]])
nn.CrossEntropyLoss(predicted, actual)

# binary_cross_entropy_with_logits: [5,1,3], [1,0,1] -> [0.0067, 1.3133, 0.0486]
# 自己写的版本
def binary_cross_entropy_with_logits(logit, label):
    sigma = 1 / (1+np.exp(-logit))  # [-inf,+inf] -> [0,1]
    loss = -label*np.log(sigma)-(1-label)*np.log(1-np.exp(-logit))
# torch 版本
F.binary_cross_entropy_with_logits(logit,label,reduction='none')
```

## 常见函数

```python {.line-numbers}
# tells your model that you are training the model. This helps inform layers such as Dropout and BatchNorm, which are designed to behave differently during training and evaluation.
model.train()
model.eval()
```

