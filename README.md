# SQLiteVec Hybrid Search Demo

A Jupyter notebook demonstrating **SQLiteVec** integration with **Microsoft Semantic Kernel** for hybrid search implementation. Uses a hotel search scenario to showcase combining FTS5 keyword search with vector semantic search.

## 🎯 Purpose

This notebook demonstrates:
- **SQLiteVec** vector storage and retrieval with Semantic Kernel
- **Hybrid Search** using Reciprocal Rank Fusion (RRF) algorithm
- **VectorData attributes** for schema definition
- **FTS5** and vector search combination
- **Dependency injection** patterns with .NET hosting

## 🗂️ Project Structure

```
SemanticKernel_SqliteVec.ipynb    # Main demonstration notebook
README.md                         # This file
```

## 📋 Prerequisites

- **.NET 8.0 SDK**
- **Jupyter Notebooks** support (.NET Interactive)
- **OpenAI API Key** (for embedding generation)
- **Visual Studio Code** or **Visual Studio** with .NET Interactive extension

## 🚀 Quick Start

### 1. Setup Environment

```bash
# Install .NET Interactive (if not already installed)
dotnet tool install -g Microsoft.dotnet-interactive

# Set your OpenAI API key
export OPENAI_API_KEY="your-openai-api-key-here"
```

### 2. Open the Notebook

```bash
# Clone repository
git clone https://github.com/your-org/sqlitevec-demo.git
cd sqlitevec-demo

# Open in VS Code
code SemanticKernel_SqliteVec.ipynb
```

### 3. Run the Demo

Execute the notebook cells sequentially to see:
1. Package installation and setup
2. Data model definition with VectorData attributes  
3. Service implementation with hybrid search
4. Dependency injection configuration
5. Database initialization and data seeding
6. Search demonstrations with different query types
7. Performance analysis
8. Advanced features and filtering

## 🧩 Key SQLiteVec Concepts Demonstrated

### Vector Data Model Definition

```csharp
public class Hotel
{
    [VectorStoreKey]
    public int HotelId { get; set; }
    
    [VectorStoreData]
    public string? HotelName { get; set; }
    
    [VectorStoreVector(Dimensions: 1536, DistanceFunction = DistanceFunction.CosineDistance)]
    public ReadOnlyMemory<float>? DescriptionEmbedding { get; set; }
}
```

### SQLiteVec Collection Setup

```csharp
var vectorCollection = new SqliteCollection<int, Hotel>(connectionString, "hotels");
await vectorCollection.EnsureCollectionExistsAsync();
```

### Hybrid Search with RRF

The notebook demonstrates the complete RRF algorithm:
- Parallel execution of FTS5 keyword search and vector similarity search
- Score fusion using `weight / (k + rank)` formula
- Result ranking and filtering

### Service Layer Integration

```csharp
services.AddSingleton<SqliteCollection<int, Hotel>>();
services.AddOpenAIEmbeddingGenerator("text-embedding-3-small", apiKey);
services.AddSingleton<IHotelSearchService, HotelSearchService>();
```

## 📊 Notebook Sections

| Section | Description |
|---------|-------------|
| **1. Package Installation** | Install required NuGet packages |
| **2. Imports and Configuration** | Namespace imports and setup |
| **3. Data Model Definition** | Hotel class with VectorData attributes |
| **4. Hotel Search Service** | Complete service implementation with RRF |
| **5. Dependency Injection Setup** | .NET hosting and service registration |
| **6. Data Initialization** | Database schema and sample data |
| **7. Hybrid Search Demo** | Various search query examples |
| **8. Performance Analysis** | Timing and memory usage measurement |
| **9. Advanced Features** | Filtering and batch operations |
| **10. Cleanup and Best Practices** | Resource management patterns |

## 🔧 Configuration

The notebook uses these key configurations:

```csharp
public record SearchConfiguration
{
    public double KeywordWeight { get; init; } = 0.6;    // 60% weight for keyword search
    public double VectorWeight { get; init; } = 0.4;     // 40% weight for vector search
    public int RrfConstant { get; init; } = 60;          // RRF algorithm constant
}
```

## 🧪 Running the Demo

1. **Execute cells sequentially** - Each cell builds on the previous ones
2. **Set OpenAI API key** - Required for embedding generation
3. **Watch the output** - Each section shows results and performance metrics
4. **Experiment with queries** - Try different search terms in section 7

## 📈 Search Examples Shown

The notebook demonstrates these search scenarios:

- **"luxury spa resort"** - Exact keyword matches + semantic similarity
- **"mountain hiking"** - Semantic understanding of outdoor activities  
- **"business conference"** - Finds hotels with meeting facilities
- **"relaxation wellness"** - Wellness-focused semantic search
- **"historic charm"** - Character and ambiance-based matching

## 🎓 Learning Outcomes

By running this notebook, you'll understand:

1. **SQLiteVec Setup**: How to configure vector collections with Semantic Kernel
2. **Data Modeling**: Using VectorData attributes for schema definition
3. **Hybrid Search**: Implementing RRF to combine keyword and vector results
4. **Performance Patterns**: Measuring and optimizing search operations
5. **Production Patterns**: Dependency injection and service architecture

## 🔄 Extending the Demo

To explore further:

1. **Modify search weights** in `SearchConfiguration`
2. **Add new hotel data** in the seeding section
3. **Experiment with different embeddings models**
4. **Try custom distance functions**
5. **Add new filtering criteria**

## 📋 System Requirements

- **.NET 8.0** runtime
- **~100MB** memory for embeddings
- **SQLite** database (created automatically)
- **Internet connection** for OpenAI API calls

## 🔍 Performance Characteristics

The notebook measures:
- **Search latency**: ~50-300ms for hybrid search
- **Memory usage**: ~171KB per search operation  
- **Vector dimensions**: 1536 (OpenAI text-embedding-3-small)
- **Sample dataset**: 5 hotels with embeddings

---

**A practical Jupyter notebook demonstration of SQLiteVec capabilities with Microsoft Semantic Kernel**
