# Chapter 3: Core Concepts and Data Modeling

## Understanding Vector Data Attributes

### VectorData Attribute System

Microsoft Semantic Kernel uses attributes to define how your C# classes map to vector database schemas. These attributes provide metadata that SQLiteVec uses to create the appropriate database structure and perform operations.

**Core Attribute Types:**
- `[VectorStoreKey]`: Identifies the primary key
- `[VectorStoreData]`: Marks regular data fields
- `[VectorStoreVector]`: Defines vector/embedding fields

### VectorStoreKey Attribute

The `[VectorStoreKey]` attribute identifies the primary key for your entity:

```csharp
public class Hotel
{
    [VectorStoreKey]
    public int HotelId { get; set; }
}
```

**Key Requirements:**
- Exactly one property must have this attribute
- Supported types: `int`, `long`, `string`, `Guid`
- Must be unique across all records
- Used for updates and deletions

**Type Considerations:**

```csharp
// Integer keys (auto-increment supported)
[VectorStoreKey]
public int Id { get; set; }

// String keys (useful for external IDs)
[VectorStoreKey]
public string ProductCode { get; set; }

// GUID keys (globally unique)
[VectorStoreKey]
public Guid DocumentId { get; set; }
```

### VectorStoreData Attribute

The `[VectorStoreData]` attribute marks properties for traditional data storage:

```csharp
public class Hotel
{
    [VectorStoreData]
    public string? HotelName { get; set; }
    
    [VectorStoreData]
    public string? Description { get; set; }
    
    [VectorStoreData]
    public double Rating { get; set; }
}
```

**Supported Data Types:**
- `string`, `int`, `long`, `double`, `float`
- `bool`, `DateTime`, `DateTimeOffset`
- `Guid`, `byte[]`
- Nullable versions of all above types

**Storage Mapping:**

```csharp
public class Product
{
    [VectorStoreData(StoragePropertyName = "product_name")]
    public string? Name { get; set; }
    
    [VectorStoreData(StoragePropertyName = "created_at")]
    public DateTime CreatedDate { get; set; }
}
```

### VectorStoreVector Attribute

The `[VectorStoreVector]` attribute defines embedding fields with specific configuration:

```csharp
public class Hotel
{
    [VectorStoreVector(
        Dimensions: 1536, 
        DistanceFunction = DistanceFunction.CosineDistance)]
    public ReadOnlyMemory<float>? DescriptionEmbedding { get; set; }
}
```

**Required Parameters:**
- `Dimensions`: Number of vector dimensions (must match embedding model)
- `DistanceFunction`: Method for calculating similarity

**Vector Data Types:**
- `ReadOnlyMemory<float>` (recommended)
- `float[]`
- `Memory<float>`

## Designing Data Models for SQLiteVec

### Single Entity Model

**Basic hotel model:**

```csharp
using Microsoft.Extensions.VectorData;

namespace HotelSearch.Core.Models;

public sealed class Hotel
{
    [VectorStoreKey]
    public int HotelId { get; set; }
    
    [VectorStoreData]
    public string HotelName { get; set; } = string.Empty;
    
    [VectorStoreData]
    public string Description { get; set; } = string.Empty;
    
    [VectorStoreData]
    public string Location { get; set; } = string.Empty;
    
    [VectorStoreData]
    public double Rating { get; set; }
    
    [VectorStoreData]
    public decimal PricePerNight { get; set; }
    
    [VectorStoreData]
    public DateTime LastUpdated { get; set; }
    
    [VectorStoreVector(
        Dimensions: 1536, 
        DistanceFunction = DistanceFunction.CosineDistance)]
    public ReadOnlyMemory<float>? NameEmbedding { get; set; }
    
    [VectorStoreVector(
        Dimensions: 1536, 
        DistanceFunction = DistanceFunction.CosineDistance)]
    public ReadOnlyMemory<float>? DescriptionEmbedding { get; set; }
}
```

### Multi-Vector Model

**Document with multiple embeddings:**

```csharp
public sealed class Document
{
    [VectorStoreKey]
    public Guid DocumentId { get; set; }
    
    [VectorStoreData]
    public string Title { get; set; } = string.Empty;
    
    [VectorStoreData]
    public string Content { get; set; } = string.Empty;
    
    [VectorStoreData]
    public string Category { get; set; } = string.Empty;
    
    [VectorStoreData]
    public string[] Tags { get; set; } = Array.Empty<string>();
    
    // Different embeddings for different purposes
    [VectorStoreVector(
        Dimensions: 1536, 
        DistanceFunction = DistanceFunction.CosineDistance)]
    public ReadOnlyMemory<float>? TitleEmbedding { get; set; }
    
    [VectorStoreVector(
        Dimensions: 1536, 
        DistanceFunction = DistanceFunction.CosineDistance)]
    public ReadOnlyMemory<float>? ContentEmbedding { get; set; }
    
    [VectorStoreVector(
        Dimensions: 768, 
        DistanceFunction = DistanceFunction.DotProduct)]
    public ReadOnlyMemory<float>? SummaryEmbedding { get; set; }
}
```

### Hierarchical Model

**Product with category and brand information:**

```csharp
public sealed class Product
{
    [VectorStoreKey]
    public string ProductId { get; set; } = string.Empty;
    
    [VectorStoreData]
    public string Name { get; set; } = string.Empty;
    
    [VectorStoreData]
    public string Description { get; set; } = string.Empty;
    
    [VectorStoreData]
    public string Brand { get; set; } = string.Empty;
    
    [VectorStoreData]
    public string Category { get; set; } = string.Empty;
    
    [VectorStoreData]
    public string[] Features { get; set; } = Array.Empty<string>();
    
    [VectorStoreData]
    public decimal Price { get; set; }
    
    [VectorStoreData]
    public bool IsActive { get; set; }
    
    // Combined product information embedding
    [VectorStoreVector(
        Dimensions: 1536, 
        DistanceFunction = DistanceFunction.CosineDistance)]
    public ReadOnlyMemory<float>? ProductEmbedding { get; set; }
    
    // Category-specific embedding for filtering
    [VectorStoreVector(
        Dimensions: 768, 
        DistanceFunction = DistanceFunction.EuclideanDistance)]
    public ReadOnlyMemory<float>? CategoryEmbedding { get; set; }
}
```

## Key, Data, and Vector Properties

### Key Property Design

**Auto-incrementing integer keys:**

```csharp
public sealed class Hotel
{
    [VectorStoreKey]
    public int HotelId { get; set; } // Auto-assigned by database
}
```

**Natural string keys:**

```csharp
public sealed class Hotel
{
    [VectorStoreKey]
    public string HotelCode { get; set; } = string.Empty; // Business-assigned
}
```

**Composite key alternative (using single string):**

```csharp
public sealed class Review
{
    [VectorStoreKey]
    public string ReviewId { get; set; } = string.Empty; // Format: "hotel123_user456_20241201"
    
    [VectorStoreData]
    public string HotelId { get; set; } = string.Empty;
    
    [VectorStoreData] 
    public string UserId { get; set; } = string.Empty;
}
```

### Data Property Best Practices

**Nullable vs Non-Nullable:**

```csharp
public sealed class Hotel
{
    // Required fields - non-nullable
    [VectorStoreData]
    public string HotelName { get; set; } = string.Empty;
    
    [VectorStoreData]
    public string Location { get; set; } = string.Empty;
    
    // Optional fields - nullable
    [VectorStoreData]
    public string? Website { get; set; }
    
    [VectorStoreData]
    public string? PhoneNumber { get; set; }
}
```

**Structured data handling:**

```csharp
public sealed class Hotel
{
    // Store as JSON string
    [VectorStoreData]
    public string AmenitiesJson { get; set; } = "[]";
    
    // Or flatten into separate properties
    [VectorStoreData]
    public bool HasPool { get; set; }
    
    [VectorStoreData]
    public bool HasSpa { get; set; }
    
    [VectorStoreData]
    public bool HasGym { get; set; }
    
    // Computed property (not stored)
    public string[] Amenities 
    { 
        get => System.Text.Json.JsonSerializer.Deserialize<string[]>(AmenitiesJson) ?? Array.Empty<string>();
        set => AmenitiesJson = System.Text.Json.JsonSerializer.Serialize(value);
    }
}
```

### Vector Property Configuration

**Multiple embeddings with different purposes:**

```csharp
public sealed class Hotel
{
    // Short, keyword-focused embedding
    [VectorStoreVector(
        Dimensions: 768, 
        DistanceFunction = DistanceFunction.DotProduct)]
    public ReadOnlyMemory<float>? NameEmbedding { get; set; }
    
    // Detailed, semantic embedding
    [VectorStoreVector(
        Dimensions: 1536, 
        DistanceFunction = DistanceFunction.CosineDistance)]
    public ReadOnlyMemory<float>? DescriptionEmbedding { get; set; }
    
    // Combined features embedding
    [VectorStoreVector(
        Dimensions: 1536, 
        DistanceFunction = DistanceFunction.CosineDistance)]
    public ReadOnlyMemory<float>? FeaturesEmbedding { get; set; }
}
```

**Property naming conventions:**

```csharp
public sealed class Document
{
    [VectorStoreData]
    public string Title { get; set; } = string.Empty;
    
    // Match property name to source + "Embedding"
    [VectorStoreVector(Dimensions: 1536, DistanceFunction = DistanceFunction.CosineDistance)]
    public ReadOnlyMemory<float>? TitleEmbedding { get; set; }
    
    [VectorStoreData]
    public string Content { get; set; } = string.Empty;
    
    [VectorStoreVector(Dimensions: 1536, DistanceFunction = DistanceFunction.CosineDistance)]
    public ReadOnlyMemory<float>? ContentEmbedding { get; set; }
}
```

## Distance Functions and Dimensions

### Distance Function Types

**Cosine Distance (Most Common):**
- **Use Case**: Text similarity, semantic search
- **Range**: 0 (identical) to 2 (opposite)
- **Benefits**: Normalized, handles different vector magnitudes well

```csharp
[VectorStoreVector(
    Dimensions: 1536, 
    DistanceFunction = DistanceFunction.CosineDistance)]
public ReadOnlyMemory<float>? TextEmbedding { get; set; }
```

**Euclidean Distance:**
- **Use Case**: Spatial data, coordinate systems
- **Range**: 0 (identical) to ∞
- **Benefits**: Intuitive geometric interpretation

```csharp
[VectorStoreVector(
    Dimensions: 3, 
    DistanceFunction = DistanceFunction.EuclideanDistance)]
public ReadOnlyMemory<float>? LocationEmbedding { get; set; }
```

**Dot Product:**
- **Use Case**: When vector magnitude matters
- **Range**: -∞ to +∞
- **Benefits**: Fast computation, good for normalized vectors

```csharp
[VectorStoreVector(
    Dimensions: 768, 
    DistanceFunction = DistanceFunction.DotProduct)]
public ReadOnlyMemory<float>? KeywordEmbedding { get; set; }
```

### Embedding Model Dimensions

**OpenAI Models:**

```csharp
// text-embedding-3-small (recommended for most use cases)
[VectorStoreVector(
    Dimensions: 1536, 
    DistanceFunction = DistanceFunction.CosineDistance)]
public ReadOnlyMemory<float>? SmallEmbedding { get; set; }

// text-embedding-3-large (higher quality, more expensive)
[VectorStoreVector(
    Dimensions: 3072, 
    DistanceFunction = DistanceFunction.CosineDistance)]
public ReadOnlyMemory<float>? LargeEmbedding { get; set; }

// text-embedding-ada-002 (legacy, still supported)
[VectorStoreVector(
    Dimensions: 1536, 
    DistanceFunction = DistanceFunction.CosineDistance)]
public ReadOnlyMemory<float>? AdaEmbedding { get; set; }
```

### Dimension Selection Strategy

**Performance vs Quality Trade-offs:**

| Dimensions | Use Case | Performance | Quality |
|------------|----------|-------------|---------|
| 384-768    | Keywords, categories | Fast | Good |
| 1536       | General text search | Moderate | Very Good |
| 3072       | High-precision search | Slow | Excellent |

**Mixed dimension approach:**

```csharp
public sealed class SearchableContent
{
    // Fast filtering embedding
    [VectorStoreVector(
        Dimensions: 768, 
        DistanceFunction = DistanceFunction.DotProduct)]
    public ReadOnlyMemory<float>? CategoryEmbedding { get; set; }
    
    // High-quality semantic embedding
    [VectorStoreVector(
        Dimensions: 1536, 
        DistanceFunction = DistanceFunction.CosineDistance)]
    public ReadOnlyMemory<float>? ContentEmbedding { get; set; }
}
```

### Model Design Patterns

**Repository Pattern Interface:**

```csharp
public interface IVectorEntity<TKey>
{
    TKey Id { get; set; }
    DateTime LastUpdated { get; set; }
}

public abstract class VectorEntityBase<TKey> : IVectorEntity<TKey>
{
    [VectorStoreKey]
    public TKey Id { get; set; } = default!;
    
    [VectorStoreData]
    public DateTime LastUpdated { get; set; } = DateTime.UtcNow;
}
```

**Derived entities:**

```csharp
public sealed class Hotel : VectorEntityBase<int>
{
    [VectorStoreData]
    public string HotelName { get; set; } = string.Empty;
    
    [VectorStoreVector(
        Dimensions: 1536, 
        DistanceFunction = DistanceFunction.CosineDistance)]
    public ReadOnlyMemory<float>? DescriptionEmbedding { get; set; }
}
```

**Validation attributes:**

```csharp
using System.ComponentModel.DataAnnotations;

public sealed class Hotel
{
    [VectorStoreKey]
    public int HotelId { get; set; }
    
    [VectorStoreData]
    [Required]
    [StringLength(200)]
    public string HotelName { get; set; } = string.Empty;
    
    [VectorStoreData]
    [Range(0.0, 5.0)]
    public double Rating { get; set; }
    
    [VectorStoreVector(
        Dimensions: 1536, 
        DistanceFunction = DistanceFunction.CosineDistance)]
    public ReadOnlyMemory<float>? DescriptionEmbedding { get; set; }
}
```

This foundation in data modeling with VectorData attributes provides the essential structure needed for implementing efficient hybrid search with SQLiteVec. The next chapter will build upon these models to create the service layer.
