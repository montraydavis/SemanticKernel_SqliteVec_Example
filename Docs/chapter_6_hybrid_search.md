# Chapter 6: Implementing Hybrid Search

## Understanding Reciprocal Rank Fusion (RRF)

### RRF Algorithm Fundamentals

Reciprocal Rank Fusion is a scientifically proven method for combining multiple ranked result lists. The algorithm assigns scores based on the rank position of items in each list, making it effective for merging keyword and vector search results.

**Core RRF Formula:**
```
RRF_Score = Σ(weight / (k + rank))
```

Where:
- `weight`: Importance weight for the search method (keyword vs vector)
- `k`: RRF constant (typically 60, prevents division by zero and balances high/low ranks)
- `rank`: Position in the ranked list (1-based)

### RRF Implementation

```csharp
namespace HotelSearch.Core.Algorithms;

public sealed class ReciprocalRankFusion
{
    private readonly SearchConfiguration _config;
    private readonly ILogger<ReciprocalRankFusion> _logger;

    public ReciprocalRankFusion(SearchConfiguration config, ILogger<ReciprocalRankFusion> logger)
    {
        _config = config;
        _logger = logger;
    }

    public List<RankedResult> FuseResults(
        List<Hotel> keywordResults,
        List<VectorSearchResult<Hotel>> vectorResults,
        SearchOptions options)
    {
        var rrfScores = new Dictionary<int, RRFScore>();

        ProcessKeywordResults(keywordResults, rrfScores);
        ProcessVectorResults(vectorResults, rrfScores);

        var results = rrfScores.Values
            .Where(score => PassesFilters(score.Hotel, options))
            .OrderByDescending(score => score.TotalScore)
            .Take(options.MaxResults)
            .Select(CreateRankedResult)
            .ToList();

        LogFusionResults(results);
        return results;
    }

    private void ProcessKeywordResults(List<Hotel> results, Dictionary<int, RRFScore> scores)
    {
        for (int i = 0; i < results.Count; i++)
        {
            var hotel = results[i];
            var rank = i + 1;
            var score = _config.KeywordWeight / (_config.RrfConstant + rank);
            
            UpdateOrCreateScore(scores, hotel, score: score, keywordRank: rank);
        }
    }

    private void ProcessVectorResults(List<VectorSearchResult<Hotel>> results, Dictionary<int, RRFScore> scores)
    {
        for (int i = 0; i < results.Count; i++)
        {
            var result = results[i];
            var rank = i + 1;
            var score = _config.VectorWeight / (_config.RrfConstant + rank);
            
            UpdateOrCreateScore(scores, result.Record, score: score, 
                vectorRank: rank, similarity: result.Score ?? 0.0);
        }
    }

    private static void UpdateOrCreateScore(
        Dictionary<int, RRFScore> scores, 
        Hotel hotel, 
        double score, 
        int? keywordRank = null, 
        int? vectorRank = null, 
        double similarity = 0.0)
    {
        if (scores.TryGetValue(hotel.HotelId, out var existing))
        {
            existing.TotalScore += score;
            existing.KeywordRank ??= keywordRank;
            existing.VectorRank ??= vectorRank;
            existing.KeywordScore += keywordRank.HasValue ? score : 0;
            existing.VectorScore += vectorRank.HasValue ? score : 0;
            existing.VectorSimilarity = Math.Max(existing.VectorSimilarity, similarity);
        }
        else
        {
            scores[hotel.HotelId] = new RRFScore
            {
                Hotel = hotel,
                KeywordRank = keywordRank,
                VectorRank = vectorRank,
                KeywordScore = keywordRank.HasValue ? score : 0,
                VectorScore = vectorRank.HasValue ? score : 0,
                TotalScore = score,
                VectorSimilarity = similarity
            };
        }
    }

    private static bool PassesFilters(Hotel hotel, SearchOptions options) =>
        (options.MinRating is null || hotel.Rating >= options.MinRating) &&
        (string.IsNullOrEmpty(options.Location) || 
         hotel.Location?.Contains(options.Location, StringComparison.OrdinalIgnoreCase) == true);

    private static RankedResult CreateRankedResult(RRFScore score) => new()
    {
        Hotel = score.Hotel,
        RRFScore = score.TotalScore,
        KeywordRank = score.KeywordRank,
        VectorRank = score.VectorRank,
        KeywordScore = score.KeywordScore,
        VectorScore = score.VectorScore,
        VectorSimilarity = score.VectorSimilarity,
        SearchType = GetSearchType(score.KeywordRank, score.VectorRank)
    };

    private static string GetSearchType(int? keywordRank, int? vectorRank) =>
        (keywordRank.HasValue, vectorRank.HasValue) switch
        {
            (true, true) => "Hybrid",
            (true, false) => "Keyword",
            (false, true) => "Vector",
            _ => "Unknown"
        };

    private void LogFusionResults(List<RankedResult> results)
    {
        if (!_logger.IsEnabled(LogLevel.Debug)) return;

        _logger.LogDebug("RRF Fusion Results (Top 3):");
        foreach (var result in results.Take(3))
        {
            _logger.LogDebug(
                "  {HotelName}: RRF={RRFScore:F4}, KW_Rank={KeywordRank}, Vec_Rank={VectorRank}, Type={SearchType}",
                result.Hotel.HotelName,
                result.RRFScore,
                result.KeywordRank?.ToString() ?? "N/A",
                result.VectorRank?.ToString() ?? "N/A",
                result.SearchType);
        }
    }
}
```

### RRF Supporting Models

```csharp
public sealed class RRFScore
{
    public Hotel Hotel { get; set; } = null!;
    public int? KeywordRank { get; set; }
    public int? VectorRank { get; set; }
    public double KeywordScore { get; set; }
    public double VectorScore { get; set; }
    public double TotalScore { get; set; }
    public double VectorSimilarity { get; set; }
}

public sealed class RankedResult
{
    public Hotel Hotel { get; set; } = null!;
    public double RRFScore { get; set; }
    public int? KeywordRank { get; set; }
    public int? VectorRank { get; set; }
    public double KeywordScore { get; set; }
    public double VectorScore { get; set; }
    public double VectorSimilarity { get; set; }
    public string SearchType { get; set; } = string.Empty;
}
```

## Keyword Search with FTS5

### FTS5 Query Builder

```csharp
namespace HotelSearch.Core.Search;

public sealed class KeywordSearchEngine
{
    private readonly string _connectionString;
    private readonly ILogger<KeywordSearchEngine> _logger;

    public KeywordSearchEngine(string connectionString, ILogger<KeywordSearchEngine> logger)
    {
        _connectionString = connectionString;
        _logger = logger;
    }

    public async Task<List<Hotel>> SearchAsync(
        string query, 
        int limit, 
        CancellationToken cancellationToken = default)
    {
        var sanitizedQuery = SanitizeQuery(query);
        var ftsQuery = BuildFtsQuery(sanitizedQuery);
        
        return await ExecuteKeywordSearchAsync(ftsQuery, limit, cancellationToken);
    }

    private string SanitizeQuery(string query)
    {
        // Remove FTS5 special characters and prepare for search
        return query
            .Replace("\"", "\"\"")
            .Replace("'", "''")
            .Replace("*", "")
            .Replace(":", "")
            .Trim();
    }

    private string BuildFtsQuery(string sanitizedQuery)
    {
        var terms = sanitizedQuery.Split(' ', StringSplitOptions.RemoveEmptyEntries);
        
        if (terms.Length == 0)
            return sanitizedQuery;

        // Build a query that searches for:
        // 1. Exact phrase (highest priority)
        // 2. All terms present (medium priority) 
        // 3. Any terms present (lowest priority)
        var exactPhrase = $"\"{string.Join(" ", terms)}\"";
        var allTerms = string.Join(" AND ", terms);
        var anyTerms = string.Join(" OR ", terms);

        return $"({exactPhrase}) OR ({allTerms}) OR ({anyTerms})";
    }

    private async Task<List<Hotel>> ExecuteKeywordSearchAsync(
        string ftsQuery, 
        int limit, 
        CancellationToken cancellationToken)
    {
        const string sql = """
            SELECT HotelId, HotelName, Description, Location, Rating, PricePerNight, LastUpdated,
                   rank as SearchScore
            FROM hotels_fts
            WHERE hotels_fts MATCH @query
            ORDER BY rank
            LIMIT @limit
            """;

        using var connection = new SqliteConnection(_connectionString);
        await connection.OpenAsync(cancellationToken);
        
        using var command = connection.CreateCommand();
        command.CommandText = sql;
        command.Parameters.AddWithValue("@query", ftsQuery);
        command.Parameters.AddWithValue("@limit", limit);
        
        var results = new List<Hotel>();
        using var reader = await command.ExecuteReaderAsync(cancellationToken);
        
        while (await reader.ReadAsync(cancellationToken))
        {
            results.Add(new Hotel
            {
                HotelId = reader.GetInt32("HotelId"),
                HotelName = reader.GetString("HotelName"),
                Description = reader.GetString("Description"),
                Location = reader.GetString("Location"),
                Rating = reader.GetDouble("Rating"),
                PricePerNight = reader.GetDecimal("PricePerNight"),
                LastUpdated = DateTime.Parse(reader.GetString("LastUpdated"))
            });
        }
        
        _logger.LogDebug("Keyword search returned {Count} results for query: {Query}", 
            results.Count, ftsQuery);
        
        return results;
    }
}
```

### Advanced FTS5 Features

```csharp
public sealed class AdvancedKeywordSearch
{
    private readonly string _connectionString;

    public AdvancedKeywordSearch(string connectionString)
    {
        _connectionString = connectionString;
    }

    public async Task<List<Hotel>> SearchWithHighlightsAsync(
        string query, 
        int limit, 
        CancellationToken cancellationToken = default)
    {
        const string sql = """
            SELECT HotelId, HotelName, Description, Location, Rating, PricePerNight, LastUpdated,
                   highlight(hotels_fts, 1, '<mark>', '</mark>') as HighlightedName,
                   highlight(hotels_fts, 2, '<mark>', '</mark>') as HighlightedDescription,
                   rank as SearchScore
            FROM hotels_fts
            WHERE hotels_fts MATCH @query
            ORDER BY rank
            LIMIT @limit
            """;

        using var connection = new SqliteConnection(_connectionString);
        await connection.OpenAsync(cancellationToken);
        
        using var command = connection.CreateCommand();
        command.CommandText = sql;
        command.Parameters.AddWithValue("@query", query);
        command.Parameters.AddWithValue("@limit", limit);
        
        var results = new List<Hotel>();
        using var reader = await command.ExecuteReaderAsync(cancellationToken);
        
        while (await reader.ReadAsync(cancellationToken))
        {
            var hotel = new Hotel
            {
                HotelId = reader.GetInt32("HotelId"),
                HotelName = reader.GetString("HotelName"),
                Description = reader.GetString("Description"),
                Location = reader.GetString("Location"),
                Rating = reader.GetDouble("Rating"),
                PricePerNight = reader.GetDecimal("PricePerNight"),
                LastUpdated = DateTime.Parse(reader.GetString("LastUpdated"))
            };
            
            // Store highlights in a separate property or handle differently
            results.Add(hotel);
        }
        
        return results;
    }

    public async Task<List<Hotel>> SearchWithSnippetsAsync(
        string query, 
        int limit, 
        CancellationToken cancellationToken = default)
    {
        const string sql = """
            SELECT HotelId, HotelName, Description, Location, Rating, PricePerNight, LastUpdated,
                   snippet(hotels_fts, 2, '<b>', '</b>', '...', 32) as Snippet,
                   rank as SearchScore
            FROM hotels_fts
            WHERE hotels_fts MATCH @query
            ORDER BY rank
            LIMIT @limit
            """;

        using var connection = new SqliteConnection(_connectionString);
        await connection.OpenAsync(cancellationToken);
        
        using var command = connection.CreateCommand();
        command.CommandText = sql;
        command.Parameters.AddWithValue("@query", query);
        command.Parameters.AddWithValue("@limit", limit);
        
        var results = new List<Hotel>();
        using var reader = await command.ExecuteReaderAsync(cancellationToken);
        
        while (await reader.ReadAsync(cancellationToken))
        {
            results.Add(new Hotel
            {
                HotelId = reader.GetInt32("HotelId"),
                HotelName = reader.GetString("HotelName"),
                Description = reader.GetString("Description"),
                Location = reader.GetString("Location"),
                Rating = reader.GetDouble("Rating"),
                PricePerNight = reader.GetDecimal("PricePerNight"),
                LastUpdated = DateTime.Parse(reader.GetString("LastUpdated"))
            });
        }
        
        return results;
    }
}
```

## Vector Search Implementation

### Vector Search Engine

```csharp
namespace HotelSearch.Core.Search;

public sealed class VectorSearchEngine
{
    private readonly SqliteCollection<int, Hotel> _vectorCollection;
    private readonly IEmbeddingGenerator<string, Embedding<float>> _embeddingService;
    private readonly ILogger<VectorSearchEngine> _logger;

    public VectorSearchEngine(
        SqliteCollection<int, Hotel> vectorCollection,
        IEmbeddingGenerator<string, Embedding<float>> embeddingService,
        ILogger<VectorSearchEngine> logger)
    {
        _vectorCollection = vectorCollection;
        _embeddingService = embeddingService;
        _logger = logger;
    }

    public async Task<List<VectorSearchResult<Hotel>>> SearchAsync(
        string query, 
        int limit, 
        CancellationToken cancellationToken = default)
    {
        var queryEmbedding = await GenerateQueryEmbeddingAsync(query, cancellationToken);
        return await PerformVectorSearchAsync(queryEmbedding, limit, cancellationToken);
    }

    public async Task<List<VectorSearchResult<Hotel>>> SearchByEmbeddingAsync(
        Embedding<float> queryEmbedding, 
        int limit, 
        CancellationToken cancellationToken = default)
    {
        return await PerformVectorSearchAsync(queryEmbedding, limit, cancellationToken);
    }

    private async Task<Embedding<float>> GenerateQueryEmbeddingAsync(
        string query, 
        CancellationToken cancellationToken)
    {
        try
        {
            return await _embeddingService.GenerateAsync(query, cancellationToken);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to generate embedding for query: {Query}", query);
            throw new EmbeddingGenerationException(query, ex);
        }
    }

    private async Task<List<VectorSearchResult<Hotel>>> PerformVectorSearchAsync(
        Embedding<float> queryEmbedding,
        int limit,
        CancellationToken cancellationToken)
    {
        var searchOptions = new VectorSearchOptions<Hotel>
        {
            VectorProperty = static v => v.DescriptionEmbedding
        };
        
        var results = new List<VectorSearchResult<Hotel>>();

        await foreach (var result in _vectorCollection
            .SearchAsync(queryEmbedding, limit, searchOptions)
            .WithCancellation(cancellationToken))
        {
            results.Add(result);
        }
        
        _logger.LogDebug("Vector search returned {Count} results", results.Count);
        
        return results;
    }
}
```

### Multi-Vector Search Strategies

```csharp
public sealed class MultiVectorSearchEngine
{
    private readonly SqliteCollection<int, Hotel> _vectorCollection;
    private readonly IEmbeddingGenerator<string, Embedding<float>> _embeddingService;

    public MultiVectorSearchEngine(
        SqliteCollection<int, Hotel> vectorCollection,
        IEmbeddingGenerator<string, Embedding<float>> embeddingService)
    {
        _vectorCollection = vectorCollection;
        _embeddingService = embeddingService;
    }

    public async Task<List<VectorSearchResult<Hotel>>> SearchMultipleFieldsAsync(
        string query,
        int limit,
        CancellationToken cancellationToken = default)
    {
        var queryEmbedding = await _embeddingService.GenerateAsync(query, cancellationToken);

        // Search both name and description embeddings
        var nameResults = await SearchFieldAsync(queryEmbedding, v => v.NameEmbedding, limit, cancellationToken);
        var descriptionResults = await SearchFieldAsync(queryEmbedding, v => v.DescriptionEmbedding, limit, cancellationToken);

        // Combine and deduplicate results
        var combinedResults = CombineMultiFieldResults(nameResults, descriptionResults, limit);
        
        return combinedResults;
    }

    private async Task<List<VectorSearchResult<Hotel>>> SearchFieldAsync(
        Embedding<float> queryEmbedding,
        Expression<Func<Hotel, ReadOnlyMemory<float>?>> vectorProperty,
        int limit,
        CancellationToken cancellationToken)
    {
        var searchOptions = new VectorSearchOptions<Hotel>
        {
            VectorProperty = vectorProperty
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

    private List<VectorSearchResult<Hotel>> CombineMultiFieldResults(
        List<VectorSearchResult<Hotel>> nameResults,
        List<VectorSearchResult<Hotel>> descriptionResults,
        int limit)
    {
        var combinedScores = new Dictionary<int, (Hotel Hotel, double MaxScore)>();

        // Process name results
        foreach (var result in nameResults)
        {
            var score = result.Score ?? 0.0;
            combinedScores[result.Record.HotelId] = (result.Record, score);
        }

        // Process description results, keeping the higher score
        foreach (var result in descriptionResults)
        {
            var score = result.Score ?? 0.0;
            if (combinedScores.TryGetValue(result.Record.HotelId, out var existing))
            {
                if (score > existing.MaxScore)
                {
                    combinedScores[result.Record.HotelId] = (result.Record, score);
                }
            }
            else
            {
                combinedScores[result.Record.HotelId] = (result.Record, score);
            }
        }

        return combinedScores.Values
            .OrderByDescending(x => x.MaxScore)
            .Take(limit)
            .Select(x => new VectorSearchResult<Hotel>(x.Hotel, x.MaxScore))
            .ToList();
    }
}
```

## Combining Search Results

### Hybrid Search Orchestrator

```csharp
namespace HotelSearch.Core.Services;

public sealed partial class HotelSearchService
{
    private async Task<(List<Hotel> keyword, List<VectorSearchResult<Hotel>> vector)> 
        ExecuteParallelSearchAsync(
            string query, 
            SearchOptions options, 
            CancellationToken cancellationToken)
    {
        var searchLimit = CalculateSearchLimit(options);
        
        // Execute searches in parallel for better performance
        var keywordTask = _keywordSearchEngine.SearchAsync(query, searchLimit, cancellationToken);
        var vectorTask = _vectorSearchEngine.SearchAsync(query, searchLimit, cancellationToken);
        
        await Task.WhenAll(keywordTask, vectorTask);

        return (await keywordTask, await vectorTask);
    }

    private int CalculateSearchLimit(SearchOptions options)
    {
        // Get more results than needed for better fusion quality
        return Math.Max(
            options.MaxResults * _config.SearchMultiplier, 
            _config.MinSearchLimit);
    }

    private List<RankedResult> ApplyReciprocalRankFusion(
        List<Hotel> keywordResults,
        List<VectorSearchResult<Hotel>> vectorResults,
        SearchOptions options)
    {
        var rrf = new ReciprocalRankFusion(_config, _logger);
        return rrf.FuseResults(keywordResults, vectorResults, options);
    }

    private SearchMetrics CreateSearchMetrics(
        TimeSpan duration,
        int keywordCount,
        int vectorCount,
        int hybridCount,
        string query)
    {
        return new SearchMetrics
        {
            SearchDuration = duration,
            KeywordResults = keywordCount,
            VectorResults = vectorCount,
            HybridResults = hybridCount,
            QueryType = DetermineQueryType(query)
        };
    }

    private string DetermineQueryType(string query)
    {
        var words = query.Split(' ', StringSplitOptions.RemoveEmptyEntries);
        return words.Length switch
        {
            1 => "Single-term",
            2 => "Two-term",
            3 to 5 => "Short-phrase",
            _ => "Long-phrase"
        };
    }
}
```

### Alternative Fusion Strategies

```csharp
namespace HotelSearch.Core.Algorithms;

public interface IResultFusionStrategy
{
    List<RankedResult> FuseResults(
        List<Hotel> keywordResults,
        List<VectorSearchResult<Hotel>> vectorResults,
        SearchOptions options);
}

public sealed class WeightedAverageFusion : IResultFusionStrategy
{
    private readonly SearchConfiguration _config;

    public WeightedAverageFusion(SearchConfiguration config)
    {
        _config = config;
    }

    public List<RankedResult> FuseResults(
        List<Hotel> keywordResults,
        List<VectorSearchResult<Hotel>> vectorResults,
        SearchOptions options)
    {
        var combinedScores = new Dictionary<int, (Hotel Hotel, double Score, string Type)>();

        // Normalize keyword scores (rank-based)
        for (int i = 0; i < keywordResults.Count; i++)
        {
            var hotel = keywordResults[i];
            var normalizedScore = (keywordResults.Count - i) / (double)keywordResults.Count;
            var weightedScore = normalizedScore * _config.KeywordWeight;
            
            combinedScores[hotel.HotelId] = (hotel, weightedScore, "Keyword");
        }

        // Add vector scores
        foreach (var result in vectorResults)
        {
            var vectorScore = (result.Score ?? 0.0) * _config.VectorWeight;
            
            if (combinedScores.TryGetValue(result.Record.HotelId, out var existing))
            {
                combinedScores[result.Record.HotelId] = (
                    existing.Hotel, 
                    existing.Score + vectorScore, 
                    "Hybrid");
            }
            else
            {
                combinedScores[result.Record.HotelId] = (result.Record, vectorScore, "Vector");
            }
        }

        return combinedScores.Values
            .OrderByDescending(x => x.Score)
            .Take(options.MaxResults)
            .Select(x => new RankedResult
            {
                Hotel = x.Hotel,
                RRFScore = x.Score,
                SearchType = x.Type
            })
            .ToList();
    }
}

public sealed class LinearCombinationFusion : IResultFusionStrategy
{
    private readonly SearchConfiguration _config;

    public LinearCombinationFusion(SearchConfiguration config)
    {
        _config = config;
    }

    public List<RankedResult> FuseResults(
        List<Hotel> keywordResults,
        List<VectorSearchResult<Hotel>> vectorResults,
        SearchOptions options)
    {
        var allHotels = GetAllUniqueHotels(keywordResults, vectorResults);
        var rankedResults = new List<RankedResult>();

        foreach (var hotel in allHotels)
        {
            var keywordScore = GetKeywordScore(hotel, keywordResults);
            var vectorScore = GetVectorScore(hotel, vectorResults);
            
            var combinedScore = (_config.KeywordWeight * keywordScore) + 
                               (_config.VectorWeight * vectorScore);

            rankedResults.Add(new RankedResult
            {
                Hotel = hotel,
                RRFScore = combinedScore,
                KeywordScore = keywordScore,
                VectorScore = vectorScore,
                SearchType = DetermineSearchType(keywordScore, vectorScore)
            });
        }

        return rankedResults
            .OrderByDescending(r => r.RRFScore)
            .Take(options.MaxResults)
            .ToList();
    }

    private List<Hotel> GetAllUniqueHotels(
        List<Hotel> keywordResults,
        List<VectorSearchResult<Hotel>> vectorResults)
    {
        var hotels = new Dictionary<int, Hotel>();
        
        foreach (var hotel in keywordResults)
        {
            hotels[hotel.HotelId] = hotel;
        }
        
        foreach (var result in vectorResults)
        {
            hotels[result.Record.HotelId] = result.Record;
        }
        
        return hotels.Values.ToList();
    }

    private double GetKeywordScore(Hotel hotel, List<Hotel> keywordResults)
    {
        var index = keywordResults.FindIndex(h => h.HotelId == hotel.HotelId);
        if (index == -1) return 0.0;
        
        return (keywordResults.Count - index) / (double)keywordResults.Count;
    }

    private double GetVectorScore(Hotel hotel, List<VectorSearchResult<Hotel>> vectorResults)
    {
        var result = vectorResults.FirstOrDefault(r => r.Record.HotelId == hotel.HotelId);
        return result?.Score ?? 0.0;
    }

    private string DetermineSearchType(double keywordScore, double vectorScore)
    {
        return (keywordScore > 0, vectorScore > 0) switch
        {
            (true, true) => "Hybrid",
            (true, false) => "Keyword",
            (false, true) => "Vector",
            _ => "Unknown"
        };
    }
}
```

This hybrid search implementation provides a robust foundation for combining keyword and vector search results using scientifically proven fusion algorithms, while maintaining flexibility for different search strategies and configurations.
