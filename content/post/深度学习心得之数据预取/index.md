---
date: 2026-01-21
tags:
- 科研
publish: true
title: 深度学习心得之数据预取
---

学习 [CS336](https://stanford-cs336.github.io/spring2025/) 时，发现[别人的代码](https://github.com/brandon-snider/cs336-a1)在取数据时有些怪：
```python
x, y = get_batch()

for step in range(max_step):
	logits = model(x)
	x, y = get_batch()
	# ....
	loss.backward()
```
相信任何人都会疑惑，为什么不是这样的呢：
```python
for step in range(max_step):
	x, y = get_batch()
	logits = model(x)
	# ....
	loss.backward()
```
这种行为叫做**数据预取**，在执行 `logits = model(x)`时，CPU 实际上只做了一件非常快的事情：**往 GPU 的任务队列里扔了一个“去跑前向传播”的命令**（Kernel Launch），然后它就认为这行代码“执行完”了，立刻往下走。

PyTorch 官方文档中有专门的[章节](https://docs.pytorch.org/docs/stable/notes/cuda.html)解释 **异步执行（Asynchronous Execution）**：
>“默认情况下，GPU 操作是异步的。当你调用一个使用 GPU 的函数时，这些操作会被放入设备的**队列**中，但在后续代码执行之前，这些操作**不一定已经执行完毕**。这允许我们并行执行更多计算，包括 CPU 上的操作或其他 GPU 上的操作。”

既然如此，两种写法的效果不是应该一致吗？难道这种异步是不能跨循环的？我们可以写一个脚本来验证：
```python
import torch
import time

device = "cuda"
N = 10000

a = torch.randn(N, N, device=device)
b = torch.randn(N, N, device=device)

t0 = time.time()

for i in range(10):
	c = torch.matmul(a, b)
	c = torch.matmul(c, a)

t1 = time.time()
torch.cuda.synchronize()
t2 = time.time()

print(t1 - t0, t2 - t1)
# 执行结果：0.11194539070129395 2.491203784942627
```
显然 $t_1 - t_0 < t_2 - t_1$，这说明这种异步是可以跨循环的。事实上，如果只是我上面那样写，相差不大。但在实际情况中，loss 通常需要被打印，这导致 CPU 必须等待 GPU 处理完毕。对于预取式写法：
```python
x, y = get_batch()

for step in range(max_step):
	logits = model(x)
	x, y = get_batch()
	# ....
	print(loss)
	loss.backward()
```
非预取式写法：
```python
for step in range(max_step):
	x, y = get_batch()
	logits = model(x)
	# ....
	print(loss)
	loss.backward()
```
预取式写法并行的是 `logits = model(x)`、 `x, y = get_batch()` 和上个循环的 `loss.backward()`，而非预取式写法并行的是 `x, y = get_batch()` 和上个循环的 `loss.backward()`。

反向传播计算的时间要比前向传播计算的时间更长。预取写法能够在这两种 GPU 计算下并行，而非预取写法则只能在反向传播计算时并行。