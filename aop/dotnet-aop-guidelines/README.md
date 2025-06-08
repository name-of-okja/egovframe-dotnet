# .NET AOP 구현 가이드라인

## 개요

.NET 8 LTS 기반 실행환경에서 AOP(Aspect Oriented Programming)를 효과적으로 구현하기 위한 가이드라인을 제시한다. 본 문서는 .NET 8의 특성을 활용한 최적의 AOP 구현 방법과 모범 사례를 다룬다.

## .NET 8 AOP 구현 원칙

### 1. 성능 우선 원칙

.NET 8의 성능 최적화를 최대한 활용하여 AOP를 구현한다.

- AOT(Ahead-of-Time) 컴파일 호환성을 고려한 구현
- 최소한의 런타임 오버헤드를 위한 소스 생성기 우선 사용
- 메모리 할당을 최소화하는 구조체 기반 설계

### 2. 타입 안전성 원칙

컴파일 타임에 최대한 많은 오류를 발견할 수 있도록 강타입을 활용한다.

```csharp
// 권장: 강타입 어트리뷰트
[LogExecution]
public async Task<User> GetUserAsync(int userId) { ... }

// 지양: 문자열 기반 설정
[Log("execution")]
public async Task<User> GetUserAsync(int userId) { ... }
```

### 3. 최소 침입 원칙

기존 비즈니스 코드에 최소한의 영향을 주는 방식으로 AOP를 적용한다.

## AOP 구현 전략

### 전략 1: 컴파일 시 위빙 (권장)

소스 생성기를 활용한 컴파일 시점 AOP 구현

**장점:**
- 런타임 오버헤드 없음
- AOT 호환성
- 디버깅 용이성

**적용 시나리오:**
- 로깅, 메트릭 수집
- 입력 값 검증
- 캐싱 로직

```csharp
[LogExecutionTime]
[ValidateInput]
public partial class UserService : IUserService
{
    public async Task<User> GetUserAsync(int id)
    {
        // 비즈니스 로직만 작성
        return await _repository.GetByIdAsync(id);
    }
}
```

### 전략 2: 미들웨어 기반 AOP

HTTP 파이프라인을 활용한 요청 레벨 AOP

**장점:**
- 웹 애플리케이션에 최적화
- 의존성 주입과 자연스러운 통합
- 필터링 및 라우팅 연계 가능

**적용 시나리오:**
- 인증/인가
- 요청/응답 로깅
- 에러 핸들링
- CORS 처리

```csharp
public class SecurityMiddleware
{
    private readonly RequestDelegate _next;
    
    public async Task InvokeAsync(HttpContext context)
    {
        if (!await IsAuthorizedAsync(context))
        {
            context.Response.StatusCode = 401;
            return;
        }
        
        await _next(context);
    }
}
```

### 전략 3: 데코레이터 패턴

의존성 주입 컨테이너를 활용한 런타임 AOP

**장점:**
- 기존 인터페이스 유지
- 조건부 적용 가능
- 테스트 용이성

**적용 시나리오:**
- 캐싱
- 재시도 로직
- 회로 차단기 패턴

```csharp
public class CachedUserService : IUserService
{
    private readonly IUserService _inner;
    private readonly IDistributedCache _cache;
    
    public async Task<User> GetUserAsync(int id)
    {
        var cacheKey = $"user:{id}";
        var cached = await _cache.GetStringAsync(cacheKey);
        
        if (cached != null)
            return JsonSerializer.Deserialize<User>(cached);
            
        var user = await _inner.GetUserAsync(id);
        await _cache.SetStringAsync(cacheKey, JsonSerializer.Serialize(user));
        
        return user;
    }
}
```

## 구현 가이드라인

### 1. 어트리뷰트 설계 가이드라인

```csharp
// 좋은 예: 명확하고 구체적인 이름
[MeasureExecutionTime]
[CacheResult(ExpirationMinutes = 5)]
[ValidateNotNull]

// 나쁜 예: 모호하고 일반적인 이름
[Process]
[Handle]
[Apply]
```

### 2. 성능 최적화 가이드라인

#### 메모리 할당 최소화

```csharp
// 권장: ValueType 활용
public readonly struct LogContext
{
    public readonly string MethodName;
    public readonly DateTime StartTime;
    
    public LogContext(string methodName, DateTime startTime)
    {
        MethodName = methodName;
        StartTime = startTime;
    }
}

// 지양: 불필요한 객체 생성
public class LogContext
{
    public string MethodName { get; set; }
    public DateTime StartTime { get; set; }
}
```

#### 조건부 컴파일 활용

```csharp
public partial class UserService
{
    [Conditional("DEBUG")]
    private static void LogDebug(string message)
    {
        Console.WriteLine($"[DEBUG] {message}");
    }
}
```

### 3. 예외 처리 가이드라인

AOP에서 발생하는 예외는 원본 메서드의 예외와 명확히 구분되어야 한다.

```csharp
public async Task<T> ExecuteWithRetryAsync<T>(Func<Task<T>> operation, int maxRetries = 3)
{
    for (int attempt = 1; attempt <= maxRetries; attempt++)
    {
        try
        {
            return await operation();
        }
        catch (Exception ex) when (attempt < maxRetries && IsRetriableException(ex))
        {
            await Task.Delay(TimeSpan.FromSeconds(Math.Pow(2, attempt)));
        }
        catch (Exception ex)
        {
            // AOP 관련 예외와 비즈니스 예외 구분
            throw new AopException($"Failed after {maxRetries} attempts", ex);
        }
    }
    
    throw new InvalidOperationException("Retry logic error");
}
```

### 4. 테스트 가이드라인

#### 단위 테스트

```csharp
[Test]
public async Task UserService_GetUser_ShouldCache()
{
    // Arrange
    var mockRepository = new Mock<IUserRepository>();
    var mockCache = new Mock<IDistributedCache>();
    var service = new CachedUserService(mockRepository.Object, mockCache.Object);
    
    // Act
    await service.GetUserAsync(1);
    await service.GetUserAsync(1); // 두 번째 호출
    
    // Assert
    mockRepository.Verify(x => x.GetByIdAsync(1), Times.Once); // 캐시로 인해 한 번만 호출
}
```

#### 통합 테스트

```csharp
[Test]
public async Task LoggingMiddleware_ShouldLogRequestResponse()
{
    // Arrange
    using var factory = new WebApplicationFactory<Program>();
    var client = factory.CreateClient();
    
    // Act
    var response = await client.GetAsync("/api/users/1");
    
    // Assert
    response.StatusCode.Should().Be(HttpStatusCode.OK);
    // 로그 검증 로직
}
```

## 모범 사례

### 1. 관심사 분리

각 Aspect는 단일 책임을 가져야 한다.

```csharp
// 좋은 예: 각각 독립적인 관심사
[LogExecution]
[MeasurePerformance]
[ValidateInput]
public async Task<User> GetUserAsync(int id) { ... }

// 나쁜 예: 여러 관심사가 혼재
[LogAndValidateAndCache]
public async Task<User> GetUserAsync(int id) { ... }
```

### 2. 설정 가능한 Aspect

```csharp
[Cache(ExpirationMinutes = 10, KeyPrefix = "user")]
[Retry(MaxAttempts = 3, DelaySeconds = 1)]
public async Task<User> GetUserAsync(int id) { ... }
```

### 3. 조건부 적용

```csharp
[LogExecution(LogLevel.Debug)]
[Conditional("ENABLE_DETAILED_LOGGING")]
public async Task<User> GetUserAsync(int id) { ... }
```

## 안티 패턴

### 1. 과도한 AOP 사용

```csharp
// 지양: AOP가 비즈니스 로직을 가리는 경우
[Log][Cache][Validate][Retry][Audit][Monitor][Trace]
public async Task<User> GetUserAsync(int id) { ... }
```

### 2. 순환 의존성

```csharp
// 지양: Aspect 간 순환 의존성
[AspectA] // AspectA가 AspectB에 의존
[AspectB] // AspectB가 AspectA에 의존
public void Method() { ... }
```

### 3. 상태 공유

```csharp
// 지양: Aspect에서 전역 상태 공유
public class StatefulAspect
{
    private static int _counter; // 전역 상태
    
    public void OnMethodExecuting()
    {
        _counter++; // 동시성 문제 발생 가능
    }
}
```

## 참고자료

- [.NET 8 Performance Improvements](https://devblogs.microsoft.com/dotnet/performance-improvements-in-net-8/)
- [Source Generators in .NET](https://docs.microsoft.com/en-us/dotnet/csharp/roslyn-sdk/source-generators-overview)
- [ASP.NET Core Middleware](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/middleware/)
- [Dependency Injection Patterns](https://docs.microsoft.com/en-us/dotnet/core/extensions/dependency-injection-guidelines)
