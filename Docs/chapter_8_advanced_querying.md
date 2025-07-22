# Chapter 8: Advanced Querying Patterns

## Filtering and Metadata Search

### Advanced Filtering System

Building sophisticated filtering that combines traditional database filters with vector search constraints:

```csharp
namespace HotelSearch.Core.Filtering;

public sealed class AdvancedFilterEngine
{
    private readonly string _connectionString;
    private readonly ILogger<AdvancedFilterEngine> _logger;

    public AdvancedFilterEngine(string connectionString, ILogger<AdvancedFilterEngine> logger)
    {
        _connectionString = connectionString;
        _logger = logger;
    }

    public async Task<List<Hotel>> ApplyFiltersAsync(
        SearchCriteria criteria,
        CancellationToken cancellationToken = default)
    {
        var filterBuilder = new FilterQueryBuilder();
        var (sql, parameters) = filterBuilder.BuildFilterQuery(criteria);

        return await ExecuteFilterQueryAsync(sql, parameters, cancellationToken);
    }

    public async Task<List<Hotel>> ApplyHybridFiltersAsync(
        SearchCriteria criteria,
        List<int> vectorResultIds,
        CancellationToken cancellationToken = default)
    {
        var filterBuilder = new FilterQueryBuilder();
        var (sql, parameters) = filterBuilder.BuildHybridFilterQuery(criteria, vectorResultIds);

        return await ExecuteFilterQueryAsync(sql, parameters, cancellationToken);
    }

    private async Task<List<Hotel>> ExecuteFilterQueryAsync(
        string sql,
        Dictionary<string, object> parameters,
        CancellationToken cancellationToken)
    {
        using var connection = new SqliteConnection(_connectionString);
        await connection.OpenAsync(cancellationToken);

        using var command = connection.CreateCommand();
        command.CommandText = sql;

        foreach (var (key, value) in parameters)
        {
            command.Parameters.AddWithValue(key, value);
        }

        var results = new List<Hotel>();
        using var reader = await command.ExecuteReaderAsync(cancellationToken);

        while (await reader.ReadAsync(cancellationToken))
        {
            results.Add(MapToHotel(reader));
        }

        _logger.LogDebug("Filter query returned {Count} results", results.Count);
        return results;
    }

    private static Hotel MapToHotel(SqliteDataReader reader)
    {
        return new Hotel
        {
            HotelId = reader.GetInt32("HotelId"),
            HotelName = reader.GetString("HotelName"),
            Description = reader.GetString("Description"),
            Location = reader.GetString("Location"),
            Rating = reader.GetDouble("Rating"),
            PricePerNight = reader.GetDecimal("PricePerNight"),
            LastUpdated = DateTime.Parse(reader.GetString("LastUpdated"))
        };
    }
}

public sealed class FilterQueryBuilder
{
    public (string Sql, Dictionary<string, object> Parameters) BuildFilterQuery(SearchCriteria criteria)
    {
        var conditions = new List<string>();
        var parameters = new Dictionary<string, object>();

        BuildRatingFilter(criteria.RatingFilter, conditions, parameters);
        BuildPriceFilter(criteria.PriceFilter, conditions, parameters);
        BuildLocationFilter(criteria.LocationFilter, conditions, parameters);
        BuildAmenityFilter(criteria.AmenityFilter, conditions, parameters);
        BuildDateFilter(criteria.DateFilter, conditions, parameters);

        var whereClause = conditions.Count > 0 ? $"WHERE {string.Join(" AND ", conditions)}" : "";
        var orderClause = BuildOrderClause(criteria.SortOptions);

        var sql = $"""
            SELECT HotelId, HotelName, Description, Location, Rating, PricePerNight, LastUpdated
            FROM hotels_fts
            {whereClause}
            {orderClause}
            LIMIT @limit
            """;

        parameters.Add("@limit", criteria.MaxResults);

        return (sql, parameters);
    }

    public (string Sql, Dictionary<string, object> Parameters) BuildHybridFilterQuery(
        SearchCriteria criteria,
        List<int> vectorResultIds)
    {
        var (baseSql, parameters) = BuildFilterQuery(criteria);
        
        if (vectorResultIds.Count > 0)
        {
            var idList = string.Join(",", vectorResultIds);
            var whereClause = baseSql.Contains("WHERE") 
                ? $" AND HotelId IN ({idList})"
                : $" WHERE HotelId IN ({idList})";
            
            baseSql = baseSql.Replace("LIMIT @limit", whereClause + " LIMIT @limit");
        }

        return (baseSql, parameters);
    }

    private void BuildRatingFilter(
        RatingFilter? filter,
        List<string> conditions,
        Dictionary<string, object> parameters)
    {
        if (filter is null) return;

        if (filter.MinRating.HasValue)
        {
            conditions.Add("Rating >= @minRating");
            parameters.Add("@minRating", filter.MinRating.Value);
        }

        if (filter.MaxRating.HasValue)
        {
            conditions.Add("Rating <= @maxRating");
            parameters.Add("@maxRating", filter.MaxRating.Value);
        }
    }

    private void BuildPriceFilter(
        PriceFilter? filter,
        List<string> conditions,
        Dictionary<string, object> parameters)
    {
        if (filter is null) return;

        if (filter.MinPrice.HasValue)
        {
            conditions.Add("PricePerNight >= @minPrice");
            parameters.Add("@minPrice", filter.MinPrice.Value);
        }

        if (filter.MaxPrice.HasValue)
        {
            conditions.Add("PricePerNight <= @maxPrice");
            parameters.Add("@maxPrice", filter.MaxPrice.Value);
        }
    }

    private void BuildLocationFilter(
        LocationFilter? filter,
        List<string> conditions,
        Dictionary<string, object> parameters)
    {
        if (filter is null) return;

        if (!string.IsNullOrWhiteSpace(filter.City))
        {
            conditions.Add("Location LIKE @city");
            parameters.Add("@city", $"%{filter.City}%");
        }

        if (!string.IsNullOrWhiteSpace(filter.State))
        {
            conditions.Add("Location LIKE @state");
            parameters.Add("@state", $"%{filter.State}%");
        }

        if (filter.Coordinates != null && filter.RadiusKm.HasValue)
        {
            // For geo-distance filtering, you'd implement spatial functions
            // This is a simplified version
            conditions.Add("1=1"); // Placeholder for geo-distance logic
        }
    }

    private void BuildAmenityFilter(
        AmenityFilter? filter,
        List<string> conditions,
        Dictionary<string, object> parameters)
    {
        if (filter?.RequiredAmenities is null || filter.RequiredAmenities.Count == 0) return;

        // This assumes amenities are stored as JSON or comma-separated values
        for (int i = 0; i < filter.RequiredAmenities.Count; i++)
        {
            var paramName = $"@amenity{i}";
            conditions.Add($"Description LIKE {paramName}");
            parameters.Add(paramName, $"%{filter.RequiredAmenities[i]}%");
        }
    }

    private void BuildDateFilter(
        DateFilter? filter,
        List<string> conditions,
        Dictionary<string, object> parameters)
    {
        if (filter is null) return;

        if (filter.UpdatedAfter.HasValue)
        {
            conditions.Add("LastUpdated >= @updatedAfter");
            parameters.Add("@updatedAfter", filter.UpdatedAfter.Value.ToString("O"));
        }
    }

    private string BuildOrderClause(SortOptions? sortOptions)
    {
        if (sortOptions is null) return "";

        return sortOptions.SortBy switch
        {
            SortField.Rating => $"ORDER BY Rating {GetSortDirection(sortOptions.Direction)}",
            SortField.Price => $"ORDER BY PricePerNight {GetSortDirection(sortOptions.Direction)}",
            SortField.Name => $"ORDER BY HotelName {GetSortDirection(sortOptions.Direction)}",
            SortField.LastUpdated => $"ORDER BY LastUpdated {GetSortDirection(sortOptions.Direction)}",
            _ => ""
        };
    }

    private static string GetSortDirection(SortDirection direction) =>
        direction == SortDirection.Ascending ? "ASC" : "DESC";
}

// Supporting models for advanced filtering
public sealed record SearchCriteria
{
    public RatingFilter? RatingFilter { get; init; }
    public PriceFilter? PriceFilter { get; init; }
    public LocationFilter? LocationFilter { get; init; }
    public AmenityFilter? AmenityFilter { get; init; }
    public DateFilter? DateFilter { get; init; }
    public SortOptions? SortOptions { get; init; }
    public int MaxResults { get; init; } = 50;
}

public sealed record RatingFilter(double? MinRating = null, double? MaxRating = null);
public sealed record PriceFilter(decimal? MinPrice = null, decimal? MaxPrice = null);
public sealed record LocationFilter(
    string? City = null, 
    string? State = null, 
    GeoCoordinates? Coordinates = null, 
    double? RadiusKm = null);
public sealed record AmenityFilter(List<string> RequiredAmenities);
public sealed record DateFilter(DateTime? UpdatedAfter = null);
public sealed record SortOptions(SortField SortBy, SortDirection Direction = SortDirection.Descending);
public sealed record GeoCoordinates(double Latitude, double Longitude);

public enum SortField { Rating, Price, Name, LastUpdated }
public enum SortDirection { Ascending, Descending }
```

### Metadata-Enhanced Search

```csharp
namespace HotelSearch.Core.Metadata;

public sealed class MetadataSearchEngine
{
    private readonly SqliteCollection<int, Hotel> _vectorCollection;
    private readonly IEmbeddingGenerator<string, Embedding<float>> _embeddingService;

    public MetadataSearchEngine(
        SqliteCollection<int, Hotel> vectorCollection,
        IEmbeddingGenerator<string, Embedding<float>> embeddingService)
    {
        _vectorCollection = vectorCollection;
        _embeddingService = embeddingService;
    }

    public async Task<List<VectorSearchResult<Hotel>>> SearchWithMetadataAsync(
        string query,
        MetadataConstraints constraints,
        int limit = 10,
        CancellationToken cancellationToken = default)
    {
        var queryEmbedding = await _embeddingService.GenerateAsync(query, cancellationToken);
        
        var searchOptions = new VectorSearchOptions<Hotel>
        {
            VectorProperty = static v => v.DescriptionEmbedding,
            Filter = CreateMetadataFilter(constraints)
        };

        var results = new List<VectorSearchResult<Hotel>>();
        await foreach (var result in _vectorCollection
            .SearchAsync(queryEmbedding, limit, searchOptions)
            .WithCancellation(cancellationToken))
        {
            results.Add(result);
        }

        return results;
    }

    private Func<Hotel, bool> CreateMetadataFilter(MetadataConstraints constraints)
    {
        return hotel =>
        {
            if (constraints.MinRating.HasValue && hotel.Rating < constraints.MinRating.Value)
                return false;

            if (constraints.MaxPrice.HasValue && hotel.PricePerNight > constraints.MaxPrice.Value)
                return false;

            if (!string.IsNullOrEmpty(constraints.LocationContains) &&
                !hotel.Location.Contains(constraints.LocationContains, StringComparison.OrdinalIgnoreCase))
                return false;

            if (constraints.UpdatedAfter.HasValue && hotel.LastUpdated < constraints.UpdatedAfter.Value)
                return false;

            return true;
        };
    }
}

public sealed record MetadataConstraints
{
    public double? MinRating { get; init; }
    public decimal? MaxPrice { get; init; }
    public string? LocationContains { get; init; }
    public DateTime? UpdatedAfter { get; init; }
}
```

## Batch Operations and Bulk Processing

### Efficient Batch Processing

```csharp
namespace HotelSearch.Core.Batch;

public sealed class BatchOperationService
{
    private readonly SqliteCollection<int, Hotel> _vectorCollection;
    private readonly IEmbeddingGenerator<string, Embedding<float>> _embeddingService;
    private readonly string _connectionString;
    private readonly ILogger<BatchOperationService> _logger;

    public BatchOperationService(
        SqliteCollection<int, Hotel> vectorCollection,
        IEmbeddingGenerator<string, Embedding<float>> embeddingService,
        string connectionString,
        ILogger<BatchOperationService> logger)
    {
        _vectorCollection = vectorCollection;
        _embeddingService = embeddingService;
        _connectionString = connectionString;
        _logger = logger;
    }

    public async Task<BatchProcessingResult> ProcessHotelsBatchAsync(
        IEnumerable<Hotel> hotels,
        BatchProcessingOptions options,
        CancellationToken cancellationToken = default)
    {
        var hotelList = hotels.ToList();
        _logger.LogInformation("Starting batch processing of {Count} hotels", hotelList.Count);

        var result = new BatchProcessingResult { TotalItems = hotelList.Count };
        var batches = CreateBatches(hotelList, options.BatchSize);

        foreach (var batch in batches)
        {
            var batchResult = await ProcessSingleBatchAsync(batch, options, cancellationToken);
            result.MergeWith(batchResult);

            if (options.DelayBetweenBatches > TimeSpan.Zero)
            {
                await Task.Delay(options.DelayBetweenBatches, cancellationToken);
            }
        }

        _logger.LogInformation(
            "Batch processing completed. Success: {Success}, Errors: {Errors}",
            result.SuccessCount, result.ErrorCount);

        return result;
    }

    private async Task<BatchProcessingResult> ProcessSingleBatchAsync(
        List<Hotel> batch,
        BatchProcessingOptions options,
        CancellationToken cancellationToken)
    {
        var result = new BatchProcessingResult();

        try
        {
            // Generate embeddings in parallel
            if (options.GenerateEmbeddings)
            {
                await GenerateEmbeddingsBatchAsync(batch, options, cancellationToken);
            }

            // Insert into vector collection
            await InsertVectorBatchAsync(batch, cancellationToken);

            // Insert into FTS table
            await InsertFtsBatchAsync(batch, cancellationToken);

            result.SuccessCount = batch.Count;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Batch processing failed for batch of {Count} items", batch.Count);
            result.ErrorCount = batch.Count;
            result.Errors.Add($"Batch error: {ex.Message}");
        }

        return result;
    }

    private async Task GenerateEmbeddingsBatchAsync(
        List<Hotel> hotels,
        BatchProcessingOptions options,
        CancellationToken cancellationToken)
    {
        var semaphore = new SemaphoreSlim(options.MaxConcurrency);
        var tasks = hotels.Select(async hotel =>
        {
            await semaphore.WaitAsync(cancellationToken);
            try
            {
                var combinedText = $"{hotel.HotelName}. {hotel.Description}. Located in {hotel.Location}";
                var embedding = await _embeddingService.GenerateAsync(combinedText, cancellationToken);
                hotel.DescriptionEmbedding = embedding.Vector;
            }
            catch (Exception ex)
            {
                _logger.LogWarning(ex, "Failed to generate embedding for hotel {HotelId}", hotel.HotelId);
                throw;
            }
            finally
            {
                semaphore.Release();
            }
        });

        await Task.WhenAll(tasks);
    }

    private async Task InsertVectorBatchAsync(List<Hotel> hotels, CancellationToken cancellationToken)
    {
        foreach (var hotel in hotels)
        {
            await _vectorCollection.UpsertAsync(hotel, cancellationToken);
        }
    }

    private async Task InsertFtsBatchAsync(List<Hotel> hotels, CancellationToken cancellationToken)
    {
        using var connection = new SqliteConnection(_connectionString);
        await connection.OpenAsync(cancellationToken);

        using var transaction = connection.BeginTransaction();
        try
        {
            const string insertSql = """
                INSERT OR REPLACE INTO hotels_fts 
                (HotelId, HotelName, Description, Location, Rating, PricePerNight, LastUpdated)
                VALUES (@id, @name, @description, @location, @rating, @price, @updated)
                """;

            foreach (var hotel in hotels)
            {
                using var command = connection.CreateCommand();
                command.CommandText = insertSql;
                command.Transaction = transaction;
                command.Parameters.AddWithValue("@id", hotel.HotelId);
                command.Parameters.AddWithValue("@name", hotel.HotelName);
                command.Parameters.AddWithValue("@description", hotel.Description);
                command.Parameters.AddWithValue("@location", hotel.Location);
                command.Parameters.AddWithValue("@rating", hotel.Rating);
                command.Parameters.AddWithValue("@price", hotel.PricePerNight);
                command.Parameters.AddWithValue("@updated", hotel.LastUpdated.ToString("O"));

                await command.ExecuteNonQueryAsync(cancellationToken);
            }

            transaction.Commit();
        }
        catch
        {
            transaction.Rollback();
            throw;
        }
    }

    private static List<List<Hotel>> CreateBatches(List<Hotel> hotels, int batchSize)
    {
        var batches = new List<List<Hotel>>();
        for (int i = 0; i < hotels.Count; i += batchSize)
        {
            var batch = hotels.Skip(i).Take(batchSize).ToList();
            batches.Add(batch);
        }
        return batches;
    }
}

public sealed record BatchProcessingOptions
{
    public int BatchSize { get; init; } = 50;
    public int MaxConcurrency { get; init; } = 5;
    public bool GenerateEmbeddings { get; init; } = true;
    public TimeSpan DelayBetweenBatches { get; init; } = TimeSpan.Zero;
}

public sealed class BatchProcessingResult
{
    public int TotalItems { get; set; }
    public int SuccessCount { get; set; }
    public int ErrorCount { get; set; }
    public List<string> Errors { get; set; } = new();

    public void MergeWith(BatchProcessingResult other)
    {
        SuccessCount += other.SuccessCount;
        ErrorCount += other.ErrorCount;
        Errors.AddRange(other.Errors);
    }

    public double SuccessRate => TotalItems > 0 ? (double)SuccessCount / TotalItems : 0;
}
```

### Bulk Search Operations

```csharp
namespace HotelSearch.Core.Batch;

public sealed class BulkSearchService
{
    private readonly IHotelSearchService _searchService;
    private readonly ILogger<BulkSearchService> _logger;

    public BulkSearchService(IHotelSearchService searchService, ILogger<BulkSearchService> logger)
    {
        _searchService = searchService;
        _logger = logger;
    }

    public async Task<BulkSearchResult> ExecuteBulkSearchAsync(
        string[] queries,
        SearchOptions searchOptions,
        BulkSearchOptions bulkOptions,
        CancellationToken cancellationToken = default)
    {
        _logger.LogInformation("Starting bulk search for {QueryCount} queries", queries.Length);

        var results = new List<QuerySearchResult>();
        var semaphore = new SemaphoreSlim(bulkOptions.MaxConcurrency);

        var tasks = queries.Select(async query =>
        {
            await semaphore.WaitAsync(cancellationToken);
            try
            {
                return await ExecuteSingleQueryAsync(query, searchOptions, cancellationToken);
            }
            finally
            {
                semaphore.Release();
            }
        });

        var queryResults = await Task.WhenAll(tasks);
        results.AddRange(queryResults);

        return new BulkSearchResult
        {
            QueryResults = results,
            TotalQueries = queries.Length,
            SuccessfulQueries = results.Count(r => r.Success),
            FailedQueries = results.Count(r => !r.Success),
            AverageResponseTime = TimeSpan.FromMilliseconds(
                results.Where(r => r.Success).Average(r => r.ResponseTime.TotalMilliseconds))
        };
    }

    private async Task<QuerySearchResult> ExecuteSingleQueryAsync(
        string query,
        SearchOptions searchOptions,
        CancellationToken cancellationToken)
    {
        var stopwatch = Stopwatch.StartNew();

        try
        {
            var result = await _searchService.SearchHotelsAsync(query, searchOptions, cancellationToken);
            stopwatch.Stop();

            return new QuerySearchResult
            {
                Query = query,
                Success = true,
                ResultCount = result.Items.Count,
                ResponseTime = stopwatch.Elapsed,
                Results = result.Items.ToList()
            };
        }
        catch (Exception ex)
        {
            stopwatch.Stop();
            _logger.LogWarning(ex, "Search failed for query: {Query}", query);

            return new QuerySearchResult
            {
                Query = query,
                Success = false,
                ResponseTime = stopwatch.Elapsed,
                ErrorMessage = ex.Message
            };
        }
    }
}

public sealed record BulkSearchOptions
{
    public int MaxConcurrency { get; init; } = 10;
    public TimeSpan Timeout { get; init; } = TimeSpan.FromSeconds(30);
}

public sealed record BulkSearchResult
{
    public List<QuerySearchResult> QueryResults { get; init; } = new();
    public int TotalQueries { get; init; }
    public int SuccessfulQueries { get; init; }
    public int FailedQueries { get; init; }
    public TimeSpan AverageResponseTime { get; init; }
}

public sealed record QuerySearchResult
{
    public string Query { get; init; } = string.Empty;
    public bool Success { get; init; }
    public int ResultCount { get; init; }
    public TimeSpan ResponseTime { get; init; }
    public List<Hotel> Results { get; init; } = new();
    public string? ErrorMessage { get; init; }
}
```

## Custom Distance Functions

### Implementing Custom Distance Metrics

```csharp
namespace HotelSearch.Core.Distance;

public interface IDistanceFunction
{
    string Name { get; }
    double CalculateDistance(ReadOnlySpan<float> vector1, ReadOnlySpan<float> vector2);
    DistanceFunction GetDistanceFunction();
}

public sealed class WeightedCosineSimilarity : IDistanceFunction
{
    private readonly float[] _weights;

    public WeightedCosineSimilarity(float[] weights)
    {
        _weights = weights ?? throw new ArgumentNullException(nameof(weights));
    }

    public string Name => "WeightedCosine";

    public double CalculateDistance(ReadOnlySpan<float> vector1, ReadOnlySpan<float> vector2)
    {
        if (vector1.Length != vector2.Length || vector1.Length != _weights.Length)
        {
            throw new ArgumentException("Vector dimensions must match weight dimensions");
        }

        double dotProduct = 0;
        double norm1 = 0;
        double norm2 = 0;

        for (int i = 0; i < vector1.Length; i++)
        {
            var weighted1 = vector1[i] * _weights[i];
            var weighted2 = vector2[i] * _weights[i];
            
            dotProduct += weighted1 * weighted2;
            norm1 += weighted1 * weighted1;
            norm2 += weighted2 * weighted2;
        }

        if (norm1 == 0 || norm2 == 0) return 1.0; // Maximum distance

        var cosineSimilarity = dotProduct / (Math.Sqrt(norm1) * Math.Sqrt(norm2));
        return 1.0 - cosineSimilarity; // Convert similarity to distance
    }

    public DistanceFunction GetDistanceFunction() => DistanceFunction.CosineDistance;
}

public sealed class SemanticSimilarity : IDistanceFunction
{
    private readonly float _conceptWeight;
    private readonly float _lexicalWeight;

    public SemanticSimilarity(float conceptWeight = 0.7f, float lexicalWeight = 0.3f)
    {
        _conceptWeight = conceptWeight;
        _lexicalWeight = lexicalWeight;
    }

    public string Name => "SemanticSimilarity";

    public double CalculateDistance(ReadOnlySpan<float> vector1, ReadOnlySpan<float> vector2)
    {
        // Split vector into conceptual and lexical components
        var conceptDimensions = vector1.Length / 2;
        
        var conceptual1 = vector1[..conceptDimensions];
        var conceptual2 = vector2[..conceptDimensions];
        var lexical1 = vector1[conceptDimensions..];
        var lexical2 = vector2[conceptDimensions..];

        var conceptualDistance = CalculateCosineDistance(conceptual1, conceptual2);
        var lexicalDistance = CalculateCosineDistance(lexical1, lexical2);

        return (_conceptWeight * conceptualDistance) + (_lexicalWeight * lexicalDistance);
    }

    private static double CalculateCosineDistance(ReadOnlySpan<float> v1, ReadOnlySpan<float> v2)
    {
        double dotProduct = 0;
        double norm1 = 0;
        double norm2 = 0;

        for (int i = 0; i < v1.Length; i++)
        {
            dotProduct += v1[i] * v2[i];
            norm1 += v1[i] * v1[i];
            norm2 += v2[i] * v2[i];
        }

        if (norm1 == 0 || norm2 == 0) return 1.0;

        var cosineSimilarity = dotProduct / (Math.Sqrt(norm1) * Math.Sqrt(norm2));
        return 1.0 - cosineSimilarity;
    }

    public DistanceFunction GetDistanceFunction() => DistanceFunction.CosineDistance;
}
```

### Custom Vector Search with Distance Functions

```csharp
namespace HotelSearch.Core.Search;

public sealed class CustomDistanceVectorSearch
{
    private readonly SqliteCollection<int, Hotel> _vectorCollection;
    private readonly IEmbeddingGenerator<string, Embedding<float>> _embeddingService;
    private readonly Dictionary<string, IDistanceFunction> _distanceFunctions;

    public CustomDistanceVectorSearch(
        SqliteCollection<int, Hotel> vectorCollection,
        IEmbeddingGenerator<string, Embedding<float>> embeddingService,
        IEnumerable<IDistanceFunction> distanceFunctions)
    {
        _vectorCollection = vectorCollection;
        _embeddingService = embeddingService;
        _distanceFunctions = distanceFunctions.ToDictionary(df => df.Name, df => df);
    }

    public async Task<List<CustomVectorResult>> SearchWithCustomDistanceAsync(
        string query,
        string distanceFunctionName,
        int limit = 10,
        CancellationToken cancellationToken = default)
    {
        if (!_distanceFunctions.TryGetValue(distanceFunctionName, out var distanceFunction))
        {
            throw new ArgumentException($"Distance function '{distanceFunctionName}' not found");
        }

        var queryEmbedding = await _embeddingService.GenerateAsync(query, cancellationToken);
        var queryVector = queryEmbedding.Vector.Span;

        // Get all vectors (in a real implementation, you'd want to use approximate nearest neighbor)
        var allHotels = await GetAllHotelsWithEmbeddingsAsync(cancellationToken);
        
        var results = new List<CustomVectorResult>();

        foreach (var hotel in allHotels)
        {
            if (hotel.DescriptionEmbedding.HasValue)
            {
                var hotelVector = hotel.DescriptionEmbedding.Value.Span;
                var distance = distanceFunction.CalculateDistance(queryVector, hotelVector);
                
                results.Add(new CustomVectorResult
                {
                    Hotel = hotel,
                    Distance = distance,
                    Similarity = 1.0 - distance, // Convert distance back to similarity
                    DistanceFunction = distanceFunctionName
                });
            }
        }

        return results
            .OrderBy(r => r.Distance)
            .Take(limit)
            .ToList();
    }

    private async Task<List<Hotel>> GetAllHotelsWithEmbeddingsAsync(CancellationToken cancellationToken)
    {
        var allHotels = new List<Hotel>();
        
        await foreach (var hotel in _vectorCollection.GetAllAsync().WithCancellation(cancellationToken))
        {
            if (hotel.DescriptionEmbedding.HasValue)
            {
                allHotels.Add(hotel);
            }
        }

        return allHotels;
    }
}

public sealed record CustomVectorResult
{
    public Hotel Hotel { get; init; } = null!;
    public double Distance { get; init; }
    public double Similarity { get; init; }
    public string DistanceFunction { get; init; } = string.Empty;
}
```

## Search Result Analytics

### Comprehensive Search Analytics

```csharp
namespace HotelSearch.Core.Analytics;

public sealed class SearchAnalyticsService
{
    private readonly ILogger<SearchAnalyticsService> _logger;
    private readonly ConcurrentDictionary<string, SearchQueryAnalytics> _queryAnalytics = new();
    private readonly ConcurrentDictionary<int, HotelPopularityAnalytics> _hotelAnalytics = new();

    public SearchAnalyticsService(ILogger<SearchAnalyticsService> logger)
    {
        _logger = logger;
    }

    public void RecordSearch(SearchAnalyticsEvent searchEvent)
    {
        RecordQueryAnalytics(searchEvent);
        RecordHotelAnalytics(searchEvent);
        RecordPerformanceMetrics(searchEvent);
    }

    private void RecordQueryAnalytics(SearchAnalyticsEvent searchEvent)
    {
        var analytics = _queryAnalytics.AddOrUpdate(
            searchEvent.Query,
            _ => new SearchQueryAnalytics { Query = searchEvent.Query },
            (_, existing) => existing);

        analytics.RecordSearch(searchEvent);
    }

    private void RecordHotelAnalytics(SearchAnalyticsEvent searchEvent)
    {
        foreach (var hotel in searchEvent.Results.Take(10)) // Top 10 results
        {
            var analytics = _hotelAnalytics.AddOrUpdate(
                hotel.HotelId,
                _ => new HotelPopularityAnalytics { HotelId = hotel.HotelId },
                (_, existing) => existing);

            analytics.RecordAppearance(searchEvent.Query, searchEvent.Timestamp);
        }
    }

    private void RecordPerformanceMetrics(SearchAnalyticsEvent searchEvent)
    {
        if (searchEvent.ResponseTime > TimeSpan.FromSeconds(2))
        {
            _logger.LogWarning(
                "Slow search detected: Query='{Query}', Time={Time}ms, Results={Results}",
                searchEvent.Query, searchEvent.ResponseTime.TotalMilliseconds, searchEvent.ResultCount);
        }
    }

    public SearchAnalyticsReport GenerateReport(TimeSpan period)
    {
        var cutoff = DateTime.UtcNow - period;
        
        var relevantQueries = _queryAnalytics.Values
            .Where(q => q.LastSearched > cutoff)
            .ToList();

        var relevantHotels = _hotelAnalytics.Values
            .Where(h => h.LastSeen > cutoff)
            .ToList();

        return new SearchAnalyticsReport
        {
            Period = period,
            GeneratedAt = DateTime.UtcNow,
            QueryAnalytics = AnalyzeQueries(relevantQueries),
            HotelAnalytics = AnalyzeHotels(relevantHotels),
            PerformanceAnalytics = AnalyzePerformance(relevantQueries)
        };
    }

    private QueryAnalyticsSummary AnalyzeQueries(List<SearchQueryAnalytics> queries)
    {
        if (queries.Count == 0)
        {
            return new QueryAnalyticsSummary();
        }

        return new QueryAnalyticsSummary
        {
            TotalUniqueQueries = queries.Count,
            TotalSearches = queries.Sum(q => q.SearchCount),
            AverageResultsPerQuery = queries.Average(q => q.AverageResultCount),
            TopQueries = queries
                .OrderByDescending(q => q.SearchCount)
                .Take(10)
                .Select(q => new TopQuery(q.Query, q.SearchCount))
                .ToList(),
            LowPerformanceQueries = queries
                .Where(q => q.AverageResultCount < 2)
                .OrderByDescending(q => q.SearchCount)
                .Take(10)
                .Select(q => new LowPerformanceQuery(q.Query, q.AverageResultCount, q.SearchCount))
                .ToList()
        };
    }

    private HotelAnalyticsSummary AnalyzeHotels(List<HotelPopularityAnalytics> hotels)
    {
        if (hotels.Count == 0)
        {
            return new HotelAnalyticsSummary();
        }

        return new HotelAnalyticsSummary
        {
            TotalHotelsInResults = hotels.Count,
            TotalAppearances = hotels.Sum(h => h.AppearanceCount),
            TopHotels = hotels
                .OrderByDescending(h => h.AppearanceCount)
                .Take(10)
                .Select(h => new PopularHotel(h.HotelId, h.AppearanceCount))
                .ToList(),
            RecentlyTrendingHotels = hotels
                .Where(h => h.RecentTrendScore > 1.5) // Trending threshold
                .OrderByDescending(h => h.RecentTrendScore)
                .Take(10)
                .Select(h => new TrendingHotel(h.HotelId, h.RecentTrendScore))
                .ToList()
        };
    }

    private PerformanceAnalyticsSummary AnalyzePerformance(List<SearchQueryAnalytics> queries)
    {
        if (queries.Count == 0)
        {
            return new PerformanceAnalyticsSummary();
        }

        var allResponseTimes = queries.SelectMany(q => q.GetRecentResponseTimes()).ToList();
        allResponseTimes.Sort();

        return new PerformanceAnalyticsSummary
        {
            AverageResponseTime = TimeSpan.FromMilliseconds(
                allResponseTimes.Average(t => t.TotalMilliseconds)),
            MedianResponseTime = GetPercentile(allResponseTimes, 0.5),
            P95ResponseTime = GetPercentile(allResponseTimes, 0.95),
            SlowQueryCount = queries.Count(q => q.AverageResponseTime > TimeSpan.FromSeconds(1)),
            TotalSearches = queries.Sum(q => q.SearchCount)
        };
    }

    private TimeSpan GetPercentile(List<TimeSpan> sortedTimes, double percentile)
    {
        if (sortedTimes.Count == 0) return TimeSpan.Zero;
        
        var index = (int)((sortedTimes.Count - 1) * percentile);
        return sortedTimes[index];
    }
}

// Supporting analytics models
public sealed class SearchQueryAnalytics
{
    private readonly object _lock = new();
    private readonly Queue<(DateTime Time, TimeSpan Duration, int Results)> _recentSearches = new();
    private const int MaxRecentSearches = 100;

    public string Query { get; init; } = string.Empty;
    public int SearchCount { get; private set; }
    public DateTime LastSearched { get; private set; }
    public double AverageResultCount { get; private set; }
    public TimeSpan AverageResponseTime { get; private set; }

    public void RecordSearch(SearchAnalyticsEvent searchEvent)
    {
        lock (_lock)
        {
            SearchCount++;
            LastSearched = searchEvent.Timestamp;

            _recentSearches.Enqueue((searchEvent.Timestamp, searchEvent.ResponseTime, searchEvent.ResultCount));
            if (_recentSearches.Count > MaxRecentSearches)
            {
                _recentSearches.Dequeue();
            }

            RecalculateAverages();
        }
    }

    private void RecalculateAverages()
    {
        if (_recentSearches.Count == 0) return;

        AverageResultCount = _recentSearches.Average(s => s.Results);
        AverageResponseTime = TimeSpan.FromMilliseconds(
            _recentSearches.Average(s => s.Duration.TotalMilliseconds));
    }

    public List<TimeSpan> GetRecentResponseTimes()
    {
        lock (_lock)
        {
            return _recentSearches.Select(s => s.Duration).ToList();
        }
    }
}

public sealed class HotelPopularityAnalytics
{
    private readonly object _lock = new();
    private readonly Queue<(DateTime Time, string Query)> _recentAppearances = new();

    public int HotelId { get; init; }
    public int AppearanceCount { get; private set; }
    public DateTime LastSeen { get; private set; }
    public double RecentTrendScore { get; private set; }

    public void RecordAppearance(string query, DateTime timestamp)
    {
        lock (_lock)
        {
            AppearanceCount++;
            LastSeen = timestamp;

            _recentAppearances.Enqueue((timestamp, query));
            
            // Keep only last 30 days of appearances
            var cutoff = timestamp.AddDays(-30);
            while (_recentAppearances.Count > 0 && _recentAppearances.Peek().Time < cutoff)
            {
                _recentAppearances.Dequeue();
            }

            CalculateTrendScore();
        }
    }

    private void CalculateTrendScore()
    {
        if (_recentAppearances.Count < 2) 
        {
            RecentTrendScore = 1.0;
            return;
        }

        var now = DateTime.UtcNow;
        var recent = _recentAppearances.Where(a => a.Time > now.AddDays(-7)).Count();
        var previous = _recentAppearances.Where(a => a.Time <= now.AddDays(-7) && a.Time > now.AddDays(-14)).Count();

        RecentTrendScore = previous > 0 ? (double)recent / previous : recent > 0 ? 2.0 : 1.0;
    }
}

public sealed record SearchAnalyticsEvent
{
    public string Query { get; init; } = string.Empty;
    public List<Hotel> Results { get; init; } = new();
    public int ResultCount => Results.Count;
    public TimeSpan ResponseTime { get; init; }
    public DateTime Timestamp { get; init; } = DateTime.UtcNow;
}

// Report models
public sealed record SearchAnalyticsReport
{
    public TimeSpan Period { get; init; }
    public DateTime GeneratedAt { get; init; }
    public QueryAnalyticsSummary QueryAnalytics { get; init; } = new();
    public HotelAnalyticsSummary HotelAnalytics { get; init; } = new();
    public PerformanceAnalyticsSummary PerformanceAnalytics { get; init; } = new();
}

public sealed record QueryAnalyticsSummary
{
    public int TotalUniqueQueries { get; init; }
    public int TotalSearches { get; init; }
    public double AverageResultsPerQuery { get; init; }
    public List<TopQuery> TopQueries { get; init; } = new();
    public List<LowPerformanceQuery> LowPerformanceQueries { get; init; } = new();
}

public sealed record HotelAnalyticsSummary
{
    public int TotalHotelsInResults { get; init; }
    public int TotalAppearances { get; init; }
    public List<PopularHotel> TopHotels { get; init; } = new();
    public List<TrendingHotel> RecentlyTrendingHotels { get; init; } = new();
}

public sealed record PerformanceAnalyticsSummary
{
    public TimeSpan AverageResponseTime { get; init; }
    public TimeSpan MedianResponseTime { get; init; }
    public TimeSpan P95ResponseTime { get; init; }
    public int SlowQueryCount { get; init; }
    public int TotalSearches { get; init; }
}

public sealed record TopQuery(string Query, int SearchCount);
public sealed record LowPerformanceQuery(string Query, double AverageResults, int SearchCount);
public sealed record PopularHotel(int HotelId, int AppearanceCount);
public sealed record TrendingHotel(int HotelId, double TrendScore);
```

This advanced querying framework provides sophisticated filtering, efficient batch processing, custom distance functions, and comprehensive analytics for production search systems.
