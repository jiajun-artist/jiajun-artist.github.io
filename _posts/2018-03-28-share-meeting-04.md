---
layout: post
title: "第四次技术分享会 分布式 CDN"
tags: [技术分享会, CDN]
comments: true
date: 2018-03-28 22:00:00+08:00
---

##### 分享会日期：2018-03-28
##### 持续时间：16:30-20:30，4个小时

>新的一年里，想让自己有些改变、有些进步、有些其他方面的尝试，于是在公司面向技术同事组织了分享会，未来很长一段时期，应该都会由我来主持和主讲，因此在此做些记录，记录下成长轨迹。

>顺序可能有点乱，因为都是即兴发挥，脱稿讲解，可能有很多不足之处，希望能包涵。。。。

一. CDN
1. 介绍CDN的概念（源站、边缘节点），CDN是为了解决什么问题（边缘节点的低成本流量去分担骨干线路和业务服务器的流量，同时可以覆盖离用户更近的线路，提高网络体验）
2. 介绍公司CDN业务的流量规模，覆盖范围（6Gps峰值，单日上亿请求量）
3. 做CDN业务时，需要考虑的问题（静态文件缓存）
4. 基于3的问题，引出静态文件的新增或者更新会有哪两种方式（主动推和被动拉），区分这两种方式的优点和缺点，对业务的影响、侵入性等等
5. 静态文件的存储空间的选择，自由服务器还是云储存（七牛等）

二. 公司现在的自研CDN架构
1. 如果自研CDN，需要解决哪些问题（静态文件缓存、源站文件高可用存储、回源配置、边缘节点的高覆盖率、域名解析的地区覆盖度等等）
2. 基于1的问题，介绍nginx的cache_module和利用nginx的upstream搭建简单的cdn节点
3. 在不主动推文件到边缘节点的情况下，如何让节点文件更新迅速，如果在边缘节点同时更新文件时做到缓解回源的流量
4. 由3的问题引出二级缓存的概念（源站和边缘节点中间的一道缓冲、类似于redis在业务中的缓存地位）

三. 在DNS层面要做的操作
1. CDN的效果很大一部分来自于DNS层面的，处于运维层面与VPS流量限制方面的考虑，传统二级域名绑定机器或者绑定LB的方式无法适用，只能采用router做302跳转或者特定地区指向特定机器的方式。无论哪一种方式，都需要DNS层面做优化，比如让东南亚的用户能够直接访问新加坡的节点，让南美洲的用户能够直接访问巴西的节点等等
2. 在1的基础上，介绍DNS相关的概念（基于UDP、DNS回溯、TTL、ANAME、CNAME、顶级一级二级域名、全球根域名服务器等等）
3. 解决方案：公司采用商业服务 CloudXNS 去做DNS层面的地域渗透，方便节点集群的部署，尽量让节点覆盖高每一个大用户地区。对主要用户及主要收入地区，可以做特殊的优化，购买线路更高的VPS做边缘节点等等

四. 更完善的运维系统
1. 在公司自研CDN的基础上，去实现更详细的运维需求和运营需求（请求状态码统计、404，5XX统计、流量统计、分域名统计、分IP统计等等）
2. 基于1的问题，现在在节点的nginx层面做了分域名相关的流量区分回源，使得CDN业务能够承接不同的需求方、不同的回源方  
同时，在海外部署了ELK集群专门用的收集边缘节点通过filebeat传入的nginx请求日志，通过nginx中的 $bytes_sent 字段可以在kibana中详细的做状态码和流量方面的统计等等
3. 开放refresh刷新缓存的api供业务或者运维系统调用（边缘节点端依赖 proxy_cache_purge module）

五. 配置文件部分示例
###### http api的purge请求
```
location ~ ^/purge(/.*){
    proxy_cache_purge cache_one $1$is_args$args;
}
```

###### 分域名回源（类似网宿的方式）
```
if ( $uri ~* "\/((?:[a-z0-9]+\.)+[a-z0-9]+)\/(.+)" ) {
    set $origin_domain $1;
    proxy_pass http://$origin_domain/$2;
}
```

###### http header相关配置
```
### cdn head ###
proxy_cache cache_one;
proxy_cache_valid 200 302 1d;
proxy_cache_valid 301 1d;
proxy_cache_valid any 1d;
proxy_cache_lock on;
proxy_cache_lock_timeout 30s;
#proxy_cache_purge $purge_method;
proxy_cache_key         $uri$is_args$args;
add_header Source-Cache "$upstream_cache_status";
more_set_headers    "Content-Disposition";
more_set_headers    "Content-Transfer-Encoding";
more_set_headers    "Access-Control-Allow-Origin";
more_set_headers    "Access-Control-Expose-Headers";
more_set_headers    "Access-Control-Max-Age";
```
###### nginx_cache_module 相关配置
```
proxy_cache_path /mnt/volume-sgp1-04 levels=1:2 keys_zone=cache_one:200m max_size=350g inactive=1d
                 use_temp_path=off;
 include /etc/nginx/conf/conf.d/*.conf;
```


> 这次分享到此为止，下周会继续，主题将是golang相关的内容