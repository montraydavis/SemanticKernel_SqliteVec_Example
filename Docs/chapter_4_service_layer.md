# Chapter 4: Building the Foundation Service Layer

## Implementing IHotelSearchService Interface

### Core Service Interface

The service interface defines the contract for hotel search operations, following the Interface Segregation Principle:

```csharp
using HotelSearch.Core.Models;

namespace HotelSearch.Core.Services;

public interface IHotelSearchService
{
    Task<SearchResult<Hotel>> SearchHotelsAsync(
        string query, 
        SearchOptions? options = null, 
        CancellationToken cancellationToken = default);

    Task<IReadOnlyList<Hotel>> GetAllHotelsAsync(
        CancellationToken cancellationToken = default);
    
    Task InitializeDataAsync(CancellationToken cancellationToken = default);
}
```

### Supporting Data Models

**Search options for filtering and configuration:**

```csharp
namespace HotelSearch.Core.Models;

public sealed record SearchOptions
{
    public int MaxResults { get; init; } = 10;
    public double? MinRating { get; init; }
    public string? Location { get; init; }
    public bool IncludeMetrics { get; init; } = false;
}
```

**Search result container with metrics:**

```csharp
public sealed record SearchResult<T>
{
    public IReadOnlyList<T> Items { get; init; } = Array.Empty<T>();
    public SearchMetrics? Metrics { get; init; }
    public int TotalCount => Items.Count;
}

public sealed record SearchMetrics
{
    public TimeSpan SearchDuration { get; init; }
    public int KeywordResults { get; init; }
    public int VectorResults { get; init; }
    public int HybridResults { get; init; }
    public string QueryType { get; init; } = string.Empty;
}
```

### Service Implementation Structure

**Core service class with dependency injection:**

```csharp
using Microsoft.Extensions.AI;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.VectorData;
using Microsoft.SemanticKernel.Connectors.SqliteVec;
using HotelSearch.Core.Configuration;

namespace HotelSearch.Core.Services;

public sealed class HotelSearchService : IHotelSearchService
{
    private readonly SqliteCollection<int, Hotel> _vectorCollection;
    private readonly IEmbeddingGenerator<string, Embedding<float>> _embeddingService;
    private readonly string _connectionString;
    private readonly ILogger<HotelSearchService> _logger;
    private readonly SearchConfiguration _config;

    public HotelSearchService(
        SqliteCollection<int, Hotel> vectorCollection,
        IEmbeddingGenerator<string, Embedding<float>> embeddingService,
        string connectionString,
        ILogger<HotelSearchService> logger,
        SearchConfiguration config)
    {
        _vectorCollection = vectorCollection ?? throw new ArgumentNullException(nameof(vectorCollection));
        _embeddingService = embeddingService ?? throw new ArgumentNullException(nameof(embeddingService));
        _connectionString = connectionString ?? throw new ArgumentNullException(nameof(connectionString));
        _logger = logger ?? throw new ArgumentNullException(nameof(logger));
        _config = config ?? throw new ArgumentNullException(nameof(config));
    }

    // Implementation methods follow...
}
```

### Search Method Implementation

**Main search orchestration:**

```csharp
public async Task<SearchResult<Hotel>> SearchHotelsAsync(
    string query, 
    SearchOptions? options = null, 
    CancellationToken cancellationToken = default)
{
    ArgumentException.ThrowIfNullOrWhiteSpace(query);
    
    options ??= new SearchOptions();
    var stopwatch = Stopwatch.StartNew();

    try
    {
        var (keywordResults, vectorResults) = await ExecuteParallelSearchAsync(
            query, options, cancellationToken);

        var rankedResults = ApplyReciprocalRankFusion(
            keywordResults, vectorResults, options);

        var metrics = options.IncludeMetrics ? CreateSearchMetrics(
            stopwatch.Elapsed, keywordResults.Count, vectorResults.Count, 
            rankedResults.Count, query) : null;

        return new SearchResult<Hotel>
        {
            Items = rankedResults.Select(r => r.Hotel).ToList(),
            Metrics = metrics
        };
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "Search failed for query: {Query}", query);
        throw;
    }
}
```

**Parallel search execution:**

```csharp
private async Task<(List<Hotel> keyword, List<VectorSearchResult<Hotel>> vector)> 
    ExecuteParallelSearchAsync(
        string query, 
        SearchOptions options, 
        CancellationToken cancellationToken)
{
    var searchLimit = Math.Max(
        options.MaxResults * _config.SearchMultiplier, 
        _config.MinSearchLimit);
    
    var keywordTask = PerformKeywordSearchAsync(query, searchLimit, cancellationToken);
    var vectorTask = GenerateEmbeddingAndSearchAsync(query, searchLimit, cancellationToken);
    
    await Task.WhenAll(keywordTask, vectorTask);

    return (await keywordTask, await vectorTask);
}
```

## Dependency Injection Configuration

### Service Registration Extension

**Clean service registration following the Extension Object pattern:**

```csharp
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.AI;
using Microsoft.SemanticKernel;
using Microsoft.SemanticKernel.Connectors.SqliteVec;
using HotelSearch.Core.Configuration;
using HotelSearch.Core.Services;
using HotelSearch.Core.Models;

namespace HotelSearch.Core.Extensions;

public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddHotelSearchServices(
        this IServiceCollection services, 
        IConfiguration configuration)
    {
        // Configure settings
        services.Configure<SearchConfiguration>(
            configuration.GetSection(SearchConfiguration.SectionName));
        
        services.Configure<OpenAIConfiguration>(
            configuration.GetSection(OpenAIConfiguration.SectionName));
        
        services.Configure<SQLiteVecConfiguration>(
            configuration.GetSection(SQLiteVecConfiguration.SectionName));

        // Register core services
        services.AddOpenAIServices(configuration);
        services.AddSQLiteVecServices(configuration);
        services.AddSearchServices(configuration);

        return services;
    }

    private static IServiceCollection AddOpenAIServices(
        this IServiceCollection services, 
        IConfiguration configuration)
    {
        var openAIConfig = configuration
            .GetSection(OpenAIConfiguration.SectionName)
            .Get<OpenAIConfiguration>() ?? new OpenAIConfiguration();

        var apiKey = Environment.GetEnvironmentVariable("OPENAI_API_KEY") 
                    ?? openAIConfig.ApiKey;

        if (string.IsNullOrWhiteSpace(apiKey))
        {
            throw new InvalidOperationException(
                "OpenAI API key not found. Set OPENAI_API_KEY environment variable.");
        }

        services.AddOpenAIEmbeddingGenerator(openAIConfig.EmbeddingModel, apiKey);

        return services;
    }

    private static IServiceCollection AddSQLiteVecServices(
        this IServiceCollection services, 
        IConfiguration configuration)
    {
        var sqliteConfig = configuration
            .GetSection(SQLiteVecConfiguration.SectionName)
            .Get<SQLiteVecConfiguration>() ?? new SQLiteVecConfiguration();

        services.AddSingleton<SqliteCollection<int, Hotel>>(provider =>
        {
            return new SqliteCollection<int, Hotel>(
                sqliteConfig.ConnectionString, 
                sqliteConfig.CollectionName);
        });

        return services;
    }

    private static IServiceCollection AddSearchServices(
        this IServiceCollection services, 
        IConfiguration configuration)
    {
        var searchConfig = configuration
            .GetSection(SearchConfiguration.SectionName)
            .Get<SearchConfiguration>() ?? new SearchConfiguration();

        var sqliteConfig = configuration
            .GetSection(SQLiteVecConfiguration.SectionName)
            .Get<SQLiteVecConfiguration>() ?? new SQLiteVecConfiguration();

        services.AddSingleton<IHotelSearchService>(provider =>
        {
            return new HotelSearchService(
                provider.GetRequiredService<SqliteCollection<int, Hotel>>(),
                provider.GetRequiredService<IEmbeddingGenerator<string, Embedding<float>>>(),
                sqliteConfig.ConnectionString,
                provider.GetRequiredService<ILogger<HotelSearchService>>(),
                searchConfig);
        });

        return services;
    }
}
```

### Configuration Validation

**Startup validation service:**

```csharp
namespace HotelSearch.Core.Services;

public interface IStartupValidationService
{
    Task ValidateAsync(CancellationToken cancellationToken = default);
}

public sealed class StartupValidationService : IStartupValidationService
{
    private readonly IEmbeddingGenerator<string, Embedding<float>> _embeddingService;
    private readonly SqliteCollection<int, Hotel> _vectorCollection;
    private readonly ILogger<StartupValidationService> _logger;

    public StartupValidationService(
        IEmbeddingGenerator<string, Embedding<float>> embeddingService,
        SqliteCollection<int, Hotel> vectorCollection,
        ILogger<StartupValidationService> logger)
    {
        _embeddingService = embeddingService;
        _vectorCollection = vectorCollection;
        _logger = logger;
    }

    public async Task ValidateAsync(CancellationToken cancellationToken = default)
    {
        await ValidateOpenAIConnectionAsync(cancellationToken);
        await ValidateDatabaseConnectionAsync(cancellationToken);
    }

    private async Task ValidateOpenAIConnectionAsync(CancellationToken cancellationToken)
    {
        try
        {
            _logger.LogInformation("Validating OpenAI connection...");
            
            var testEmbedding = await _embeddingService.GenerateAsync(
                "test connection", cancellationToken);
            
            _logger.LogInformation(
                "OpenAI connection successful. Embedding dimensions: {Dimensions}",
                testEmbedding.Vector.Length);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "OpenAI connection validation failed");
            throw new InvalidOperationException(
                "Failed to connect to OpenAI. Check API key and network connection.", ex);
        }
    }

    private async Task ValidateDatabaseConnectionAsync(CancellationToken cancellationToken)
    {
        try
        {
            _logger.LogInformation("Validating database connection...");
            
            var exists = await _vectorCollection.CollectionExistsAsync();
            
            _logger.LogInformation(
                "Database connection successful. Collection exists: {Exists}", exists);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Database connection validation failed");
            throw new InvalidOperationException(
                "Failed to connect to database. Check connection string and permissions.", ex);
        }
    }
}
```

## Service Registration Patterns

### Factory Pattern for Complex Services

**Service factory for advanced scenarios:**

```csharp
namespace HotelSearch.Core.Factories;

public interface IHotelSearchServiceFactory
{
    IHotelSearchService CreateService(SearchConfiguration? config = null);
}

public sealed class HotelSearchServiceFactory : IHotelSearchServiceFactory
{
    private readonly SqliteCollection<int, Hotel> _vectorCollection;
    private readonly IEmbeddingGenerator<string, Embedding<float>> _embeddingService;
    private readonly string _connectionString;
    private readonly ILogger<HotelSearchService> _logger;
    private readonly SearchConfiguration _defaultConfig;

    public HotelSearchServiceFactory(
        SqliteCollection<int, Hotel> vectorCollection,
        IEmbeddingGenerator<string, Embedding<float>> embeddingService,
        string connectionString,
        ILogger<HotelSearchService> logger,
        SearchConfiguration defaultConfig)
    {
        _vectorCollection = vectorCollection;
        _embeddingService = embeddingService;
        _connectionString = connectionString;
        _logger = logger;
        _defaultConfig = defaultConfig;
    }

    public IHotelSearchService CreateService(SearchConfiguration? config = null)
    {
        return new HotelSearchService(
            _vectorCollection,
            _embeddingService,
            _connectionString,
            _logger,
            config ?? _defaultConfig);
    }
}
```

### Decorator Pattern for Enhanced Functionality

**Caching decorator:**

```csharp
namespace HotelSearch.Core.Decorators;

public sealed class CachedHotelSearchService : IHotelSearchService
{
    private readonly IHotelSearchService _inner;
    private readonly IMemoryCache _cache;
    private readonly TimeSpan _cacheDuration;

    public CachedHotelSearchService(
        IHotelSearchService inner,
        IMemoryCache cache,
        TimeSpan cacheDuration)
    {
        _inner = inner;
        _cache = cache;
        _cacheDuration = cacheDuration;
    }

    public async Task<SearchResult<Hotel>> SearchHotelsAsync(
        string query, 
        SearchOptions? options = null, 
        CancellationToken cancellationToken = default)
    {
        var cacheKey = CreateCacheKey(query, options);
        
        if (_cache.TryGetValue(cacheKey, out SearchResult<Hotel>? cachedResult))
        {
            return cachedResult!;
        }

        var result = await _inner.SearchHotelsAsync(query, options, cancellationToken);
        
        _cache.Set(cacheKey, result, _cacheDuration);
        
        return result;
    }

    public Task<IReadOnlyList<Hotel>> GetAllHotelsAsync(
        CancellationToken cancellationToken = default)
    {
        return _inner.GetAllHotelsAsync(cancellationToken);
    }

    public Task InitializeDataAsync(CancellationToken cancellationToken = default)
    {
        return _inner.InitializeDataAsync(cancellationToken);
    }

    private static string CreateCacheKey(string query, SearchOptions? options)
    {
        var key = $"search:{query}";
        if (options != null)
        {
            key += $":{options.MaxResults}:{options.MinRating}:{options.Location}";
        }
        return key;
    }
}
```

### Scoped vs Singleton Registration

**Register appropriate service lifetimes:**

```csharp
public static class ServiceLifetimeExtensions
{
    public static IServiceCollection AddHotelSearchWithCaching(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        // Add base services
        services.AddHotelSearchServices(configuration);
        
        // Add caching
        services.AddMemoryCache();
        
        // Register cached decorator
        services.Decorate<IHotelSearchService>((inner, provider) =>
        {
            var cache = provider.GetRequiredService<IMemoryCache>();
            return new CachedHotelSearchService(inner, cache, TimeSpan.FromMinutes(5));
        });

        return services;
    }

    public static IServiceCollection AddHotelSearchWithValidation(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        services.AddHotelSearchServices(configuration);
        services.AddScoped<IStartupValidationService, StartupValidationService>();
        
        return services;
    }
}
```

## Error Handling Strategies

### Custom Exception Types

**Domain-specific exceptions:**

```csharp
namespace HotelSearch.Core.Exceptions;

public abstract class HotelSearchException : Exception
{
    protected HotelSearchException(string message) : base(message) { }
    protected HotelSearchException(string message, Exception innerException) 
        : base(message, innerException) { }
}

public sealed class SearchQueryException : HotelSearchException
{
    public string Query { get; }

    public SearchQueryException(string query, string message) 
        : base($"Invalid search query '{query}': {message}")
    {
        Query = query;
    }

    public SearchQueryException(string query, string message, Exception innerException) 
        : base($"Search query '{query}' failed: {message}", innerException)
    {
        Query = query;
    }
}

public sealed class EmbeddingGenerationException : HotelSearchException
{
    public string Text { get; }

    public EmbeddingGenerationException(string text, Exception innerException)
        : base($"Failed to generate embedding for text: {text[..Math.Min(50, text.Length)]}", innerException)
    {
        Text = text;
    }
}

public sealed class DatabaseConnectionException : HotelSearchException
{
    public string ConnectionString { get; }

    public DatabaseConnectionException(string connectionString, Exception innerException)
        : base("Failed to connect to database", innerException)
    {
        ConnectionString = connectionString;
    }
}
```

### Resilience Patterns

**Retry policy implementation:**

```csharp
namespace HotelSearch.Core.Services;

public sealed class ResilientHotelSearchService : IHotelSearchService
{
    private readonly IHotelSearchService _inner;
    private readonly ILogger<ResilientHotelSearchService> _logger;
    private readonly RetryConfiguration _retryConfig;

    public ResilientHotelSearchService(
        IHotelSearchService inner,
        ILogger<ResilientHotelSearchService> logger,
        RetryConfiguration retryConfig)
    {
        _inner = inner;
        _logger = logger;
        _retryConfig = retryConfig;
    }

    public async Task<SearchResult<Hotel>> SearchHotelsAsync(
        string query, 
        SearchOptions? options = null, 
        CancellationToken cancellationToken = default)
    {
        return await ExecuteWithRetryAsync(
            () => _inner.SearchHotelsAsync(query, options, cancellationToken),
            $"SearchHotels({query})",
            cancellationToken);
    }

    private async Task<T> ExecuteWithRetryAsync<T>(
        Func<Task<T>> operation,
        string operationName,
        CancellationToken cancellationToken)
    {
        var attempt = 0;
        Exception? lastException = null;

        while (attempt < _retryConfig.MaxAttempts)
        {
            try
            {
                return await operation();
            }
            catch (Exception ex) when (ShouldRetry(ex, attempt))
            {
                lastException = ex;
                attempt++;
                
                var delay = CalculateDelay(attempt);
                _logger.LogWarning(
                    "Operation {OperationName} failed on attempt {Attempt}. Retrying in {Delay}ms. Error: {Error}",
                    operationName, attempt, delay.TotalMilliseconds, ex.Message);

                await Task.Delay(delay, cancellationToken);
            }
        }

        throw new OperationFailedException(
            $"Operation {operationName} failed after {_retryConfig.MaxAttempts} attempts",
            lastException!);
    }

    private bool ShouldRetry(Exception exception, int attempt)
    {
        if (attempt >= _retryConfig.MaxAttempts)
            return false;

        return exception switch
        {
            HttpRequestException => true,
            TaskCanceledException => false,
            ArgumentException => false,
            _ => true
        };
    }

    private TimeSpan CalculateDelay(int attempt)
    {
        return TimeSpan.FromMilliseconds(
            _retryConfig.BaseDelayMs * Math.Pow(2, attempt - 1));
    }
}

public sealed record RetryConfiguration
{
    public int MaxAttempts { get; init; } = 3;
    public int BaseDelayMs { get; init; } = 1000;
}
```

### Error Boundary Pattern

**Service wrapper for graceful degradation:**

```csharp
public sealed class ErrorBoundaryHotelSearchService : IHotelSearchService
{
    private readonly IHotelSearchService _inner;
    private readonly ILogger<ErrorBoundaryHotelSearchService> _logger;

    public ErrorBoundaryHotelSearchService(
        IHotelSearchService inner,
        ILogger<ErrorBoundaryHotelSearchService> logger)
    {
        _inner = inner;
        _logger = logger;
    }

    public async Task<SearchResult<Hotel>> SearchHotelsAsync(
        string query, 
        SearchOptions? options = null, 
        CancellationToken cancellationToken = default)
    {
        try
        {
            return await _inner.SearchHotelsAsync(query, options, cancellationToken);
        }
        catch (EmbeddingGenerationException ex)
        {
            _logger.LogError(ex, "Embedding generation failed, falling back to keyword search");
            return await FallbackToKeywordSearch(query, options, cancellationToken);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Search operation failed completely");
            return new SearchResult<Hotel> { Items = Array.Empty<Hotel>() };
        }
    }

    private async Task<SearchResult<Hotel>> FallbackToKeywordSearch(
        string query,
        SearchOptions? options,
        CancellationToken cancellationToken)
    {
        // Implement keyword-only search fallback
        // This would use direct SQL queries against FTS5 tables
        return new SearchResult<Hotel> { Items = Array.Empty<Hotel>() };
    }
}
```

This service layer foundation provides a robust, testable, and maintainable architecture for the hotel search functionality, following SOLID principles and modern C# practices.
