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

![数据存储](images/vm-index-data.png)

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

+ metricName -> TSID：`http_request_total{status="200",method="GET"}` -> `{metricGroup=0,jobID=0,instanceID=0,metricID="1241516442"}`
+ metricID -> metricName：`1241516442` -> `http_request_total{status="200",method="GET"}`
+ metricID -> TSID：`1241516442` -> `{metricGroup=0,jobID=0,instanceID=0,metricID="1241516442"}`
+ tag -> TSID：`status="200"` -> `{metricGroup=0,jobID=0,instanceID=0,metricID="1241516442"}`, `method=GET` -> `{metricGroup=0,jobID=0,instanceID=0,metricID="1241516442"}`, `__name__=http_request_total` -> `{metricGroup=0,jobID=0,instanceID=0,metricID="1241516442"}`
+ tag -> metricID：`status="200"` -> `1241516442`, `method=GET` -> `1241516442`, `__name__=http_request_total` -> `1241516442`

在查询时，先根据给定的 tag 条件查找每个 tag 的 metricID 列表，然后计算所有 tag 的 metircID 的交集，再去索引文件中检索出 TSID，根据 TSID 到数据文件中查询数据，返回结果前根据 TSID 中的 metricID 查询原始 metricName。
tag -> metricIDs -> TSID -> data -> metricID -> metricName
1. 根据 tag 查询 metricID 列表
2. 根据 metricsID 查询 TSID
3. 根据 TSID 查询数据
4. 根据 TSID 中的 metricID 查询写入时的原始 MetricName

在查询时，需要 tag->metricID 的索引，因此需要多次查询才能将一个 tag 对应的所有 metricID 才能查询出来，查询出结果后仍需查询索引文件才能查询文件数据，多了一个 IO开销。

### rawRow
```golang
// rawRow 原始时间序列行
type rawRow struct {
    TSID          TSID
    Timestamp     int64 # 时间戳
    Value         int64 # 当前 timestamp 的时间序列值
    PrecisionBits uint8 # 每个值要存储的精度位数，可能值为 [1,64]，低精度可能会减少磁盘占用，但
}

// MetricRow 插入到存储中的指标
type MetricRow struct {
    MetricNameRaw []byte
    Timestamp     int64
    Value         float64
}
```

### Addrows 方法
``` golang
func (s *Storage) AddRows(mrs []MetricRow, precisionBits uint8) {
	if len(mrs) == 0 {
		return
	}

	// Add rows to the storage in blocks with limited size in order to reduce memory usage.
	ic := getMetricRowsInsertCtx()
	maxBlockLen := len(ic.rrs)
	for len(mrs) > 0 {
		mrsBlock := mrs
		if len(mrs) > maxBlockLen {
			mrsBlock = mrs[:maxBlockLen]
			mrs = mrs[maxBlockLen:]
		} else {
			mrs = nil
		}
		s.add(ic.rrs, ic.tmpMrs, mrsBlock, precisionBits)
		s.rowsAddedTotal.Add(uint64(len(mrsBlock)))
	}
	putMetricRowsInsertCtx(ic)
}
```

### 获取TSID
```golang
// lib/storage/storage.go
func (s *Storage) add(rows []rawRow, dstMrs []*MetricRow, mrs []MetricRow, precisionBits uint8) {
  // 获取当前使用的索引
	idb := s.idb()

	var (
		// 用于加速同一 metricName 的多个相邻行的批量插入
		prevTSID          TSID
		prevMetricNameRaw []byte
	)
	var metricNameBuf []byte

	var slowInsertsCount uint64
	var newSeriesCount uint64
	var seriesRepopulated uint64

	// 获取该数据块的最小时间和最大时间
	minTimestamp, maxTimestamp := s.tb.getMinMaxTimestamps()

	// 带有第几代索引信息的 TSID 对象
	var genTSID generationTSID

	// 只返回第一个错误，返回的错误没有意义？
	var firstWarn error

	j := 0
	for i := range mrs {
		mr := &mrs[i]
		var isStaleNan bool
		if math.IsNaN(mr.Value) {
		  // 如果指标值为 NaN(not a number) 则跳过
			if !decimal.IsStaleNaN(mr.Value) {
				continue
			}
			isStaleNan = true
		}
		// 如果指标已经过期，则跳过
		if mr.Timestamp < minTimestamp {
			continue
		}
		// 同样跳过最大时间戳的数据，maxTimestamp = now + 2d
		if mr.Timestamp > maxTimestamp {
			continue
		}
		dstMrs[j] = mr
		r := &rows[j]
		j++
		r.Timestamp = mr.Timestamp
		r.Value = mr.Value
		r.PrecisionBits = precisionBits

		// 快速路径 -> 当前 mr 与前一个 mr 指标名称相同 -> tsid 相同
		if string(mr.MetricNameRaw) == string(prevMetricNameRaw) {
			r.TSID = prevTSID
			continue
		}
		// 判断 tsid 是否在缓存中(命中缓存)
		if s.getTSIDFromCache(&genTSID, mr.MetricNameRaw) {
		  // 触发限流
			if !s.registerSeriesCardinality(r.TSID.MetricID, mr.MetricNameRaw) {
				j--
				continue
			}
			// 快速路径，通过 metricName 找到 TSID，并且未删除
			// 更新前一个 TSID&MetricName 的值
			r.TSID = genTSID.TSID
			prevTSID = r.TSID
			prevMetricNameRaw = mr.MetricNameRaw

			// 找到的 TSID 不是当前代的索引，重新创建索引
			if genTSID.generation < generation {

				createAllIndexesForMetricName(is, mn, &genTSID.TSID, date)
				// is.createGlobalIndexes(tsid, mn)
				// is.createPerDayIndexes(date, tsid, mn)
				// 更新当前代数
				genTSID.generation = generation
				// 更新缓存
				s.putSeriesToCache(mr.MetricNameRaw, &genTSID, date)
			}
			continue
		}

		// 慢速路径，缓存中未找到 TSID
		slowInsertsCount++

		// 在索引文件中通过M MetricName 查找 TSID
		if is.getTSIDByMetricName(&genTSID, metricNameBuf, date) {
			// 慢速路径 - 在索引文件中找到 TSID

			// 触发限流
			if !s.registerSeriesCardinality(genTSID.TSID.MetricID, mr.MetricNameRaw) {
				continue
			}

			if genTSID.generation < generation {
				// TSID.generation 不是当前代，重新创建
				createAllIndexesForMetricName(is, mn, &genTSID.TSID, date)
				genTSID.generation = generation
				seriesRepopulated++
			}
			// 更新索引
			s.putSeriesToCache(mr.MetricNameRaw, &genTSID, date)

			// 更新
			r.TSID = genTSID.TSID
			prevTSID = genTSID.TSID
			prevMetricNameRaw = mr.MetricNameRaw
			continue
		}

		// If sample is stale and its TSID wasn't found in cache and in indexdb,
		// then we skip it. See https://github.com/VictoriaMetrics/VictoriaMetrics/issues/5069
		// cache&indexdb 中未找到TSID，跳过 Prometheus 过时标记以外的 NaN，因为底层编码不知道如何使用它们。
		if isStaleNan {
			j--
			continue
		}

		// 慢速路径 - 重新生成 TSID
		generateTSID(&genTSID.TSID, mn)

		// 触发限流
		if !s.registerSeriesCardinality(genTSID.TSID.MetricID, mr.MetricNameRaw) {
			j--
			continue
		}

		// 创建索引
		createAllIndexesForMetricName(is, mn, &genTSID.TSID, date)
		// 更新当前数据
		genTSID.generation = generation
		s.putSeriesToCache(mr.MetricNameRaw, &genTSID, date)
		newSeriesCount++

		r.TSID = genTSID.TSID
		prevTSID = r.TSID
		prevMetricNameRaw = mr.MetricNameRaw

		if logNewSeries {
			logger.Infof("new series created: %s", mn.String())
		}
	}

	s.slowRowInserts.Add(slowInsertsCount)
	s.newTimeseriesCreated.Add(newSeriesCount)
	s.timeseriesRepopulated.Add(seriesRepopulated)

	dstMrs = dstMrs[:j]
	rows = rows[:j]

	// TSID 填充完成，可以插入数据
	if err := s.prefillNextIndexDB(rows, dstMrs); err != nil {
		if firstWarn == nil {
			firstWarn = fmt.Errorf("cannot prefill next indexdb: %w", err)
		}
	}
	if err := s.updatePerDateData(rows, dstMrs); err != nil {
		if firstWarn == nil {
			firstWarn = fmt.Errorf("cannot not update per-day index: %w", err)
		}
	}

	if firstWarn != nil {
		storageAddRowsLogger.Warnf("warn occurred during rows addition: %s", firstWarn)
	}

	s.tb.MustAddRows(rows)
}
```
循环数据，将不符合预期的时间戳过滤掉，然后获取指标的 TSID：
+ 快速路径 -> 当前的 MetricRows[i] 与 MetricRows[i - 1] 的指标名称相同，因此具有相同的 TSID，因此MetricRows[i].TSID = MetricRows[i - 1].TSID
+ MetricRows[i].MetricName != MetricRows[i - 1].MetricName
  + 命中缓存：更新内存中 prev* 相关数据，并更新 MetricRows[i].TSID
  + 未命中缓存：
    + 命中 indexdb：更新内存中 prev* 相关数据，并更新 MetricRows[i].TSID
    + 未命中 indexdb：生成 TSID，更新内存中 prev* 相关数据，并更新 MetricRows[i].TSID
+ 写入数据

### indexdb 中获取 TSID
```golang
func (is *indexSearch) getTSIDByMetricName(dst *generationTSID, metricName []byte, date uint64) bool {
	if is.getTSIDByMetricNameNoExtDB(&dst.TSID, metricName, date) {
	  // 快速路径 - 在当前的 indexdb 中找到 TSID
		dst.generation = is.db.generation
		return true
	}

	// 慢速路径 - 在之前的 indexdb 中找到 TSID
	ok := false
	deadline := is.deadline
	is.db.doExtDB(func(extDB *indexDB) {
		is := extDB.getIndexSearch(0, 0, deadline)
		ok = is.getTSIDByMetricNameNoExtDB(&dst.TSID, metricName, date)
		extDB.putIndexSearch(is)
		if ok {
			dst.generation = extDB.generation
		}
	})
	return ok
}
```
生成 TSID 的方法如下
```golang
func generateTSID(dst *TSID, mn *MetricName) {
	dst.AccountID = mn.AccountID
	dst.ProjectID = mn.ProjectID
	dst.MetricGroupID = xxhash.Sum64(mn.MetricGroup)
	if len(mn.Tags) > 0 {
		dst.JobID = uint32(xxhash.Sum64(mn.Tags[0].Value))
	}
	if len(mn.Tags) > 1 {
		dst.InstanceID = uint32(xxhash.Sum64(mn.Tags[1].Value))
	}
	dst.MetricID = generateUniqueMetricID()
}
```
