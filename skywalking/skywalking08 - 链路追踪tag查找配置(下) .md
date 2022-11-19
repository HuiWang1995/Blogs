# skywalking08 - 链路追踪tag查找配置(下)

> ​		在上篇中，已经讲述了如何配置tag进行查找，以及一些坑点，比如H2数据库会导致无法查找。那这一篇主要讲讲代码实现，比较轻松，简单的crud。

## 查询入口方法

通过一个org.apache.skywalking.oap.query.graphql.resolver.TraceQuery#queryBasicTraces方法，来获取的查询服务进行查找

```java
    public TraceBrief queryBasicTraces(final TraceQueryCondition condition) throws IOException {
        long startSecondTB = 0;
        long endSecondTB = 0;
        String traceId = Const.EMPTY_STRING;

        if (!Strings.isNullOrEmpty(condition.getTraceId())) {
            // 有链路流水号
            traceId = condition.getTraceId();
        } else if (nonNull(condition.getQueryDuration())) {
            // 开始结束时间
            startSecondTB = condition.getQueryDuration().getStartTimeBucketInSec();
            endSecondTB = condition.getQueryDuration().getEndTimeBucketInSec();
        } else {
            throw new UnexpectedException("The condition must contains either queryDuration or traceId.");
        }
		// 各种条件的获取
        int minDuration = condition.getMinTraceDuration();
        int maxDuration = condition.getMaxTraceDuration();
        String endpointName = condition.getEndpointName();
        String endpointId = condition.getEndpointId();
        TraceState traceState = condition.getTraceState();
        QueryOrder queryOrder = condition.getQueryOrder();
        Pagination pagination = condition.getPaging();
		// 获取查询服务，进行查询Traces （一个方法这么多参数也是有点离谱）
        return getQueryService().queryBasicTraces(
            condition.getServiceId(), condition.getServiceInstanceId(), endpointId, traceId, endpointName, minDuration,
            maxDuration, traceState, queryOrder, pagination, startSecondTB, endSecondTB, condition.getTags()
        );
    }

```

## 获取查询Dao类进行查询

通过org.apache.skywalking.oap.server.core.query.TraceQueryService#queryBasicTraces来屏蔽底层数据源的不同

```java
    public TraceBrief queryBasicTraces(final String serviceId,
                                       final String serviceInstanceId,
                                       final String endpointId,
                                       final String traceId,
                                       final String endpointName,
                                       final int minTraceDuration,
                                       int maxTraceDuration,
                                       final TraceState traceState,
                                       final QueryOrder queryOrder,
                                       final Pagination paging,
                                       final long startTB,
                                       final long endTB,
                                       final List<Tag> tags) throws IOException {
        PaginationUtils.Page page = PaginationUtils.INSTANCE.exchange(paging);
		// 获取具体的数据源进行查询
        return getTraceQueryDAO().queryBasicTraces(
            startTB, endTB, minTraceDuration, maxTraceDuration, endpointName, serviceId, serviceInstanceId, endpointId,
            traceId, page.getLimit(), page.getFrom(), traceState, queryOrder, tags
        );
    }
```

可以看到具体的Dao的实现类的继承关系如下：

![image-20211013165223452](/home/wanglh/Documents/Blogs/skywalking/pic/c08_2/01.png)

可以看到对应的数据库的选型对应有dao进行查询，像h2、mysql、influxdb、es等均有对应dao。我们选择es6的进行查看大致代码。

```java
    @Override
    public TraceBrief queryBasicTraces(long startSecondTB,
                                       long endSecondTB,
                                       long minDuration,
                                       long maxDuration,
                                       String endpointName,
                                       String serviceId,
                                       String serviceInstanceId,
                                       String endpointId,
                                       String traceId,
                                       int limit,
                                       int from,
                                       TraceState traceState,
                                       QueryOrder queryOrder,
                                       final List<Tag> tags) throws IOException {
        // 查询条件构建
        SearchSourceBuilder sourceBuilder = SearchSourceBuilder.searchSource();

        BoolQueryBuilder boolQueryBuilder = QueryBuilders.boolQuery();
        sourceBuilder.query(boolQueryBuilder);
        List<QueryBuilder> mustQueryList = boolQueryBuilder.must();

        if (startSecondTB != 0 && endSecondTB != 0) {
            mustQueryList.add(QueryBuilders.rangeQuery(SegmentRecord.TIME_BUCKET).gte(startSecondTB).lte(endSecondTB));
        }

        if (minDuration != 0 || maxDuration != 0) {
            RangeQueryBuilder rangeQueryBuilder = QueryBuilders.rangeQuery(SegmentRecord.LATENCY);
            if (minDuration != 0) {
                rangeQueryBuilder.gte(minDuration);
            }
            if (maxDuration != 0) {
                rangeQueryBuilder.lte(maxDuration);
            }
            boolQueryBuilder.must().add(rangeQueryBuilder);
        }
        if (!Strings.isNullOrEmpty(endpointName)) {
            String matchCName = MatchCNameBuilder.INSTANCE.build(SegmentRecord.ENDPOINT_NAME);
            mustQueryList.add(QueryBuilders.matchPhraseQuery(matchCName, endpointName));
        }
        if (StringUtil.isNotEmpty(serviceId)) {
            boolQueryBuilder.must().add(QueryBuilders.termQuery(SegmentRecord.SERVICE_ID, serviceId));
        }
        if (StringUtil.isNotEmpty(serviceInstanceId)) {
            boolQueryBuilder.must().add(QueryBuilders.termQuery(SegmentRecord.SERVICE_INSTANCE_ID, serviceInstanceId));
        }
        if (!Strings.isNullOrEmpty(endpointId)) {
            boolQueryBuilder.must().add(QueryBuilders.termQuery(SegmentRecord.ENDPOINT_ID, endpointId));
        }
        if (!Strings.isNullOrEmpty(traceId)) {
            boolQueryBuilder.must().add(QueryBuilders.termQuery(SegmentRecord.TRACE_ID, traceId));
        }
        switch (traceState) {
            case ERROR:
                mustQueryList.add(QueryBuilders.matchQuery(SegmentRecord.IS_ERROR, BooleanUtils.TRUE));
                break;
            case SUCCESS:
                mustQueryList.add(QueryBuilders.matchQuery(SegmentRecord.IS_ERROR, BooleanUtils.FALSE));
                break;
        }
        switch (queryOrder) {
            case BY_START_TIME:
                sourceBuilder.sort(SegmentRecord.START_TIME, SortOrder.DESC);
                break;
            case BY_DURATION:
                sourceBuilder.sort(SegmentRecord.LATENCY, SortOrder.DESC);
                break;
        }
        if (CollectionUtils.isNotEmpty(tags)) {
            BoolQueryBuilder tagMatchQuery = QueryBuilders.boolQuery();
            // 将TAGS添加到查询条件中
            tags.forEach(tag -> tagMatchQuery.must(QueryBuilders.termQuery(SegmentRecord.TAGS, tag.toString())));
            mustQueryList.add(tagMatchQuery);
        }
        sourceBuilder.size(limit);
        sourceBuilder.from(from);
        // 执行查询
        SearchResponse response = getClient().search(new TimeRangeIndexNameMaker(SegmentRecord.INDEX_NAME, startSecondTB, endSecondTB), sourceBuilder);
        TraceBrief traceBrief = new TraceBrief();
        traceBrief.setTotal((int) response.getHits().totalHits);

        for (SearchHit searchHit : response.getHits().getHits()) {
            BasicTrace basicTrace = new BasicTrace();

            basicTrace.setSegmentId((String) searchHit.getSourceAsMap().get(SegmentRecord.SEGMENT_ID));
            basicTrace.setStart(String.valueOf(searchHit.getSourceAsMap().get(SegmentRecord.START_TIME)));
            basicTrace.getEndpointNames().add((String) searchHit.getSourceAsMap().get(SegmentRecord.ENDPOINT_NAME));
            basicTrace.setDuration(((Number) searchHit.getSourceAsMap().get(SegmentRecord.LATENCY)).intValue());
            basicTrace.setError(
                BooleanUtils.valueToBoolean(
                    ((Number) searchHit.getSourceAsMap().get(SegmentRecord.IS_ERROR)).intValue()
                )
            );
            basicTrace.getTraceIds().add((String) searchHit.getSourceAsMap().get(SegmentRecord.TRACE_ID));
            traceBrief.getTraces().add(basicTrace);
        }

        return traceBrief;
    }
```

## 总结

总体来说，这个接口即是将查询条件组装到查询语句中，最后将查询结果组装为trace对象，是个CRUD的接口，技术含量并不高。但是它用来接入多数据源、同数据源不同的版本的设计模式值得参考学习。



