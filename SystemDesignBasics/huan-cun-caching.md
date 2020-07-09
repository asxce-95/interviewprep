# 缓存 Caching



## 概念

缓存效率远比数据库高，使用缓存可以减少对数据库的请求次数，提高系统效率，减少延时。


## 缓存的模式

不同的缓存有不同的运行模式，一般来说是根据缓存读和写的时间来做区分，下面是常见的缓存模式。

#### Cache-aside
Cache-aside是非常经典的模式，并没有一个官方的中文名。其运行原理很简单。

* 读的时候，先检查数据是否存在缓存里面，如果存在，直接返回。如果不存在，则从数据库中读取，读取后同时写入缓存。
* 写的时候，写入数据库，同时淘汰(Delete)已有缓存（这里有争议，有的地方认为直接替换(set)现有缓存，[讨论可以参考这里](http://www.mobabel.net/%E8%BD%AC%E7%BC%93%E5%AD%98%E6%9E%84%E6%9E%B6%E7%BB%8F%E9%AA%8C%E6%80%BB%E7%BB%93-5-2-cache-aside-pattern/)。
* 适用情况：无法预测哪些数据会被读取的时候，Cache-aside模式根据应用需求加载数据。

#### Read-through
另一种缓存模式是 Cache-as-SOR(System of Record)。也就是把缓存，除了Read-through之外，下面的 Write-through 和 Write-behind都是这种模式，只是类型不同。

Read-trhough模式和 Cache-aside模式非常类似，当应用层向数据存储系统请求数据的时候，如果数据存在缓存，则返回，否则去数据库存储数据。区别是Read-through一般是数据存储系统自带的，应用层（请求方）不需要自己处理缓存不存在的情况。

#### Write-through
Write-through模式是在写入数据的时候，应用层写入数据的时候，缓存系统将数据写入缓存，然后将数据写入数据库，最后才返回。这样之后要读取刚写入的数据可以直接从缓存中提取，性能较好。

缺点是很多时候写入缓存的数据并不会被读到，可以给缓存加上TTL来优化。 写的时候同时更新缓存和数据库，降低性能。

#### Write-behind/Write-back
当应用层往缓存写入数据的时候，缓存系统间隔一段时候之后异步将数据写入数据库。这种模式相对于Write-through来说性能有所提升，同时因为是异步写入，降低了数据库的负载。如果一个数据在写入数据库之前被更新，只有缓存被更新，最终写入数据库的是最新的数据。

缺点是因为缓存和数据库是异步更新，需要保证缓存写入数据库成功，否则出现数据丢失，或者不一致的问题。

## 常见的缓存
上面提到了常见的缓存模式，这里介绍一些常见的缓存应用，希望这些对于系统设计问题有所帮助。

#### CDN
CDN本身就是缓存，它在不同的节点存储较大的静态文件（比如视频，图片），这样在不同地区的用户可以访问到距离他们最近的节点，提高效率。

#### Client Side Cache
客户端多种多样，最常见的就是浏览器和手机App。客户端和服务器沟通的时候，为了提升响应熟读，服务器肯定不能一次性返回所有数据，否则当数据量很大的时候，客户端需要花很多时间加载，影响用户体验。

有的时候我们通过异步通信来解决，先加载一部分数据，然后再加载其他，同样我们也能用缓存来提升响应速度，对于一些时效性不高的数据可以提前写入缓存，比如搜索引擎一般有自动提示(type ahead/auto suggestion)功能，有时候自动提示的内容可以预先加载到缓存里面，用户输入的时候直接从缓存载入相应提示。再比如说一些图片、视频广告、音效等，可以放在缓存，这样不用每次都从服务器加载，占用带宽。

#### Server Side Cache
通常来说，服务器收到用户（客户端）请求之后，需要根据请求内容，做出一些处理，比如从数据库读数据，业务逻辑处理数据，最后生成返回内容，对于一个网站来说，返回内容可能是渲染好的html页面，或是json数据。一个请求经过这整个流程最后变成页面返回，需要调用很多资源，假设有很多用户发相同的请求，那么实际上就可以把请求(request)本身所对应的结果(html/json)存入缓存，这样下次遇到相同请求的时候可以直接返回对应结果，而不需要调用数据库，业务逻辑等等。

同样的道理，在应用层(Application-level)也可以做缓存，缓存的内容不限于渲染好的html页面，可以根据需求来调整缓存的颗粒度，比如可以存某个query的返回结果等等，不过需要做好权衡，更细颗粒度可能带来更复杂的业务逻辑。

#### Database Cache
通常很多数据库都会自带缓存功能，比如一些SQL数据库，就能缓存SQL Query的结果，下次相同的query被执行的时候直接返回结果。

#### Distributed Cache
现在许多大型网站都使用分布式缓存，实际上就是用多台机器（物理或者虚拟）来存放数据，把缓存作为一个服务（service）来使用。因为大型网站为了满足高并发的需求，防止单点失效，必须要使用分布式系统，同时因为大型网站业务逻辑复杂，将缓存拆分出来作为一个中间件能提升可维护性等，常见的有Redis，Memcached等。

## 缓存的优缺点

总的来说，不管缓存模式是什么，他们都是为了解决类似的问题，各个模式虽然不同，优缺点大致类似，在此做出总结。

优点

* 访问速度快，能提升系统响应速度（性能）。
* 作为缓冲层，可以减少数据库的负载。

缺点：

* 不容易保持缓存和数据库之间数据的一致性，选择合适的更新时机和淘汰算法并不容易
* 增加一层缓存就增加了一层复杂（another layer of complexity），增加工作量。

## 缓存使用场景

缓存可以再任何地方使用，系统设计过程中，往往会涉及到客户端\(client\)，服务器端\(server\)，和数据库端的设计，在任何一个环节我们都可以考虑使用缓存来提高响应速度。同时，缓存也可以作为一个单独的层级\(layer\)存在。一般情况下，在以下场景可以考虑使用缓存：

* 对数据一致性实时性要求不高的时候。
* 对性能要求高，短时间需要处理大量请求的时候（突发活动，抢票...）

## 缓存淘汰算法 - Eviction Policies

缓存容量有限，当容量到达上线的时候，新写入的数据肯定需要按照某种算法淘汰旧的数据，这些算法就是缓存的淘汰算法\(eviction policy\)，常见的淘汰算法有：

* First In First Out \(FIFO\): 把缓存看成一个队列，遵循先进先出原则，不考虑数据访问是否频繁，当缓存满的时候，把最先进入缓存的数据给淘汰掉。 
  * set\(key, value\), 如果缓存中存在该key，则重置value，如果不存在，则将该key插入，如果缓存已满，淘汰最先进入数据
* Last In First Out \(LIFO\): 和FIFO相反。
* Least Recently Used \(LRU\): 非常常用的缓存算法，基于“如果一个数据在最近一段时间没有被访问到，那么在将来它被访问的可能性也很小”的思路，Leetcode上有LRU的题目，建议自己实现。
* Least Frequently Used \(LFU\): 使用最不频繁的最先淘汰，基于“如果一个数据在最近一段时间内使用次数很少，那么在将来一段时间内被使用的可能性也很小”的思路。Leetcode上有LFU的题目，建议自己实现。

当然还有其他的，MRU\(Most Recently Used\)、MFU\(Most Frequently Used\), Random Replacement等等，看名字大概就知道什么意思了，具体可以参考[维基百科 Cache Replacement Policies](https://en.wikipedia.org/wiki/Cache_replacement_policies)

## 常用缓存系统使用经验总结 
关于缓存的热点，惊群效应，击穿，并发，一致性等等，一般面试不会涉及这些内容，但是这些内容对实际应用很有帮助。参考此文《[常用缓存系统使用经验总结](https://www.jianshu.com/p/c1b9ec30b994)》。

## 参考文章

* http://www.mobabel.net/%E8%BD%AC%E7%BC%93%E5%AD%98%E6%9E%84%E6%9E%B6%E7%BB%8F%E9%AA%8C%E6%80%BB%E7%BB%93-5-2-cache-aside-pattern/
* http://www.ehcache.org/documentation/3.6/caching-patterns.html

