---
title: victoriametrics-vmstorage 存储索引原理
date: 2024-08-27 14:36:53
tags:
---

## 存储格式
prometheus 写入示例如下

```
http_request_total{status="200",method="GET"} 1724740700000 12345
http_request_total{status="200",method="GET"} 1724740800000 23456
```
当 vm 接收到请求时，会对请求中的时序数据做转换处理，根据 metric 和label.Name 生成 TSID，然后将`metric(指标名称__name__) + labels + TSID` 作为index，`TSID + timestamp + value` 作为data, index 和 data 分别进行存储和索引。

### TSID
vm 中的 metricName 结构如下：
```golang
type MetricName struct {
    AccountID uint32
    ProjectID uint32
    MetricGroup []byte # 指标名称 __name__
    Tags []Tag
}

type Tag struct {
    Key   []byte
    Value []byte
}

```
TSID 结构如下，除 MetricID 外其他字段都是可选的。
```golang
type TSID struct{
    AccountID     uint32
    ProjectID     uint32
    # 相同名称的一组指标，例如 http_request_total，有不同的标签
    # 但仍属于一个组
    MetricGroupID uint64
    # 采集任务 id 及，instanceid id，由 job 和 instance hash 生成
    JobID         uint32
    InstanceID    uint32
    # 进程启动时的系统纳秒时间戳自动生成
    MetricID      uint64
}
```
假设写入 `http_request_total{status="200",method="GET"}` 为例，则 metricName 为 `http_request_total{status="200",method="GET"}`，假设生成的 TSID 为`{metricGroup=0,jobID=0,instanceID=0,metricID="1241516442"}`则 vm 在写入时会构建几种类型的索引，其他类型的索引是再后台或者查询是构建的。

+ metricName -> TSID
+ metricID -> metricName
+ metricID -> TSID
+ tag -> TSID
+ tag -> metricID

在查询时，先根据给定的 tag 条件查找每个 tag 的 metricID 列表，然后计算所有 tag 的 metircID 的交集，再去索引文件中检索出 TSID，根据 TSID 到数据文件中查询数据，返回结果前根据 TSID 中的 metricID 查询原始 metricName。
tag -> metricID -> TSID -> data -> metricID -> metricName

### rawRow
```golang
type rawRow struct {
    TSID          TSID
    Timestamp     int64 # 时间戳
    Value         int64 # 当前 timestamp 的时间序列值
    PrecisionBits uint8 # 每个值要存储的精度位数，可能值为 [1,64]，低精度可能会减少磁盘占用，但
}

type MetricRow struct {
    MetricNameRaw []byte
    Timestamp     int64
    Value         float64
}
```

### Addrows 方法
查询
