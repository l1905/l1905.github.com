---
title: golang微服务熔断(breaker)
description: golang微服务熔断(breaker)
categories:
 - golang
 - 熔断
tags:
 - golang
 - braker
---


# gobreaker熔断源码解析

## 背景

公司内部项目最近需要接入熔断， 正好调研到`gobreaker`，和公司内部实现的熔断器，代码量不大， 且非常清晰， 这里记录下其源码分析， 帮助自己更好的理解

## 熔断架构设计

我们可以看到，主要存在三种状态：

1. Close(正常): 此刻正常阶段， 不会影响用户请求
2. Open(熔断): 此刻处于熔断阶段， 内部服务不可用
3. HalfOpen(恢复阶段): 此刻处于恢复阶段，放行零星请求


状态机流程图如下：


所以我们需要重点关注

1.  在什么条件下， 会从[正常阶段]， 切换到[熔断中阶段] ---> 即动作B
2. 在什么条件下， 会从[熔断中]，切换到[恢复阶段]  ---> 即动作B
3. 在什么条件下， 会从[恢复阶段], 重新切换到[熔断中阶段]---> 即动作
4. 在什么条件下， 会从[恢复阶段], 重新切换到[正常状态] 


<img src="/assets/images/blog/breaker.png" class="full-image" />



### 对gobraker进行源码注释分析：

```
package gobreaker

import (
	"errors"
	"fmt"
	"sync"
	"time"
)

// State is a type that represents a state of CircuitBreaker.
// 定义状态
type State int

// These constants are states of CircuitBreaker.
// 常量状态
const (
	StateClosed State = iota
	StateHalfOpen
	StateOpen
)

var (
	// ErrTooManyRequests is returned when the CB state is half open and the requests count is over the cb maxRequests
	ErrTooManyRequests = errors.New("too many requests")
	// ErrOpenState is returned when the CB state is open
	ErrOpenState = errors.New("circuit breaker is open")
)

// String implements stringer interface.
// 获取当前状态
func (s State) String() string {
	switch s {
	case StateClosed:
		return "closed"
	case StateHalfOpen:
		return "half-open"
	case StateOpen:
		return "open"
	default:
		return fmt.Sprintf("unknown state: %d", s)
	}
}

// Counts holds the numbers of requests and their successes/failures.
// CircuitBreaker clears the internal Counts either
// on the change of the state or at the closed-state intervals.
// Counts ignores the results of the requests sent before clearing.
// 计数器： 总请求数， 总成功请求数， 总失败请求数， 连续成功总数， 连续失败总数
type Counts struct {
	Requests             uint32
	TotalSuccesses       uint32
	TotalFailures        uint32
	ConsecutiveSuccesses uint32
	ConsecutiveFailures  uint32
}

// 总请求数+1
func (c *Counts) onRequest() {
	c.Requests++
}

// 如果成功， 则成功计数加1，失败次数归零
func (c *Counts) onSuccess() {
	c.TotalSuccesses++
	c.ConsecutiveSuccesses++
	c.ConsecutiveFailures = 0
}

// 如果失败，则失败计数加1， 成功次数归零
func (c *Counts) onFailure() {
	c.TotalFailures++
	c.ConsecutiveFailures++
	c.ConsecutiveSuccesses = 0
}

// 成功，失败，总计清零
func (c *Counts) clear() {
	c.Requests = 0
	c.TotalSuccesses = 0
	c.TotalFailures = 0
	c.ConsecutiveSuccesses = 0
	c.ConsecutiveFailures = 0
}

// Settings configures CircuitBreaker:
//
// Name is the name of the CircuitBreaker.
//
// MaxRequests is the maximum number of requests allowed to pass through
// when the CircuitBreaker is half-open.
// If MaxRequests is 0, the CircuitBreaker allows only 1 request.
//
// Interval is the cyclic period of the closed state
// for the CircuitBreaker to clear the internal Counts.
// If Interval is 0, the CircuitBreaker doesn't clear internal Counts during the closed state.
//
// Timeout is the period of the open state,
// after which the state of the CircuitBreaker becomes half-open.
// If Timeout is 0, the timeout value of the CircuitBreaker is set to 60 seconds.
//
// ReadyToTrip is called with a copy of Counts whenever a request fails in the closed state.
// If ReadyToTrip returns true, the CircuitBreaker will be placed into the open state.
// If ReadyToTrip is nil, default ReadyToTrip is used.
// Default ReadyToTrip returns true when the number of consecutive failures is more than 5.
//
// OnStateChange is called whenever the state of the CircuitBreaker changes.
type Settings struct {
	Name          string // 熔断器名称
	MaxRequests   uint32 // 在恢复中 允许通过的最多请求数
	Interval      time.Duration // 在close状态，多长时间内清空计数。
	Timeout       time.Duration  // 熔断中， 冷却时间秒数， 超过此秒数，则进入到 恢复阶段
	ReadyToTrip   func(counts Counts) bool //  从正常状态 切换到 熔断时的条件判断
	OnStateChange func(name string, from State, to State) // 状态更换时的回调函数
}

// CircuitBreaker is a state machine to prevent sending requests that are likely to fail.
type CircuitBreaker struct {
	name          string  // 熔断器名称， 从setting中获得
	maxRequests   uint32  // 在恢复中， 允许通过的最多请求数
	interval      time.Duration // 在close状态， 多长时间内清空计数
	timeout       time.Duration  // 同步自setting, 冷却时间秒数， 超过此秒数，则自动进入到 恢复阶段
	readyToTrip   func(counts Counts) bool // 同步自setting
	onStateChange func(name string, from State, to State) // 同步自setting

	mutex      sync.Mutex //  s锁
	state      State // 当前状态
	generation uint64 // 版本号， 类似乐观锁
	counts     Counts // 关联的计数器
	expiry     time.Time // 有效期, 通过interval 和timeout来驱动修改
}

// TwoStepCircuitBreaker is like CircuitBreaker but instead of surrounding a function
// with the breaker functionality, it only checks whether a request can proceed and
// expects the caller to report the outcome in a separate step using a callback.
type TwoStepCircuitBreaker struct {
	cb *CircuitBreaker
}

// NewCircuitBreaker returns a new CircuitBreaker configured with the given Settings.
func NewCircuitBreaker(st Settings) *CircuitBreaker {
	cb := new(CircuitBreaker)

	cb.name = st.Name
	cb.interval = st.Interval
	cb.onStateChange = st.OnStateChange

	if st.MaxRequests == 0 {
		cb.maxRequests = 1
	} else {
		cb.maxRequests = st.MaxRequests
	}

	if st.Timeout == 0 {
		cb.timeout = defaultTimeout
	} else {
		cb.timeout = st.Timeout
	}

	if st.ReadyToTrip == nil {
		cb.readyToTrip = defaultReadyToTrip
	} else {
		cb.readyToTrip = st.ReadyToTrip
	}

	cb.toNewGeneration(time.Now())

	return cb
}

// NewTwoStepCircuitBreaker returns a new TwoStepCircuitBreaker configured with the given Settings.
func NewTwoStepCircuitBreaker(st Settings) *TwoStepCircuitBreaker {
	return &TwoStepCircuitBreaker{
		cb: NewCircuitBreaker(st),
	}
}

const defaultTimeout = time.Duration(60) * time.Second

func defaultReadyToTrip(counts Counts) bool {
	return counts.ConsecutiveFailures > 5
}

// Name returns the name of the CircuitBreaker.
func (cb *CircuitBreaker) Name() string {
	return cb.name
}

// State returns the current state of the CircuitBreaker.
func (cb *CircuitBreaker) State() State {
	cb.mutex.Lock()
	defer cb.mutex.Unlock()

	now := time.Now()
	state, _ := cb.currentState(now)
	return state
}

// Execute runs the given request if the CircuitBreaker accepts it.
// Execute returns an error instantly if the CircuitBreaker rejects the request.
// Otherwise, Execute returns the result of the request.
// If a panic occurs in the request, the CircuitBreaker handles it as an error
// and causes the same panic again.
// 执行函数
func (cb *CircuitBreaker) Execute(req func() (interface{}, error)) (interface{}, error) {
	// 在方法执行前需要做的事情
	generation, err := cb.beforeRequest()
	if err != nil {
		return nil, err
	}

	defer func() {
		e := recover()
		if e != nil {
			cb.afterRequest(generation, false)
			panic(e)
		}
	}()
	// 进行接口请求
	result, err := req()
	// 回调函数
	cb.afterRequest(generation, err == nil)
	return result, err
}

// Name returns the name of the TwoStepCircuitBreaker.
func (tscb *TwoStepCircuitBreaker) Name() string {
	return tscb.cb.Name()
}

// State returns the current state of the TwoStepCircuitBreaker.
func (tscb *TwoStepCircuitBreaker) State() State {
	return tscb.cb.State()
}

// Allow checks if a new request can proceed. It returns a callback that should be used to
// register the success or failure in a separate step. If the circuit breaker doesn't allow
// requests, it returns an error.
func (tscb *TwoStepCircuitBreaker) Allow() (done func(success bool), err error) {
	generation, err := tscb.cb.beforeRequest()
	if err != nil {
		return nil, err
	}

	return func(success bool) {
		tscb.cb.afterRequest(generation, success)
	}, nil
}

// 请求之前 需要做的事情
func (cb *CircuitBreaker) beforeRequest() (uint64, error) {
	cb.mutex.Lock()
	defer cb.mutex.Unlock()

	now := time.Now()
	// 获取当前状态
	state, generation := cb.currentState(now)

	// 如果是熔断状态， 则直接返回错误，提前中止本次请求
	if state == StateOpen {
		return generation, ErrOpenState
	} else if state == StateHalfOpen && cb.counts.Requests >= cb.maxRequests {
		// 因为限定了[恢复中]阶段的最大请求数， 这里需要做通过的最大请求数判断， 即只允许通过多少条
		return generation, ErrTooManyRequests
	}

	// 总请求数+1
	cb.counts.onRequest()

	return generation, nil
}

// 请求结束后， 需要做的事情
func (cb *CircuitBreaker) afterRequest(before uint64, success bool) {
	cb.mutex.Lock()
	defer cb.mutex.Unlock()

	now := time.Now()
	// 再次获取当前的状态， 如果当前请求周期太长，导致统计状态 已经变化，则需要将此次请求丢掉， 不不加入到成功，或者失败总数中
	state, generation := cb.currentState(now)
	if generation != before {
		return
	}

	// 如果成功，则成功计数， 并且这只状态
	if success {
		// 成功计数
		cb.onSuccess(state, now)
	} else {
		// 失败计数
		cb.onFailure(state, now)
	}
}

func (cb *CircuitBreaker) onSuccess(state State, now time.Time) {
	switch state {
	case StateClosed:
		cb.counts.onSuccess()
	case StateHalfOpen:
		cb.counts.onSuccess()
		if cb.counts.ConsecutiveSuccesses >= cb.maxRequests {
			cb.setState(StateClosed, now)
		}
	}
}

func (cb *CircuitBreaker) onFailure(state State, now time.Time) {
	switch state {
	case StateClosed:
		cb.counts.onFailure()
		if cb.readyToTrip(cb.counts) {
			cb.setState(StateOpen, now)
		}
	case StateHalfOpen:
		cb.setState(StateOpen, now)
	}
}

// 获取当前状态
func (cb *CircuitBreaker) currentState(now time.Time) (State, uint64) {
	switch cb.state {
	case StateClosed:
		if !cb.expiry.IsZero() && cb.expiry.Before(now) {
			cb.toNewGeneration(now)
		}
	case StateOpen:
		if cb.expiry.Before(now) {
			cb.setState(StateHalfOpen, now)
		}
	}
	return cb.state, cb.generation
}

func (cb *CircuitBreaker) setState(state State, now time.Time) {
	if cb.state == state {
		return
	}

	prev := cb.state
	cb.state = state

	cb.toNewGeneration(now)

	if cb.onStateChange != nil {
		cb.onStateChange(cb.name, prev, state)
	}
}

// 主要是更新expiry字段，并且加上时间戳
func (cb *CircuitBreaker) toNewGeneration(now time.Time) {
	cb.generation++
	cb.counts.clear()

	var zero time.Time
	switch cb.state {
	case StateClosed:
		if cb.interval == 0 {
			cb.expiry = zero
		} else {
			cb.expiry = now.Add(cb.interval)
		}
	case StateOpen:
		cb.expiry = now.Add(cb.timeout)
	default: // StateHalfOpen
		cb.expiry = zero
	}
}
```

解答上面的四个问题：

我们进行依次解答：

解答1:  

在正常状态下， 当连续失败次数>5次时， 则切换到[熔断中]；  
这里提供 回调函数， 用户可自己切换条件。

解答2:  
在[熔断中]会有冷却期， 当经过固定的冷却期，会自动切换到 [恢复阶段]
冷却期通过`timeout`参数进行控制， 在请求前检测，如果达到出发条件，则切换到[恢复阶段]

解答3:

当在[恢复阶段] 陆续检测部分请求，当存在失败情况，或者成功率小于多少，则切换到 熔断中

在[恢复阶段], 如果进行请求， 探测仍然失败， 则继续切换到[熔断中]

解答4， 

当在[恢复阶段]陆续检测部分请求，当连续成功次数大于固定值， 或者成功率大于多少，则切换到[正常阶段]

在[恢复阶段], 如果持续进行请求， 并且每次请求都成功， 连续请求成功次数> 最大请求数， 则切换到[正常阶段]


其中的问题有：

1. 如果请求时间过长， 请求前后状态已改变， 则对此次请求不进行计数。
2. 是按时间段来计数， 并不太精准
3. 在恢复阶段， 无法自定义方法更改状态。 


### 对公司内熔断器进行解析：

```
package client
import (
	"math/rand"
	"sync"
	"time"
)
// 每秒一个桶, 记录该秒的请求成功、失败次数
type StatsBucket struct {
	success int
	fail int
}

// 健康统计, 维护最近N秒的滑动窗口
type HealthStats struct {
	buckets []StatsBucket // 滑动窗口, 每个桶1秒
	curTime int64 // 当前窗口末尾的秒级unix时间戳， 桶内最大时间， 即最近的一个桶
	minStats int // 少于该打点数量直接返回健康,  配置信息
	healthRate float64 // 健康阀值， 配置信息
}

// 熔断器状态
type CircuitStatus int
const (
	CIRCUIT_NORMAL CircuitStatus = 1	// 正常
	CIRCUIT_BREAK = 2	// 熔断
	CIRCUIT_RECOVER = 3	// 恢复中
)

// 熔断器
type CircuitBreaker struct {
	mutex sync.Mutex
	healthStats *HealthStats // 健康统计
	status CircuitStatus // 熔断状态
	breakTime int64 // 熔断的时间点(秒)
	breakPeriod int // 熔断封锁时间， 处于熔断的时间秒数，比如3秒， 属于配置信息 , 数据来自: breakPeriod
	recoverPeriod int // 熔断恢复时间， 处于恢复阶段持续的秒数， 比如2秒，属于配置信息，数据来自: CircuitBreakerInfo
}

// 熔断配置
type CircuitBreakerInfo struct {
	BreakPeriod int // 熔断封锁时间
	RecoverPeriod int // 熔断恢复时间
	WinSize int // 滑动窗口大小
	MinStats int // 最小统计样本
	HealthRate float64 // 健康阀值
}

// 创建健康统计器
func createHealthStats(info *CircuitBreakerInfo) (healthStats *HealthStats) {
	healthStats = &HealthStats{
		minStats: info.MinStats,
		healthRate: info.HealthRate,
	}
	healthStats.buckets = make([]StatsBucket, info.WinSize)
	healthStats.resetBuckets(healthStats.buckets[:])
	healthStats.curTime = CurUnixSecond()
	return
}

// 获取当前时间戳
func CurUnixSecond() int64 {
	return time.Now().Unix()
}

// 重置桶状态， 将桶内成功数， 失败数置零
func (healthStats *HealthStats) resetBuckets(buckets []StatsBucket) {
	for idx, _ := range buckets {
		buckets[idx].success = 0
		buckets[idx].fail = 0
	}
}
// 窗口滑动
func (healthStats *HealthStats) shiftBuckets() {
	now := CurUnixSecond()
	// 当前时间减去桶中最大值时间
	timeDiff := int(now - healthStats.curTime)
	if timeDiff <= 0 {
		return
	}

	// 如果时间差 大于桶的长度， 则需要重置桶内所有数据
	// 应用场景： 比如经过很长时间没有请求接口，突然来请求接口，这个时候滑动桶，则把所有的内容滑动没了
	if timeDiff >= len(healthStats.buckets) {
		healthStats.resetBuckets(healthStats.buckets[:])
	} else {
		// 如果当前时间大于 记录的最大时间， 并且小于桶的的个数, 则滑动桶
		healthStats.buckets = append(healthStats.buckets[:0], healthStats.buckets[timeDiff:]...)
		for i := 0; i < timeDiff; i++ {
			healthStats.buckets = append(healthStats.buckets, StatsBucket{})
		}
	}
	healthStats.curTime = now
}
// 成功打点
func (healthStats *HealthStats) success() {
	healthStats.shiftBuckets()
	healthStats.buckets[len(healthStats.buckets) - 1].success++
}
// 失败打点
func (healthStats *HealthStats) fail() {
	healthStats.shiftBuckets()
	healthStats.buckets[len(healthStats.buckets) - 1].fail++
}
// 判断是否健康
func (healthStats *HealthStats) isHealthy() (bool, float64) {
	healthStats.shiftBuckets()
	success := 0
	fail := 0
	for _, bucket := range healthStats.buckets {
		success += bucket.success
		fail += bucket.fail
	}
	total := success + fail
	// 没有样本
	if total == 0 {
		return true, 1
	}
	rate :=  (float64(success) / float64(total))
	// 样本不足
	if total < healthStats.minStats {
		return true, rate
	}
	// 样本充足
	return rate >= healthStats.healthRate, rate
}
// 创建熔断器
func CreateCircuitBreaker(info *CircuitBreakerInfo) (circuitBreaker *CircuitBreaker)  {
	circuitBreaker = &CircuitBreaker{
		healthStats: createHealthStats(info),
		status: CIRCUIT_NORMAL,
		breakTime: 0,
		breakPeriod: info.BreakPeriod,
		recoverPeriod: info.RecoverPeriod,
	}
	return
}

// 请求成功， 对外提供调用接口
func (circuitBreaker *CircuitBreaker) Success() {
	circuitBreaker.mutex.Lock()
	defer circuitBreaker.mutex.Unlock()
	circuitBreaker.healthStats.success()
}

// 请求失败， 对外提供调用接口
func (circuitBreaker *CircuitBreaker) Fail()  {
	circuitBreaker.mutex.Lock()
	defer circuitBreaker.mutex.Unlock()
	circuitBreaker.healthStats.fail()
}
// 熔断器判定, 判读是否熔断
func (circuitBreaker *CircuitBreaker) IsBreak() (isBreak bool, isHealthy bool, healthRate float64) {
	// 最外层锁
	circuitBreaker.mutex.Lock()
	defer circuitBreaker.mutex.Unlock()
	now := CurUnixSecond()

	// 现在时间减去熔断开始时间， 作为熔断冷却期，
	breakLastTime := now - circuitBreaker.breakTime
	// 判断当前是否是健康状态
	isHealthy, healthRate = circuitBreaker.healthStats.isHealthy()
	isBreak = false
	// 状态机运转， 更改状态
	switch circuitBreaker.status {
	case CIRCUIT_NORMAL:
		// 非健康状态， 则轮转替换为[熔断中]
		if !isHealthy {
			circuitBreaker.status = CIRCUIT_BREAK
			circuitBreaker.breakTime = now
			isBreak = true
		}
	case CIRCUIT_BREAK:
		// 熔断中, 如果在熔断中， 或者不健康
		if breakLastTime < int64(circuitBreaker.breakPeriod) || !isHealthy {
			isBreak = true
		} else {
			// 否则切换到[恢复中]
			circuitBreaker.status = CIRCUIT_RECOVER
		}
	case CIRCUIT_RECOVER:
		// 在恢复中，如果是不健康状态， 则直接切换到[熔断中]
		if !isHealthy {
			circuitBreaker.status = CIRCUIT_BREAK
			circuitBreaker.breakTime = now
			isBreak = true
		} else {
			// 如果熔断的时间 已经大于 (熔断时间+恢复时间), 则切换到 正常状态
			if breakLastTime >= int64(circuitBreaker.breakPeriod + circuitBreaker.recoverPeriod) {
				circuitBreaker.status = CIRCUIT_NORMAL
			} else {
				// 随着时间间隔增加， 放行比例主键 需要逐渐增大
				passRate := float64(breakLastTime) /  float64(circuitBreaker.breakPeriod + circuitBreaker.recoverPeriod)
				// 随机数随机生成0-1之间的数， 因为passRate逐渐增大， 所以break禁止的越来越少
				if rand.Float64() > passRate {
					isBreak = true
				}
			}
		}
	}
	return
}
```


公司内部实现， 同样采用状态机，在3种状态中轮转。

较`gobreak` 改进的地方是， 从按固定时间段统计 调整为 按滑动窗口方式来统计。

每秒作为一个桶， 记录每秒钟的成功次数和失败次数。随着时间变化， 每次都将时间窗口向前滑动。

因此我们重新解答上面我们遇到的四个主要问题：

解答1:  

通过统计时间窗口中 所有桶的失败数占(失败数+成功数)比例， 统计次数有下限， 比如> xxx次， 如果失败率达到此， 则切换到[熔断中]

解答2:  
在[熔断中]会有冷却期， 当超过固定的冷却期，会自动切换到 [恢复阶段]

解答3:

在[恢复阶段], 如果进行请求， 探测达到时间窗口内的失败率， 则继续切换到[熔断中]。
这里最开始放行次数较少， 随着时间增加，放行次数也逐渐增加

解答4， 

在[恢复阶段], 探测达到时间窗口内的失败率较小，并且持续固定时间的【恢复阶段】， 则切换到[正常阶段]


这里使用滑动窗口的概念， 因此需要时刻进行滑动窗口：

1. 是否需要熔断
2. 成功计数时
3. 失败计数时


## 参考文章

1. https://zhuanlan.zhihu.com/p/58428026



















