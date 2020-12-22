# 数据安全保障
数据安全是最为重要的点，如果无法保障数据的安全，那么后续的其他功能就失去了意义。  
首先数据复制的流程可以分为两种场景：
1. 两个 zone 之间正常复制。
2. 切换 zone 的时候数据复制。
3. 数据从零开始全量复制。

而这两种场景优惠面对三种 case：
1. 新增数据的时候。
2. 删除数据的时候。
3. 更新数据的时候。

三种场景加上三种 case 在加上各种一场情况下如何处理。

## 可能出现的异常
在上面的场景和 case 下，可能会出现的异常情况会有那些，那些异常不会出现，这里做一个梳理。

### 负责同步数据的 MQ 是否会丢失消息？
### 手动修改了数据？校验失败
### 消息延迟
### 消息乱序
### 消息过期被丢弃