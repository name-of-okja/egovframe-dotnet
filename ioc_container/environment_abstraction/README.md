# Environment Abstraction

## 개요

.NET 8 LTS에서는 애플리케이션 환경에 대한 추상화를 `IHostEnvironment`와 `IConfiguration` 인터페이스를 통해 제공한다.

이러한 인터페이스들은 애플리케이션 환경의 두 가지 주요 측면을 모델링하는 통합된 추상화로, **환경별 구성(Environment-specific Configuration)**과 **구성 속성(Configuration Properties)**을 관리할 때 사용한다.

**Environment**는 개발(Development), 스테이징(Staging), 프로덕션(Production) 등 특정 환경이 활성화된 경우에만 특정 서비스나 구성이 등록되는 논리적 그룹이다.

**Configuration**은 거의 모든 애플리케이션에서 중요한 역할을 하며, appsettings.json 파일, 환경 변수, 명령줄 인수, 외부 구성 저장소, 사용자 비밀(User Secrets) 등 다양한 소스에서 생성될 수 있다.

.NET의 구성 시스템은 사용자에게 구성 소스를 조합하고 강타입 구성 객체로 바인딩하기 위한 편리한 서비스 인터페이스를 제공한다.

## 설명

### 환경별 서비스 등록

.NET에서는 다음과 같은 상황에서 서로 다른 환경에서 서로 다른 서비스를 등록할 수 있는 메커니즘을 제공한다:

- 개발 시에는 인메모리 데이터베이스를 사용하고, QA나 프로덕션에서는 실제 데이터베이스 연결을 사용
- 프로덕션 환경에서만 모니터링 및 로깅 인프라를 등록
- 고객별 또는 테넌트별 서비스의 사용자 정의 구현을 등록

#### 환경별 조건부 서비스 등록

.NET에서는 `IHostEnvironment`를 사용하여 환경에 따른 조건부 서비스 등록을 할 수 있다:

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// 개발 환경에서만 사용되는 데이터베이스 구성
if (builder.Environment.IsDevelopment())
{
    builder.Services.AddDbContext<AppDbContext>(options =>
        options.UseInMemoryDatabase("InMemoryDb"));
    
    builder.Services.AddScoped<IDataService, MockDataService>();
}
else if (builder.Environment.IsProduction())
{
    builder.Services.AddDbContext<AppDbContext>(options =>
        options.UseSqlServer(builder.Configuration.GetConnectionString("ProductionDb")));
    
    builder.Services.AddScoped<IDataService, SqlDataService>();
}

var app = builder.Build();
```

#### 환경별 구성 클래스

환경별 구성을 별도 클래스로 분리할 수 있다:

```csharp
// DevelopmentConfiguration.cs
public static class DevelopmentConfiguration
{
    public static IServiceCollection AddDevelopmentServices(this IServiceCollection services)
    {
        services.AddDbContext<AppDbContext>(options =>
            options.UseInMemoryDatabase("DevDb"));
        
        services.AddSingleton<IEmailService, MockEmailService>();
        services.AddScoped<IDataService, MockDataService>();
        
        return services;
    }
}

// ProductionConfiguration.cs
public static class ProductionConfiguration
{
    public static IServiceCollection AddProductionServices(this IServiceCollection services, 
        IConfiguration configuration)
    {
        services.AddDbContext<AppDbContext>(options =>
            options.UseSqlServer(configuration.GetConnectionString("DefaultConnection")));
        
        services.AddSingleton<IEmailService, SendGridEmailService>();
        services.AddScoped<IDataService, EntityFrameworkDataService>();
        
        return services;
    }
}
```

#### 환경별 구성 사용

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// 환경에 따른 서비스 등록
if (builder.Environment.IsDevelopment())
{
    builder.Services.AddDevelopmentServices();
}
else
{
    builder.Services.AddProductionServices(builder.Configuration);
}

var app = builder.Build();
```

### Configuration 시스템 활용

.NET의 구성 시스템은 다양한 구성 소스를 조합하여 강타입 구성 객체를 제공한다.

#### appsettings.json 구성

먼저 `appsettings.json` 파일에 구성을 정의한다:

```json
{
  "DatabaseSettings": {
    "ConnectionString": "Server=localhost;Database=MyApp;",
    "CommandTimeout": 30,
    "EnableRetryOnFailure": true
  },
  "EmailSettings": {
    "SmtpServer": "smtp.gmail.com",
    "Port": 587,
    "Username": "noreply@myapp.com"
  }
}
```

#### 강타입 구성 클래스

구성을 강타입 클래스로 바인딩한다:

```csharp
// DatabaseSettings.cs
public class DatabaseSettings
{
    public string ConnectionString { get; set; } = string.Empty;
    public int CommandTimeout { get; set; } = 30;
    public bool EnableRetryOnFailure { get; set; } = true;
}

// EmailSettings.cs
public class EmailSettings
{
    public string SmtpServer { get; set; } = string.Empty;
    public int Port { get; set; } = 587;
    public string Username { get; set; } = string.Empty;
}
```

#### 구성 등록 및 사용

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// 강타입 구성 등록
builder.Services.Configure<DatabaseSettings>(
    builder.Configuration.GetSection("DatabaseSettings"));
builder.Services.Configure<EmailSettings>(
    builder.Configuration.GetSection("EmailSettings"));

// 구성을 사용하는 서비스 등록
builder.Services.AddScoped<IDatabaseService, DatabaseService>();

var app = builder.Build();

// DatabaseService.cs
public class DatabaseService : IDatabaseService
{
    private readonly DatabaseSettings _settings;

    public DatabaseService(IOptions<DatabaseSettings> options)
    {
        _settings = options.Value;
    }

    public async Task<string> GetConnectionStringAsync()
    {
        return _settings.ConnectionString;
    }
}
```

### 환경별 구성 파일

.NET은 환경별 구성 파일을 자동으로 로드한다:

```
appsettings.json                    // 기본 구성
appsettings.Development.json        // 개발 환경 구성
appsettings.Staging.json           // 스테이징 환경 구성  
appsettings.Production.json        // 프로덕션 환경 구성
```

#### 환경별 구성 예시

```json
// appsettings.Development.json
{
  "DatabaseSettings": {
    "ConnectionString": "Server=(localdb)\\mssqllocaldb;Database=MyApp_Dev;",
    "EnableRetryOnFailure": false
  },
  "Logging": {
    "LogLevel": {
      "Default": "Debug"
    }
  }
}

// appsettings.Production.json  
{
  "DatabaseSettings": {
    "ConnectionString": "Server=prod-server;Database=MyApp_Prod;",
    "CommandTimeout": 60,
    "EnableRetryOnFailure": true
  },
  "Logging": {
    "LogLevel": {
      "Default": "Warning"
    }
  }
}
```

### 구성 유효성 검사

.NET에서는 구성 유효성 검사를 통해 런타임 오류를 방지할 수 있다:

```csharp
// DatabaseSettings.cs with validation
public class DatabaseSettings
{
    [Required]
    public string ConnectionString { get; set; } = string.Empty;
    
    [Range(1, 300)]
    public int CommandTimeout { get; set; } = 30;
    
    public bool EnableRetryOnFailure { get; set; } = true;
}

// Program.cs
builder.Services.AddOptions<DatabaseSettings>()
    .BindConfiguration("DatabaseSettings")
    .ValidateDataAnnotations()
    .ValidateOnStart(); // 애플리케이션 시작 시 유효성 검사
```

### 환경 변수와 사용자 비밀

#### 환경 변수 사용

```csharp
// 환경 변수를 통한 구성 재정의
// 환경 변수: DatabaseSettings__ConnectionString
var connectionString = builder.Configuration["DatabaseSettings:ConnectionString"];

// 또는 강타입 구성에서 자동으로 바인딩됨
```

#### 사용자 비밀 (개발 환경)

```csharp
// 개발 환경에서 민감한 정보를 위한 사용자 비밀
if (builder.Environment.IsDevelopment())
{
    builder.Configuration.AddUserSecrets<Program>();
}
```

### 외부 구성 저장소 통합 (프로덕션)

```csharp
// 프로덕션 환경에서 외부 구성 저장소 사용
if (builder.Environment.IsProduction())
{
    // 외부 구성 저장소 URL (예: HashiCorp Vault, etcd, Consul 등)
    var configStoreUrl = builder.Configuration["ExternalConfigStore:Url"];
    var configStoreToken = builder.Configuration["ExternalConfigStore:Token"];
    
    if (!string.IsNullOrEmpty(configStoreUrl))
    {
        // 사용자 정의 구성 프로바이더 추가
        builder.Configuration.AddExternalConfigurationStore(configStoreUrl, configStoreToken);
    }
}

// 또는 Docker/Kubernetes 환경에서 ConfigMap 및 Secret 사용
if (builder.Environment.IsProduction())
{
    // Kubernetes ConfigMap 기반 구성
    var configMapPath = "/etc/config";
    if (Directory.Exists(configMapPath))
    {
        builder.Configuration.AddKeyPerFile(configMapPath, optional: true);
    }
    
    // Kubernetes Secret 기반 구성
    var secretPath = "/etc/secrets";
    if (Directory.Exists(secretPath))
    {
        builder.Configuration.AddKeyPerFile(secretPath, optional: true);
    }
}
```

### 구성 변경 감지

.NET에서는 구성 변경을 실시간으로 감지할 수 있다:

```csharp
// IOptionsMonitor를 사용한 실시간 구성 변경 감지
public class ConfigurableService
{
    private readonly IOptionsMonitor<DatabaseSettings> _optionsMonitor;

    public ConfigurableService(IOptionsMonitor<DatabaseSettings> optionsMonitor)
    {
        _optionsMonitor = optionsMonitor;
        
        // 구성 변경 시 콜백 등록
        _optionsMonitor.OnChange(OnSettingsChanged);
    }

    private void OnSettingsChanged(DatabaseSettings newSettings)
    {
        // 구성이 변경되었을 때 실행될 로직
        Console.WriteLine($"Connection string changed to: {newSettings.ConnectionString}");
    }

    public DatabaseSettings GetCurrentSettings()
    {
        return _optionsMonitor.CurrentValue;
    }
}
```

## 참고자료

- [.NET Configuration](https://learn.microsoft.com/dotnet/core/extensions/configuration)
- [Options pattern in .NET](https://learn.microsoft.com/dotnet/core/extensions/options)
- [.NET Environments](https://learn.microsoft.com/aspnet/core/fundamentals/environments)
- [Configuration in ASP.NET Core](https://learn.microsoft.com/aspnet/core/fundamentals/configuration/)
