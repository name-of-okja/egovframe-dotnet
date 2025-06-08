# AOP 서비스

## 개요

AOP 서비스는 관점지향 프로그래밍(Aspect Oriented Programming: AOP) 사상을 구현하고 지원한다. .NET 8 LTS 기반 실행환경 AOP 서비스는 ASP.NET Core 미들웨어와 정적 소스 생성기(Source Generator)를 활용한다. 본 장에서는 AOP의 개요 및 .NET 8에서의 AOP 구현 방법을 중심으로 살펴본다.

## 설명

### AOP 개요

개별 프로그래밍 언어는 프로그램 개발을 위해 고유한 관심사 분리(Separation of Concerns) 패러다임을 갖는다. 예를 들면 절차적 프로그래밍은 상태값을 갖지 않는 연속된 함수들의 실행을 프로그램으로 이해하고 모듈을 주요 분리 단위로 정의한다. 객체지향 프로그래밍은 일련의 함수 실행이 아닌 상호작용하는 객체들의 집합으로 보며 클래스를 주요 단위로 한다.

객체지향 프로그래밍은 많은 장점에도 불구하고, 다수의 객체들에 분산되어 중복적으로 존재하는 공통 관심사가 존재한다. 이들은 프로그램을 복잡하게 만들고, 코드의 변경을 어렵게 한다.

관점 지향 프로그래밍(AOP, Aspect-oriented programming)은 이러한 객체지향 프로그래밍의 문제점을 보완하는 방법으로 핵심 관심사를 분리하여 프로그램 모듈화를 향상시키는 프로그래밍 스타일이다. AOP는 객체를 핵심 관심사와 횡단 관심사로 분리하고, 횡단 관심사를 관점(Aspect)이라는 모듈로 정의하고 핵심 관심사와 엮어서 처리할 수 있는 방법을 제공한다.

- **관점(Aspect)**: 프로그램의 핵심 관심사에 걸쳐 적용되는 공통 프로그램 영역을 의미한다. 예를 들면 로깅, 인증, 권한확인, 트랜잭션은 비즈니스 기능 구현 시에 공통적으로 적용되는 요소이며 하나의 관점으로 정의될 수 있다.
- **핵심 관심사(Core concern)**: 프로그램을 작성하려는 핵심 가치와 목적이 드러난 관심 영역으로 보통 핵심 비즈니스 기능에 해당한다.
- **횡단 관심사(Cross-cutting concern)**: 핵심 관심에 영향을 주는 프로그램의 영역으로, 로깅과 트랜잭션, 인증처리와 같은 시스템 공통 처리 영역이 해당된다.

다음 그림은 객체지향 프로그래밍 개발에서 핵심 관심사와 횡단 관심사가 하나의 코드로 통합되어 개발된 사례를 보여준다.

![AOP 개념 다이어그램 1](images/aop-concept-01.png)

객체지향 프로그래밍 코드에 AOP를 적용하면 다음 그림처럼 각 코드에 분산되어 있던 횡단 관심사는 관점으로 분리되어 정의된다. AOP는 엮기(Weaving)라는 방식을 이용하여 분리된 관점을 핵심 관심사와 엮는다.

![AOP 개념 다이어그램 2](images/aop-concept-02.png)

### AOP 주요 개념

관점 지향 프로그래밍은 횡단 관심사를 분리하고 핵심 관심사와 엮어 사용할 수 있는 방법을 제공하며 다음의 몇 가지 새로운 개념을 포함한다.

#### 관점(Aspect)

관점은 구현하고자 하는 횡단 관심사의 기능이다. .NET에서는 어트리뷰트(Attribute), 미들웨어(Middleware), 또는 인터셉터(Interceptor) 형태로 구현된다.

#### 결합점(Join point)

결합점은 관점(Aspect)를 삽입하여 실행 가능한 애플리케이션의 특정 지점을 말한다. .NET에서는 메서드 호출, 속성 접근, 예외 발생 등이 결합점이 될 수 있다.

#### 포인트컷(Pointcut)

포인트컷은 결합점 집합을 의미한다. .NET에서는 어트리뷰트를 통한 메타데이터 기반 선택이나 네임스페이스, 타입명 패턴 매칭을 통해 정의한다.

다음 그림은 .NET 8에서 어트리뷰트와 소스 생성기를 이용하여 횡단 관심사를 적용하는 예제를 보여준다.

![포인트컷 예제](images/dotnet-aop-pointcut.png)

#### 충고(Advice)

충고(Advice)는 관점(Aspect)의 실제 구현체로 결합점에 삽입되어 동작할 수 있는 코드이다. .NET 8에서는 다음과 같은 방식으로 구현된다:

- **Before advice**: 메서드 실행 전에 수행되는 코드 (`OnMethodExecuting` 구현)
- **After returning advice**: 메서드가 성공적으로 완료된 후에 동작하는 코드 (`OnMethodExecuted` 구현)
- **After throwing advice**: 예외가 발생했을 때 수행되는 코드 (`OnException` 구현)
- **After (finally) advice**: 메서드 실행 완료 여부에 상관없이 수행되는 코드 (`finally` 블록 활용)
- **Around advice**: 메서드 실행 전후를 모두 제어하는 코드 (미들웨어 패턴 활용)

#### 엮기(Weaving)

.NET 8에서는 다음과 같은 엮기 방식을 지원한다:

- **컴파일 시 엮기**: 정적 소스 생성기(Source Generator)를 통해 컴파일 시점에 횡단 관심사 코드가 생성되어 포함되는 방식
- **런타임 엮기**:
  - 미들웨어 파이프라인을 통한 요청/응답 처리 시점에서의 AOP 적용
  - 동적 프록시나 인터셉터를 이용한 런타임 위빙
  - Dependency Injection 컨테이너의 데코레이터 패턴 활용

#### 도입(Introduction)

.NET에서는 확장 메서드(Extension Methods)나 인터페이스 믹스인을 통해 기존 타입에 새로운 기능을 추가할 수 있다.

#### AOP 프록시(Proxy)

.NET에서는 `DispatchProxy` 클래스나 `Castle.DynamicProxy` 같은 라이브러리를 통해 동적 프록시를 생성할 수 있다.

#### 대상 객체(Target Object)

대상 객체는 충고(Advice)를 받는 객체이다. .NET의 DI 컨테이너에서는 인터페이스 기반의 프록시 객체가 주입된다.

### .NET 8에서의 AOP 구현 방법

.NET 8 LTS에서는 다음과 같은 두 가지 주요 방식으로 AOP를 구현할 수 있다.

#### ASP.NET Core 미들웨어를 이용한 AOP 구현

ASP.NET Core의 미들웨어 파이프라인을 활용하여 횡단 관심사를 구현하는 방식이다.

```csharp
// 로깅 미들웨어 예제
public class LoggingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<LoggingMiddleware> _logger;

    public LoggingMiddleware(RequestDelegate next, ILogger<LoggingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        // Before advice
        _logger.LogInformation($"Request: {context.Request.Method} {context.Request.Path}");
        
        try
        {
            // 핵심 로직 실행
            await _next(context);
            
            // After returning advice
            _logger.LogInformation($"Response: {context.Response.StatusCode}");
        }
        catch (Exception ex)
        {
            // After throwing advice
            _logger.LogError(ex, "Exception occurred during request processing");
            throw;
        }
    }
}
```

미들웨어 등록:

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.UseMiddleware<LoggingMiddleware>();
app.MapControllers();

app.Run();
```

**특징:**

- HTTP 요청/응답 처리에 특화
- 파이프라인 기반의 순차적 처리
- 의존성 주입과 자연스럽게 통합
- 성능 최적화가 용이

#### 정적 소스 생성기를 통한 AOP 구현

컴파일 시점에 코드를 생성하여 AOP를 구현하는 방식이다.

```csharp
// AOP 어트리뷰트 정의
[AttributeUsage(AttributeTargets.Method)]
public class LogExecutionTimeAttribute : Attribute
{
}

// 소스 생성기에서 생성할 코드 예제
[LogExecutionTime]
public partial class UserService
{
    public async Task<User> GetUserAsync(int id)
    {
        // 비즈니스 로직
        return await _userRepository.GetByIdAsync(id);
    }
}

// 생성된 코드 (소스 생성기에 의해 자동 생성)
public partial class UserService
{
    public async Task<User> GetUserAsync(int id)
    {
        var stopwatch = System.Diagnostics.Stopwatch.StartNew();
        try
        {
            var result = await GetUserAsyncCore(id);
            return result;
        }
        finally
        {
            stopwatch.Stop();
            _logger.LogInformation($"Method GetUserAsync executed in {stopwatch.ElapsedMilliseconds}ms");
        }
    }

    private async Task<User> GetUserAsyncCore(int id)
    {
        // 원본 비즈니스 로직
        return await _userRepository.GetByIdAsync(id);
    }
}
```

**특징:**

- 컴파일 시점에 코드 생성으로 런타임 오버헤드 최소화
- 강타입 지원으로 컴파일 타임 안전성 확보
- 디버깅 시 생성된 코드 확인 가능
- 복잡한 로직도 효율적으로 처리 가능

### 의존성 주입과 AOP 통합

.NET 8의 DI 컨테이너와 AOP를 통합하여 사용하는 방법:

```csharp
// 서비스 등록
builder.Services.AddScoped<IUserService, UserService>();
builder.Services.Decorate<IUserService, CachedUserService>(); // 데코레이터 패턴
builder.Services.AddSingleton<IMemoryCache, MemoryCache>();

// 캐싱 데코레이터 구현
public class CachedUserService : IUserService
{
    private readonly IUserService _inner;
    private readonly IMemoryCache _cache;

    public CachedUserService(IUserService inner, IMemoryCache cache)
    {
        _inner = inner;
        _cache = cache;
    }

    public async Task<User> GetUserAsync(int id)
    {
        var cacheKey = $"user_{id}";
        if (_cache.TryGetValue(cacheKey, out User cachedUser))
        {
            return cachedUser;
        }

        var user = await _inner.GetUserAsync(id);
        _cache.Set(cacheKey, user, TimeSpan.FromMinutes(5));
        return user;
    }
}
```

### 성능 고려사항

.NET 8에서 AOP 구현 시 성능 최적화 방안:

1. **미들웨어 방식**: HTTP 파이프라인에서 한 번만 실행되므로 오버헤드가 적음
2. **소스 생성기 방식**: 컴파일 시점 코드 생성으로 런타임 오버헤드 없음
3. **동적 프록시 방식**: 리플렉션 사용으로 상대적으로 느리지만 유연함

### 실행환경 AOP 가이드라인

- [.NET AOP 구현 가이드라인](./dotnet-aop-guidelines/index.md)
- [미들웨어 기반 AOP 패턴](./middleware-aop-patterns/index.md)
- [소스 생성기 AOP 구현](./source-generator-aop/index.md)

## 참고자료

- [ASP.NET Core 미들웨어](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/middleware/)
- [.NET Source Generators](https://docs.microsoft.com/en-us/dotnet/csharp/roslyn-sdk/source-generators-overview)
- [Dependency Injection in .NET](https://docs.microsoft.com/en-us/dotnet/core/extensions/dependency-injection)
- [Performance Best Practices for .NET 8](https://docs.microsoft.com/en-us/dotnet/core/whats-new/dotnet-8)
