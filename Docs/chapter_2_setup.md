# Chapter 2: Setting Up the Development Environment

## Prerequisites and Dependencies

### Required Software

**Development Environment:**
- **.NET 8.0 SDK** or later
- **Visual Studio 2022 17.8+** or **Visual Studio Code** with C# extension
- **Git** for version control

**External Services:**
- **OpenAI API Key** for embedding generation
- **Internet Connection** for NuGet package downloads

### Verify Prerequisites

```bash
# Check .NET version
dotnet --version

# Should return 8.0.x or later
```

```bash
# Check Git installation
git --version
```

## NuGet Package Installation

### Create New Project

```bash
# Create solution
dotnet new sln -n HotelSearch

# Create console application
dotnet new console -n HotelSearch.Console
dotnet sln add HotelSearch.Console

# Create class library for core services
dotnet new classlib -n HotelSearch.Core
dotnet sln add HotelSearch.Core

# Add project reference
dotnet add HotelSearch.Console reference HotelSearch.Core
```

### Install Required Packages

**Core Semantic Kernel Packages:**

```bash
cd HotelSearch.Core

# Semantic Kernel foundation
dotnet add package Microsoft.SemanticKernel --version 1.60.0

# SQLiteVec connector (preview)
dotnet add package Microsoft.SemanticKernel.Connectors.SqliteVec --version 1.60.0-preview

# OpenAI connector
dotnet add package Microsoft.SemanticKernel.Connectors.OpenAI --version 1.60.0

# Vector Data abstractions
dotnet add package Microsoft.Extensions.VectorData.Abstractions --version 9.7.0
```

**Data and Infrastructure Packages:**

```bash
# SQLite data provider
dotnet add package Microsoft.Data.Sqlite --version 9.0.7

# Dependency injection
dotnet add package Microsoft.Extensions.DependencyInjection --version 9.0.7
dotnet add package Microsoft.Extensions.Hosting --version 9.0.7

# Logging
dotnet add package Microsoft.Extensions.Logging --version 9.0.7
dotnet add package Microsoft.Extensions.Logging.Console --version 9.0.7
```

**Console Application Packages:**

```bash
cd ../HotelSearch.Console

# Configuration
dotnet add package Microsoft.Extensions.Configuration --version 9.0.7
dotnet add package Microsoft.Extensions.Configuration.Json --version 9.0.7
dotnet add package Microsoft.Extensions.Configuration.EnvironmentVariables --version 9.0.7
```

### Verify Package Installation

```xml
<!-- HotelSearch.Core/HotelSearch.Core.csproj -->
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <LangVersion>latest</LangVersion>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.Data.Sqlite" Version="9.0.7" />
    <PackageReference Include="Microsoft.Extensions.DependencyInjection" Version="9.0.7" />
    <PackageReference Include="Microsoft.Extensions.Hosting" Version="9.0.7" />
    <PackageReference Include="Microsoft.Extensions.Logging" Version="9.0.7" />
    <PackageReference Include="Microsoft.Extensions.VectorData.Abstractions" Version="9.7.0" />
    <PackageReference Include="Microsoft.SemanticKernel" Version="1.60.0" />
    <PackageReference Include="Microsoft.SemanticKernel.Connectors.OpenAI" Version="1.60.0" />
    <PackageReference Include="Microsoft.SemanticKernel.Connectors.SqliteVec" Version="1.60.0-preview" />
  </ItemGroup>

</Project>
```

## Project Structure and Configuration

### Recommended Project Structure

```
HotelSearch/
├── HotelSearch.sln
├── HotelSearch.Core/
│   ├── Models/
│   │   └── Hotel.cs
│   ├── Services/
│   │   ├── IHotelSearchService.cs
│   │   └── HotelSearchService.cs
│   ├── Configuration/
│   │   └── SearchConfiguration.cs
│   └── Extensions/
│       └── ServiceCollectionExtensions.cs
├── HotelSearch.Console/
│   ├── Program.cs
│   ├── appsettings.json
│   └── appsettings.Development.json
└── README.md
```

### Configuration Setup

**Create appsettings.json:**

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft": "Warning",
      "Microsoft.Hosting.Lifetime": "Information"
    }
  },
  "OpenAI": {
    "ApiKey": "",
    "EmbeddingModel": "text-embedding-3-small",
    "ChatModel": "gpt-4o-mini"
  },
  "SQLiteVec": {
    "ConnectionString": "Data Source=hotels.db",
    "CollectionName": "hotels"
  },
  "Search": {
    "KeywordWeight": 0.6,
    "VectorWeight": 0.4,
    "RrfConstant": 60,
    "SearchMultiplier": 2,
    "MinSearchLimit": 20
  }
}
```

**Create appsettings.Development.json:**

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Debug",
      "HotelSearch": "Debug"
    }
  }
}
```

### Environment Variables Setup

**Create .env file (for development):**

```bash
# .env
OPENAI_API_KEY=your_openai_api_key_here
SQLITEVEC_CONNECTION_STRING=Data Source=hotels.db
```

**Add to .gitignore:**

```gitignore
# Environment files
.env
*.db
*.db-shm
*.db-wal

# API Keys
appsettings.*.json
!appsettings.json
!appsettings.Development.json

# Build outputs
bin/
obj/
*.user
```

### Configuration Models

**Create Configuration/SearchConfiguration.cs:**

```csharp
namespace HotelSearch.Core.Configuration;

public sealed record SearchConfiguration
{
    public const string SectionName = "Search";
    
    public double KeywordWeight { get; init; } = 0.6;
    public double VectorWeight { get; init; } = 0.4;
    public int RrfConstant { get; init; } = 60;
    public int SearchMultiplier { get; init; } = 2;
    public int MinSearchLimit { get; init; } = 20;
}
```

**Create Configuration/OpenAIConfiguration.cs:**

```csharp
namespace HotelSearch.Core.Configuration;

public sealed record OpenAIConfiguration
{
    public const string SectionName = "OpenAI";
    
    public string ApiKey { get; init; } = string.Empty;
    public string EmbeddingModel { get; init; } = "text-embedding-3-small";
    public string ChatModel { get; init; } = "gpt-4o-mini";
}
```

**Create Configuration/SQLiteVecConfiguration.cs:**

```csharp
namespace HotelSearch.Core.Configuration;

public sealed record SQLiteVecConfiguration
{
    public const string SectionName = "SQLiteVec";
    
    public string ConnectionString { get; init; } = "Data Source=hotels.db";
    public string CollectionName { get; init; } = "hotels";
}
```

## OpenAI API Setup

### Obtain API Key

1. **Visit OpenAI Platform**: https://platform.openai.com/
2. **Create Account**: Sign up or sign in
3. **Navigate to API Keys**: Go to API section
4. **Create New Key**: Generate a new secret key
5. **Copy Key**: Save securely (you won't see it again)

### Configure API Key

**Option 1: Environment Variable (Recommended)**

```bash
# Windows
set OPENAI_API_KEY=your_api_key_here

# macOS/Linux
export OPENAI_API_KEY=your_api_key_here
```

**Option 2: User Secrets (Development)**

```bash
# Navigate to console project
cd HotelSearch.Console

# Initialize user secrets
dotnet user-secrets init

# Set the API key
dotnet user-secrets set "OpenAI:ApiKey" "your_api_key_here"
```

**Option 3: Configuration File (Not Recommended for Production)**

```json
{
  "OpenAI": {
    "ApiKey": "your_api_key_here"
  }
}
```

### Verify API Access

**Create simple verification:**

```csharp
// Test/OpenAIConnectionTest.cs
using Microsoft.Extensions.AI;
using Microsoft.SemanticKernel.Connectors.OpenAI;

public static class OpenAIConnectionTest
{
    public static async Task VerifyConnectionAsync(string apiKey)
    {
        try
        {
            var embeddingService = new OpenAIEmbeddingGenerator(
                "text-embedding-3-small", 
                apiKey);
            
            var embedding = await embeddingService.GenerateAsync("test connection");
            
            Console.WriteLine($"✅ OpenAI connection successful");
            Console.WriteLine($"   Embedding dimensions: {embedding.Vector.Length}");
        }
        catch (Exception ex)
        {
            Console.WriteLine($"❌ OpenAI connection failed: {ex.Message}");
            throw;
        }
    }
}
```

### Usage Monitoring

**Track API usage:**

```csharp
public sealed class OpenAIUsageTracker
{
    private readonly ILogger<OpenAIUsageTracker> _logger;
    private int _embeddingRequests;
    private int _totalTokens;

    public OpenAIUsageTracker(ILogger<OpenAIUsageTracker> logger)
    {
        _logger = logger;
    }

    public void TrackEmbeddingRequest(int tokens)
    {
        Interlocked.Increment(ref _embeddingRequests);
        Interlocked.Add(ref _totalTokens, tokens);
        
        _logger.LogDebug(
            "Embedding request #{RequestCount}, Tokens: {Tokens}, Total: {TotalTokens}",
            _embeddingRequests, tokens, _totalTokens);
    }

    public (int Requests, int Tokens) GetUsage() => (_embeddingRequests, _totalTokens);
}
```

## Development Environment Validation

### Create Basic Program Structure

**Program.cs:**

```csharp
using HotelSearch.Core.Extensions;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;

var builder = Host.CreateDefaultBuilder(args);

builder.ConfigureServices((context, services) =>
{
    services.AddHotelSearchServices(context.Configuration);
});

using var host = builder.Build();

var logger = host.Services.GetRequiredService<ILogger<Program>>();
logger.LogInformation("🚀 Hotel Search application starting...");

// Add startup validation here
await ValidateEnvironmentAsync(host.Services);

logger.LogInformation("✅ Environment validation completed");

static async Task ValidateEnvironmentAsync(IServiceProvider services)
{
    var logger = services.GetRequiredService<ILogger<Program>>();
    
    // Test database connection
    logger.LogInformation("🔍 Testing database connection...");
    // Implementation in next chapter
    
    // Test OpenAI connection
    logger.LogInformation("🔍 Testing OpenAI connection...");
    // Implementation in next chapter
}
```

### Build and Test

**Build the solution:**

```bash
# From solution root
dotnet build

# Should complete without errors
```

**Run basic application:**

```bash
cd HotelSearch.Console
dotnet run
```

**Expected output:**

```
info: Program[0]
      🚀 Hotel Search application starting...
info: Program[0]
      🔍 Testing database connection...
info: Program[0]
      🔍 Testing OpenAI connection...
info: Program[0]
      ✅ Environment validation completed
```

## Common Setup Issues

### Package Version Conflicts

**Issue**: Conflicting package versions
**Solution**: Use consistent version ranges

```xml
<PackageReference Include="Microsoft.SemanticKernel*" Version="1.60.0" />
<PackageReference Include="Microsoft.Extensions.*" Version="9.0.7" />
```

### Missing SQLite Native Libraries

**Issue**: SQLite native library not found
**Solution**: Add runtime packages

```bash
dotnet add package Microsoft.Data.Sqlite.Core --version 9.0.7
dotnet add package SQLitePCLRaw.bundle_e_sqlite3 --version 2.1.10
```

### OpenAI API Key Issues

**Issue**: Invalid or missing API key
**Solutions**:
1. Verify key format (starts with `sk-`)
2. Check billing account status
3. Verify key permissions
4. Test with simple HTTP request

### Preview Package Warnings

**Issue**: Preview package warnings
**Solution**: Suppress in project file

```xml
<PropertyGroup>
  <NoWarn>$(NoWarn);SKEXP0001;SKEXP0020</NoWarn>
</PropertyGroup>
```

The development environment is now ready for implementing SQLiteVec hybrid search functionality. The next chapter will cover core concepts and data modeling.
