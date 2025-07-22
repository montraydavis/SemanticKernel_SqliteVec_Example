# Chapter 5: Database Schema and Initialization

## Creating SQLiteVec Collections

### Collection Setup and Configuration

SQLiteVec collections automatically handle the vector storage schema based on your data model attributes. The collection creation is declarative and type-safe:

```csharp
using Microsoft.SemanticKernel.Connectors.SqliteVec;
using HotelSearch.Core.Models;

namespace HotelSearch.Core.Data;

public sealed class DatabaseInitializer
{
    private readonly SqliteCollection<int, Hotel> _hotelCollection;
    private readonly string _connectionString;
    private readonly ILogger<DatabaseInitializer> _logger;

    public DatabaseInitializer(
        SqliteCollection<int, Hotel> hotelCollection,
        string connectionString,
        ILogger<DatabaseInitializer> logger)
    {
        _hotelCollection = hotelCollection;
        _connectionString = connectionString;
        _logger = logger;
    }

    public async Task InitializeAsync(CancellationToken cancellationToken = default)
    {
        await CreateCollectionAsync(cancellationToken);
        await CreateIndexesAsync(cancellationToken);
        await VerifySchemaAsync(cancellationToken);
    }

    private async Task CreateCollectionAsync(CancellationToken cancellationToken)
    {
        _logger.LogInformation("Creating SQLiteVec collection...");
        
        await _hotelCollection.EnsureCollectionExistsAsync();
        
        _logger.LogInformation("SQLiteVec collection created successfully");
    }
}
```

### Collection Schema Verification

**Verify the generated schema matches expectations:**

```csharp
public async Task VerifySchemaAsync(CancellationToken cancellationToken = default)
{
    using var connection = new SqliteConnection(_connectionString);
    await connection.OpenAsync(cancellationToken);

    // Check vector tables exist
    var vectorTables = await GetTableInfoAsync(connection, "hotels");
    LogTableSchema("Vector Collection", vectorTables);

    // Verify vector dimensions
    await VerifyVectorDimensionsAsync(connection);
}

private async Task<List<TableColumn>> GetTableInfoAsync(
    SqliteConnection connection, 
    string tableName)
{
    var command = connection.CreateCommand();
    command.CommandText = "PRAGMA table_info(@tableName)";
    command.Parameters.AddWithValue("@tableName", tableName);

    var columns = new List<TableColumn>();
    using var reader = await command.ExecuteReaderAsync();
    
    while (await reader.ReadAsync())
    {
        columns.Add(new TableColumn(
            Name: reader.GetString("name"),
            Type: reader.GetString("type"),
            NotNull: reader.GetBoolean("notnull"),
            DefaultValue: reader.IsDBNull("dflt_value") ? null : reader.GetString("dflt_value"),
            PrimaryKey: reader.GetBoolean("pk")));
    }

    return columns;
}

private record TableColumn(
    string Name,
    string Type,
    bool NotNull,
    string? DefaultValue,
    bool PrimaryKey);
```

### Multiple Collection Management

**Managing multiple entity types:**

```csharp
public sealed class CollectionManager
{
    private readonly string _connectionString;
    private readonly ILogger<CollectionManager> _logger;

    public CollectionManager(string connectionString, ILogger<CollectionManager> logger)
    {
        _connectionString = connectionString;
        _logger = logger;
    }

    public async Task InitializeAllCollectionsAsync(CancellationToken cancellationToken = default)
    {
        var collections = new[]
        {
            ("hotels", typeof(Hotel)),
            ("reviews", typeof(Review)),
            ("amenities", typeof(Amenity))
        };

        foreach (var (name, type) in collections)
        {
            await InitializeCollectionAsync(name, type, cancellationToken);
        }
    }

    private async Task InitializeCollectionAsync(
        string collectionName,
        Type entityType,
        CancellationToken cancellationToken)
    {
        _logger.LogInformation("Initializing collection: {CollectionName}", collectionName);

        // Use reflection to create the appropriate collection type
        var collectionType = typeof(SqliteCollection<,>).MakeGenericType(GetKeyType(entityType), entityType);
        var collection = Activator.CreateInstance(collectionType, _connectionString, collectionName);

        var ensureMethod = collectionType.GetMethod("EnsureCollectionExistsAsync");
        await (Task)ensureMethod!.Invoke(collection, Array.Empty<object>())!;

        _logger.LogInformation("Collection initialized: {CollectionName}", collectionName);
    }

    private static Type GetKeyType(Type entityType)
    {
        var keyProperty = entityType.GetProperties()
            .FirstOrDefault(p => p.GetCustomAttribute<VectorStoreKeyAttribute>() != null);

        return keyProperty?.PropertyType ?? typeof(int);
    }
}
```

## FTS5 Table Configuration

### Creating Full-Text Search Tables

FTS5 provides the keyword search component of hybrid search. The schema should mirror the vector collection data:

```csharp
public sealed class FtsTableManager
{
    private readonly string _connectionString;
    private readonly ILogger<FtsTableManager> _logger;

    public FtsTableManager(string connectionString, ILogger<FtsTableManager> logger)
    {
        _connectionString = connectionString;
        _logger = logger;
    }

    public async Task CreateHotelsFtsTableAsync(CancellationToken cancellationToken = default)
    {
        const string createTableSql = """
            CREATE VIRTUAL TABLE IF NOT EXISTS hotels_fts USING fts5(
                HotelId UNINDEXED,
                HotelName,
                Description,
                Location,
                Rating UNINDEXED,
                PricePerNight UNINDEXED,
                LastUpdated UNINDEXED,
                content='hotels',
                content_rowid='HotelId'
            )
            """;

        await ExecuteSqlAsync(createTableSql, cancellationToken);
        await CreateFtsTriggersAsync(cancellationToken);
    }

    private async Task CreateFtsTriggersAsync(CancellationToken cancellationToken)
    {
        var triggers = new[]
        {
            // Insert trigger
            """
            CREATE TRIGGER IF NOT EXISTS hotels_fts_insert AFTER INSERT ON hotels BEGIN
                INSERT INTO hotels_fts(HotelId, HotelName, Description, Location, Rating, PricePerNight, LastUpdated)
                VALUES (new.HotelId, new.HotelName, new.Description, new.Location, new.Rating, new.PricePerNight, new.LastUpdated);
            END
            """,
            
            // Update trigger
            """
            CREATE TRIGGER IF NOT EXISTS hotels_fts_update AFTER UPDATE ON hotels BEGIN
                UPDATE hotels_fts SET 
                    HotelName = new.HotelName,
                    Description = new.Description,
                    Location = new.Location,
                    Rating = new.Rating,
                    PricePerNight = new.PricePerNight,
                    LastUpdated = new.LastUpdated
                WHERE HotelId = new.HotelId;
            END
            """,
            
            // Delete trigger
            """
            CREATE TRIGGER IF NOT EXISTS hotels_fts_delete AFTER DELETE ON hotels BEGIN
                DELETE FROM hotels_fts WHERE HotelId = old.HotelId;
            END
            """
        };

        foreach (var trigger in triggers)
        {
            await ExecuteSqlAsync(trigger, cancellationToken);
        }
    }

    private async Task ExecuteSqlAsync(string sql, CancellationToken cancellationToken)
    {
        using var connection = new SqliteConnection(_connectionString);
        await connection.OpenAsync(cancellationToken);
        
        using var command = connection.CreateCommand();
        command.CommandText = sql;
        await command.ExecuteNonQueryAsync(cancellationToken);
    }
}
```

### Advanced FTS5 Configuration

**Customized FTS5 configuration for better search:**

```csharp
public async Task CreateAdvancedFtsTableAsync(CancellationToken cancellationToken = default)
{
    const string createTableSql = """
        CREATE VIRTUAL TABLE IF NOT EXISTS hotels_fts USING fts5(
            HotelId UNINDEXED,
            HotelName,
            Description,
            Location,
            Rating UNINDEXED,
            PricePerNight UNINDEXED,
            LastUpdated UNINDEXED,
            tokenize='porter unicode61 remove_diacritics 1',
            prefix='2 3 4'
        )
        """;

    await ExecuteSqlAsync(createTableSql, cancellationToken);
    await ConfigureFtsRankingAsync(cancellationToken);
}

private async Task ConfigureFtsRankingAsync(CancellationToken cancellationToken)
{
    // Create custom ranking function for FTS5
    const string rankingSql = """
        INSERT INTO hotels_fts(hotels_fts, rank) VALUES('rank', 'bm25(2.0, 1.0, 0.5)');
        """;

    await ExecuteSqlAsync(rankingSql, cancellationToken);
}
```

### FTS5 Query Optimization

**Optimized FTS5 query methods:**

```csharp
public sealed class FtsQueryBuilder
{
    public static string BuildHotelSearchQuery(string query, SearchOptions options)
    {
        var sanitizedQuery = SanitizeQuery(query);
        var whereClause = BuildWhereClause(options);
        
        return $"""
            SELECT HotelId, HotelName, Description, Location, Rating, 
                   rank as SearchScore
            FROM hotels_fts
            WHERE hotels_fts MATCH @query
            {whereClause}
            ORDER BY rank
            LIMIT @limit
            """;
    }

    private static string SanitizeQuery(string query)
    {
        // Remove FTS5 special characters and escape quotes
        return query
            .Replace("\"", "\"\"")
            .Replace("'", "''")
            .Trim();
    }

    private static string BuildWhereClause(SearchOptions options)
    {
        var conditions = new List<string>();

        if (options.MinRating.HasValue)
        {
            conditions.Add("AND Rating >= @minRating");
        }

        if (!string.IsNullOrWhiteSpace(options.Location))
        {
            conditions.Add("AND Location LIKE @location");
        }

        return string.Join(" ", conditions);
    }
}
```

## Data Seeding and Migration Patterns

### Initial Data Seeding

**Structured approach to seeding sample data:**

```csharp
public sealed class HotelDataSeeder
{
    private readonly SqliteCollection<int, Hotel> _vectorCollection;
    private readonly IEmbeddingGenerator<string, Embedding<float>> _embeddingService;
    private readonly string _connectionString;
    private readonly ILogger<HotelDataSeeder> _logger;

    public HotelDataSeeder(
        SqliteCollection<int, Hotel> vectorCollection,
        IEmbeddingGenerator<string, Embedding<float>> embeddingService,
        string connectionString,
        ILogger<HotelDataSeeder> logger)
    {
        _vectorCollection = vectorCollection;
        _embeddingService = embeddingService;
        _connectionString = connectionString;
        _logger = logger;
    }

    public async Task SeedAsync(CancellationToken cancellationToken = default)
    {
        var sampleHotels = GetSampleHotels();
        
        _logger.LogInformation("Seeding {Count} sample hotels...", sampleHotels.Length);

        await GenerateEmbeddingsAsync(sampleHotels, cancellationToken);
        await InsertIntoVectorCollectionAsync(sampleHotels, cancellationToken);
        await InsertIntoFtsTableAsync(sampleHotels, cancellationToken);

        _logger.LogInformation("Seeding completed successfully");
    }

    private static Hotel[] GetSampleHotels()
    {
        return new[]
        {
            new Hotel
            {
                HotelId = 1,
                HotelName = "Grand Luxury Resort",
                Description = "Luxury beachfront resort with world-class spa, multiple dining options, and pristine ocean views",
                Location = "Miami Beach, Florida",
                Rating = 4.8,
                PricePerNight = 450.00m,
                LastUpdated = DateTime.UtcNow
            },
            new Hotel
            {
                HotelId = 2,
                HotelName = "Mountain View Lodge",
                Description = "Cozy mountain retreat featuring hiking trails, scenic overlooks, and rustic luxury accommodations",
                Location = "Aspen, Colorado",
                Rating = 4.5,
                PricePerNight = 320.00m,
                LastUpdated = DateTime.UtcNow
            },
            new Hotel
            {
                HotelId = 3,
                HotelName = "Urban Business Hotel",
                Description = "Modern downtown hotel with state-of-the-art conference facilities and convenient city access",
                Location = "Manhattan, New York",
                Rating = 4.2,
                PricePerNight = 280.00m,
                LastUpdated = DateTime.UtcNow
            },
            new Hotel
            {
                HotelId = 4,
                HotelName = "Seaside Wellness Retreat",
                Description = "Holistic wellness destination offering spa treatments, meditation gardens, and organic cuisine",
                Location = "Big Sur, California",
                Rating = 4.7,
                PricePerNight = 380.00m,
                LastUpdated = DateTime.UtcNow
            },
            new Hotel
            {
                HotelId = 5,
                HotelName = "Historic Boutique Inn",
                Description = "Charming 19th-century inn with antique furnishings, local artisan touches, and authentic regional character",
                Location = "Stowe, Vermont",
                Rating = 4.3,
                PricePerNight = 220.00m,
                LastUpdated = DateTime.UtcNow
            }
        };
    }

    private async Task GenerateEmbeddingsAsync(Hotel[] hotels, CancellationToken cancellationToken)
    {
        var embeddingTasks = hotels.Select(async hotel =>
        {
            var combinedText = $"{hotel.HotelName}. {hotel.Description}. Located in {hotel.Location}";
            var embedding = await _embeddingService.GenerateAsync(combinedText, cancellationToken);
            hotel.DescriptionEmbedding = embedding.Vector;
        });

        await Task.WhenAll(embeddingTasks);
    }

    private async Task InsertIntoVectorCollectionAsync(Hotel[] hotels, CancellationToken cancellationToken)
    {
        foreach (var hotel in hotels)
        {
            await _vectorCollection.UpsertAsync(hotel, cancellationToken);
        }
    }

    private async Task InsertIntoFtsTableAsync(Hotel[] hotels, CancellationToken cancellationToken)
    {
        using var connection = new SqliteConnection(_connectionString);
        await connection.OpenAsync(cancellationToken);

        const string insertSql = """
            INSERT OR REPLACE INTO hotels_fts 
            (HotelId, HotelName, Description, Location, Rating, PricePerNight, LastUpdated)
            VALUES (@id, @name, @description, @location, @rating, @price, @updated)
            """;

        foreach (var hotel in hotels)
        {
            using var command = connection.CreateCommand();
            command.CommandText = insertSql;
            command.Parameters.AddWithValue("@id", hotel.HotelId);
            command.Parameters.AddWithValue("@name", hotel.HotelName);
            command.Parameters.AddWithValue("@description", hotel.Description);
            command.Parameters.AddWithValue("@location", hotel.Location);
            command.Parameters.AddWithValue("@rating", hotel.Rating);
            command.Parameters.AddWithValue("@price", hotel.PricePerNight);
            command.Parameters.AddWithValue("@updated", hotel.LastUpdated);
            
            await command.ExecuteNonQueryAsync(cancellationToken);
        }
    }
}
```

### Database Migration System

**Version-based migration framework:**

```csharp
public sealed class DatabaseMigrator
{
    private readonly string _connectionString;
    private readonly ILogger<DatabaseMigrator> _logger;

    public DatabaseMigrator(string connectionString, ILogger<DatabaseMigrator> logger)
    {
        _connectionString = connectionString;
        _logger = logger;
    }

    public async Task MigrateAsync(CancellationToken cancellationToken = default)
    {
        await EnsureMigrationTableAsync(cancellationToken);
        
        var currentVersion = await GetCurrentVersionAsync(cancellationToken);
        var migrations = GetMigrations().Where(m => m.Version > currentVersion).OrderBy(m => m.Version);

        foreach (var migration in migrations)
        {
            await ExecuteMigrationAsync(migration, cancellationToken);
        }
    }

    private async Task EnsureMigrationTableAsync(CancellationToken cancellationToken)
    {
        const string createTableSql = """
            CREATE TABLE IF NOT EXISTS __migrations (
                version INTEGER PRIMARY KEY,
                name TEXT NOT NULL,
                executed_at TEXT NOT NULL
            )
            """;

        await ExecuteSqlAsync(createTableSql, cancellationToken);
    }

    private async Task<int> GetCurrentVersionAsync(CancellationToken cancellationToken)
    {
        const string sql = "SELECT COALESCE(MAX(version), 0) FROM __migrations";
        
        using var connection = new SqliteConnection(_connectionString);
        await connection.OpenAsync(cancellationToken);
        
        using var command = connection.CreateCommand();
        command.CommandText = sql;
        
        var result = await command.ExecuteScalarAsync(cancellationToken);
        return Convert.ToInt32(result);
    }

    private static Migration[] GetMigrations()
    {
        return new[]
        {
            new Migration(1, "CreateHotelsTable", """
                CREATE TABLE IF NOT EXISTS hotels (
                    HotelId INTEGER PRIMARY KEY AUTOINCREMENT,
                    HotelName TEXT NOT NULL,
                    Description TEXT NOT NULL,
                    Location TEXT NOT NULL,
                    Rating REAL NOT NULL,
                    LastUpdated TEXT NOT NULL
                )
                """),
            
            new Migration(2, "AddPricePerNightColumn", """
                ALTER TABLE hotels ADD COLUMN PricePerNight REAL NOT NULL DEFAULT 0.0
                """),
            
            new Migration(3, "CreateHotelsFtsTable", """
                CREATE VIRTUAL TABLE IF NOT EXISTS hotels_fts USING fts5(
                    HotelId UNINDEXED,
                    HotelName,
                    Description,
                    Location,
                    Rating UNINDEXED,
                    PricePerNight UNINDEXED,
                    LastUpdated UNINDEXED
                )
                """)
        };
    }

    private async Task ExecuteMigrationAsync(Migration migration, CancellationToken cancellationToken)
    {
        _logger.LogInformation("Executing migration {Version}: {Name}", migration.Version, migration.Name);

        using var connection = new SqliteConnection(_connectionString);
        await connection.OpenAsync(cancellationToken);
        
        using var transaction = connection.BeginTransaction();
        
        try
        {
            // Execute migration SQL
            using var migrationCommand = connection.CreateCommand();
            migrationCommand.CommandText = migration.Sql;
            migrationCommand.Transaction = transaction;
            await migrationCommand.ExecuteNonQueryAsync(cancellationToken);

            // Record migration
            using var recordCommand = connection.CreateCommand();
            recordCommand.CommandText = """
                INSERT INTO __migrations (version, name, executed_at) 
                VALUES (@version, @name, @executedAt)
                """;
            recordCommand.Transaction = transaction;
            recordCommand.Parameters.AddWithValue("@version", migration.Version);
            recordCommand.Parameters.AddWithValue("@name", migration.Name);
            recordCommand.Parameters.AddWithValue("@executedAt", DateTime.UtcNow.ToString("O"));
            await recordCommand.ExecuteNonQueryAsync(cancellationToken);

            transaction.Commit();
            
            _logger.LogInformation("Migration {Version} completed successfully", migration.Version);
        }
        catch
        {
            transaction.Rollback();
            throw;
        }
    }

    private async Task ExecuteSqlAsync(string sql, CancellationToken cancellationToken)
    {
        using var connection = new SqliteConnection(_connectionString);
        await connection.OpenAsync(cancellationToken);
        
        using var command = connection.CreateCommand();
        command.CommandText = sql;
        await command.ExecuteNonQueryAsync(cancellationToken);
    }

    private record Migration(int Version, string Name, string Sql);
}
```

## Index Optimization

### Vector Index Configuration

**Optimize vector search performance:**

```csharp
public sealed class IndexOptimizer
{
    private readonly string _connectionString;
    private readonly ILogger<IndexOptimizer> _logger;

    public IndexOptimizer(string connectionString, ILogger<IndexOptimizer> logger)
    {
        _connectionString = connectionString;
        _logger = logger;
    }

    public async Task OptimizeVectorIndexesAsync(CancellationToken cancellationToken = default)
    {
        await OptimizeVectorSearchAsync(cancellationToken);
        await OptimizeFtsIndexAsync(cancellationToken);
        await CreateCompositeIndexesAsync(cancellationToken);
    }

    private async Task OptimizeVectorSearchAsync(CancellationToken cancellationToken)
    {
        // Configure vector search parameters for optimal performance
        const string optimizeSql = """
            PRAGMA vector_cache_size = 10000;
            PRAGMA vector_index_type = 'HNSW';
            """;

        await ExecutePragmasAsync(optimizeSql, cancellationToken);
    }

    private async Task OptimizeFtsIndexAsync(CancellationToken cancellationToken)
    {
        const string optimizeSql = """
            INSERT INTO hotels_fts(hotels_fts) VALUES('optimize');
            """;

        await ExecuteSqlAsync(optimizeSql, cancellationToken);
    }

    private async Task CreateCompositeIndexesAsync(CancellationToken cancellationToken)
    {
        var indexes = new[]
        {
            "CREATE INDEX IF NOT EXISTS idx_hotels_rating_location ON hotels(Rating, Location)",
            "CREATE INDEX IF NOT EXISTS idx_hotels_price_rating ON hotels(PricePerNight, Rating)",
            "CREATE INDEX IF NOT EXISTS idx_hotels_updated ON hotels(LastUpdated)"
        };

        foreach (var indexSql in indexes)
        {
            await ExecuteSqlAsync(indexSql, cancellationToken);
        }
    }

    private async Task ExecutePragmasAsync(string sql, CancellationToken cancellationToken)
    {
        using var connection = new SqliteConnection(_connectionString);
        await connection.OpenAsync(cancellationToken);
        
        var statements = sql.Split(';', StringSplitOptions.RemoveEmptyEntries);
        
        foreach (var statement in statements)
        {
            if (string.IsNullOrWhiteSpace(statement)) continue;
            
            using var command = connection.CreateCommand();
            command.CommandText = statement.Trim();
            await command.ExecuteNonQueryAsync(cancellationToken);
        }
    }

    private async Task ExecuteSqlAsync(string sql, CancellationToken cancellationToken)
    {
        using var connection = new SqliteConnection(_connectionString);
        await connection.OpenAsync(cancellationToken);
        
        using var command = connection.CreateCommand();
        command.CommandText = sql;
        await command.ExecuteNonQueryAsync(cancellationToken);
    }
}
```

### Performance Monitoring

**Monitor database performance:**

```csharp
public sealed class DatabasePerformanceMonitor
{
    private readonly string _connectionString;
    private readonly ILogger<DatabasePerformanceMonitor> _logger;

    public DatabasePerformanceMonitor(string connectionString, ILogger<DatabasePerformanceMonitor> logger)
    {
        _connectionString = connectionString;
        _logger = logger;
    }

    public async Task<DatabaseStats> GetDatabaseStatsAsync(CancellationToken cancellationToken = default)
    {
        using var connection = new SqliteConnection(_connectionString);
        await connection.OpenAsync(cancellationToken);

        var stats = new DatabaseStats
        {
            VectorTableStats = await GetTableStatsAsync(connection, "hotels"),
            FtsTableStats = await GetTableStatsAsync(connection, "hotels_fts"),
            IndexStats = await GetIndexStatsAsync(connection),
            DatabaseSize = await GetDatabaseSizeAsync(connection)
        };

        LogDatabaseStats(stats);
        return stats;
    }

    private async Task<TableStats> GetTableStatsAsync(SqliteConnection connection, string tableName)
    {
        var command = connection.CreateCommand();
        command.CommandText = $"SELECT COUNT(*) FROM {tableName}";
        var rowCount = (long)(await command.ExecuteScalarAsync() ?? 0);

        command.CommandText = $"SELECT page_count * page_size FROM pragma_page_count(), pragma_page_size()";
        var tableSize = (long)(await command.ExecuteScalarAsync() ?? 0);

        return new TableStats(tableName, rowCount, tableSize);
    }

    private async Task<List<IndexStats>> GetIndexStatsAsync(SqliteConnection connection)
    {
        var command = connection.CreateCommand();
        command.CommandText = """
            SELECT name, tbl_name, sql 
            FROM sqlite_master 
            WHERE type = 'index' AND name NOT LIKE 'sqlite_%'
            """;

        var indexes = new List<IndexStats>();
        using var reader = await command.ExecuteReaderAsync();
        
        while (await reader.ReadAsync())
        {
            indexes.Add(new IndexStats(
                reader.GetString("name"),
                reader.GetString("tbl_name"),
                reader.IsDBNull("sql") ? null : reader.GetString("sql")));
        }

        return indexes;
    }

    private async Task<long> GetDatabaseSizeAsync(SqliteConnection connection)
    {
        var command = connection.CreateCommand();
        command.CommandText = "SELECT page_count * page_size FROM pragma_page_count(), pragma_page_size()";
        return (long)(await command.ExecuteScalarAsync() ?? 0);
    }

    private void LogDatabaseStats(DatabaseStats stats)
    {
        _logger.LogInformation("Database Statistics:");
        _logger.LogInformation("  Vector Table: {RowCount} rows, {SizeKB} KB", 
            stats.VectorTableStats.RowCount, stats.VectorTableStats.SizeBytes / 1024);
        _logger.LogInformation("  FTS Table: {RowCount} rows, {SizeKB} KB", 
            stats.FtsTableStats.RowCount, stats.FtsTableStats.SizeBytes / 1024);
        _logger.LogInformation("  Total Database Size: {SizeMB} MB", stats.DatabaseSize / 1024 / 1024);
        _logger.LogInformation("  Indexes: {IndexCount}", stats.IndexStats.Count);
    }
}

public record DatabaseStats(
    TableStats VectorTableStats,
    TableStats FtsTableStats,
    List<IndexStats> IndexStats,
    long DatabaseSize);

public record TableStats(string Name, long RowCount, long SizeBytes);
public record IndexStats(string Name, string TableName, string? Definition);
```

This database initialization framework provides a solid foundation for managing SQLiteVec collections, FTS5 tables, and performance optimization, setting the stage for implementing hybrid search functionality.
