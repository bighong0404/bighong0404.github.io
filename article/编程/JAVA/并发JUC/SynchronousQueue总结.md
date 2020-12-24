

# SynchronousQueue总结



## 1. 非公平模式



### 1.1 原理图



### 1.2 数据结构



TransferStack

Snode（mode: REQUEST, DATA, FULFILLING）



### 1.3 重要的细节



- match = this 表示 cancel状态;
- mode = FULFILLING|REQUEST或者FULFILLING|DATA





## 2. 公平模式

### 1.1 原理图



### 1.2 数据结构

TransferQeueu

QNode



### 1.3 重要的细节



- item= this 表示 cancel状态;



## 3. `TransferQueue`与`TransferStack`的`awaitFulfill()`差异在哪?