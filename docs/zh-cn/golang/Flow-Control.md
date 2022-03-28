# 流量控制 (Sentinel Go)

## Overview

流量控制(flow control)，其原理是监控资源(`Resource`)的统计指标，然后根据Token计算策略来计算资源的可用Token(也就是阈值)，然后根据流量控制策略对请求进行控制，避免被瞬时的流量高峰冲垮，从而保障应用的高可用性。

Sentinel Go 的流量控制实现代码参考：[https://github.com/alibaba/sentinel-golang/tree/master/core/flow](https://github.com/alibaba/sentinel-golang/tree/master/core/flow)

Sentinel 通过定义流控规则来实现对 `Resource` 的流量控制。在 Sentinel 内部会在加载流控规则时候将每个 flow.Rule 都会被转换成流量控制器(TrafficShapingController)。 每个流量控制器实例都会有自己独立的统计结构，这里统计结构是一个滑动窗口。Sentinel 内部会尽可能复用 Resource 级别的全局滑动窗口，如果流控规则的统计设置没法复用Resource的全局统计结构，那么Sentinel会为流量控制器创建一个全新的私有的滑动窗口，然后通过 flow.StandaloneStatSlot 这个统计Slot来维护统计指标。

Sentinel 的流量控制组件对Resource的检查结果要么通过，要么会block，对于block的流量相当于拒绝。

## 流控规则

流控规则的[定义](https://github.com/alibaba/sentinel-golang/blob/master/core/flow/rule.go)如下：

```go
type Rule struct {
	// ID represents the unique ID of the rule (optional).
	ID string `json:"id,omitempty"`
	// Resource represents the resource name.
	Resource               string                 `json:"resource"`
	TokenCalculateStrategy TokenCalculateStrategy `json:"tokenCalculateStrategy"`
	ControlBehavior        ControlBehavior        `json:"controlBehavior"`
	// Threshold means the threshold during StatIntervalInMs
	// If StatIntervalInMs is 1000(1 second), Threshold means QPS
	Threshold         float64          `json:"threshold"`
	RelationStrategy  RelationStrategy `json:"relationStrategy"`
	RefResource       string           `json:"refResource"`
	MaxQueueingTimeMs uint32           `json:"maxQueueingTimeMs"`
	WarmUpPeriodSec   uint32           `json:"warmUpPeriodSec"`
	WarmUpColdFactor  uint32           `json:"warmUpColdFactor"`
	// StatIntervalInMs indicates the statistic interval and it's the optional setting for flow Rule.
	// If user doesn't set StatIntervalInMs, that means using default metric statistic of resource.
	// If the StatIntervalInMs user specifies can not reuse the global statistic of resource,
	// 		sentinel will generate independent statistic structure for this rule.
	StatIntervalInMs uint32 `json:"statIntervalInMs"`
}
```

一条流控规则主要由下面几个因素组成，我们可以组合这些元素来实现不同的限流效果：

- `Resource`：资源名，即规则的作用目标。
- `TokenCalculateStrategy`: 当前流量控制器的Token计算策略。Direct表示直接使用字段 Threshold 作为阈值；WarmUp表示使用预热方式计算Token的阈值。
- `ControlBehavior`: 表示流量控制器的控制策略；Reject表示超过阈值直接拒绝，Throttling表示匀速排队。
- `Threshold`: 表示流控阈值；如果字段 StatIntervalInMs 是1000(也就是1秒)，那么Threshold就表示QPS，流量控制器也就会依据资源的QPS来做流控。
- `RelationStrategy`: 调用关系限流策略，CurrentResource表示使用当前规则的resource做流控；AssociatedResource表示使用关联的resource做流控，关联的resource在字段 `RefResource` 定义；
- `RefResource`: 关联的resource；
- `WarmUpPeriodSec`: 预热的时间长度，该字段仅仅对 `WarmUp` 的TokenCalculateStrategy生效；
- `WarmUpColdFactor`: 预热的因子，默认是3，该值的设置会影响预热的速度，该字段仅仅对 `WarmUp` 的TokenCalculateStrategy生效；
- `MaxQueueingTimeMs`: 匀速排队的最大等待时间，该字段仅仅对 `Throttling` ControlBehavior生效；
- `StatIntervalInMs`: 规则对应的流量控制器的独立统计结构的统计周期。如果StatIntervalInMs是1000，也就是统计QPS。

这里特别强调一下`StatIntervalInMs`和`Threshold`这两个字段，这两个字段决定了流量控制器的灵敏度。以 Direct + Reject 的流控策略为例，流量控制器的行为就是在`StatIntervalInMs`周期内，允许的最大请求数量是`Threshold`。比如，如果`StatIntervalInMs`是10000，`Threshold`是10000，那么流量控制器的行为就是10s内运行最多10000次访问。

## 流量控制策略

Sentinel的流量控制策略由规则中的 `TokenCalculateStrategy` 和 `ControlBehavior` 两个字段决定。`TokenCalculateStrategy` 表示流量控制器的Token计算方式，目前Sentinel支持两种：
1. Direct表示直接使用规则中的 `Threshold` 表示当前统计周期内的最大Token数量。
2. WarmUp表示通过预热的方式计算当前统计周期内的最大Token数量，预热的计算方式会根据规则中的字段 `WarmUpPeriodSec` 和 `WarmUpColdFactor` 来决定预热的曲线。

WarmUp方式，即预热/冷启动方式。当系统长期处于低水位的情况下，当流量突然增加时，直接把系统拉升到高水位可能瞬间把系统压垮。通过"冷启动"，让通过的流量缓慢增加，在一定时间内逐渐增加到阈值上限，给冷系统一个预热的时间，避免冷系统被压垮。这块设计和Java类似，可以参考[限流 冷启动](https://github.com/alibaba/Sentinel/wiki/%E9%99%90%E6%B5%81---%E5%86%B7%E5%90%AF%E5%8A%A8)
通常冷启动的过程系统允许通过的 QPS 曲线如下图所示：
![](https://user-images.githubusercontent.com/9434884/68292392-b5b0aa00-00c6-11ea-86e1-ecacff8aab51.png)

字段`ControlBehavior`表示表示流量控制器的控制行为，目前Sentinel支持两种：
1. Reject：表示如果当前统计周期内，统计结构统计的请求数超过了阈值，就直接拒绝。
2. Throttling：表示匀速排队的统计策略。它的中心思想是，以固定的间隔时间让请求通过。当请求到来的时候，如果当前请求距离上个通过的请求通过的时间间隔不小于预设值，则让当前请求通过；否则，计算当前请求的预期通过时间，如果该请求的预期通过时间小于规则预设的 timeout 时间，则该请求会等待直到预设时间到来通过（排队等待处理）；若预期的通过时间超出最大排队时长，则直接拒接这个请求。

匀速排队方式会严格控制请求通过的间隔时间，也即是让请求以均匀的速度通过，对应的是漏桶算法。该方式的作用如下图所示：
![](https://user-images.githubusercontent.com/9434884/68292442-d4af3c00-00c6-11ea-8251-d0977366d9b4.png)

这种方式主要用于处理间隔性突发的流量，例如消息队列。想象一下这样的场景，在某一秒有大量的请求到来，而接下来的几秒则处于空闲状态，我们希望系统能够在接下来的空闲期间逐渐处理这些请求，而不是在第一秒直接拒绝多余的请求。

以下规则代表每 100ms 最多通过一个请求，多余的请求将会排队等待通过，若排队时队列长度大于 500ms 则直接拒绝：

```go
{
	Resource:          "some-test",
        TokenCalculateStrategy: flow.Direct,
	ControlBehavior:   flow.Throttling, // 流控效果为匀速排队
        Threshold:             10, // 请求的间隔控制在 1000/10=100 ms
	MaxQueueingTimeMs: 500, // 最长排队等待时间
}
```
上面Threshold是10，Sentinel默认使用1s作为控制周期，表示1秒内10个请求匀速排队，所以排队时间就是1000ms/10 = 100ms；

`MaxQueueingTimeMs` 设为 0 时代表不允许排队，只控制请求时间间隔，多余的请求将会直接拒绝。

## 流量控制器的统计结构
每个流量控制器实例都会有自己独立的统计结构。流量控制器的统计结构由规则中的 `StatIntervalInMs` 字段设置，`StatIntervalInMs`表示统计结构的统计周期。Sentinel 默认会为每个Resource创建一个全局的滑动窗口统计结构，这个全局的统计结构默认是一个间隔为10s,20个格子的滑动窗口，也就是每个统计窗口长度是500ms。

流量控制器实例会尽可能复用这个Resource级别的全局统计结构，复用逻辑原则是：优先复用Resource级别的全局统计结构，如果不可复用，就重新创建一个独立的滑动窗口统计结构(BucketLeapArray)，具体的逻辑细节如下：

1. 如果`StatIntervalInMs`大于全局滑动窗口的间隔(默认10s)，那么将不可复用全局统计结构。Sentinel会给流量控制器创建一个长度是`StatIntervalInMs`，格子数是1的全新统计结构，这个全新的统计结构由Sentinel内部的`StandaloneStatSlot`来维护统计。
2. 如果`StatIntervalInMs`小于全局滑动窗口的窗口长度(默认是500ms), 那么将不可复用全局统计结构。Sentinel会给流量控制器创建一个长度是`StatIntervalInMs`，格子数是1的全新统计结构，这个全新的统计结构由Sentinel内部的`StandaloneStatSlot`来维护统计。
3. 如果`StatIntervalInMs`在集合[全局滑动窗口的窗口长度，全局滑动窗口的间隔]之间，首先需要计算格子数：如果`StatIntervalInMs`可以被全局滑动窗口的窗口长度(默认是500ms)整除，那么格子数就为 `StatIntervalInMs`/`GlobalStatisticBucketLengthInMs`，如果不可整除，格子数是1。然后会调用 `core/base/CheckValidityForReuseStatistic`函数来判断当前统计结构间隔和格子数是否可以复用全局统计结构。如果可以复用，就会基于resource级别的全局统计结构`ResourceNode`创建`SlidingWindow`，SlidingWindow是一个虚拟结构，SlidingWindow只可读，而且读的数据是通过聚合`ResourceNode`数据得到的。如果不可复用，就使用统计结构间隔和格子数创建全新的滑动窗口(BucketLeapArray)。

CheckValidityForReuseStatistic函数参考：[https://github.com/alibaba/sentinel-golang/blob/master/core/base/stat.go#func CheckValidityForReuseStatistic](https://github.com/alibaba/sentinel-golang/blob/master/core/base/stat.go)

基于规则创建统计结构的逻辑参考：[https://github.com/alibaba/sentinel-golang/blob/master/core/flow/rule_manager.go#func generateStatFor](https://github.com/alibaba/sentinel-golang/blob/master/core/flow/rule_manager.go)

## 基于调用关系的流量控制
Sentinel 支持关联流量控制策略。当两个资源之间具有资源争抢或者依赖关系的时候，这两个资源便具有了关联。比如对数据库同一个字段的读操作和写操作存在争抢，读的速度过高会影响写得速度，写的速度过高会影响读的速度。如果放任读写操作争抢资源，则争抢本身带来的开销会降低整体的吞吐量。可使用关联限流来避免具有关联关系的资源之间过度的争抢。

## 核心接口

在流控模块（flow）中，Sentinel 会为每条流控规则（`flow.Rule`）生成相应的流量调配器（`TrafficShapingController`）。TrafficShapingController 由两部分构成：

- `TrafficShapingCalculator`: 根据规则的计算策略计算出当前的流量阈值（如部分策略在一段时间内逐步抬升 QPS 阈值）
- `TrafficShapingChecker`: 根据阈值执行相应的检查和调配策略，返回 `base.TokenResult` 指示如何进行调配。

## 常见场景的规则配置：

### 1.基于QPS对某个资源限流
基于对某个资源访问的QPS来做流控，这个是非常常见的场景。流控规则的配置下面有个sample：
```go
{
	Resource:                "some-test",
        TokenCalculateStrategy:  flow.Direct,
	ControlBehavior:         flow.Reject,
	Threshold:               500,
        StatIntervalInMs:        1000,
}
```
上面sample中的5个字段是必填的。其中StatIntervalInMs必须是1000，表示统计周期是1s，那么Threshold所配置的值也就是QPS的阈值。

### 2.基于一定统计间隔时间来控制总的请求数

这个场景就是想在一定统计周期内控制请求的总量。比如`StatIntervalInMs`配置10000，`Threshold`配置10000，这种配置意思就是控制10s内最大请求数是10000。sample:
```go
{
	Resource:                "some-test",
        TokenCalculateStrategy:  flow.Direct,
	ControlBehavior:         flow.Reject,
	Threshold:               10000,
        StatIntervalInMs:        10000,
}
```
注意：这种流控配置对于脉冲类型的流量抵抗力很弱，有极大潜在风险压垮系统。比如流量表现形式是10s内请求数最大是9800，实际上流量可能是脉冲形式，比如下图：

![脉冲流量](https://user-images.githubusercontent.com/9346473/94354820-a907bc80-00b1-11eb-8831-a3f67eda908c.png)

这种类型的流量，其实在前一秒的QPS达到了4500，很可能直接把系统打挂了。对于这种类型流量除非是抗脉冲场景，一般建议使用毫秒级别的流控。

注意：这种大周期的配置其实也有好处，就是能够做到流量的无损，前提是保证系统能够抗住这种周期内的脉冲流量，当然如果流量曲线在秒级别比较平顺，也就不存在脉冲问题，我们是建议统计周期可以稍微调大。

### 3.毫秒级别流控
针对一些流量曲在毫秒级别波动非常大的场景(类似于脉冲)，建议`StatIntervalInMs`的配置在毫秒级别，除非特殊场景，建议配置的值为100ms的倍数，比如100，200这种。这种相当于缩小了统计周期，将QPS的周期缩小了10倍，控制周期降低到了100ms。这种配置能够很好的应对脉冲流量，保障系统稳定性，比如sample:
```go
{
	Resource:                "some-test",
        TokenCalculateStrategy:  flow.Direct,
	ControlBehavior:         flow.Reject,
	Threshold:               80,
        StatIntervalInMs:        100,
}
```
上面限制了100ms的阈值是80，实际QPS大概是800。

注意：这种配置也是有缺点的，脉冲流量很大可能造成有损(会拒绝很多流量)。

### 4.脉冲流量无损
针对前面第三点场景，如果既想控制流量曲线，又想无损，一般做法是通过匀速排队的控制策略，平滑掉流量。

## Example

[flow 示例](https://github.com/alibaba/sentinel-golang/tree/master/example/flow)