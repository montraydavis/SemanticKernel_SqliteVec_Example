# Microsoft Semantic Kernel with SQLiteVec
## A Complete Hybrid Search Tutorial Collection

> **Learn to build production-ready hybrid search with SQLiteVec and Microsoft Semantic Kernel through multiple comprehensive learning formats.**

<div align="center">

[![.NET 8.0](https://img.shields.io/badge/.NET-8.0-purple.svg)](https://dotnet.microsoft.com/download/dotnet/8.0)
[![Semantic Kernel](https://img.shields.io/badge/Semantic%20Kernel-1.60.0-blue.svg)](https://github.com/microsoft/semantic-kernel)
[![SQLiteVec](https://img.shields.io/badge/SQLiteVec-Preview-green.svg)](https://github.com/asg017/sqlite-vss)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

</div>

---

## 🎯 What You'll Master

This comprehensive tutorial collection teaches you to build **hybrid search systems** that combine the precision of keyword search with the semantic understanding of vector embeddings. You'll learn through multiple formats designed for different learning styles.

### Core Technologies
- **SQLiteVec**: Lightweight vector database extension for SQLite
- **Microsoft Semantic Kernel**: AI orchestration framework
- **Hybrid Search**: Reciprocal Rank Fusion (RRF) algorithm
- **OpenAI Embeddings**: Text-to-vector transformation
- **Production Patterns**: Scalable architecture design

---

## 📚 Learning Resources

### 🎧 Audio Tutorial
**[Microsoft Semantic Kernel with SQLiteVec: A Hybrid Search Guide](./Assets/Microsoft%20Semantic%20Kernel%20with%20SQLiteVec_A%20Hybrid%20Search%20Guide.mp3)**
> *Perfect for commuting or multitasking learners*

A comprehensive audio walkthrough covering the entire hybrid search implementation from concept to production.

### 📖 Complete Guide (9 Chapters)
**[Browse the full guide chapters](./Docs/)**

A comprehensive technical guide covering everything from basics to production deployment:

<details>
<summary><strong>📑 Chapter Overview</strong></summary>

#### **Part I: Foundations**
- [Chapter 1: Introduction to Vector Databases and Hybrid Search](./Docs/chapter_1_intro.md)
- [Chapter 2: Setting Up the Development Environment](./Docs/chapter_2_setup.md)  
- [Chapter 3: Core Concepts and Data Modeling](./Docs/chapter_3_data_modeling.md)

#### **Part II: Implementation**
- [Chapter 4: Building the Foundation Service Layer](./Docs/chapter_4_service_layer.md)
- [Chapter 5: Database Schema and Initialization](./Docs/chapter_5_database_schema.md)
- [Chapter 6: Implementing Hybrid Search](./Docs/chapter_6_hybrid_search.md)

#### **Part III: Advanced Features & Production**
- [Chapter 7: Search Optimization and Performance](./Docs/chapter_7_optimization.md)
- [Chapter 8: Advanced Querying Patterns](./Docs/chapter_8_advanced_querying.md)
- [Chapter 9: Production Considerations](./Docs/chapter_9_production.md)

</details>

### 🔬 Interactive Jupyter Notebook
**[SemanticKernel_SqliteVec.ipynb](./SemanticKernel_SqliteVec.ipynb)**
> *Hands-on learning with live code execution*

Step-by-step implementation with running code, performance analysis, and interactive examples.

---

## 🚀 Quick Start

### Prerequisites
- **.NET 8.0 SDK** or later
- **OpenAI API Key** (for embeddings)
- **Visual Studio Code** or **Visual Studio 2022**

### 🔧 Setup

1. **Clone the repository**
   ```bash
   git clone https://github.com/your-org/semantic-kernel-sqlitevec-tutorial.git
   cd semantic-kernel-sqlitevec-tutorial
   ```

2. **Set your OpenAI API key**
   ```bash
   # Windows
   set OPENAI_API_KEY=your_openai_api_key_here

   # macOS/Linux  
   export OPENAI_API_KEY=your_openai_api_key_here
   ```

3. **Choose your learning path**
   - 🎧 **Audio**: Play the MP3 tutorial
   - 📖 **Reading**: Start with [Chapter 1](./Docs/chapter_1_intro.md)
   - 🔬 **Interactive**: Open the [Jupyter notebook](./SemanticKernel_SqliteVec.ipynb)

---

## 🏗️ What You'll Build

### Hotel Search Engine Demo
A complete hybrid search system that demonstrates:

```csharp
// Combine keyword precision with semantic understanding
var results = await hotelSearchService.SearchHotelsAsync(
    "luxury spa resort",           // User query
    new SearchOptions { 
        MaxResults = 10,
        IncludeMetrics = true      // See the fusion in action
    }
);

// Results intelligently ranked using RRF algorithm
foreach (var hotel in results.Items)
{
    Console.WriteLine($"{hotel.HotelName} ({hotel.Rating}★)");
    Console.WriteLine($"📍 {hotel.Location}");
    Console.WriteLine($"💬 {hotel.Description}");
}
```

### Key Features Implemented
- **📊 Hybrid Search**: 60% keyword + 40% vector weights (configurable)
- **🔄 RRF Algorithm**: Scientifically proven result fusion
- **⚡ Performance**: ~50-300ms search latency  
- **🎯 Filtering**: Rating, location, and metadata filters
- **📈 Analytics**: Detailed search metrics and performance tracking
- **🏭 Production Ready**: Connection pooling, caching, error handling

---

## 🎓 Learning Path Recommendations

### 👶 **Beginner** (New to vector search)
1. 🎧 Listen to the audio tutorial for foundational concepts
2. 📖 Read Chapters 1-3 for core understanding
3. 🔬 Follow the Jupyter notebook sections 1-6

### 👨‍💻 **Intermediate** (Familiar with embeddings)
1. 📖 Focus on Chapters 4-6 for implementation patterns
2. 🔬 Run the complete Jupyter notebook
3. 🛠️ Experiment with the RRF algorithm parameters

### 🚀 **Advanced** (Ready for production)
1. 📖 Study Chapters 7-9 for optimization and production patterns
2. 🏗️ Implement the connection pooling and caching strategies
3. 📊 Build your own distance functions and fusion algorithms

---

## 🔑 Key Concepts Covered

### **Hybrid Search Architecture**
- Combining FTS5 (SQLite full-text search) with vector embeddings
- Reciprocal Rank Fusion for intelligent result merging
- Configurable weighting between search methods

### **SQLiteVec Integration** 
- VectorData attributes for schema definition
- Efficient vector storage and retrieval
- Distance function configuration (Cosine, Euclidean, Dot Product)

### **Production Patterns**
- Dependency injection and clean architecture
- Connection pooling and resource management
- Performance monitoring and optimization
- Error handling and resilience patterns

---

## 📊 Performance Benchmarks

From the notebook demonstrations:
- **Search Latency**: 50-300ms for hybrid search
- **Memory Usage**: ~171KB per search operation  
- **Embedding Model**: OpenAI text-embedding-3-small (1536 dimensions)
- **Dataset**: Demonstrates with 5 sample hotels

---

## 🤝 Learning Support

### 💬 Discussion Topics
- RRF parameter tuning strategies
- Custom distance function implementations  
- Scaling SQLiteVec for larger datasets
- Integration with other embedding providers

### 🐛 Common Issues & Solutions
- [OpenAI API key configuration](./Docs/chapter_2_setup.md#openai-api-setup)
- [SQLite connection management](./Docs/chapter_9_production.md#connection-management-and-pooling)
- [Performance optimization tips](./Docs/chapter_7_optimization.md)

---

## 🔗 Related Resources

- [Microsoft Semantic Kernel Documentation](https://docs.microsoft.com/en-us/semantic-kernel/)
- [SQLiteVec Extension](https://github.com/asg017/sqlite-vss)
- [OpenAI Embeddings Guide](https://platform.openai.com/docs/guides/embeddings)
- [Reciprocal Rank Fusion Paper](https://plg.uwaterloo.ca/~gvcormac/cormacksigir09-rrf.pdf)

---

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

---

<div align="center">

### 🌟 Ready to build intelligent search?

**[🎧 Start with Audio](./Assets/Microsoft%20Semantic%20Kernel%20with%20SQLiteVec_A%20Hybrid%20Search%20Guide.mp3)** | **[📖 Read Chapter 1](./Docs/chapter_1_intro.md)** | **[🔬 Try the Notebook](./SemanticKernel_SqliteVec.ipynb)**

*Master hybrid search with the most comprehensive SQLiteVec tutorial available*

</div>
