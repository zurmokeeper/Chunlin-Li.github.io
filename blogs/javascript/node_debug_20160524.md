Debug 纪实
============

> 请求被卡住, node 进程的堆外内存不断升高.
> 使用 zlib 对请求/响应中的数据进行解压/压缩时的性能问题.
> 同步 vs 异步

似乎很多难以排查的 Bug 都或多或少和内存问题有些关系. 又一次遇到内存问题, 而且这次似乎更加严重了.

这几天只要请求量变大, 就会导致所有的实例的 RSS 内存开始不断上涨. 但是通过 debug 接口查看内部的堆内存却并不是太高. 极端情况下, RSS 内存是堆内存的10倍甚至更多.

请求我们的上游服务发起的请求都严重超时, 导致服务器大量 socket 资源被耗尽, 最终导致无法正常运行. 

通过之前做的一些 debug 接口, 发现服务的 qps 时高时低; 而实例内部并发的请求数有时甚至达到 8000, 是正常值的百倍以上. 

根据[之前的经验](blogs/javascript/node_debug_20160305.md)打开实例的堆内存 dump 开关, 捕捉到出问题的时候的内存 snapshot 进行分析.

抓到的 dump 有1200MB, 但是从 chrome 的 dev tool 的 summary 中能看到的 retain size 只有总大小的 20%. 而 statistic 中显示 95% 都是 System Objects.

缺乏明确指引的情况下, 只能靠肉眼逐个排查了, 最后发现 ZLIB 的实例数量非常多. 这是一个很重要的线索. 我们业务中会使用 ZLIB 的 inflate 和 deflate 对业务中的一些数据
进行压缩解压操作, 不知道会不会是因为 zlib 执行效率太低导致积压了大量请求呢? 

我们写了一个临时的 debug 接口, 将压缩解压的操作拿出来放在这个接口中, 然后发布到线上进行测试(本地测试都是没问题的). 发布后一直观察并发数, 一旦并发数上去, 就使用测试
接口查看 zlib 的处理是否正常, 结果如预期一样, 请求直接被卡住, 要等待很久(甚至超过1分钟)才能返回. 
 
前段时间为了优化性能还专门将原本同步的 zlib 方法改成了异步的, 怎么并没有达到预期的效果呢? 这时突然想到了以前遇到的一个类似的[同步异步性能问题](blogs/javascript/node_performance_benchmark.md).
在这个问题中, 同样的 murmurhash 实现, 使用异步的性能反而不如同步. 受此启发, 给程序又加了一个测试接口, 使用同步的 zlib 方法. 再次测试后发现当异步方法 hang 住的时候,
同步方法依然能很快的响应.

于是接下来将所有 zlib 的调用从异步方式又改回了同步方式. 问题解决!

可是为什么呢? 只能用实测来找出答案了.

在本地单独写了段测试代码, 测试异步串行方式, 异步多并发方式, 同步方式, 对比它门的吞吐量, 响应时间, CPU占用. 结果同步方式在各个方面完胜异步方式.
在同样的环境下测试, 同步方式具有更高的吞吐量(近2倍), 更低的超时次数(1/10), 以及更低的 CPU 占用率. 这样, 就完全没有任何理由去使用异步方式了... 


据此推断: 对于任何计算密集型的任务, 如果每次的计算量较小, 而调用次数非常多, 那么千万不要用异步方式来处理. 异步方式只适用于较低并发, 且每次计算量较大的情形.
