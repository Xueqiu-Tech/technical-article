雪球,聪明的投资者都在这里 - 4300万投资者都在用的投资社区，沪深港美全球市场实时行情，股票基金债券免费资讯，与投资高手实战交流。

# 背景

雪球为用户提供稳定的投资实时资讯，内部对于沪深港美全球市场实时行情进行了双活改造，其中涉及到架构升级问题，而跨机房的用户鉴权是其中重要的一环。

现有的鉴权方式是通过客户端 SDK 集成，由基础组件完成 RPC 的调用、缓存、解析、处理等一系列的 Token 校验流程。而在双活架构下，跨机房时延的存在使得既有的校验模式，对已有的 SLA 产生重大影响，同时还需升级鉴权模式，减少业务端代码侵入、降低耦合。

不支持在 Docs 外粘贴 block

# 方案选型

## 一、基于已有的Openresty演绎

早期，雪球网关基于 Openresty 自主研发设计。但当前存在众多挑战，如网关代码模块的维护以及整体的可观测性等。另外，因历史原因出现断层、继续迭代维护较难、迭代升级需要投入较多人力成本以及难以满足当前业务提出的众多需求等因素，使得该方案不适合当前架构设计。

## 二、使用Java或Go编写gateway

当前，雪球内部某业务线存在一套基于Spring Cloud Zuul编写的Gateway网关。基于现状和未来规划评估，方案的选型要考虑到后续的传承和整体投入，需要满足当前业务需求的同时，不引入过多的开发成本，同时还要经得起流量的考验；基于 Java 或 Golang 的方案设计只是换了种方式，继续走了自己造轮子的老路，同样也不适合当前的架构设计。

## 三、采用开源的APISIX或Kong

### Kong

Kong 的功能强大且丰富，在国内有较高的社区活跃度，在各个互联网公司具有很多落地案例，是当前较好的选择之一。同时，当前需要的基本功能插件都有，如：gRPC Proxy、gRPC Transcode、 HTTP(S)、HTTP2、QUIC等。但是，官方主推的 PostgreSQL 关系型数据库是单点，无法支持弹性扩缩容，是架构选型的硬伤。

### APISIX

目前，雪球内部架构向着云原生方向演进，因此想到在 CNCF 内部选择适合的设计，根据社区的推荐找到了APISIX。其依赖于 etcd 解决了单点问题，可用性更高；满足了可用性的同时，APISIX的社区活跃度和性能指标根据官方数据也满足了当前需求；除了上述 Kong 里简述的功能插件外，对于一些开源或者 SaaS 服务：SkyWalking、Prometheus等也提供了很好的支持，方便接入雪球既有的云原生基础设施；最后一个就是社区支持力度，在下文中将会有样例和描述。

基于上述方案的对比，最终选定使用Apisix作为使用方案。

# 在雪球的落地应用

## 一、网关鉴权

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d8e7f0000a9a44ae99e0638e40080644~tplv-k3u1fbpfcp-zoom-1.image)

基于APISIX网关鉴权三种实现方案：

1.  Lua 直接 gRPC 调用。但此方案需要 Load Balancing, Dynamic Upstream 等相关工作，故舍弃。
1.  通过 Golang 完成相关工作，Lua 协程回调。但此方案我司缺少相关具体实现经验，故舍弃。
1.  Lua 进行 Http 调用，gRPC 接口采用 [grpc-transcode](https://apisix.apache.org/zh/docs/apisix/plugins/grpc-transcode) 代理实现。其特点如下：

好处：

-   借助社区的能力，实现方式上也最简单。
-   根据实际压测性能指标满足既有业务场景。

坏处：

-   使用 APISIX 代理了gRPC之后，对于 proto 文件的管理目前手动配置，鉴权服务的 proto 文件变更和网关的联动还是留有一定风险。

综上所述，鉴权部分采用官方的既有插件，同时结合自身业务改造。该部分已经在公司内部实现了业务迭代跟进。同时丰富的社区内容和架构设计，过程中有社区帮助改进也少走了很多弯路，关于 gRPC 服务代理部分会在下文中具体阐述。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0e807335c19a49ebaf5a2a653d4896ae~tplv-k3u1fbpfcp-zoom-1.image)

## 二、可观测性

根据官方 [prometheus](https://apisix.apache.org/zh/docs/apisix/plugins/prometheus) 配置，只需在 conf.yaml 文件中添加：

```
plugin_attr:

  prometheus:

    export_addr:

      ip: 192.168.X.X

      port: 9091
```

相关的安全规则，可以按照自身的基础设施配置，对应的 grafana dashboard 配置官方提供的有具体样例，可实现监控系统快速接入。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/76b825e11bce43cb832bbcb083199e95~tplv-k3u1fbpfcp-zoom-1.image)

对于日志部分，官方提供了Http、Tcp、Udp、Kafka、[阿里云sls](https://apisix.apache.org/zh/docs/apisix/plugins/sls-logger)等若干常用实现，基于插件快速接入，同时各项指标与 Nginx 规则相同，可复用既有的基础设施。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ee28d0ebe93e4d948f342e0b7c693370~tplv-k3u1fbpfcp-zoom-1.image)

## 三、扩展自定义插件

雪球行情应用业务在某阶段对于部分服务在某机房有用户 UID 白名单的需求，其中 UID 需要做到动态可配置。

根据社区[插件开发](https://apisix.apache.org/zh/docs/apisix/plugin-develop)文档，参照既有的插件源码，可快速完成功能开发。在开发过程需注意以下几个方面：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2638cd7a386a4b4eaac63eb0ce602b1d~tplv-k3u1fbpfcp-zoom-1.image)

另外，结合实践发现一个小窍门，官方的源码库下已经覆盖了大量工具库和方法，在编写过程中尽可能复用减少不少的工作。

注意：自定义插件在 APISIX Dashboard 中的显示，需要手动的将 APISIX 下的 scheme [导入过去](https://github.com/apache/apisix-dashboard/blob/master/docs/en/latest/FAQ.md)。

实现效果：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f142296b05bc4ca8b72c26f063acc051~tplv-k3u1fbpfcp-zoom-1.image)

## 四、TLS双向认证

APISIX 支持通过 TLS 扩展 [SNI](https://apisix.apache.org/zh/docs/apisix/certificate/) 加载特定的 SSL 证书，以实现对 https 的支持。使用时遇见一个小插曲，雪球由于历史原因，Android、IOS、Web三端的TLS调用认证有部分差别。

IOS 客户端上报的没有 SNI：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cbef4b74ec8c4e69ba54af621598d402~tplv-k3u1fbpfcp-zoom-1.image)

Android 客户端上报的有 SNI：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e5aff87203ec4d3095b2338d3ce7ef5c~tplv-k3u1fbpfcp-zoom-1.image)

由于IOS的 SNI 上报问题，导致网关一直报错：

> failed to fetch ssl config: failed to find SNI: please check if the client requests via IP or uses an outdated protocol

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3d2c970d8d434432b544ae12e61aeed4~tplv-k3u1fbpfcp-zoom-1.image)

针对该问题，进行了网络抓包和代码调试，耗时很久没有定位到问题原因。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/47c1e7cdeaa5408ab693116f7697ae32~tplv-k3u1fbpfcp-zoom-1.image)

最后，在官方社区帮助下，发现通过增加一个默认配置 [fallback SNI](https://github.com/apache/apisix/blob/master/conf/config-default.yaml#L150)，就能给 SNI 为空的 ssl 连接设置对应的证书了，fallback 到诸如 ios.xueqiu.com 的域名，这样含有 *.xueqiu.com 的证书就能匹配到了。

```
function _M.server_name()

    local sni, err = ngx_ssl.server_name()

    if err then

        return nil, err

    end

    -- sni为空，fallback到默认SNI

    if not sni then

        local local_conf = core.config.local_conf()

        sni = core.table.try_read_attr(local_conf, "apisix", "ssl", "fallback_sni")

        if not sni then

            return nil

        end

    end



    sni = str_lower(sni)

    return sni

end
```

## 五、管理gRPC服务

目前，雪球内部现有的 gRPC 服务调用是基于 Zookeeper 注册中心自研的 RPC 框架实现。对于负载均衡、限流、熔断降级等功能主要依赖于框架和组件实现，现有的业务方在使用的便捷性上存在一定的局限。

接上文的服务代理部分，根据官方提供的[集成服务发现注册中心](https://apisix.apache.org/zh/docs/apisix/discovery)文档，进行基于 Zookeeper 注册中心扩展，实现流程较为清晰，基本步骤如下：

1.  在 apisix/discovery/ 目录中添加 Zookeeper 注册中心客户端的实现；
1.  实现用于初始化的 _M.init_worker() 函数以及用于获取服务实例节点列表的 _M.nodes(service_name) 函数；
1.  将 Zookeeper 注册中心数据转换为 APISIX 格式的数据；

```
local function fetch_full_registry(premature)

    ...



    for _, service_info in ipairs(infos) do

        local data, err



        local query_path = 'v1/ls/' .. service_info.service_name

        log. info("query_path:", query_path)

        data, err = get_url(base_uri, query_path)

        if err then

            log.error('get_url:', query_path, ' err:', err)

            goto CONTINUE

        end



        log. info("state:", data.state)

        for _, children in ipairs(data.childrens) do

            log. info("children:", children)



            query_path = 'v1/get/' .. service_info.service_name .. '/'  .. children .. '+json'

            log. info("query_path_2:", query_path)

            data, err = get_url(base_uri, query_path)

            if not data.data then

                log.error('get_url:', query_path, ' not found')

                goto CONTINUE

            end



            ...

            -- 解析注册信息的地址和端口

            if service_info.upstream_scheme == 'http' then

                instance_port = instance.port

                flag = true

            end

            if service_info.upstream_scheme == 'grpc' then

                instance_port = instance.metadata.grpc.port

                flag = true

            end



            if flag then

                log. info("host: ", instance_host, ",port:", instance_port)

                core.table.insert(nodes, {

                    host = instance_host,

                    port = instance_port,

                    weight = 100,

                })

            end

        end

        :: CONTINUE ::

    end

    applications = up_apps

end
```

实现的难点在于 Zookeeper 注册中在 Openresty 体系下很难找到具体的实现。应对这一问题我司采用的实现方式为，通过 Zookeeper REST 代理将注册中心的数据暴露出来，其中的 watcher 机制使用定时轮询的方式替代。这个实现目前虽说不很完美，但是一定程度上解决了眼前问题，目前社区里也有很多同学遇见类似的问题，欢迎大家[一起参与讨论](https://github.com/apache/apisix/issues/3231)，最终能够给出开源的解决方案。

# 未来规划

## 一、Kubernetes使用 APISIX Ingress Controller

在部署 Kubernetes 的时候，一般都会选择 Nginx-Ingress 来处理外网请求，但因为当前Nginx 存在一些缺陷，我司正在调研使用 APISIX Ingress Controller。

## 二、兼容全部既有网关功能

目前 APISIX 网关在雪球主要承载了鉴权和 gRPC 调用相关的功能模块，其他大部分功能模块还是由原有的 Openresty 体系承载。其中涉及到大量的历史细节，在后续的规划和梳理中会伴随着需求迭代，一步步地通过统一的技术栈实现。

## 三、管理全域南北流量

当前 APISIX 网关在雪球主要是在新项目中应用，其他大部分历史项目还是由多个 Nginx 集群进行处理，一直处于稳定运行状态。面临着大量配置梳理等问题，下一步借助开源社区的能力实现投入与受益的最佳平衡。

# 参考连接

[Blog | Apache APISIX® -- Cloud-Native API Gateway](https://apisix.apache.org/zh/blog/)

[Lua 级别 CPU 火焰图简介 -- 章亦春](https://blog.openresty.com.cn/cn/lua-cpu-flame-graph/)

[API 网关 Apache APISIX 和 Kong 的选型对比 -- 温铭](https://zhuanlan.zhihu.com/p/103236688)

[又拍云·再谈 APISIX 高性能实践 -- 王院生](https://www.upyun.com/tech/article/454/OpenResty%20%E7%A4%BE%E5%8C%BA%E7%8E%8B%E9%99%A2%E7%94%9F%EF%BC%9AAPISIX%20%E7%9A%84%E9%AB%98%E6%80%A7%E8%83%BD%E5%AE%9E%E8%B7%B5.html)

[深入聊聊微服务架构的身份认证问题 -- 王海龙](https://www.infoq.cn/news/identity-authentication-of-architecture-in-micro-service)