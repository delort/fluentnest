# FluentNest

LINQ-like query language for ElasticSearch built on top of NEST, aimed at simplifying analytical queries. Compatible with ElastichSearch 1 and 2.

| Elastic Search & Nest Version | Build | Nuget | Branch |
| ----------------------------- |:-----:| :----:|:------:|
| 1.7.* and greater| [![Build 2.0.0](https://ci.appveyor.com/api/projects/status/wrorpoekyw416hn1/branch/1.0.0?svg=true)](https://ci.appveyor.com/project/hoonzis/fluentnest) | [![Nuget Package](https://img.shields.io/badge/nuget-v1.0.180-blue.svg)](https://www.nuget.org/packages/FluentNest/1.0.166) | [1.0.0](https://github.com/hoonzis/fluentnest/tree/1.0.0) |
| 2.0.* and greater| [![Build 2.0.0](https://ci.appveyor.com/api/projects/status/wrorpoekyw416hn1?svg=true)](https://ci.appveyor.com/project/hoonzis/fluentnest)|[![Nuget Package](https://img.shields.io/nuget/v/fluentnest.svg)](https://www.nuget.org/packages/fluentnest) | [master](https://github.com/hoonzis/fluentnest/) |

NEST for querying ElasticSearch is great, but complex queries are hard to read and reason about. The same can be said about the basic JSON ElasticSearch query language. This library contains set of methods that give you more LINQ-like feeling. Currently mainly aggregations and filters are covered. This page contains few examples, more details are available in the [wiki](https://github.com/hoonzis/fluentnest/wiki/FluentNest-wiki). Motivation and few more implementation details are described in [this blog post.](http://www.hoonzis.com/fluent-interface-for-elastic-search/).


### Statistics
```Csharp
var result = client.Search<Car>(search => search.Aggregations(agg => agg
	.SumBy<Car>(x => x.Price)
	.CardinalityBy(x => x.EngineType)
);

var priceSum = result.Aggs.GetSum<Car, decimal>(x => x.Price);
var engines = result.Aggs.GetCardinality<Car>(x => x.EngineType);
```

Since all the queries are always done on the same entity type, one can also convert the result into a typed container:

```Csharp
var container = result.Aggs.AsContainer<Car>();
var priceSum = container.GetSum(x => x.Price);
var engines = container.GetCardinality(x => x.EngineType);
```

### Conditional statistics
Conditional sums can be quite complicated with NEST. One has to define a **Filter** aggregation with nested inner **Sum** or other aggregation. Here is quicker way with FluentNest:

```CSharp
var result = client.Search<Car>(search => search.Aggregations(aggs => aggs
	.SumBy(x=>x.Price, c => c.EngineType == EngineType.Diesel)
	.SumBy(x=>x.Sales, c => c.CarType == "Car1"))
);
```

### Filtering and expressions to queries compilation
Filtering on multiple conditions might be complicated since you have to compose filters using *Or*, *And*, *Range* methods, often resulting in huge lambdas. *FluentNest* can compile small expressions into NEST query language. Examples:

```CSharp
client.Search<Car>(s => s.FilterOn(f => f.Timestamp > startDate && f.Timestamp < endDate));
client.Search<Car>(s => s.FilterOn(f=> f.Ranking.HasValue || f.IsAllowed);
client.Search<Car>(s => s.FilterOn(f=> f.Ranking!=null || f.IsAllowed == true);
```
*HasValue* or null-checks are compiled into an **Exists** filter. Boolean values or expressions are compiled into **Bool** filters. Note that the same expressions can be used for conditional statistics as well as for general filters which affect the whole query. Comparisons of values are compiled into **Terms** filters.

### Grouped statistics
Quite often you might want to calculate a sum per group. With FluentNest you can write:

```CSharp
var result = client.Search<Car>(search => search.Aggregations(agg => agg
	.SumBy<Car>(s => s.Price)
	.GroupBy(s => s.EngineType)
);
```

Nested groups are very easy as well:

```Csharp
agg => agg
	.SumBy<Car>(s => s.Price)
	.GroupBy(s => s.CarType)
	.GroupBy(s => s.EngineType)
```

### Hitograms
Histogram is another useful aggregation supported by ElasticSearch. Here is a way to get a **Sum** by month.

```CSharp
var result = client.Search<Car>(s => s.Aggregations(a =>agg.
	.SumBy<Car>(x => x.Price)
	.IntoDateHistogram(date => date.Timestamp, DateInterval.Month)
);
var histogram = result.Aggs.GetDateHistogram<Car>(x => x.Timestamp);
```
