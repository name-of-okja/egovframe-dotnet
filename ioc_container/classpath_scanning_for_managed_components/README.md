# Assembly scanning for managed components

## 개요

.NET 8 LTS에서는 의존성 주입 컨테이너에 서비스를 등록하기 위해 다양한 방법을 제공합니다. 이전 section [Attribute-based configuration](../annotation-based_configuration/index.md)에서는 특성(Attributes)을 사용한 의존성 주입을 다뤘습니다.

.NET에서는 Spring의 XML 설정과 달리 **코드 기반 설정**을 기본으로 하며, **어셈블리 스캐닝**과 **리플렉션**을 통해 서비스를 자동으로 발견하고 등록할 수 있습니다. 하지만 **.NET의 기본 철학은 명시적 서비스 등록**이며, 특히 **Native AOT**와 같은 현대적 배포 시나리오에서는 리플렉션 기반 접근법이 제한될 수 있습니다.

이번 섹션에서는 어셈블리를 검색하고 **필터링**을 통해 **대상 컴포넌트(candidate components)**를 검출하는 방법을 소개하되, 각 접근법의 장단점과 적절한 사용 시나리오를 함께 설명합니다.

## .NET의 서비스 등록 철학: 명시적 등록 우선

.NET은 Spring Framework와 달리 **명시적 서비스 등록**을 기본 철학으로 합니다. 이는 다음과 같은 이유 때문입니다:

### 컴파일 타임 안전성

```csharp
// Program.cs - 권장하는 명시적 등록 방식
var builder = WebApplication.CreateBuilder(args);

// 컴파일 타임에 타입 안전성 보장
builder.Services.AddScoped<IMovieFinder, EntityFrameworkMovieFinder>();
builder.Services.AddScoped<IMovieService, MovieService>();
builder.Services.AddSingleton<IMovieCache, RedisMovieCache>();

var app = builder.Build();
```

### Native AOT 호환성

.NET 8의 Native AOT는 애플리케이션 시작 성능과 메모리 사용량을 크게 개선하지만, 리플렉션에 제약이 있습니다:

```csharp
// Native AOT 친화적 - 명시적 등록
builder.Services.AddScoped<IMovieFinder>(provider => 
    new EntityFrameworkMovieFinder(provider.GetRequiredService<MovieDbContext>()));

// Native AOT에서 문제가 될 수 있음 - 리플렉션 기반 스캐닝
// builder.Services.ScanAndRegisterServices(typeof(Program).Assembly);
```

## Native AOT와 리플렉션 제약의 기술적 배경

**Native AOT (Ahead-of-Time) 컴파일**은 .NET 애플리케이션을 기계어로 미리 컴파일하여 런타임 없이 실행 가능한 네이티브 바이너리를 생성합니다. 이 과정에서 다음과 같은 근본적인 변화가 발생합니다:

### 정적 컴파일과 메타데이터 제거

```csharp
// 일반 .NET (JIT 컴파일)
// 런타임에 모든 타입 메타데이터가 메모리에 존재
Type movieServiceType = typeof(MovieService);
var properties = movieServiceType.GetProperties(); // ✅ 항상 작동

// Native AOT
// 컴파일 시점에 사용되지 않는 메타데이터는 제거됨
Type movieServiceType = typeof(MovieService);
var properties = movieServiceType.GetProperties(); // ❌ 메타데이터가 없으면 실패
```

### 트리 셰이킹과 코드 제거

Native AOT 컴파일러는 실제로 사용되는 코드만 최종 바이너리에 포함시킵니다:

```csharp
// 컴파일 시점에 호출되지 않는 것으로 판단되는 코드
public class UnusedService 
{
    public void UnusedMethod() { } // 트리 셰이킹으로 제거될 수 있음
}

// 어셈블리 스캐닝으로 발견되어야 하는 서비스
[Service]
public class DiscoveredService 
{
    // 컴파일러가 "사용되지 않음"으로 판단하여 메타데이터 제거 가능
}
```

### 어셈블리 스캐닝 실패 사례

```csharp
// 이 코드는 Native AOT에서 예상과 다르게 작동할 수 있음
public static IServiceCollection ScanAndRegisterServices(this IServiceCollection services)
{
    var assembly = Assembly.GetExecutingAssembly();
    
    // ❌ Native AOT에서 문제가 되는 부분들:
    var types = assembly.GetTypes(); // 일부 타입 누락 가능
    
    foreach (var type in types)
    {
        // ❌ 속성 메타데이터가 제거되어 null 반환 가능
        var attribute = type.GetCustomAttribute<ServiceAttribute>();
        
        if (attribute != null)
        {
            // ❌ 생성자 메타데이터가 없어 실패 가능
            var constructor = type.GetConstructors().FirstOrDefault();
            
            // ❌ 인터페이스 정보가 제거되어 빈 배열 반환 가능
            var interfaces = type.GetInterfaces();
        }
    }
}
```

### 성능과 크기 비교

```csharp
// 명시적 등록 (권장)
// - 바이너리 크기: ~15MB
// - 시작 시간: ~50ms
// - 메모리 사용량: ~20MB
builder.Services.AddScoped<IMovieService, MovieService>();

// 리플렉션 기반 스캐닝 + Native AOT
// - 바이너리 크기: ~45MB (메타데이터 보존으로 인한 증가)
// - 시작 시간: ~200ms (리플렉션 오버헤드)
// - 메모리 사용량: ~60MB
builder.Services.ScanAndRegisterServices();
```

### Source Generator 대안

Native AOT 호환성을 유지하면서 자동화를 원한다면 Source Generator를 사용할 수 있습니다:

```csharp
// 컴파일 시점에 생성되는 코드 (Source Generator)
public static partial class GeneratedServiceRegistration
{
    public static IServiceCollection AddGeneratedServices(this IServiceCollection services)
    {
        // 컴파일 시점에 자동 생성됨
        services.AddScoped<IMovieService, MovieService>();
        services.AddScoped<IMovieFinder, EntityFrameworkMovieFinder>();
        return services;
    }
}
```

이러한 기술적 제약으로 인해 .NET Native AOT 환경에서는 **명시적 서비스 등록**이 강력히 권장됩니다.

### 성능 최적화

명시적 등록은 애플리케이션 시작 시 리플렉션 오버헤드를 제거합니다:

```csharp
// 빠른 시작 - 리플렉션 없음
builder.Services.AddScoped<IMovieService, MovieService>();

// 느린 시작 - 어셈블리 스캐닝 필요
// builder.Services.ScanAndRegisterServices();
```

### 명확한 의존성 추적

```csharp
// 한눈에 보이는 모든 서비스 등록
builder.Services.AddScoped<IMovieFinder, EntityFrameworkMovieFinder>();
builder.Services.AddScoped<IMovieService, MovieService>();
builder.Services.AddScoped<INotificationService, EmailNotificationService>();
// 프로젝트의 모든 의존성이 명시적으로 드러남
```

## 권장 사항: 계층별 서비스 등록 패턴

.NET에서는 다음과 같은 패턴으로 서비스를 구조화하는 것이 권장됩니다:

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// 계층별로 구분된 서비스 등록
builder.Services.AddRepositories();
builder.Services.AddDomainServices();
builder.Services.AddApplicationServices();
builder.Services.AddInfrastructureServices();

var app = builder.Build();
```

```csharp
// ServiceCollectionExtensions.cs
public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddRepositories(this IServiceCollection services)
    {
        services.AddScoped<IMovieRepository, EntityFrameworkMovieRepository>();
        services.AddScoped<IUserRepository, EntityFrameworkUserRepository>();
        return services;
    }

    public static IServiceCollection AddDomainServices(this IServiceCollection services)
    {
        services.AddScoped<IMovieService, MovieService>();
        services.AddScoped<IUserService, UserService>();
        return services;
    }

    public static IServiceCollection AddApplicationServices(this IServiceCollection services)
    {
        services.AddScoped<IMovieApplicationService, MovieApplicationService>();
        return services;
    }

    public static IServiceCollection AddInfrastructureServices(this IServiceCollection services)
    {
        services.AddSingleton<IEmailService, SendGridEmailService>();
        services.AddSingleton<ICacheService, RedisCacheService>();
        return services;
    }
}
```

## 자동 스캐닝을 고려할 수 있는 시나리오

다음과 같은 특수한 경우에는 자동 스캐닝을 고려할 수 있습니다:

- **레거시 Spring 애플리케이션 마이그레이션**
- **플러그인 아키텍처 구현**
- **런타임 어셈블리 로딩이 필요한 경우**

**단, 이러한 경우에도 Native AOT 호환성을 포기해야 함을 인지해야 합니다.**

## .NET의 서비스 등록 방식 (참고용)

.NET에서는 Spring의 `@Component`, `@Service`, `@Repository` 어노테이션과 달리 **명시적 서비스 등록**을 기본으로 합니다. 하지만 **사용자 정의 특성**과 **리플렉션**을 통해 자동 등록을 구현할 수 있습니다.

**⚠️ 주의: 다음 예제는 교육 목적이며, 프로덕션에서는 명시적 등록을 권장합니다.**

```csharp
// 사용자 정의 서비스 특성들
[AttributeUsage(AttributeTargets.Class)]
public class ServiceAttribute : Attribute
{
    public ServiceLifetime Lifetime { get; }
    public string? Name { get; }

    public ServiceAttribute(ServiceLifetime lifetime = ServiceLifetime.Scoped, string? name = null)
    {
        Lifetime = lifetime;
        Name = name;
    }
}

[AttributeUsage(AttributeTargets.Class)]
public class RepositoryAttribute : ServiceAttribute
{
    public RepositoryAttribute(ServiceLifetime lifetime = ServiceLifetime.Scoped, string? name = null) 
        : base(lifetime, name) { }
}

[AttributeUsage(AttributeTargets.Class)]
public class ControllerServiceAttribute : ServiceAttribute
{
    public ControllerServiceAttribute(ServiceLifetime lifetime = ServiceLifetime.Scoped, string? name = null) 
        : base(lifetime, name) { }
}
```

## 자동 컴포넌트 검색 (참고용)

**⚠️ Native AOT 비호환: 다음 코드는 Native AOT 환경에서 작동하지 않습니다.**

.NET에서는 어셈블리 스캐닝을 통해 특성이 적용된 클래스를 자동으로 탐지하고 DI 컨테이너에 등록할 수 있습니다.

```csharp
[Service]
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

```csharp
[Repository]
public class EntityFrameworkMovieFinder : IMovieFinder
{
    private readonly MovieDbContext _context;

    public EntityFrameworkMovieFinder(MovieDbContext context)
    {
        _context = context;
    }

    // implementation elided for clarity
}
```

실제로 위 두 클래스를 자동 탐지하고 상응하는 서비스를 등록하기 위해서는, `Program.cs`에서 어셈블리 스캐닝을 설정해야 합니다:

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// 자동 서비스 등록을 위한 확장 메서드 호출
builder.Services.ScanAndRegisterServices(typeof(Program).Assembly);

var app = builder.Build();
```

## 어셈블리 스캐닝 구현 (참고용)

**⚠️ 성능 고려사항: 어셈블리 스캐닝은 애플리케이션 시작 성능에 영향을 줄 수 있습니다.**

어셈블리 스캐닝을 구현하는 확장 메서드:

```csharp
public static class ServiceCollectionExtensions
{
    public static IServiceCollection ScanAndRegisterServices(
        this IServiceCollection services, 
        params Assembly[] assemblies)
    {
        foreach (var assembly in assemblies)
        {
            var serviceTypes = assembly.GetTypes()
                .Where(type => type.IsClass && !type.IsAbstract)
                .Where(type => type.GetCustomAttribute<ServiceAttribute>() != null);

            foreach (var serviceType in serviceTypes)
            {
                var serviceAttribute = serviceType.GetCustomAttribute<ServiceAttribute>();
                var interfaces = serviceType.GetInterfaces();

                if (interfaces.Any())
                {
                    foreach (var interfaceType in interfaces)
                    {
                        RegisterService(services, interfaceType, serviceType, serviceAttribute);
                    }
                }
                else
                {
                    RegisterService(services, serviceType, serviceType, serviceAttribute);
                }
            }
        }

        return services;
    }

    private static void RegisterService(
        IServiceCollection services,
        Type serviceType,
        Type implementationType,
        ServiceAttribute attribute)
    {
        var serviceDescriptor = attribute.Lifetime switch
        {
            ServiceLifetime.Singleton => ServiceDescriptor.Singleton(serviceType, implementationType),
            ServiceLifetime.Transient => ServiceDescriptor.Transient(serviceType, implementationType),
            _ => ServiceDescriptor.Scoped(serviceType, implementationType)
        };

        services.Add(serviceDescriptor);
    }
}
```

## 서드파티 라이브러리 활용

실제 자동 스캐닝이 필요한 경우, **Scrutor** 같은 검증된 서드파티 라이브러리를 사용하는 것이 권장됩니다.

```csharp
// NuGet: Scrutor
builder.Services.Scan(scan => scan
    .FromAssemblyOf<Program>()
    .AddClasses(classes => classes.Where(type => 
        type.GetCustomAttribute<ServiceAttribute>() != null))
    .AsImplementedInterfaces()
    .WithScopedLifetime());
```

## Microsoft.Extensions.DependencyInjection.Abstractions의 활용

.NET 8에서는 `[FromKeyedServices]` 특성을 활용한 더 정교한 의존성 주입이 가능합니다:

```csharp
[Service]
public class MovieRecommendationEngine
{
    private readonly IMovieFinder _primaryFinder;
    private readonly IMovieFinder _cachingFinder;

    public MovieRecommendationEngine(
        IMovieFinder primaryFinder,
        [FromKeyedServices("caching")] IMovieFinder cachingFinder)
    {
        _primaryFinder = primaryFinder;
        _cachingFinder = cachingFinder;
    }

    // ...existing code...
}
```

## 결론: .NET 서비스 등록 모범 사례

1. **기본 원칙**: 명시적 서비스 등록 사용
2. **구조화**: 계층별 확장 메서드로 서비스 그룹화
3. **성능**: Native AOT 호환성 고려
4. **예외**: 특수한 경우에만 자동 스캐닝 고려
5. **도구**: 자동 스캐닝이 필요한 경우 검증된 라이브러리 사용

```csharp
// 권장하는 최종 패턴
var builder = WebApplication.CreateBuilder(args);

// 명시적이고 구조화된 서비스 등록
builder.Services.AddDataLayer(builder.Configuration);
builder.Services.AddBusinessLayer();
builder.Services.AddPresentationLayer();

var app = builder.Build();
```

## 참고자료

- [.NET Dependency injection](https://learn.microsoft.com/dotnet/core/extensions/dependency-injection)
- [Native AOT deployment](https://learn.microsoft.com/dotnet/core/deploying/native-aot/)
- [Keyed services in .NET 8](https://learn.microsoft.com/dotnet/core/extensions/dependency-injection#keyed-services)
- [Scrutor - Assembly scanning and decoration extensions for Microsoft.Extensions.DependencyInjection](https://github.com/khellang/Scrutor)
