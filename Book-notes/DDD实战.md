
# DDD 

## 分包


## Entity
复杂Entity对象使用Factory模式创建

## Value Object

## Aggregate

## Aggregate Root

## Bounded Context

## Event Storming 事件风暴
Event：代表了事件风暴中的核心概念，表示系统中的一个业务行为
Command：有了事件，就会产生事件的对象，即命令
User/Actor：发出命令和事件的用户
Policy：规则，用户发出命令，产生事件时，需要遵守一定的业务规则
Entity：事件产生后，最终会生成一些物(记录)
ReadModel/ScreenLayout：读模型与页面布局。在事件产生后，往往需要我们读取一些已有的模型，在页面展示给用户。这里进一步指示了UI设计。
>人-命令-事件-规则-物  

`用户`执行了`命令`，产生了`事件`，这个过程需要遵守一定的`规则`，最终会形成本次事件的`记录(物)`。
在事件风暴中形成大家有共识的`统一语言`

## CQRS(Command Query Responsibility Segregation)
采用读写分离的方式，将读写分离为两套独立的系统和储存，由`领域事件`通知查询系统更新数据，这样比较难的是事务的一致性。
所以在实践中，针对同一个领域的读写，基本还是在同一个服务中，采用同一个储存(DB等)，针对特殊查询使用ES等。
