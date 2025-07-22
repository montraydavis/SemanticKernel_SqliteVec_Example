# Chapter 9: Production Considerations

## Connection Management and Pooling

### SQLite Connection Pooling

```csharp
namespace HotelSearch.Core.Infrastructure;

public interface IConnectionPool : IDisposable
{
    Task<IDbConnection> GetConnectionAsync(CancellationToken cancellationToken = default);
    void ReturnConnection(IDbConnection connection);
    ConnectionPoolStats GetStats();
}

public sealed class SqliteConnectionPool : IConnectionPool
{
    private readonly string _connectionString;
    private readonly SemaphoreSlim _semaphore;
    private readonly ConcurrentQueue<SqliteConnection> _availableConnections = new();
    private readonly ConcurrentDictionary<SqliteConnection, DateTime> _activeConnections = new();
    private readonly ILogger<SqliteConnectionPool> _logger;
    private readonly Timer _cleanupTimer;
    private readonly ConnectionPoolConfiguration _config;

    private volatile bool _disposed;
    private long _totalConnectionsCreated;
    private long _totalConnectionsReused;

    public SqliteConnectionPool(
        string connectionString,
        ConnectionPoolConfiguration config,
        ILogger<SqliteConnectionPool> logger)
    {
        _connectionString = connectionString ?? throw new ArgumentNullException(nameof(connectionString));
        _config = config ?? throw new ArgumentNullException(nameof(config));
        _logger = logger ?? throw new ArgumentNullException(nameof(logger));
        
        _semaphore = new SemaphoreSlim(_config.MaxPoolSize, _config.MaxPoolSize);
        
        // Initialize minimum connections
        InitializeMinimumConnections();
        
        // Start cleanup timer
        _cleanupTimer = new Timer(CleanupIdleConnections, null, 
            _config.CleanupInterval, _config.CleanupInterval);

        _logger.LogInformation(
            "Connection pool initialized: Min={MinSize}, Max={MaxSize}, Timeout={Timeout}s",
            _config.MinPoolSize, _config.MaxPoolSize, _config.ConnectionTimeout.TotalSeconds);
    }

    public async Task<IDbConnection> GetConnectionAsync(CancellationToken cancellationToken = default)
    {
        if (_disposed)
            throw new ObjectDisposedException(nameof(SqliteConnectionPool));

        await _semaphore.WaitAsync(cancellationToken);

        try
        {
            if (_availableConnections.TryDequeue(out var existingConnection))
            {
                if (IsConnectionValid(existingConnection))
                {
                    _activeConnections[existingConnection] = DateTime.UtcNow;
                    Interlocked.Increment(ref _totalConnectionsReused);
                    return existingConnection;
                }
                else
                {
                    existingConnection.Dispose();
                }
            }

            var newConnection = await CreateNewConnectionAsync(cancellationToken);
            _activeConnections[newConnection] = DateTime.UtcNow;
            Interlocked.Increment(ref _totalConnectionsCreated);

            return newConnection;
        }
        catch
        {
            _semaphore.Release();
            throw;
        }
    }

    public void ReturnConnection(IDbConnection connection)
    {
        if (_disposed || connection is not SqliteConnection sqliteConnection)
        {
            connection?.Dispose();
            return;
        }

        _activeConnections.TryRemove(sqliteConnection, out _);

        if (IsConnectionValid(sqliteConnection) && 
            _availableConnections.Count < _config.MaxPoolSize)
        {
            _availableConnections.Enqueue(sqliteConnection);
        }
        else
        {
            sqliteConnection.Dispose();
        }

        _semaphore.Release();
    }

    private void InitializeMinimumConnections()
    {
        for (int i = 0; i < _config.MinPoolSize; i++)
        {
            try
            {
                var connection = CreateNewConnectionAsync(CancellationToken.None).Result;
                _availableConnections.Enqueue(connection);
            }
            catch (Exception ex)
            {
                _logger.LogWarning(ex, "Failed to create initial connection {Index}", i);
            }
        }
    }

    private async Task<SqliteConnection> CreateNewConnectionAsync(CancellationToken cancellationToken)
    {
        var connection = new SqliteConnection(_connectionString);
        
        using var timeout = new CancellationTokenSource(_config.ConnectionTimeout);
        using var combined = CancellationTokenSource.CreateLinkedTokenSource(cancellationToken, timeout.Token);

        await connection.OpenAsync(combined.Token);
        
        // Configure connection for optimal performance
        await ConfigureConnectionAsync(connection, combined.Token);
        
        return connection;
    }

    private async Task ConfigureConnectionAsync(SqliteConnection connection, CancellationToken cancellationToken)
    {
        var pragmas = new[]
        {
            "PRAGMA journal_mode=WAL",
            "PRAGMA synchronous=NORMAL", 
            "PRAGMA cache_size=10000",
            "PRAGMA temp_store=MEMORY",
            "PRAGMA mmap_size=268435456" // 256MB
        };

        foreach (var pragma in pragmas)
        {
            using var command = connection.CreateCommand();
            command.CommandText = pragma;
            await command.ExecuteNonQueryAsync(cancellationToken);
        }
    }

    private bool IsConnectionValid(SqliteConnection connection)
    {
        try
        {
            return connection.State == ConnectionState.Open;
        }
        catch
        {
            return false;
        }
    }

    private void CleanupIdleConnections(object? state)
    {
        try
        {
            var cutoff = DateTime.UtcNow - _config.IdleTimeout;
            var toRemove = new List<SqliteConnection>();

            // Find idle connections
            foreach (var kvp in _activeConnections)
            {
                if (kvp.Value < cutoff)
                {
                    toRemove.Add(kvp.Key);
                }
            }

            // Remove and dispose idle connections
            foreach (var connection in toRemove)
            {
                if (_activeConnections.TryRemove(connection, out _))
                {
                    connection.Dispose();
                    _semaphore.Release();
                }
            }

            // Cleanup available connections that have been idle too long
            var availableToCleanup = new List<SqliteConnection>();
            while (_availableConnections.TryDequeue(out var conn))
            {
                if (IsConnectionValid(conn) && _availableConnections.Count >= _config.MinPoolSize)
                {
                    availableToCleanup.Add(conn);
                }
                else
                {
                    _availableConnections.Enqueue(conn);
                    break;
                }
            }

            foreach (var conn in availableToCleanup)
            {
                conn.Dispose();
            }

            if (toRemove.Count > 0 || availableToCleanup.Count > 0)
            {
                _logger.LogDebug(
                    "Cleaned up {ActiveCount} active and {AvailableCount} available idle connections",
                    toRemove.Count, availableToCleanup.Count);
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error during connection cleanup");
        }
    }

    public ConnectionPoolStats GetStats()
    {
        return new ConnectionPoolStats
        {
            TotalConnectionsCreated = _totalConnectionsCreated,
            TotalConnectionsReused = _totalConnectionsReused,
            ActiveConnections = _activeConnections.Count,
            AvailableConnections = _availableConnections.Count,
            PoolUtilization = (double)_activeConnections.Count / _config.MaxPoolSize
        };
    }

    public void Dispose()
    {
        if (_disposed) return;
        _disposed = true;

        _cleanupTimer?.Dispose();
        _semaphore?.Dispose();

        // Dispose all connections
        while (_availableConnections.TryDequeue(out var connection))
        {
            connection.Dispose();
        }

        foreach (var connection in _activeConnections.Keys)
        {
            connection.Dispose();
        }

        _logger.LogInformation("Connection pool disposed");
    }
}

public sealed record ConnectionPoolConfiguration
{
    public int MinPoolSize { get; init; } = 5;
    public int MaxPoolSize { get; init; } = 20;
    public TimeSpan ConnectionTimeout { get; init; } = TimeSpan.FromSeconds(30);
    public TimeSpan IdleTimeout { get; init; } = TimeSpan.FromMinutes(10);
    public TimeSpan CleanupInterval { get; init; } = TimeSpan.FromMinutes(5);
}

public sealed record ConnectionPoolStats
{
    public long TotalConnectionsCreated { get; init; }
    public long TotalConnectionsReused { get; init; }
    public int ActiveConnections { get; init; }
    public int AvailableConnections { get; init; }
    public double PoolUtilization { get; init; }
    public double ReuseRate => TotalConnectionsCreated > 0 
        ? (double)TotalConnectionsReused / (TotalConnectionsCreated + TotalConnectionsReused) 
        : 0;
}
```

### Connection-Aware Service Implementation

```csharp
namespace HotelSearch.Core.Services;

public sealed class PooledHotelSearchService : IHotelSearchService
{
    private readonly IConnectionPool _connectionPool;
    private readonly SqliteCollection<int, Hotel> _vectorCollection;
    private readonly IEmbeddingGenerator<string, Embedding<float>> _embeddingService;
    private readonly ILogger<PooledHotelSearchService> _logger;
    private readonly SearchConfiguration _config;

    public PooledHotelSearchService(
        IConnectionPool connectionPool,
        SqliteCollection<int, Hotel> vectorCollection,
        IEmbeddingGenerator<string, Embedding<float>> embeddingService,
        ILogger<PooledHotelSearchService> logger,
        SearchConfiguration config)
    {
        _connectionPool = connectionPool;
        _vectorCollection = vectorCollection;
        _embeddingService = embeddingService;
        _logger = logger;
        _config = config;
    }

    public async Task<SearchResult<Hotel>> SearchHotelsAsync(
        string query,
        SearchOptions? options = null,
        CancellationToken cancellationToken = default)
    {
        var connection = await _connectionPool.GetConnectionAsync(cancellationToken);
        try
        {
            return await ExecuteSearchWithConnectionAsync(query, options, connection, cancellationToken);
        }
        finally
        {
            _connectionPool.ReturnConnection(connection);
        }
    }

    private async Task<SearchResult<Hotel>> ExecuteSearchWithConnectionAsync(
        string query,
        SearchOptions? options,
        IDbConnection connection,
        CancellationToken cancellationToken)
    {
        // Implementation using the provided connection
        // This ensures we don't create additional connections unnecessarily
        options ??= new SearchOptions();
        
        var stopwatch = Stopwatch.StartNew();
        var (keywordResults, vectorResults) = await ExecuteParallelSearchAsync(
            query, options, connection, cancellationToken);
        
        var fusionAlgorithm = new ReciprocalRankFusion(_config, _logger);
        var rankedResults = fusionAlgorithm.FuseResults(keywordResults, vectorResults, options);
        
        var metrics = options.IncludeMetrics ? new SearchMetrics
        {
            SearchDuration = stopwatch.Elapsed,
            KeywordResults = keywordResults.Count,
            VectorResults = vectorResults.Count,
            HybridResults = rankedResults.Count,
            QueryType = DetermineQueryType(query)
        } : null;

        return new SearchResult<Hotel>
        {
            Items = rankedResults.Select(r => r.Hotel).ToList(),
            Metrics = metrics
        };
    }

    private async Task<(List<Hotel>, List<VectorSearchResult<Hotel>>)> ExecuteParallelSearchAsync(
        string query,
        SearchOptions options,
        IDbConnection connection,
        CancellationToken cancellationToken)
    {
        var searchLimit = Math.Max(options.MaxResults * _config.SearchMultiplier, _config.MinSearchLimit);
        
        // Use the same connection for keyword search, generate embedding separately
        var keywordTask = PerformKeywordSearchAsync(query, searchLimit, connection, cancellationToken);
        var vectorTask = GenerateEmbeddingAndSearchAsync(query, searchLimit, cancellationToken);
        
        await Task.WhenAll(keywordTask, vectorTask);
        return (await keywordTask, await vectorTask);
    }
}
```

## Caching Strategies

### Multi-Level Caching System

```csharp
namespace HotelSearch.Core.Caching;

public interface ISearchCache
{
    Task<SearchResult<Hotel>?> GetAsync(string cacheKey, CancellationToken cancellationToken = default);
    Task SetAsync(string cacheKey, SearchResult<Hotel> result, TimeSpan expiry, CancellationToken cancellationToken = default);
    Task InvalidateAsync(string pattern, CancellationToken cancellationToken = default);
    Task<CacheStatistics> GetStatisticsAsync();
}

public sealed class HybridSearchCache : ISearchCache
{
    private readonly IMemoryCache _l1Cache; // Fast, small capacity
    private readonly IDistributedCache _l2Cache; // Slower, large capacity
    private readonly ILogger<HybridSearchCache> _logger;
    private readonly CacheConfiguration _config;
    private readonly CacheStatistics _stats = new();

    public HybridSearchCache(
        IMemoryCache memoryCache,
        IDistributedCache distributedCache,
        ILogger<HybridSearchCache> logger,
        CacheConfiguration config)
    {
        _l1Cache = memoryCache;
        _l2Cache = distributedCache;
        _logger = logger;
        _config = config;
    }

    public async Task<SearchResult<Hotel>?> GetAsync(string cacheKey, CancellationToken cancellationToken = default)
    {
        Interlocked.Increment(ref _stats._totalRequests);

        // Try L1 cache first (in-memory)
        if (_l1Cache.TryGetValue(cacheKey, out SearchResult<Hotel>? l1Result))
        {
            Interlocked.Increment(ref _stats._l1Hits);
            _logger.LogDebug("Cache L1 hit for key: {CacheKey}", cacheKey);
            return l1Result;
        }

        // Try L2 cache (distributed)
        try
        {
            var l2Data = await _l2Cache.GetAsync(cacheKey, cancellationToken);
            if (l2Data != null)
            {
                var l2Result = DeserializeSearchResult(l2Data);
                
                // Promote to L1 cache
                var l1Options = new MemoryCacheEntryOptions
                {
                    AbsoluteExpirationRelativeToNow = _config.L1CacheExpiry,
                    Priority = CacheItemPriority.High
                };
                _l1Cache.Set(cacheKey, l2Result, l1Options);

                Interlocked.Increment(ref _stats._l2Hits);
                _logger.LogDebug("Cache L2 hit for key: {CacheKey}", cacheKey);
                return l2Result;
            }
        }
        catch (Exception ex)
        {
            _logger.LogWarning(ex, "Error accessing L2 cache for key: {CacheKey}", cacheKey);
        }

        Interlocked.Increment(ref _stats._misses);
        return null;
    }

    public async Task SetAsync(
        string cacheKey, 
        SearchResult<Hotel> result, 
        TimeSpan expiry, 
        CancellationToken cancellationToken = default)
    {
        // Set in L1 cache
        var l1Options = new MemoryCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = TimeSpan.FromTicks(Math.Min(expiry.Ticks, _config.L1CacheExpiry.Ticks)),
            Priority = CacheItemPriority.High
        };
        _l1Cache.Set(cacheKey, result, l1Options);

        // Set in L2 cache
        try
        {
            var serializedData = SerializeSearchResult(result);
            var l2Options = new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = expiry
            };
            await _l2Cache.SetAsync(cacheKey, serializedData, l2Options, cancellationToken);

            _logger.LogDebug("Cached result for key: {CacheKey}, Expiry: {Expiry}", cacheKey, expiry);
        }
        catch (Exception ex)
        {
            _logger.LogWarning(ex, "Error setting L2 cache for key: {CacheKey}", cacheKey);
        }
    }

    public async Task InvalidateAsync(string pattern, CancellationToken cancellationToken = default)
    {
        // For memory cache, we'd need to track keys or use a different approach
        // For distributed cache, this depends on the implementation (Redis supports patterns)
        
        _logger.LogInformation("Cache invalidation requested for pattern: {Pattern}", pattern);
        
        // This is a simplified implementation
        // In practice, you'd need more sophisticated key tracking
        await Task.CompletedTask;
    }

    public Task<CacheStatistics> GetStatisticsAsync()
    {
        return Task.FromResult(new CacheStatistics
        {
            TotalRequests = _stats._totalRequests,
            L1Hits = _stats._l1Hits,
            L2Hits = _stats._l2Hits,
            Misses = _stats._misses,
            HitRate = _stats._totalRequests > 0 
                ? (double)(_stats._l1Hits + _stats._l2Hits) / _stats._totalRequests 
                : 0
        });
    }

    private byte[] SerializeSearchResult(SearchResult<Hotel> result)
    {
        var json = JsonSerializer.Serialize(result, new JsonSerializerOptions
        {
            PropertyNamingPolicy = JsonNamingPolicy.CamelCase
        });
        return Encoding.UTF8.GetBytes(json);
    }

    private SearchResult<Hotel> DeserializeSearchResult(byte[] data)
    {
        var json = Encoding.UTF8.GetString(data);
        return JsonSerializer.Deserialize<SearchResult<Hotel>>(json, new JsonSerializerOptions
        {
            PropertyNamingPolicy = JsonNamingPolicy.CamelCase
        }) ?? new SearchResult<Hotel>();
    }
}

public sealed record CacheConfiguration
{
    public TimeSpan L1CacheExpiry { get; init; } = TimeSpan.FromMinutes(5);
    public TimeSpan L2CacheExpiry { get; init; } = TimeSpan.FromHours(1);
    public TimeSpan DefaultExpiry { get; init; } = TimeSpan.FromMinutes(15);
}

public sealed class CacheStatistics
{
    internal long _totalRequests;
    internal long _l1Hits;
    internal long _l2Hits;
    internal long _misses;

    public long TotalRequests => _totalRequests;
    public long L1Hits => _l1Hits;
    public long L2Hits => _l2Hits;
    public long Misses => _misses;
    public double HitRate { get; init; }
    public double L1HitRate => TotalRequests > 0 ? (double)L1Hits / TotalRequests : 0;
    public double L2HitRate => TotalRequests > 0 ? (double)L2Hits / TotalRequests : 0;
}
```

### Intelligent Cache Key Generation

```csharp
namespace HotelSearch.Core.Caching;

public sealed class SearchCacheKeyBuilder
{
    private readonly ILogger<SearchCacheKeyBuilder> _logger;

    public SearchCacheKeyBuilder(ILogger<SearchCacheKeyBuilder> logger)
    {
        _logger = logger;
    }

    public string BuildCacheKey(string query, SearchOptions options)
    {
        var normalizedQuery = NormalizeQuery(query);
        var optionsHash = ComputeOptionsHash(options);
        
        return $"search:{normalizedQuery}:{optionsHash}";
    }

    public string BuildEmbeddingCacheKey(string text)
    {
        var normalizedText = NormalizeQuery(text);
        var hash = ComputeHash(normalizedText);
        
        return $"embedding:{hash}";
    }

    private string NormalizeQuery(string query)
    {
        return query
            .ToLowerInvariant()
            .Trim()
            .Replace("  ", " ") // Multiple spaces to single space
            .Replace("&", "and")
            .Replace("@", "at");
    }

    private string ComputeOptionsHash(SearchOptions options)
    {
        var optionsString = JsonSerializer.Serialize(options, new JsonSerializerOptions
        {
            PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
            DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull
        });
        
        return ComputeHash(optionsString);
    }

    private string ComputeHash(string input)
    {
        using var sha256 = SHA256.Create();
        var hashBytes = sha256.ComputeHash(Encoding.UTF8.GetBytes(input));
        return Convert.ToHexString(hashBytes)[..16]; // Take first 16 characters
    }
}
```

## Error Handling and Resilience

### Circuit Breaker Pattern

```csharp
namespace HotelSearch.Core.Resilience;

public interface ICircuitBreaker
{
    Task<T> ExecuteAsync<T>(Func<Task<T>> operation, CancellationToken cancellationToken = default);
    CircuitBreakerState State { get; }
    CircuitBreakerMetrics GetMetrics();
}

public sealed class CircuitBreaker : ICircuitBreaker
{
    private readonly CircuitBreakerConfiguration _config;
    private readonly ILogger<CircuitBreaker> _logger;
    private readonly object _lock = new();

    private CircuitBreakerState _state = CircuitBreakerState.Closed;
    private int _failureCount;
    private DateTime _lastFailureTime;
    private DateTime _nextAttemptTime;
    private long _totalOperations;
    private long _successfulOperations;
    private long _failedOperations;

    public CircuitBreaker(CircuitBreakerConfiguration config, ILogger<CircuitBreaker> logger)
    {
        _config = config ?? throw new ArgumentNullException(nameof(config));
        _logger = logger ?? throw new ArgumentNullException(nameof(logger));
    }

    public CircuitBreakerState State
    {
        get
        {
            lock (_lock)
            {
                return _state;
            }
        }
    }

    public async Task<T> ExecuteAsync<T>(Func<Task<T>> operation, CancellationToken cancellationToken = default)
    {
        Interlocked.Increment(ref _totalOperations);

        lock (_lock)
        {
            if (_state == CircuitBreakerState.Open)
            {
                if (DateTime.UtcNow < _nextAttemptTime)
                {
                    throw new CircuitBreakerOpenException("Circuit breaker is open");
                }
                
                _state = CircuitBreakerState.HalfOpen;
                _logger.LogInformation("Circuit breaker transitioning to half-open state");
            }
        }

        try
        {
            var result = await operation();
            OnSuccess();
            return result;
        }
        catch (Exception ex)
        {
            OnFailure(ex);
            throw;
        }
    }

    private void OnSuccess()
    {
        Interlocked.Increment(ref _successfulOperations);

        lock (_lock)
        {
            if (_state == CircuitBreakerState.HalfOpen)
            {
                _state = CircuitBreakerState.Closed;
                _failureCount = 0;
                _logger.LogInformation("Circuit breaker reset to closed state after successful operation");
            }
        }
    }

    private void OnFailure(Exception exception)
    {
        Interlocked.Increment(ref _failedOperations);

        lock (_lock)
        {
            _failureCount++;
            _lastFailureTime = DateTime.UtcNow;

            if (_failureCount >= _config.FailureThreshold)
            {
                _state = CircuitBreakerState.Open;
                _nextAttemptTime = DateTime.UtcNow.Add(_config.OpenTimeout);
                
                _logger.LogWarning(
                    "Circuit breaker opened due to {FailureCount} failures. Next attempt at {NextAttempt}. Last error: {Error}",
                    _failureCount, _nextAttemptTime, exception.Message);
            }
        }
    }

    public CircuitBreakerMetrics GetMetrics()
    {
        lock (_lock)
        {
            return new CircuitBreakerMetrics
            {
                State = _state,
                FailureCount = _failureCount,
                TotalOperations = _totalOperations,
                SuccessfulOperations = _successfulOperations,
                FailedOperations = _failedOperations,
                LastFailureTime = _lastFailureTime,
                NextAttemptTime = _nextAttemptTime,
                SuccessRate = _totalOperations > 0 ? (double)_successfulOperations / _totalOperations : 0
            };
        }
    }
}

public sealed record CircuitBreakerConfiguration
{
    public int FailureThreshold { get; init; } = 5;
    public TimeSpan OpenTimeout { get; init; } = TimeSpan.FromMinutes(1);
}

public enum CircuitBreakerState
{
    Closed,   // Normal operation
    Open,     // Failing fast
    HalfOpen  // Testing if service has recovered
}

public sealed record CircuitBreakerMetrics
{
    public CircuitBreakerState State { get; init; }
    public int FailureCount { get; init; }
    public long TotalOperations { get; init; }
    public long SuccessfulOperations { get; init; }
    public long FailedOperations { get; init; }
    public DateTime LastFailureTime { get; init; }
    public DateTime NextAttemptTime { get; init; }
    public double SuccessRate { get; init; }
}

public sealed class CircuitBreakerOpenException : Exception
{
    public CircuitBreakerOpenException(string message) : base(message) { }
}
```

### Comprehensive Retry Policy

```csharp
namespace HotelSearch.Core.Resilience;

public sealed class RetryPolicy
{
    private readonly RetryConfiguration _config;
    private readonly ILogger<RetryPolicy> _logger;

    public RetryPolicy(RetryConfiguration config, ILogger<RetryPolicy> logger)
    {
        _config = config;
        _logger = logger;
    }

    public async Task<T> ExecuteAsync<T>(
        Func<Task<T>> operation,
        string operationName,
        CancellationToken cancellationToken = default)
    {
        var attempt = 0;
        var exceptions = new List<Exception>();

        while (attempt < _config.MaxAttempts)
        {
            try
            {
                var result = await operation();
                
                if (attempt > 0)
                {
                    _logger.LogInformation(
                        "Operation {OperationName} succeeded on attempt {Attempt}",
                        operationName, attempt + 1);
                }
                
                return result;
            }
            catch (Exception ex)
            {
                attempt++;
                exceptions.Add(ex);

                if (!ShouldRetry(ex) || attempt >= _config.MaxAttempts)
                {
                    _logger.LogError(
                        "Operation {OperationName} failed after {Attempts} attempts. Final error: {Error}",
                        operationName, attempt, ex.Message);
                    
                    throw new AggregateException(
                        $"Operation {operationName} failed after {attempt} attempts", 
                        exceptions);
                }

                var delay = CalculateDelay(attempt);
                
                _logger.LogWarning(
                    "Operation {OperationName} failed on attempt {Attempt}/{MaxAttempts}. Retrying in {Delay}ms. Error: {Error}",
                    operationName, attempt, _config.MaxAttempts, delay.TotalMilliseconds, ex.Message);

                await Task.Delay(delay, cancellationToken);
            }
        }

        throw new InvalidOperationException("This should never be reached");
    }

    private bool ShouldRetry(Exception exception)
    {
        return exception switch
        {
            CircuitBreakerOpenException => false,
            OperationCanceledException => false,
            ArgumentException => false,
            SqliteException sqlite => sqlite.SqliteErrorCode != SqliteErrorCode.Corrupt,
            HttpRequestException => true,
            TimeoutException => true,
            _ => true
        };
    }

    private TimeSpan CalculateDelay(int attempt)
    {
        return _config.DelayStrategy switch
        {
            RetryDelayStrategy.Fixed => _config.BaseDelay,
            RetryDelayStrategy.Linear => TimeSpan.FromMilliseconds(_config.BaseDelay.TotalMilliseconds * attempt),
            RetryDelayStrategy.Exponential => TimeSpan.FromMilliseconds(
                _config.BaseDelay.TotalMilliseconds * Math.Pow(2, attempt - 1)),
            RetryDelayStrategy.ExponentialWithJitter => AddJitter(
                TimeSpan.FromMilliseconds(_config.BaseDelay.TotalMilliseconds * Math.Pow(2, attempt - 1))),
            _ => _config.BaseDelay
        };
    }

    private TimeSpan AddJitter(TimeSpan delay)
    {
        var jitter = Random.Shared.NextDouble() * 0.1; // 10% jitter
        var jitterMs = delay.TotalMilliseconds * jitter;
        return TimeSpan.FromMilliseconds(delay.TotalMilliseconds + jitterMs);
    }
}

public sealed record RetryConfiguration
{
    public int MaxAttempts { get; init; } = 3;
    public TimeSpan BaseDelay { get; init; } = TimeSpan.FromSeconds(1);
    public RetryDelayStrategy DelayStrategy { get; init; } = RetryDelayStrategy.ExponentialWithJitter;
}

public enum RetryDelayStrategy
{
    Fixed,
    Linear,
    Exponential,
    ExponentialWithJitter
}
```

## Monitoring and Logging

### Structured Logging Implementation

```csharp
namespace HotelSearch.Core.Logging;

public static partial class LoggerExtensions
{
    [LoggerMessage(
        EventId = 1001,
        Level = LogLevel.Information,
        Message = "Search executed: Query='{Query}', Results={ResultCount}, Duration={Duration}ms")]
    public static partial void LogSearchExecuted(
        this ILogger logger, string query, int resultCount, double duration);

    [LoggerMessage(
        EventId = 1002,
        Level = LogLevel.Warning,
        Message = "Slow search detected: Query='{Query}', Duration={Duration}ms, Threshold={Threshold}ms")]
    public static partial void LogSlowSearch(
        this ILogger logger, string query, double duration, double threshold);

    [LoggerMessage(
        EventId = 1003,
        Level = LogLevel.Error,
        Message = "Search failed: Query='{Query}', Error={Error}")]
    public static partial void LogSearchFailed(
        this ILogger logger, string query, string error, Exception exception);

    [LoggerMessage(
        EventId = 2001,
        Level = LogLevel.Information,
        Message = "Embedding generated: TextLength={TextLength}, Duration={Duration}ms")]
    public static partial void LogEmbeddingGenerated(
        this ILogger logger, int textLength, double duration);

    [LoggerMessage(
        EventId = 2002,
        Level = LogLevel.Warning,
        Message = "Embedding generation failed: TextLength={TextLength}, Error={Error}")]
    public static partial void LogEmbeddingFailed(
        this ILogger logger, int textLength, string error, Exception exception);

    [LoggerMessage(
        EventId = 3001,
        Level = LogLevel.Information,
        Message = "Cache hit: Key={CacheKey}, Layer={CacheLayer}")]
    public static partial void LogCacheHit(
        this ILogger logger, string cacheKey, string cacheLayer);

    [LoggerMessage(
        EventId = 3002,
        Level = LogLevel.Debug,
        Message = "Cache miss: Key={CacheKey}")]
    public static partial void LogCacheMiss(
        this ILogger logger, string cacheKey);
}

public sealed class SearchOperationLogger
{
    private readonly ILogger<SearchOperationLogger> _logger;
    private readonly IMetrics _metrics;

    public SearchOperationLogger(ILogger<SearchOperationLogger> logger, IMetrics metrics)
    {
        _logger = logger;
        _metrics = metrics;
    }

    public IDisposable BeginSearchOperation(string query, SearchOptions options)
    {
        return new SearchOperationScope(query, options, _logger, _metrics);
    }

    private sealed class SearchOperationScope : IDisposable
    {
        private readonly string _query;
        private readonly SearchOptions _options;
        private readonly ILogger _logger;
        private readonly IMetrics _metrics;
        private readonly Stopwatch _stopwatch;
        private readonly Dictionary<string, object> _context;

        public SearchOperationScope(
            string query, 
            SearchOptions options, 
            ILogger logger, 
            IMetrics metrics)
        {
            _query = query;
            _options = options;
            _logger = logger;
            _metrics = metrics;
            _stopwatch = Stopwatch.StartNew();
            _context = new Dictionary<string, object>
            {
                ["query"] = query,
                ["maxResults"] = options.MaxResults,
                ["hasFilters"] = options.MinRating.HasValue || !string.IsNullOrEmpty(options.Location)
            };

            using var scope = _logger.BeginScope(_context);
            _logger.LogDebug("Search operation started: {Query}", query);
        }

        public void SetResult(SearchResult<Hotel> result)
        {
            _context["resultCount"] = result.Items.Count;
            _context["searchDuration"] = _stopwatch.Elapsed.TotalMilliseconds;
        }

        public void SetError(Exception exception)
        {
            _context["error"] = exception.Message;
            _context["errorType"] = exception.GetType().Name;
        }

        public void Dispose()
        {
            _stopwatch.Stop();
            var duration = _stopwatch.Elapsed.TotalMilliseconds;

            // Log the operation result
            if (_context.ContainsKey("error"))
            {
                _logger.LogSearchFailed(_query, _context["error"].ToString()!, (Exception)_context["errorType"]!);
                _metrics.Counter("search.errors").Increment(new Dictionary<string, string>
                {
                    ["error_type"] = _context["errorType"].ToString()!
                });
            }
            else
            {
                var resultCount = (int)_context.GetValueOrDefault("resultCount", 0);
                _logger.LogSearchExecuted(_query, resultCount, duration);

                // Record metrics
                _metrics.Histogram("search.duration").Record(duration);
                _metrics.Histogram("search.results").Record(resultCount);

                if (duration > 1000) // Slow search threshold
                {
                    _logger.LogSlowSearch(_query, duration, 1000);
                    _metrics.Counter("search.slow").Increment();
                }
            }
        }
    }
}
```

### Health Checks and Monitoring

```csharp
namespace HotelSearch.Core.Health;

public sealed class SearchServiceHealthCheck : IHealthCheck
{
    private readonly IHotelSearchService _searchService;
    private readonly IConnectionPool _connectionPool;
    private readonly ISearchCache _cache;
    private readonly ILogger<SearchServiceHealthCheck> _logger;

    public SearchServiceHealthCheck(
        IHotelSearchService searchService,
        IConnectionPool connectionPool,
        ISearchCache cache,
        ILogger<SearchServiceHealthCheck> logger)
    {
        _searchService = searchService;
        _connectionPool = connectionPool;
        _cache = cache;
        _logger = logger;
    }

    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        var healthData = new Dictionary<string, object>();

        try
        {
            // Check database connectivity
            await CheckDatabaseHealthAsync(healthData, cancellationToken);
            
            // Check cache functionality
            await CheckCacheHealthAsync(healthData, cancellationToken);
            
            // Check search functionality
            await CheckSearchHealthAsync(healthData, cancellationToken);
            
            // Check connection pool health
            CheckConnectionPoolHealth(healthData);

            return HealthCheckResult.Healthy("All systems operational", healthData);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Health check failed");
            return HealthCheckResult.Unhealthy("Health check failed", ex, healthData);
        }
    }

    private async Task CheckDatabaseHealthAsync(
        Dictionary<string, object> healthData,
        CancellationToken cancellationToken)
    {
        var connection = await _connectionPool.GetConnectionAsync(cancellationToken);
        try
        {
            using var command = connection.CreateCommand();
            command.CommandText = "SELECT COUNT(*) FROM hotels_fts LIMIT 1";
            var result = await command.ExecuteScalarAsync(cancellationToken);
            
            healthData["database_status"] = "healthy";
            healthData["database_record_count"] = result ?? 0;
        }
        finally
        {
            _connectionPool.ReturnConnection(connection);
        }
    }

    private async Task CheckCacheHealthAsync(
        Dictionary<string, object> healthData,
        CancellationToken cancellationToken)
    {
        var testKey = $"health_check_{Guid.NewGuid()}";
        var testResult = new SearchResult<Hotel> { Items = new List<Hotel>() };
        
        await _cache.SetAsync(testKey, testResult, TimeSpan.FromMinutes(1), cancellationToken);
        var retrieved = await _cache.GetAsync(testKey, cancellationToken);
        
        var cacheStats = await _cache.GetStatisticsAsync();
        
        healthData["cache_status"] = retrieved != null ? "healthy" : "degraded";
        healthData["cache_hit_rate"] = cacheStats.HitRate;
    }

    private async Task CheckSearchHealthAsync(
        Dictionary<string, object> healthData,
        CancellationToken cancellationToken)
    {
        var stopwatch = Stopwatch.StartNew();
        var result = await _searchService.SearchHotelsAsync(
            "test health check", 
            new SearchOptions { MaxResults = 1 }, 
            cancellationToken);
        stopwatch.Stop();

        healthData["search_status"] = "healthy";
        healthData["search_response_time_ms"] = stopwatch.Elapsed.TotalMilliseconds;
        healthData["search_test_results"] = result.Items.Count;
    }

    private void CheckConnectionPoolHealth(Dictionary<string, object> healthData)
    {
        var poolStats = _connectionPool.GetStats();
        
        healthData["connection_pool_utilization"] = poolStats.PoolUtilization;
        healthData["connection_pool_reuse_rate"] = poolStats.ReuseRate;
        healthData["active_connections"] = poolStats.ActiveConnections;
        healthData["available_connections"] = poolStats.AvailableConnections;
    }
}

public interface IMetrics
{
    ICounter Counter(string name);
    IHistogram Histogram(string name);
    IGauge Gauge(string name);
}

public interface ICounter
{
    void Increment(Dictionary<string, string>? tags = null);
}

public interface IHistogram
{
    void Record(double value, Dictionary<string, string>? tags = null);
}

public interface IGauge
{
    void Set(double value, Dictionary<string, string>? tags = null);
}
```

This production considerations framework provides comprehensive connection pooling, multi-level caching, circuit breaker resilience patterns, structured logging, and health monitoring for a robust production deployment.
