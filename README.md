
# 一、概述


      上篇文章介绍了[基于surging的木舟平台如何构建起微服务](https://github.com/fanliang11/p/18541040) ，那么此篇文章将介绍基于木舟平台浅谈surging 的热点KEY的解决方法


      木舟 (Kayak) 是什么?


       木舟(Kayak)是基于.NET6\.0软件环境下的surging微服务引擎进行开发的, 平台包含了微服务和物联网平台。支持异步和响应式编程开发，功能包含了物模型,设备,产品,网络组件的统一管理和微服务平台下的注册中心，服务路由，模块，中间服务等管理。还有多协议适配(TCP,MQTT,UDP,CoAP,HTTP,Grpc,websocket,rtmp,httpflv,webservice,等),通过灵活多样的配置适配能够接入不同厂家不同协议等设备。并且通过设备告警,消息通知,数据可视化等功能。能够让你能快速建立起微服务物联网平台系统。 


      木舟kayal 平台开源地址：[https://github.com/microsurging/](https://github.com)


      surging 微服务引擎开源地址：[https://github.com/fanliang11/surging](https://github.com):[蓝猫机场加速器](https://dahelaoshi.com)（后面surging 会移动到[microsurging](https://github.com)进行维护）


## 二、缓存热点Key的问题


1. 什么是热点key的问题
就是某个瞬间有大量的请求去访问Redis上某个固定的key，导致缓存击穿，请求都打到了DB上，压垮了缓存服务和DB服务，从而影响到服务的可用性；
2. 怎么样会成为热点Key


          (1\)、 QPS 集中访问频次占比比较高的会被称为热点Key,木舟平台会添加基于routepath访问频次统计，让技术人员查找出排名靠前的热点KEY，


        （2\)、Value数据集合非常大导致带宽占用比较高会被称为热点KEY.


         3\.热点KEY的危害


          (1\)、占用带宽影响其它服务调用


          (2\)、请求过大，降低了其它缓存调用性能


           (3\)、缓存击穿，DB被压垮，引起业务雪崩。


## 三、基于surging 如何解决热点Key的问题


       1\.基于MemoryCache缓存拦截


     访问频次比较高，数据不经常修改，而无需其它微服务共享调用的时候就可以使用MemoryCache进行缓存在本地，就比如木舟平台首页的产品，设备，设备消息统计，如下图


![](https://img2024.cnblogs.com/blog/192878/202411/192878-20241120205027632-1550011638.png)


 你可以添加以下特性就能开启缓存拦截，Mode选择CacheTargetType.MemoryCache




```
        [ServiceCacheIntercept(CachingMethod.Get, Key = "GetProductStatistics", CacheSectionType = "ddlCache", EnableL2Cache = false, Mode = CacheTargetType.MemoryCache, Time = 1, EnableStageCache = true)]
        Task> GetProductStatistics();
```


删除的时候就可以使用CachingMethod.Remove，传入"GetProducts", "GetProductStatistics"， 如果需要传入其它参数值就可以添加\_{0}\_{1} ，比如 GetProductsByName\_{0}




```
        [ServiceCacheIntercept(CachingMethod.Remove, "GetProducts", "GetProductStatistics", CacheSectionType = "ddlCache", Mode = CacheTargetType.MemoryCache, EnableStageCache = true)]
        [ServiceLogIntercept]
        Taskbool>> DeleteById(List<int> ids);
```


2\. 基于redis 缓存


     访问频次比较高，数据不经常修改，但需其它微服务共享调用的时候就可以使用Redis进行缓存，就比如获取Token,就需要开启redis缓存拦截，可以在添加上添加，修改代码：Mode \= CacheTargetType.Redis，如下图：


 




```
        [ServiceCacheIntercept(CachingMethod.Get, Key = "GetProductStatistics", CacheSectionType = "ddlCache", EnableL2Cache = false, Mode = CacheTargetType.Redis, Time = 1, EnableStageCache = true)]
        Task> GetProductStatistics();
```


 


3\.二级缓存


访问频次比较高，数据会经常修改，Value数据集合非常大会导致占用带宽，这时候使用二级缓存是最适合的，因为大的数据集合会通过二级本地缓存读取，一级缓存存储标志位来管理二级缓存的失效，代码如下




```
        [Metadatas.ServiceCacheIntercept(Metadatas.CachingMethod.Get, Key = "GetUserId_{0}", CacheSectionType = "ddlCache", L2Key= "GetUserId_{0}",  EnableL2Cache = true, Mode = Metadatas.CacheTargetType.Redis, Time = 480,EnableStageCache =true)]
```


4\. 缓存中间件的分片处理


缓存中间件使用了哈希一致性负载分流算法，这样就可以把不同的KEY分散到不同的服务节点上，也保证热点KEY的集中访问的问题，可以在cacheSettings配置文件中添加redis服务节点,配置文件代码如下：




```
{
  "CachingSettings": [
    {
      "Id": "ddlCache",
      "Class": "Surging.Core.Caching.RedisCache.RedisContext,Surging.Core.Caching",
      "InitMethod": "",
      "Maps": null,
      "Properties": [
        {
          "Name": "appRuleFile",
          "Ref": "rule",
          "Value": "",
          "Maps": null
        },
        {
          "Name": "dataContextPool",
          "Ref": "ddls_sample",
          "Value": "",
          "Maps": [
            {
              "Name": "Redis",
              "Properties": [
                {
                  "Name": null,
                  "Ref": null,
                  "Value": "127.0.0.1:6379::1",
                  "Maps": null
                },
                {
                  "Name": null,
                  "Ref": null,
                  "Value": "127.0.0.1:6379::1",
                  "Maps": null
                },
                {
                  "Name": null,
                  "Ref": null,
                  "Value": "127.0.0.1:6379::1",
                  "Maps": null
                }
              ]
            },
            {
              "Name": "MemoryCache",
              "Properties": null
            }
          ]
        },
        {
          "Name": "defaultExpireTime",
          "Ref": "",
          "Value": "120",
          "Maps": null
        },
        {
          "Name": "connectTimeout",
          "Ref": "",
          "Value": "120",
          "Maps": null
        },
        {
          "Name": "minSize",
          "Ref": "",
          "Value": "1",
          "Maps": null
        },
        {
          "Name": "maxSize",
          "Ref": "",
          "Value": "10",
          "Maps": null
        }
      ]
    },
    {
      "Id": "userCache",
      "Class": "Surging.Core.Caching.RedisCache.RedisContext,Surging.Core.Caching",
      "InitMethod": "",
      "Maps": null,
      "Properties": [
        {
          "Name": "appRuleFile",
          "Ref": "rule",
          "Value": "",
          "Maps": null
        },
        {
          "Name": "dataContextPool",
          "Ref": "ddls_sample",
          "Value": "",
          "Maps": [
            {
              "Name": "Redis",
              "Properties": [
                {
                  "Name": null,
                  "Ref": null,
                  "Value": "127.0.0.1:7000::1",
                  "Maps": null
                },
                {
                  "Name": null,
                  "Ref": null,
                  "Value": "127.0.0.1:7005::1",
                  "Maps": null
                },
                {
                  "Name": null,
                  "Ref": null,
                  "Value": "127.0.0.1:6379::1",
                  "Maps": null
                }
              ]
            },
            {
              "Name": "MemoryCache",
              "Properties": null
            }
          ]
        },
        {
          "Name": "defaultExpireTime",
          "Ref": "",
          "Value": "120",
          "Maps": null
        },
        {
          "Name": "connectTimeout",
          "Ref": "",
          "Value": "120",
          "Maps": null
        },
        {
          "Name": "minSize",
          "Ref": "",
          "Value": "1",
          "Maps": null
        },
        {
          "Name": "maxSize",
          "Ref": "",
          "Value": "10",
          "Maps": null
        }
      ]
    }
  ]
}
```


## 四、总结


  木舟平台api,ui已经开源发布，后面陆续更新，等完成mqtt和国标28181设备接入，会搭建官方网站和DEMO，敬请期待。


 


