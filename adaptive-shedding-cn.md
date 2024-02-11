## 1. go-zero 稳定性能力概览

经过这么多年大流量服务端架构设计的沉淀，go-zero 在保护服务的稳定性上下足了功夫，不管是 CPU 密集型还是 IO 密集型服务，go-zero 都能很好的保护服务在如下场景不被拖垮或卡死：
- 远超服务容量的突发大流量
- CPU 打满
- 上下游故障或者超时
- MySQL、MongoDB、Redis 等中间件故障或者超负载（典型的是 CPU 飙高）
<img width="807" alt="image" src="https://github.com/kevwan/microservices-in-action/assets/1918356/46b8a223-68ce-446c-abe6-8c2d2ada8b80">

如图，我们从三个方面来保护系统的稳定性：
- 服务端自适应过载保护
- 服务段自适应熔断
- 客户端自适应熔断

当然，我们还有自动适配后端服务能力的负载均衡算法，对稳定性进一步保驾护航。本文主要讲解自适应过载的原理、场景和表现。

## 2. 自适应过载保护

用过 Windows 的同学对这个界面应该都不陌生，这就是典型 CPU 打满服务不可用的表现。此时，我们一般都是心里默默骂一句，然后点左边那个按钮，对吧？
![image](https://github.com/kevwan/microservices-in-action/assets/1918356/35436275-8077-49ee-9616-5b3ad0b6adde)

那我们想想，如果我们的服务 CPU 被打满了，是不是后面所有的请求也都被卡住了？等服务处理完请求的时候，用户那里总就超时离开了，结果服务器很忙，但都是做的无用功。如果这里不能理解，停下来好好思考一番，如果还不懂的话，可以来 go-zero 群里讨论讨论。。。

### 2.1 模拟 CPU 密集型服务

有人可能会问 CPU 密集型服务怎么定义？你的服务 CPU 会打满吗？处理请求会包含复杂的计算逻辑吗？你经常需要通过 cpu profiling 来优化性能吗？可以理解为服务的 IO 比较快，或者比较少，瓶颈是在 CPU 消耗上。

你可以直接用 `goctl quickstart -t mono` 命令生成一个 HTTP 服务，然后在 logic 代码里加上模拟 CPU 负载的请求处理代码。

模拟 CPU 计算的代码：https://gist.github.com/kevwan/ccfaf45aa190ac44003d93c094a12c3f
1. fib(n int) []int: 生成一个斐波那契数列，长度为参数n。它使用迭代的方式生成斐波那契数列。
2. prime(n int) bool: 用来判断一个数是否为质数。它使用了简单的质数判断算法，遍历2到sqrt(n)之间的数，判断是否有能整除n的数。
3. reversePrime(slice []int) []int: 接受一个整数切片作为参数，然后返回其中的质数，但是返回的顺序是逆向的。
4. expand(slice []int) []int: 接受一个整数切片作为参数，然后将切片中的每个元素扩展为10个，并返回新的切片。扩展的规则是每个元素增加100。
5. bubble(slice []int) []int: 是一个冒泡排序算法，用来对传入的整数切片进行排序。
6. CpuBounded(times int) []int: 接受一个整数times作为参数，然后进行一系列操作。具体操作包括：生成一个长度为40的斐波那契数列，反转后筛选出其中的质数，对质数进行扩展，然后对扩展后的序列进行冒泡排序，最后返回排序后的结果。这个过程会重复times次，并将每次排序后的结果以切片的形式返回。

### 2.2 压测场景
#### 2.2.1 场景一（不开启过载保护）

```yaml
Timeout: 1000
Middlewares:
  Shedding: false
```

- 服务跑在两核的容器内
- 不开启过载保护
- 超时 1s
- `loops 2 hey -c 200 -z 60m "http://localhost:8888/ping"`
  - `loops` 是我的一个 alias：`loops='fs() {for i in {1..$1}; do ${@:2} & done; wait}; fs'`，用来并行执行给定命令指定的次数
![image](https://github.com/kevwan/microservices-in-action/assets/1918356/96f72b7e-d085-4c78-9cbf-3536c537b434)

- 可以看到系统总共只处理了大概 500qps 的请求，其中 400qps 多一点是成功的，近 100qps 是超时的（返回了 503 状态码）
![image](https://github.com/kevwan/microservices-in-action/assets/1918356/7b54b46b-74cd-4f68-9675-f933e72a09e8)

- 随着请求的堆积，很快就会大量请求都超时了，并且p99，甚至p90都已经超过 1s 了
- 这里进一步解释一下，超时的请求意味着对系统资源的浪费，比如接受到一个请求，花了不少cpu时间处理完了，然后返回结果时，发现请求已经超时了，用户已经收到了类似“服务器繁忙，请稍后再试！”的提示。这里可能有用户会说，go不是有超时控制吗？这里有两点：
  - 超时不是在代码的每个指令处都能先判断是否超时再执行，比如我们有一个for循环，不会每次都先判断ctx是否超时，然后再执行下一次迭代，如果你真的这样写了，性能可能需要特别关注，得看你每次循环的计算和判断超时的开销对比
  - 即使能够比较好的判断超时，在侦测到超时之前也已经白费了一些系统资源处理请求了

#### 2.2.2 场景二（开启过载保护）
```yaml
Timeout: 1000
```

- 开启过载保护（默认）
- 超时 1s
- `loops 2 hey -c 200 -z 60m "http://localhost:8888/ping"`
  - `loops` 是我的一个 alias：`loops='fs() {for i in {1..$1}; do ${@:2} & done; wait}; fs'`，用来并行执行给定命令指定的次数
![image](https://github.com/kevwan/microservices-in-action/assets/1918356/0599e7db-0ba0-4914-8849-23e286304b5c)

- 总 qps 大概在 4000-5000 左右
- 拒绝了约 90% 的过载请求
![image](https://github.com/kevwan/microservices-in-action/assets/1918356/44169738-7a3d-4cb6-af1b-8534e5b9d8d4)

- 成功处理请求在 400-450 qps 左右
![image](https://github.com/kevwan/microservices-in-action/assets/1918356/4a3cae8f-4ba3-4e0d-8801-e1c498d88300)

- 处理时延 p99 约 600ms，p90 约 150ms

#### 2.2.3 结论：
- 流量未知的情况下，保障系统不卡死（无过载保护情况下，CPU 满载一般表现为大量请求超时），且保证了系统容量的 400 qps 正常提供服务
- 自动拒绝了过量的请求，避免过量请求浪费系统资源（即使处理，系统最后返回给用户的也是不可用错误、超时错误等）
