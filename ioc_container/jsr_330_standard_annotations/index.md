# .NET 표준 의존성 주입 사용하기

## 개요

.NET에서는 Java의 JSR-330과 같은 표준화된 의존성 주입 어노테이션이 별도로 존재하지 않습니다. 대신 **.NET Core 1.0부터 도입된 Microsoft.Extensions.DependencyInjection**이 사실상의 표준으로 자리잡았으며, .NET 8 LTS에서는 이를 기반으로 한 풍부한 의존성 주입 기능을 제공합니다.

Spring의 JSR-330 지원과 달리, .NET은 **단일 표준 DI 컨테이너**를 제공하여 일관성 있는 개발 경험을 보장합니다. 이는 생태계 전반에 걸쳐 동일한 의존성 주입 패턴을 사용할 수 있게 해줍니다.

.NET의 의존성 주입을 사용하기 위해서는 다음 NuGet 패키지가 필요합니다:

```xml
<PackageReference Include="Microsoft.Extensions.DependencyInjection" Version="8.0.0" />
<PackageReference Include="Microsoft.Extensions.Hosting" Version="8.0.0" />
```

## 설명

### 1. 생성자 주입과 FromServices 특성을 이용한 의존성 주입

.NET에서는 **생성자 주입**이 기본이며, 특별한 경우에만 `[FromServices]` 특성을 사용합니다:

```csharp
using Microsoft.Extensions.DependencyInjection;

public class SimpleMovieLister
{
    private readonly IMovieFinder _movieFinder;

    // 생성자 주입 (권장 방식)
    public SimpleMovieLister(IMovieFinder movieFinder)
    {
        _movieFinder = movieFinder ?? throw new ArgumentNullException(nameof(movieFinder));
    }

    public void ListMovies()
    {
        _movieFinder.FindMovies();
        // ...
    }
}
```

메서드 파라미터에서 직접 서비스를 주입받는 경우 (주로 컨트롤러나 Minimal API):

```csharp
using Microsoft.AspNetCore.Mvc;

[ApiController]
public class MoviesController : ControllerBase
{
    // 액션 메서드에서 직접 주입
    [HttpGet]
    public IActionResult GetMovies([FromServices] IMovieFinder movieFinder)
    {
        var movies = movieFinder.FindMovies();
        return Ok(movies);
    }
}

// Minimal API에서의 사용
app.MapGet("/movies", ([FromServices] IMovieFinder movieFinder) => 
    movieFinder.FindMovies());
```

지연 로딩을 위해서는 `Lazy<T>`나 팩토리 패턴을 사용합니다:

```csharp
public class SimpleMovieLister
{
    private readonly Lazy<IMovieFinder> _movieFinder;

    public SimpleMovieLister(Lazy<IMovieFinder> movieFinder)
    {
        _movieFinder = movieFinder;
    }

    public void ListMovies()
    {
        _movieFinder.Value.FindMovies();
        // ...
    }
}
```

### 2. 키 기반 서비스와 명명된 서비스

.NET 8에서는 **키 기반 서비스**를 통해 특정 구현체를 선택할 수 있습니다:

```csharp
using Microsoft.Extensions.DependencyInjection;

public class SimpleMovieLister
{
    private readonly IMovieFinder _movieFinder;

    public SimpleMovieLister([FromKeyedServices("main")] IMovieFinder movieFinder)
    {
        _movieFinder = movieFinder;
    }

    // ...
}
```

서비스 등록 시 키를 지정합니다:

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// 키 기반 서비스 등록
builder.Services.AddKeyedScoped<IMovieFinder, DatabaseMovieFinder>("main");
builder.Services.AddKeyedScoped<IMovieFinder, CacheMovieFinder>("cache");

var app = builder.Build();
```

옵션 패턴을 통한 명명된 구성도 가능합니다:

```csharp
public class MovieSettings
{
    public string ConnectionString { get; set; } = string.Empty;
    public int TimeoutSeconds { get; set; }
}

public class SimpleMovieLister
{
    private readonly MovieSettings _settings;

    public SimpleMovieLister(IOptionsSnapshot<MovieSettings> options)
    {
        _settings = options.Get("MovieDatabase"); // 명명된 옵션
    }

    // ...
}
```

### 3. 서비스 등록과 컴포넌트 스캐닝

.NET에서는 **명시적 서비스 등록**이 기본입니다:

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// 명시적 서비스 등록
builder.Services.AddScoped<IMovieFinder, DatabaseMovieFinder>();
builder.Services.AddScoped<SimpleMovieLister>();

// 구성 옵션 등록
builder.Services.Configure<MovieSettings>("MovieDatabase", 
    builder.Configuration.GetSection("MovieDatabase"));

var app = builder.Build();
```

계층별 서비스 등록을 위한 확장 메서드:

```csharp
public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddMovieServices(this IServiceCollection services)
    {
        services.AddScoped<IMovieFinder, DatabaseMovieFinder>();
        services.AddScoped<SimpleMovieLister>();
        return services;
    }
}

// 사용
builder.Services.AddMovieServices();
```

### 4. .NET 의존성 주입의 특징과 제약사항

Java JSR-330과 .NET의 의존성 주입을 비교하면 다음과 같습니다:

| 기능 | JSR-330 | .NET DI | .NET의 특징 및 제약사항 |
|------|---------|---------|------------------------|
| 기본 주입 방식 | @Inject | 생성자 주입 | .NET은 생성자 주입을 강력히 권장하며, 컴파일 타임 안전성 제공 |
| 컴포넌트 표시 | @Named / @Component | 명시적 등록 | .NET은 어노테이션 기반 자동 스캐닝보다 명시적 등록을 선호 |
| 스코프 관리 | @Singleton / @Scope | ServiceLifetime | .NET은 Singleton, Scoped, Transient 세 가지 기본 생명주기 제공 |
| 한정자 | @Qualifier / @Named | FromKeyedServices | .NET 8에서 키 기반 서비스로 특정 구현체 선택 가능 |
| 옵션 주입 | @Value 상당 | IOptions&lt;T&gt; | .NET은 강타입 구성 객체를 통한 옵션 패턴 제공 |
| 필수 의존성 | @Inject(required) | 생성자 주입 | .NET은 생성자 주입으로 필수 의존성을 자연스럽게 표현 |
| 지연 로딩 | Provider&lt;T&gt; | Lazy&lt;T&gt; / Factory | .NET은 Lazy&lt;T&gt;나 팩토리 패턴으로 지연 로딩 구현 |

### 5. .NET 8의 고급 의존성 주입 기능

#### 키 기반 서비스의 고급 활용

```csharp
public interface INotificationService
{
    Task SendAsync(string message);
}

public class EmailNotificationService : INotificationService
{
    public Task SendAsync(string message) => Task.CompletedTask;
}

public class SmsNotificationService : INotificationService
{
    public Task SendAsync(string message) => Task.CompletedTask;
}

// 등록
builder.Services.AddKeyedScoped<INotificationService, EmailNotificationService>("email");
builder.Services.AddKeyedScoped<INotificationService, SmsNotificationService>("sms");

// 사용
public class NotificationManager
{
    private readonly INotificationService _emailService;
    private readonly INotificationService _smsService;

    public NotificationManager(
        [FromKeyedServices("email")] INotificationService emailService,
        [FromKeyedServices("sms")] INotificationService smsService)
    {
        _emailService = emailService;
        _smsService = smsService;
    }
}
```

#### 조건부 서비스 등록

```csharp
// 환경에 따른 조건부 등록
if (builder.Environment.IsDevelopment())
{
    builder.Services.AddScoped<IMovieFinder, MockMovieFinder>();
}
else
{
    builder.Services.AddScoped<IMovieFinder, DatabaseMovieFinder>();
}

// 구성에 따른 조건부 등록
var useCache = builder.Configuration.GetValue<bool>("UseCache");
if (useCache)
{
    builder.Services.Decorate<IMovieFinder, CachedMovieFinder>();
}
```

#### 팩토리 패턴과 서비스 로케이터

```csharp
public interface IMovieFinderFactory
{
    IMovieFinder Create(string type);
}

public class MovieFinderFactory : IMovieFinderFactory
{
    private readonly IServiceProvider _serviceProvider;

    public MovieFinderFactory(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }

    public IMovieFinder Create(string type)
    {
        return type switch
        {
            "database" => _serviceProvider.GetRequiredKeyedService<IMovieFinder>("database"),
            "cache" => _serviceProvider.GetRequiredKeyedService<IMovieFinder>("cache"),
            _ => throw new ArgumentException($"Unknown type: {type}")
        };
    }
}
```

### 6. 성능과 메모리 최적화

.NET의 DI 컨테이너는 성능을 위해 최적화되었습니다:

```csharp
// 싱글톤으로 등록하여 성능 최적화
builder.Services.AddSingleton<IExpensiveService, ExpensiveService>();

// 미리 빌드된 서비스 제공자 사용
var serviceProvider = builder.Services.BuildServiceProvider();
var service = serviceProvider.GetRequiredService<IMovieFinder>();

// 스코프를 명시적으로 관리
using var scope = serviceProvider.CreateScope();
var scopedService = scope.ServiceProvider.GetRequiredService<IScopedService>();
```

## 결론

.NET은 JSR-330과 같은 별도 표준 없이도 **Microsoft.Extensions.DependencyInjection**을 통해 강력하고 일관된 의존성 주입 경험을 제공합니다.

주요 장점은 다음과 같습니다.

- **단일 표준**: 생태계 전반의 일관성
- **컴파일 타임 안전성**: 생성자 주입 중심의 안전한 설계
- **성능**: Native AOT 지원과 최적화된 런타임
- **유연성**: 키 기반 서비스, 옵션 패턴 등 다양한 기능

.NET의 접근 방식은 **명시적이고 안전한 의존성 관리**를 통해 maintainable한 애플리케이션 개발을 지원합니다.

## 참고자료

- [.NET Dependency injection](https://learn.microsoft.com/dotnet/core/extensions/dependency-injection)
- [Options pattern in .NET](https://learn.microsoft.com/dotnet/core/extensions/options)
- [Keyed services in .NET 8](https://learn.microsoft.com/dotnet/core/extensions/dependency-injection#keyed-services)
- [Configuration in .NET](https://learn.microsoft.com/dotnet/core/extensions/configuration)
