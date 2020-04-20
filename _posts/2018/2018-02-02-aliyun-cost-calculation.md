---
layout: post
title: 阿里云的日常费用计算
tags: aliyun
---

> 阿里云的服务多如牛毛，其计费规则也是纷繁复杂。这里我总结了几个比较常用的服务，做了一个快速计算日常费用的简要说明。

### CDN

#### 首先，回顾一下 CDN 的计费说明，如下：

![cdn-details](/assets/img/posts/2018/ali-cost/cdn-details.png "CDN cost details")

#### 增值服务是针对 HTTPS 请求的特殊计费，说明如下：

![https-details](/assets/img/posts/2018/ali-cost/https-details.png "CDN HTTPS details")

#### 然后我们来看一下如何统计日常的 CDN 流量费用

* #### 进入控制台，选择 CDN -> 监控

![cdn-panel](/assets/img/posts/2018/ali-cost/cdn-panel.png "CDN panel")

* #### 根据需要，选择域名，地区（国内、外），时间段

![cdn-cctv](/assets/img/posts/2018/ali-cost/cdn-cctv.png "CDN CCTV")

* #### 把得到的流量填入下表（我选的时间段是一天）

![cdn-cal](/assets/img/posts/2018/ali-cost/cdn-cal.jpeg "CDN calculation")
注：HTTPS 请求数是选择 “访问次数” 子菜单，然后点击右侧下载 Excel 表格后，加总最后一列 “Total HTTPS Visits” 得到的

#### 这样你就能初步得到或者预估一年的流量开销了。然后基于此使用情况，再合理选择购买流量包，详情如下：

![cdn-package](/assets/img/posts/2018/ali-cost/cdn-package.png "CDN package")
![https-package](/assets/img/posts/2018/ali-cost/https-package.png "HTTPS package")
注：目前，CDN 的流量包只能抵国内的流量，国外的要另外按流量收费

#### 此外，对于 HTTPS 增值服务费，以 24.66 万次／天 请求为界，超过则选择请求包划算。计算过程如下表所示：

![https-cal](/assets/img/posts/2018/ali-cost/https-cal.jpeg "HTTPS calculation")

### OSS

OSS 的 “标准型” 存储包和 Snapshot 是共用的，这个放到 Snapshot 中讲，这里按流量计算

![oss-details](/assets/img/posts/2018/ali-cost/oss-details.png "OSS cost details")

* 进入控制台，选择 “对象存储OSS” -> 选中 bucket

![oss-panel](/assets/img/posts/2018/ali-cost/oss-panel.png "OSS panel")

* 查看概览的数据，“外网流量” 中可以选择 “流入”、“流出” 或 “回源”

![oss-cal](/assets/img/posts/2018/ali-cost/oss-cal.jpeg "OSS calculation")

### Snapshot

Snapshot 和 OSS 的“标准型” 存储包 是共用的

![snapshot-details](/assets/img/posts/2018/ali-cost/snapshot-details.png "Snapshot cost details")

* 进入控制台，选择 ECS -> 实例 -> 选择实例进入实例详情

![snapshot-example-1](/assets/img/posts/2018/ali-cost/snapshot-example-1.png "Snapshot Panel")

* 点击 “磁盘” 右边的数字（“1”）进入本实例云盘列表，记住该磁盘ID

![snapshot-example-2](/assets/img/posts/2018/ali-cost/snapshot-example-2.png "Snapshot Panel")

* 左边菜单栏选择 “快照链”，根据 “磁盘ID” 即可定位到目标 ECS 实例的快照容量

![snapshot-example-3](/assets/img/posts/2018/ali-cost/snapshot-example-3.png "Snapshot Panel")

另，通过菜单栏 “快照容量” 的显示可以直接指导目标区域所有的快照所占用的存储量。

---

## 参考资料：

* [Ali CDN 计费详细说明](https://www.aliyun.com/price/product?spm=5176.7933777.598288.btn3.7df156f5pK2gO2#/cdn/detail)
* [Ali OSS 计费详细说明](https://cn.aliyun.com/price/product?spm=5176.doc27271.2.4.6MFEZS#/oss/detail)
* [Ali Snapshot 计费详细说明](https://cn.aliyun.com/price/product?spm=5176.doc27271.2.4.6MFEZS#/snapshot/detail)