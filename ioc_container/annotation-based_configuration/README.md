# Attribute-based configuration

## 개요

.NET 8 LTS에서는 의존성 주입을 위해 다양한 특성(Attributes)을 사용할 수 있습니다.  
.NET Core 1.0부터 내장 DI 컨테이너가 도입되었고, .NET 5부터는 더욱 강화된 기능들이 추가되었으며, .NET 6 이상에서는 소스 생성기(Source Generators)를 활용한 컴파일 타임 의존성 주입 검증이 가능해졌습니다.

Spring Framework의 `@Autowired`와 달리, .NET은 **생성자 주입을 우선하는 철학**을 가지고 있으며, 특성 기반 주입은 제한적이고 특수한 상황에서만 사용됩니다.

## 설명

.NET의 의존성 주입은 **생성자 주입을 기본 원칙**으로 하며, 컴파일 타임 안전성과 명시적 의존성 선언을 중시합니다.  
특성을 통한 주입(`[FromServices]`, `[FromKeyedServices]`, `[Required]` 등)은 ASP.NET Core의 특정 시나리오나 속성 주입이 필요한 경우에만 제한적으로 사용됩니다.

이러한 접근 방식은 Spring의 유연한 어노테이션 기반 주입과는 근본적으로 다른 설계 철학을 반영합니다.

```csharp
// Program.cs 또는 Startup.cs에서 서비스 등록
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddScoped<IMovieFinder, MovieFinder>();
builder.Services.AddScoped<IMovieService, MovieService>();

var app = builder.Build();
```

### Spring의 @Autowired vs .NET의 접근 방식

.NET과 Spring의 의존성 주입 방식에는 근본적인 철학적 차이가 있습니다.

#### Spring의 @Autowired 방식

Spring에서는 필드, 메서드, 생성자에 `@Autowired`를 사용하여 유연한 의존성 주입을 제공합니다:

```java
// Spring 예시 (Java)
@Component
public class MovieRecommender {
    @Autowired
    private MovieCatalog movieCatalog;  // 필드 주입
    
    @Autowired
    public void setCustomerPreferenceDao(CustomerPreferenceDao customerPreferenceDao) {
        this.customerPreferenceDao = customerPreferenceDao;  // 세터 주입
    }
}
```

#### .NET의 특성 기반 주입

.NET에서는 특성(Attributes)이 제한적으로 사용되며, 주로 특수한 상황에서만 활용됩니다:

```csharp
public class MovieController : ControllerBase
{
    // ASP.NET Core의 컨트롤러 액션에서만 사용 가능
    public IActionResult GetMovies([FromServices] IMovieService movieService)
    {
        return Ok(movieService.GetAllMovies());
    }
}

public class MovieRecommender
{
    [Required]  // 속성 주입 (제한적 사용)
    public IMovieCatalog MovieCatalog { get; set; }
}
```

#### 주요 차이점

| 구분 | Spring (@Autowired) | .NET (생성자 우선) |
|------|---------------------|-------------------|
| **설계 철학** | 유연성과 편의성 중시 | 안전성과 명시성 중시 |
| **지원 범위** | 필드, 메서드, 생성자 전반 | 생성자 주입 우선, 특성은 제한적 |
| **기본 방식** | 필드/메서드 주입 선호 | 생성자 주입 강력 권장 |
| **컴파일 타임 검증** | 런타임에 검증 | 생성자 주입 시 컴파일 타임 검증 |
| **테스트 용이성** | Mock 객체 설정 복잡 | 생성자 주입으로 쉬운 테스트 |
| **불변성** | 필드 변경 가능 | readonly 필드로 불변성 보장 |

### .NET에서 생성자 주입을 우선하는 이유

.NET이 생성자 주입을 권장하는 데에는 명확한 기술적 이유들이 있습니다.

#### 1. **불변성 보장 (Immutability)**

생성자 주입은 의존성을 readonly 필드로 선언할 수 있어 객체의 불변성을 보장합니다:

```csharp
public class MovieService
{
    private readonly IMovieRepository _movieRepository;  // 불변 필드
    
    public MovieService(IMovieRepository movieRepository)
    {
        _movieRepository = movieRepository ?? throw new ArgumentNullException(nameof(movieRepository));
    }
    
    // _movieRepository는 생성 후 변경 불가
}
```

#### 2. **컴파일 타임 안전성**

생성자 주입은 컴파일 시점에 의존성을 검증할 수 있습니다:

```csharp
// 컴파일 타임에 의존성 누락 감지
var movieService = new MovieService(); // ❌ 컴파일 오류
var movieService = new MovieService(movieRepository); // ✅ 정상
```

#### 3. **명시적 의존성 선언**

생성자를 통해 클래스의 의존성을 명확하게 드러냅니다:

```csharp
// 생성자를 보면 이 클래스의 모든 의존성을 한눈에 파악 가능
public class MovieRecommender
{
    private readonly IMovieCatalog _movieCatalog;
    private readonly ICustomerPreferenceDao _customerPreferenceDao;

    public MovieRecommender(
        IMovieCatalog movieCatalog,
        ICustomerPreferenceDao customerPreferenceDao)
    {
        _movieCatalog = movieCatalog;
        _customerPreferenceDao = customerPreferenceDao;
        // 모든 의존성이 명시적으로 드러남
    }

    // ...existing code...
}
```

#### 4. **쉬운 단위 테스트**

생성자 주입은 Mock 객체를 쉽게 주입할 수 있어 테스트가 용이합니다:

```csharp
[Test]
public void Should_Return_Recommendations()
{
    // Arrange
    var mockCatalog = new Mock<IMovieCatalog>();
    var mockDao = new Mock<ICustomerPreferenceDao>();
    var movieRecommender = new MovieRecommender(mockCatalog.Object, mockDao.Object);
    
    // Act & Assert
    // 쉬운 테스트 작성
}
```

#### 5. **순환 의존성 방지**

생성자 주입은 순환 의존성을 컴파일 타임에 감지할 수 있습니다:

```csharp
// 순환 의존성이 있다면 컴파일 시점에 감지됨
public class ServiceA
{
    public ServiceA(ServiceB serviceB) { }  // ServiceB를 의존
}

public class ServiceB
{
    public ServiceB(ServiceA serviceA) { }  // ServiceA를 의존 - 순환 의존성!
}
```

#### 6. **Fail-Fast 원칙**

생성자 주입은 애플리케이션 시작 시점에 모든 의존성 문제를 발견할 수 있습니다:

```csharp
// 애플리케이션 시작 시 모든 의존성이 해결되지 않으면 즉시 실패
var app = builder.Build();  // 여기서 의존성 문제 발견
```

### 생성자 주입 (Constructor Injection)

.NET에서 가장 권장되는 의존성 주입 방식은 생성자 주입입니다.

```csharp
public class SimpleMovieLister
{
    private readonly IMovieFinder _movieFinder;

    public SimpleMovieLister(IMovieFinder movieFinder)
    {
        _movieFinder = movieFinder ?? throw new ArgumentNullException(nameof(movieFinder));
    }

    // ...existing code...
}
```

### 특성 기반 주입의 제한적 사용 사례

.NET에서 특성 기반 주입은 생성자 주입이 적합하지 않은 특수한 경우에만 사용됩니다:

```csharp
// 1. ASP.NET Core 컨트롤러 액션 (메서드 파라미터)
public class MoviesController : ControllerBase
{
    public IActionResult GetMovies([FromServices] IMovieService movieService)
    {
        return Ok(movieService.GetAllMovies());
    }
}

// 2. Minimal API (람다 파라미터)
app.MapGet("/movies", ([FromServices] IMovieService movieService) => 
    movieService.GetAllMovies());

// 3. 키 기반 서비스 (특정 구현체 선택)
public class MovieProcessor
{
    public MovieProcessor([FromKeyedServices("premium")] IMovieService movieService)
    {
        // 특정 키로 등록된 서비스만 주입
    }
}
```

### Required 속성을 통한 속성 주입

.NET 7부터 도입된 `[Required]` 특성을 사용하여 필수 속성을 지정할 수 있습니다. 단, 이는 생성자 주입이 불가능한 특수한 상황에서만 사용하는 것이 권장됩니다:

```csharp
public class SimpleMovieLister
{
    [Required]
    public IMovieFinder MovieFinder { get; set; }

    // 주의: 이 방식은 생성자 주입을 사용할 수 없는 특별한 경우에만 사용
}
```

### 컬렉션 주입

.NET DI 컨테이너는 자동으로 같은 인터페이스를 구현한 모든 서비스를 컬렉션으로 주입할 수 있습니다.

```csharp
public class MovieRecommender
{
    private readonly IEnumerable<IMovieCatalog> _movieCatalogs;

    public MovieRecommender(IEnumerable<IMovieCatalog> movieCatalogs)
    {
        _movieCatalogs = movieCatalogs;
    }

    // ...existing code...
}
```

### 키 기반 서비스 주입

.NET 8에서 도입된 키 기반 서비스를 사용하여 특정 구현체를 선택할 수 있습니다.

```csharp
public class MovieRecommender
{
    private readonly IMovieCatalog _mainMovieCatalog;

    public MovieRecommender([FromKeyedServices("main")] IMovieCatalog movieCatalog)
    {
        _mainMovieCatalog = movieCatalog;
    }

    // ...existing code...
}
```

키 기반 서비스 등록:

```csharp
builder.Services.AddKeyedScoped<IMovieCatalog, MainMovieCatalog>("main");
builder.Services.AddKeyedScoped<IMovieCatalog, ActionMovieCatalog>("action");
```

### 옵션 패턴

.NET에서는 구성 값 주입을 위해 옵션 패턴을 사용합니다.

```csharp
public class MovieSettings
{
    public string ApiUrl { get; set; }
    public int TimeoutSeconds { get; set; }
}

public class MovieService
{
    private readonly MovieSettings _settings;

    public MovieService(IOptions<MovieSettings> options)
    {
        _settings = options.Value;
    }

    // ...existing code...
}
```

appsettings.json 구성:

```json
{
  "MovieSettings": {
    "ApiUrl": "https://api.movies.com",
    "TimeoutSeconds": 30
  }
}
```

서비스 등록:

```csharp
builder.Services.Configure<MovieSettings>(builder.Configuration.GetSection("MovieSettings"));
```

### 생명주기 관리

.NET에서는 `IDisposable` 및 `IAsyncDisposable`을 구현하여 리소스 정리를 수행합니다.

```csharp
public class CachingMovieLister : IDisposable, IAsyncDisposable
{
    private readonly IMemoryCache _cache;
    private bool _disposed;

    public CachingMovieLister(IMemoryCache cache)
    {
        _cache = cache;
        PopulateMovieCache();
    }

    private void PopulateMovieCache()
    {
        // 초기화 시 영화 캐시를 채움
    }

    public void Dispose()
    {
        if (!_disposed)
        {
            ClearMovieCache();
            _disposed = true;
        }
    }

    public async ValueTask DisposeAsync()
    {
        if (!_disposed)
        {
            await ClearMovieCacheAsync();
            _disposed = true;
        }
    }

    private void ClearMovieCache()
    {
        // 캐시 정리
    }

    private async Task ClearMovieCacheAsync()
    {
        // 비동기 캐시 정리
    }
}
```

### 호스팅된 서비스

백그라운드 작업을 위해 `IHostedService` 또는 `BackgroundService`를 사용합니다.

```csharp
public class MovieCacheService : BackgroundService
{
    private readonly IServiceProvider _serviceProvider;
    private readonly ILogger<MovieCacheService> _logger;

    public MovieCacheService(IServiceProvider serviceProvider, ILogger<MovieCacheService> logger)
    {
        _serviceProvider = serviceProvider;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            using var scope = _serviceProvider.CreateScope();
            var movieService = scope.ServiceProvider.GetRequiredService<IMovieService>();
            
            // 주기적인 캐시 업데이트 작업
            await movieService.UpdateCacheAsync();
            
            await Task.Delay(TimeSpan.FromHours(1), stoppingToken);
        }
    }
}
```

## 참고자료

- [.NET Dependency injection](https://learn.microsoft.com/dotnet/core/extensions/dependency-injection)
- [Options pattern in .NET](https://learn.microsoft.com/dotnet/core/extensions/options)
- [Keyed services in .NET 8](https://learn.microsoft.com/dotnet/core/extensions/dependency-injection#keyed-services)
