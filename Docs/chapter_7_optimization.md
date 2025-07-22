# Chapter 7: Search Optimization and Performance

## RRF Algorithm Deep Dive

### Mathematical Foundation and Tuning

The RRF algorithm's effectiveness depends on proper parameter tuning and understanding its mathematical properties. The core formula can be optimized for different use cases:

```csharp
namespace HotelSearch.Core.Algorithms;

public sealed class OptimizedRRF
{
    private readonly RRFConfiguration _config;
    private readonly ILogger<OptimizedRRF> _logger;

    public OptimizedRRF(RRFConfiguration config, ILogger<OptimizedRRF> logger)
    {
        _config = config;
        _logger = logger;
    }

    public List<RankedResult> FuseWithAdvancedScoring(
        List<Hotel> keywordResults,
        List<VectorSearchResult<Hotel>> vectorResults,
        SearchOptions options)
    {
        var rrfScores = new Dictionary<int, AdvancedRRFScore>();

        ProcessKeywordResultsWithBoost(keywordResults, rrfScores);
        ProcessVectorResultsWithDecay(vectorResults, rrfScores);
        ApplyQueryTypeBoosts(rrfScores, options);

        return rrfScores.Values
            .Where(score => PassesAdvancedFilters(score.Hotel, options))
            .OrderByDescending(score => score.FinalScore)
            .Take(options.MaxResults)
            .Select(CreateAdvancedRankedResult)
            .ToList();
    }

    private void ProcessKeywordResultsWithBoost(
        List<Hotel> results, 
        Dictionary<int, AdvancedRRFScore> scores)
    {
        for (int i = 0; i < results.Count; i++)
        {
            var hotel = results[i];
            var rank = i + 1;
            
            // Apply position-dependent boost for top results
            var positionBoost = CalculatePositionBoost(rank, results.Count);
            var baseScore = _config.KeywordWeight / (_config.RrfConstant + rank);
            var boostedScore = baseScore * positionBoost;
            
            UpdateAdvancedScore(scores, hotel, keywordScore: boostedScore, keywordRank: rank);
        }
    }

    private void ProcessVectorResultsWithDecay(
        List<VectorSearchResult<Hotel>> results,
        Dictionary<int, AdvancedRRFScore> scores)
    {
        for (int i = 0; i < results.Count; i++)
        {
            var result = results[i];
            var rank = i + 1;
            var similarity = result.Score ?? 0.0;
            
            // Apply similarity-based weighting
            var similarityWeight = CalculateSimilarityWeight(similarity);
            var baseScore = _config.VectorWeight / (_config.RrfConstant + rank);
            var weightedScore = baseScore * similarityWeight;
            
            UpdateAdvancedScore(scores, result.Record, 
                vectorScore: weightedScore, vectorRank: rank, similarity: similarity);
        }
    }

    private double CalculatePositionBoost(int rank, int totalResults)
    {
        // Boost top 3 results more aggressively
        return rank switch
        {
            1 => _config.TopResultBoost,
            2 => _config.TopResultBoost * 0.8,
            3 => _config.TopResultBoost * 0.6,
            _ => 1.0
        };
    }

    private double CalculateSimilarityWeight(double similarity)
    {
        // Apply sigmoid function to emphasize high similarity scores
        return 1.0 / (1.0 + Math.Exp(-_config.SimilaritySharpness * (similarity - _config.SimilarityThreshold)));
    }

    private void ApplyQueryTypeBoosts(
        Dictionary<int, AdvancedRRFScore> scores, 
        SearchOptions options)
    {
        foreach (var score in scores.Values)
        {
            // Apply hotel-specific boosts
            var qualityBoost = CalculateQualityBoost(score.Hotel);
            var freshnessBoost = CalculateFreshnessBoost(score.Hotel.LastUpdated);
            
            score.FinalScore = score.TotalScore * qualityBoost * freshnessBoost;
        }
    }

    private double CalculateQualityBoost(Hotel hotel)
    {
        // Boost higher-rated hotels
        return 1.0 + (hotel.Rating - 3.0) * _config.RatingBoostFactor;
    }

    private double CalculateFreshnessBoost(DateTime lastUpdated)
    {
        var age = DateTime.UtcNow - lastUpdated;
        var daysSinceUpdate = age.TotalDays;
        
        // Apply time decay - fresher content gets slight boost
        return Math.Max(0.8, 1.0 - (daysSinceUpdate * _config.FreshnessDecayRate));
    }
}

public sealed record RRFConfiguration
{
    public double KeywordWeight { get; init; } = 0.6;
    public double VectorWeight { get; init; } = 0.4;
    public int RrfConstant { get; init; } = 60;
    public double TopResultBoost { get; init; } = 1.2;
    public double SimilaritySharpness { get; init; } = 10.0;
    public double SimilarityThreshold { get; init; } = 0.7;
    public double RatingBoostFactor { get; init; } = 0.1;
    public double FreshnessDecayRate { get; init; } = 0.001;
}

public sealed class AdvancedRRFScore
{
    public Hotel Hotel { get; set; } = null!;
    public int? KeywordRank { get; set; }
    public int? VectorRank { get; set; }
    public double KeywordScore { get; set; }
    public double VectorScore { get; set; }
    public double TotalScore { get; set; }
    public double FinalScore { get; set; }
    public double VectorSimilarity { get; set; }
}
```

### Performance Benchmarking

```csharp
namespace HotelSearch.Core.Performance;

public sealed class SearchPerformanceBenchmark
{
    private readonly IHotelSearchService _searchService;
    private readonly ILogger<SearchPerformanceBenchmark> _logger;

    public SearchPerformanceBenchmark(
        IHotelSearchService searchService,
        ILogger<SearchPerformanceBenchmark> logger)
    {
        _searchService = searchService;
        _logger = logger;
    }

    public async Task<BenchmarkResults> RunBenchmarkAsync(
        string[] testQueries,
        int iterations = 10,
        CancellationToken cancellationToken = default)
    {
        var results = new List<QueryBenchmark>();

        foreach (var query in testQueries)
        {
            var queryBenchmark = await BenchmarkQueryAsync(query, iterations, cancellationToken);
            results.Add(queryBenchmark);
        }

        var aggregateResults = CalculateAggregateMetrics(results);
        LogBenchmarkResults(aggregateResults);

        return aggregateResults;
    }

    private async Task<QueryBenchmark> BenchmarkQueryAsync(
        string query,
        int iterations,
        CancellationToken cancellationToken)
    {
        var times = new List<TimeSpan>();
        var resultCounts = new List<int>();
        
        // Warm-up run
        await _searchService.SearchHotelsAsync(query, cancellationToken: cancellationToken);

        for (int i = 0; i < iterations; i++)
        {
            var stopwatch = Stopwatch.StartNew();
            
            var searchResult = await _searchService.SearchHotelsAsync(
                query, 
                new SearchOptions { MaxResults = 10, IncludeMetrics = true },
                cancellationToken);
            
            stopwatch.Stop();
            
            times.Add(stopwatch.Elapsed);
            resultCounts.Add(searchResult.Items.Count);
        }

        return new QueryBenchmark
        {
            Query = query,
            Iterations = iterations,
            AverageTime = TimeSpan.FromMilliseconds(times.Average(t => t.TotalMilliseconds)),
            MinTime = times.Min(),
            MaxTime = times.Max(),
            MedianTime = CalculateMedian(times),
            AverageResultCount = resultCounts.Average(),
            StandardDeviation = CalculateStandardDeviation(times)
        };
    }

    private TimeSpan CalculateMedian(List<TimeSpan> times)
    {
        var sorted = times.OrderBy(t => t.TotalMilliseconds).ToList();
        var middle = sorted.Count / 2;
        
        if (sorted.Count % 2 == 0)
        {
            var avg = (sorted[middle - 1].TotalMilliseconds + sorted[middle].TotalMilliseconds) / 2;
            return TimeSpan.FromMilliseconds(avg);
        }
        
        return sorted[middle];
    }

    private double CalculateStandardDeviation(List<TimeSpan> times)
    {
        var average = times.Average(t => t.TotalMilliseconds);
        var sumOfSquares = times.Sum(t => Math.Pow(t.TotalMilliseconds - average, 2));
        return Math.Sqrt(sumOfSquares / times.Count);
    }

    private BenchmarkResults CalculateAggregateMetrics(List<QueryBenchmark> results)
    {
        return new BenchmarkResults
        {
            QueryBenchmarks = results,
            OverallAverageTime = TimeSpan.FromMilliseconds(
                results.Average(r => r.AverageTime.TotalMilliseconds)),
            FastestQuery = results.OrderBy(r => r.AverageTime).First(),
            SlowestQuery = results.OrderByDescending(r => r.AverageTime).First(),
            TotalQueriesExecuted = results.Sum(r => r.Iterations)
        };
    }

    private void LogBenchmarkResults(BenchmarkResults results)
    {
        _logger.LogInformation("=== Search Performance Benchmark Results ===");
        _logger.LogInformation("Overall Average Time: {AverageTime}ms", 
            results.OverallAverageTime.TotalMilliseconds);
        _logger.LogInformation("Fastest Query: '{Query}' - {Time}ms", 
            results.FastestQuery.Query, results.FastestQuery.AverageTime.TotalMilliseconds);
        _logger.LogInformation("Slowest Query: '{Query}' - {Time}ms", 
            results.SlowestQuery.Query, results.SlowestQuery.AverageTime.TotalMilliseconds);
        
        foreach (var benchmark in results.QueryBenchmarks.OrderBy(b => b.AverageTime))
        {
            _logger.LogInformation(
                "Query: '{Query}' | Avg: {Avg}ms | Min: {Min}ms | Max: {Max}ms | StdDev: {StdDev:F2}ms",
                benchmark.Query,
                benchmark.AverageTime.TotalMilliseconds,
                benchmark.MinTime.TotalMilliseconds,
                benchmark.MaxTime.TotalMilliseconds,
                benchmark.StandardDeviation);
        }
    }
}

public sealed record QueryBenchmark
{
    public string Query { get; init; } = string.Empty;
    public int Iterations { get; init; }
    public TimeSpan AverageTime { get; init; }
    public TimeSpan MinTime { get; init; }
    public TimeSpan MaxTime { get; init; }
    public TimeSpan MedianTime { get; init; }
    public double AverageResultCount { get; init; }
    public double StandardDeviation { get; init; }
}

public sealed record BenchmarkResults
{
    public List<QueryBenchmark> QueryBenchmarks { get; init; } = new();
    public TimeSpan OverallAverageTime { get; init; }
    public QueryBenchmark FastestQuery { get; init; } = null!;
    public QueryBenchmark SlowestQuery { get; init; } = null!;
    public int TotalQueriesExecuted { get; init; }
}
```

## Configurable Weight Systems

### Dynamic Weight Adjustment

```csharp
namespace HotelSearch.Core.Configuration;

public interface ISearchWeightCalculator
{
    SearchWeights CalculateWeights(string query, SearchContext context);
}

public sealed class AdaptiveSearchWeightCalculator : ISearchWeightCalculator
{
    private readonly ILogger<AdaptiveSearchWeightCalculator> _logger;

    public AdaptiveSearchWeightCalculator(ILogger<AdaptiveSearchWeightCalculator> logger)
    {
        _logger = logger;
    }

    public SearchWeights CalculateWeights(string query, SearchContext context)
    {
        var baseWeights = new SearchWeights();
        
        // Adjust based on query characteristics
        var queryAnalysis = AnalyzeQuery(query);
        var adjustedWeights = ApplyQueryAdjustments(baseWeights, queryAnalysis);
        
        // Adjust based on user context
        var contextWeights = ApplyContextAdjustments(adjustedWeights, context);
        
        _logger.LogDebug(
            "Calculated weights for query '{Query}': Keyword={Keyword:F2}, Vector={Vector:F2}",
            query, contextWeights.KeywordWeight, contextWeights.VectorWeight);
        
        return contextWeights;
    }

    private QueryAnalysis AnalyzeQuery(string query)
    {
        var words = query.Split(' ', StringSplitOptions.RemoveEmptyEntries);
        var hasQuotes = query.Contains('"');
        var hasSpecialTerms = words.Any(w => w.All(char.IsDigit) || w.Contains('@'));
        var avgWordLength = words.Average(w => w.Length);
        
        return new QueryAnalysis
        {
            WordCount = words.Length,
            HasQuotes = hasQuotes,
            HasSpecialTerms = hasSpecialTerms,
            AverageWordLength = avgWordLength,
            IsLikelyNaturalLanguage = avgWordLength > 4 && words.Length > 2 && !hasSpecialTerms
        };
    }

    private SearchWeights ApplyQueryAdjustments(SearchWeights baseWeights, QueryAnalysis analysis)
    {
        var keywordWeight = baseWeights.KeywordWeight;
        var vectorWeight = baseWeights.VectorWeight;

        // Boost keyword search for:
        if (analysis.HasQuotes)
        {
            keywordWeight += 0.2; // Exact phrase matching
        }
        
        if (analysis.HasSpecialTerms)
        {
            keywordWeight += 0.3; // IDs, codes, emails
        }
        
        if (analysis.WordCount == 1)
        {
            keywordWeight += 0.1; // Single term searches
        }

        // Boost vector search for:
        if (analysis.IsLikelyNaturalLanguage)
        {
            vectorWeight += 0.2; // Natural language queries
        }
        
        if (analysis.WordCount > 5)
        {
            vectorWeight += 0.1; // Long descriptive queries
        }

        // Normalize weights
        var total = keywordWeight + vectorWeight;
        return new SearchWeights
        {
            KeywordWeight = keywordWeight / total,
            VectorWeight = vectorWeight / total
        };
    }

    private SearchWeights ApplyContextAdjustments(SearchWeights weights, SearchContext context)
    {
        var adjustedWeights = weights;

        // Adjust based on user preference patterns
        if (context.UserPreference == UserSearchPreference.Precise)
        {
            adjustedWeights = adjustedWeights with 
            { 
                KeywordWeight = Math.Min(0.8, adjustedWeights.KeywordWeight + 0.1) 
            };
        }
        else if (context.UserPreference == UserSearchPreference.Exploratory)
        {
            adjustedWeights = adjustedWeights with 
            { 
                VectorWeight = Math.Min(0.8, adjustedWeights.VectorWeight + 0.1) 
            };
        }

        // Adjust based on time of day or seasonal factors
        if (context.IsBusinessHours)
        {
            // During business hours, users might prefer more precise results
            adjustedWeights = adjustedWeights with 
            { 
                KeywordWeight = adjustedWeights.KeywordWeight + 0.05 
            };
        }

        // Renormalize
        var total = adjustedWeights.KeywordWeight + adjustedWeights.VectorWeight;
        return new SearchWeights
        {
            KeywordWeight = adjustedWeights.KeywordWeight / total,
            VectorWeight = adjustedWeights.VectorWeight / total
        };
    }
}

public sealed record SearchWeights
{
    public double KeywordWeight { get; init; } = 0.6;
    public double VectorWeight { get; init; } = 0.4;
}

public sealed record QueryAnalysis
{
    public int WordCount { get; init; }
    public bool HasQuotes { get; init; }
    public bool HasSpecialTerms { get; init; }
    public double AverageWordLength { get; init; }
    public bool IsLikelyNaturalLanguage { get; init; }
}

public sealed record SearchContext
{
    public UserSearchPreference UserPreference { get; init; } = UserSearchPreference.Balanced;
    public bool IsBusinessHours { get; init; }
    public string? UserLocation { get; init; }
    public DateTime SearchTime { get; init; } = DateTime.UtcNow;
}

public enum UserSearchPreference
{
    Precise,
    Balanced,
    Exploratory
}
```

## Search Result Fusion Strategies

### A/B Testing Framework for Fusion Algorithms

```csharp
namespace HotelSearch.Core.Testing;

public interface IFusionStrategy
{
    string Name { get; }
    List<RankedResult> FuseResults(
        List<Hotel> keywordResults,
        List<VectorSearchResult<Hotel>> vectorResults,
        SearchOptions options);
}

public sealed class FusionTestingService
{
    private readonly Dictionary<string, IFusionStrategy> _strategies;
    private readonly ILogger<FusionTestingService> _logger;
    private readonly Random _random = new();

    public FusionTestingService(
        IEnumerable<IFusionStrategy> strategies,
        ILogger<FusionTestingService> logger)
    {
        _strategies = strategies.ToDictionary(s => s.Name, s => s);
        _logger = logger;
    }

    public async Task<FusionTestResult> RunComparativeTestAsync(
        string[] testQueries,
        CancellationToken cancellationToken = default)
    {
        var results = new Dictionary<string, List<SearchMetrics>>();
        
        foreach (var strategyName in _strategies.Keys)
        {
            results[strategyName] = new List<SearchMetrics>();
        }

        foreach (var query in testQueries)
        {
            var queryResults = await TestQueryAcrossStrategiesAsync(query, cancellationToken);
            
            foreach (var (strategyName, metrics) in queryResults)
            {
                results[strategyName].Add(metrics);
            }
        }

        return AnalyzeTestResults(results);
    }

    private async Task<Dictionary<string, SearchMetrics>> TestQueryAcrossStrategiesAsync(
        string query,
        CancellationToken cancellationToken)
    {
        var results = new Dictionary<string, SearchMetrics>();
        
        // Get base search results once
        var keywordResults = await GetKeywordResultsAsync(query, cancellationToken);
        var vectorResults = await GetVectorResultsAsync(query, cancellationToken);
        
        // Test each fusion strategy
        foreach (var (name, strategy) in _strategies)
        {
            var stopwatch = Stopwatch.StartNew();
            
            var fusedResults = strategy.FuseResults(
                keywordResults, 
                vectorResults, 
                new SearchOptions { MaxResults = 10 });
            
            stopwatch.Stop();
            
            results[name] = new SearchMetrics
            {
                SearchDuration = stopwatch.Elapsed,
                KeywordResults = keywordResults.Count,
                VectorResults = vectorResults.Count,
                HybridResults = fusedResults.Count,
                QueryType = "Test"
            };
        }
        
        return results;
    }

    private FusionTestResult AnalyzeTestResults(Dictionary<string, List<SearchMetrics>> results)
    {
        var strategyComparisons = new List<StrategyComparison>();
        
        foreach (var (strategyName, metricsList) in results)
        {
            var avgDuration = TimeSpan.FromMilliseconds(
                metricsList.Average(m => m.SearchDuration.TotalMilliseconds));
            
            var avgResults = metricsList.Average(m => m.HybridResults);
            
            var consistency = CalculateConsistency(metricsList);
            
            strategyComparisons.Add(new StrategyComparison
            {
                StrategyName = strategyName,
                AverageResponseTime = avgDuration,
                AverageResultCount = avgResults,
                ConsistencyScore = consistency,
                TotalQueries = metricsList.Count
            });
        }
        
        return new FusionTestResult
        {
            StrategyComparisons = strategyComparisons,
            TestDate = DateTime.UtcNow,
            RecommendedStrategy = strategyComparisons
                .OrderByDescending(s => s.ConsistencyScore)
                .ThenBy(s => s.AverageResponseTime)
                .First().StrategyName
        };
    }

    private double CalculateConsistency(List<SearchMetrics> metrics)
    {
        var durations = metrics.Select(m => m.SearchDuration.TotalMilliseconds).ToList();
        var resultCounts = metrics.Select(m => (double)m.HybridResults).ToList();
        
        var durationConsistency = 1.0 / (1.0 + CalculateStandardDeviation(durations));
        var resultConsistency = 1.0 / (1.0 + CalculateStandardDeviation(resultCounts));
        
        return (durationConsistency + resultConsistency) / 2.0;
    }

    private double CalculateStandardDeviation(List<double> values)
    {
        var average = values.Average();
        var sumOfSquares = values.Sum(v => Math.Pow(v - average, 2));
        return Math.Sqrt(sumOfSquares / values.Count);
    }
}

public sealed record FusionTestResult
{
    public List<StrategyComparison> StrategyComparisons { get; init; } = new();
    public DateTime TestDate { get; init; }
    public string RecommendedStrategy { get; init; } = string.Empty;
}

public sealed record StrategyComparison
{
    public string StrategyName { get; init; } = string.Empty;
    public TimeSpan AverageResponseTime { get; init; }
    public double AverageResultCount { get; init; }
    public double ConsistencyScore { get; init; }
    public int TotalQueries { get; init; }
}
```

## Performance Metrics and Monitoring

### Real-time Performance Monitoring

```csharp
namespace HotelSearch.Core.Monitoring;

public sealed class SearchPerformanceMonitor
{
    private readonly IMetricsCollector _metricsCollector;
    private readonly ILogger<SearchPerformanceMonitor> _logger;
    private readonly ConcurrentDictionary<string, PerformanceStats> _queryStats = new();

    public SearchPerformanceMonitor(
        IMetricsCollector metricsCollector,
        ILogger<SearchPerformanceMonitor> logger)
    {
        _metricsCollector = metricsCollector;
        _logger = logger;
    }

    public IDisposable TrackSearch(string query, SearchOptions options)
    {
        return new SearchTracker(query, options, this);
    }

    internal void RecordSearch(string query, SearchOptions options, TimeSpan duration, int resultCount)
    {
        var stats = _queryStats.AddOrUpdate(
            query,
            _ => new PerformanceStats { Query = query },
            (_, existing) => existing);

        stats.RecordExecution(duration, resultCount);

        _metricsCollector.RecordSearchMetrics(new SearchMetricsData
        {
            Query = query,
            Duration = duration,
            ResultCount = resultCount,
            MaxResults = options.MaxResults,
            HasFilters = options.MinRating.HasValue || !string.IsNullOrEmpty(options.Location)
        });

        CheckPerformanceThresholds(query, duration, resultCount);
    }

    private void CheckPerformanceThresholds(string query, TimeSpan duration, int resultCount)
    {
        const int SlowSearchThresholdMs = 1000;
        const int LowResultsThreshold = 2;

        if (duration.TotalMilliseconds > SlowSearchThresholdMs)
        {
            _logger.LogWarning(
                "Slow search detected: Query='{Query}', Duration={Duration}ms",
                query, duration.TotalMilliseconds);
        }

        if (resultCount < LowResultsThreshold)
        {
            _logger.LogInformation(
                "Low result count: Query='{Query}', Results={ResultCount}",
                query, resultCount);
        }
    }

    public SearchPerformanceReport GenerateReport(TimeSpan period)
    {
        var cutoff = DateTime.UtcNow - period;
        var relevantStats = _queryStats.Values
            .Where(s => s.LastExecuted > cutoff)
            .ToList();

        return new SearchPerformanceReport
        {
            Period = period,
            TotalSearches = relevantStats.Sum(s => s.ExecutionCount),
            AverageResponseTime = TimeSpan.FromMilliseconds(
                relevantStats.Average(s => s.AverageResponseTime.TotalMilliseconds)),
            SlowestQueries = relevantStats
                .OrderByDescending(s => s.AverageResponseTime)
                .Take(10)
                .ToList(),
            MostFrequentQueries = relevantStats
                .OrderByDescending(s => s.ExecutionCount)
                .Take(10)
                .ToList(),
            PerformanceDistribution = CalculatePerformanceDistribution(relevantStats)
        };
    }

    private PerformanceDistribution CalculatePerformanceDistribution(List<PerformanceStats> stats)
    {
        var durations = stats.SelectMany(s => s.GetRecentDurations()).ToList();
        durations.Sort();

        if (durations.Count == 0)
        {
            return new PerformanceDistribution();
        }

        return new PerformanceDistribution
        {
            P50 = GetPercentile(durations, 0.5),
            P75 = GetPercentile(durations, 0.75),
            P90 = GetPercentile(durations, 0.9),
            P95 = GetPercentile(durations, 0.95),
            P99 = GetPercentile(durations, 0.99)
        };
    }

    private TimeSpan GetPercentile(List<TimeSpan> sortedDurations, double percentile)
    {
        var index = (int)((sortedDurations.Count - 1) * percentile);
        return sortedDurations[index];
    }

    private sealed class SearchTracker : IDisposable
    {
        private readonly string _query;
        private readonly SearchOptions _options;
        private readonly SearchPerformanceMonitor _monitor;
        private readonly Stopwatch _stopwatch;
        private int _resultCount;

        public SearchTracker(string query, SearchOptions options, SearchPerformanceMonitor monitor)
        {
            _query = query;
            _options = options;
            _monitor = monitor;
            _stopwatch = Stopwatch.StartNew();
        }

        public void SetResultCount(int count)
        {
            _resultCount = count;
        }

        public void Dispose()
        {
            _stopwatch.Stop();
            _monitor.RecordSearch(_query, _options, _stopwatch.Elapsed, _resultCount);
        }
    }
}

public sealed class PerformanceStats
{
    private readonly object _lock = new();
    private readonly Queue<(DateTime Time, TimeSpan Duration)> _recentExecutions = new();
    private const int MaxRecentExecutions = 100;

    public string Query { get; init; } = string.Empty;
    public int ExecutionCount { get; private set; }
    public TimeSpan AverageResponseTime { get; private set; }
    public DateTime LastExecuted { get; private set; }
    public int AverageResultCount { get; private set; }

    public void RecordExecution(TimeSpan duration, int resultCount)
    {
        lock (_lock)
        {
            ExecutionCount++;
            LastExecuted = DateTime.UtcNow;

            _recentExecutions.Enqueue((LastExecuted, duration));
            if (_recentExecutions.Count > MaxRecentExecutions)
            {
                _recentExecutions.Dequeue();
            }

            // Update averages
            var totalMs = AverageResponseTime.TotalMilliseconds * (ExecutionCount - 1) + duration.TotalMilliseconds;
            AverageResponseTime = TimeSpan.FromMilliseconds(totalMs / ExecutionCount);

            var totalResults = AverageResultCount * (ExecutionCount - 1) + resultCount;
            AverageResultCount = totalResults / ExecutionCount;
        }
    }

    public List<TimeSpan> GetRecentDurations()
    {
        lock (_lock)
        {
            return _recentExecutions.Select(e => e.Duration).ToList();
        }
    }
}

public sealed record SearchPerformanceReport
{
    public TimeSpan Period { get; init; }
    public int TotalSearches { get; init; }
    public TimeSpan AverageResponseTime { get; init; }
    public List<PerformanceStats> SlowestQueries { get; init; } = new();
    public List<PerformanceStats> MostFrequentQueries { get; init; } = new();
    public PerformanceDistribution PerformanceDistribution { get; init; } = new();
}

public sealed record PerformanceDistribution
{
    public TimeSpan P50 { get; init; }
    public TimeSpan P75 { get; init; }
    public TimeSpan P90 { get; init; }
    public TimeSpan P95 { get; init; }
    public TimeSpan P99 { get; init; }
}

public interface IMetricsCollector
{
    void RecordSearchMetrics(SearchMetricsData data);
}

public sealed record SearchMetricsData
{
    public string Query { get; init; } = string.Empty;
    public TimeSpan Duration { get; init; }
    public int ResultCount { get; init; }
    public int MaxResults { get; init; }
    public bool HasFilters { get; init; }
}
```

This optimization framework provides comprehensive tools for tuning RRF algorithms, benchmarking performance, implementing adaptive weight systems, and monitoring search performance in production environments.
