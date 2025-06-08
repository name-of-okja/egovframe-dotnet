# 미들웨어 기반 AOP 패턴

## 개요

ASP.NET Core 미들웨어를 활용한 AOP(Aspect Oriented Programming) 구현 패턴을 다룬다. 미들웨어는 HTTP 요청 파이프라인에서 횡단 관심사를 처리하는 데 최적화된 .NET의 핵심 기능이다.

## 미들웨어 AOP 기본 개념

### 미들웨어 파이프라인

ASP.NET Core의 미들웨어 파이프라인은 요청과 응답을 처리하는 일련의 컴포넌트들로 구성된다.

```
Request → Middleware1 → Middleware2 → ... → Controller Action
          ↓            ↓                     ↓
Response ← Middleware1 ← Middleware2 ← ... ← Controller Action
```

### 미들웨어 기반 AOP의 특징

1. **요청/응답 중심**: HTTP 컨텍스트에서 동작
2. **순차 실행**: 파이프라인 순서에 따른 실행
3. **조건부 실행**: 특정 조건에서만 다음 미들웨어 실행
4. **전역 적용**: 모든 요청에 일괄 적용 가능

## 핵심 미들웨어 AOP 패턴

### 1. 로깅 미들웨어 패턴

요청과 응답에 대한 로깅을 처리하는 패턴

```csharp
public class RequestLoggingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<RequestLoggingMiddleware> _logger;

    public RequestLoggingMiddleware(RequestDelegate next, ILogger<RequestLoggingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var requestId = Guid.NewGuid().ToString();
        var stopwatch = Stopwatch.StartNew();

        // 요청 로깅
        _logger.LogInformation(
            "Request {RequestId}: {Method} {Path} {QueryString}",
            requestId, 
            context.Request.Method, 
            context.Request.Path, 
            context.Request.QueryString);

        // 요청 본문 로깅 (필요시)
        if (ShouldLogRequestBody(context))
        {
            var requestBody = await ReadRequestBodyAsync(context);
            _logger.LogDebug("Request {RequestId} Body: {Body}", requestId, requestBody);
        }

        try
        {
            await _next(context);

            // 성공 응답 로깅
            stopwatch.Stop();
            _logger.LogInformation(
                "Response {RequestId}: {StatusCode} in {ElapsedMs}ms",
                requestId,
                context.Response.StatusCode,
                stopwatch.ElapsedMilliseconds);
        }
        catch (Exception ex)
        {
            // 예외 발생 시 로깅
            stopwatch.Stop();
            _logger.LogError(ex,
                "Exception {RequestId}: {Message} in {ElapsedMs}ms",
                requestId,
                ex.Message,
                stopwatch.ElapsedMilliseconds);
            throw;
        }
    }

    private static bool ShouldLogRequestBody(HttpContext context)
    {
        return context.Request.ContentType?.Contains("application/json") == true
               && context.Request.ContentLength > 0
               && context.Request.ContentLength < 10240; // 10KB 미만만 로깅
    }

    private static async Task<string> ReadRequestBodyAsync(HttpContext context)
    {
        context.Request.EnableBuffering();
        var buffer = new byte[Convert.ToInt32(context.Request.ContentLength)];
        await context.Request.Body.ReadAsync(buffer, 0, buffer.Length);
        context.Request.Body.Position = 0;
        return Encoding.UTF8.GetString(buffer);
    }
}
```

### 2. 인증/인가 미들웨어 패턴

보안 관련 횡단 관심사를 처리하는 패턴

```csharp
public class CustomAuthenticationMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IAuthenticationService _authService;
    private readonly ILogger<CustomAuthenticationMiddleware> _logger;

    public CustomAuthenticationMiddleware(
        RequestDelegate next, 
        IAuthenticationService authService,
        ILogger<CustomAuthenticationMiddleware> logger)
    {
        _next = next;
        _authService = authService;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        // 인증이 필요 없는 경로 확인
        if (IsPublicEndpoint(context.Request.Path))
        {
            await _next(context);
            return;
        }

        // 토큰 추출
        var token = ExtractToken(context.Request);
        if (string.IsNullOrEmpty(token))
        {
            await HandleUnauthorizedAsync(context, "Missing authentication token");
            return;
        }

        try
        {
            // 토큰 검증 및 사용자 정보 설정
            var principal = await _authService.ValidateTokenAsync(token);
            context.User = principal;

            // 권한 확인
            if (!await HasRequiredPermissionAsync(context, principal))
            {
                await HandleForbiddenAsync(context, "Insufficient permissions");
                return;
            }

            await _next(context);
        }
        catch (SecurityTokenException ex)
        {
            _logger.LogWarning(ex, "Invalid token provided");
            await HandleUnauthorizedAsync(context, "Invalid token");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Authentication error");
            await HandleUnauthorizedAsync(context, "Authentication failed");
        }
    }

    private static bool IsPublicEndpoint(PathString path)
    {
        var publicPaths = new[] { "/health", "/api/auth/login", "/api/auth/register" };
        return publicPaths.Any(p => path.StartsWithSegments(p));
    }

    private static string ExtractToken(HttpRequest request)
    {
        var authHeader = request.Headers["Authorization"].ToString();
        return authHeader?.StartsWith("Bearer ") == true 
            ? authHeader.Substring("Bearer ".Length).Trim() 
            : null;
    }

    private async Task<bool> HasRequiredPermissionAsync(HttpContext context, ClaimsPrincipal principal)
    {
        var endpoint = context.GetEndpoint();
        var requiredPermission = endpoint?.Metadata.GetMetadata<RequirePermissionAttribute>();
        
        if (requiredPermission == null)
            return true;

        return await _authService.HasPermissionAsync(principal, requiredPermission.Permission);
    }

    private static async Task HandleUnauthorizedAsync(HttpContext context, string message)
    {
        context.Response.StatusCode = 401;
        context.Response.ContentType = "application/json";
        
        var response = new { error = "Unauthorized", message };
        await context.Response.WriteAsync(JsonSerializer.Serialize(response));
    }

    private static async Task HandleForbiddenAsync(HttpContext context, string message)
    {
        context.Response.StatusCode = 403;
        context.Response.ContentType = "application/json";
        
        var response = new { error = "Forbidden", message };
        await context.Response.WriteAsync(JsonSerializer.Serialize(response));
    }
}
```

### 3. 예외 처리 미들웨어 패턴

전역 예외 처리를 담당하는 패턴

```csharp
public class ExceptionHandlingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ExceptionHandlingMiddleware> _logger;
    private readonly IWebHostEnvironment _environment;

    public ExceptionHandlingMiddleware(
        RequestDelegate next, 
        ILogger<ExceptionHandlingMiddleware> logger,
        IWebHostEnvironment environment)
    {
        _next = next;
        _logger = logger;
        _environment = environment;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (Exception ex)
        {
            await HandleExceptionAsync(context, ex);
        }
    }

    private async Task HandleExceptionAsync(HttpContext context, Exception exception)
    {
        var (statusCode, message, details) = exception switch
        {
            ValidationException validationEx => (400, "Validation failed", validationEx.Errors),
            NotFoundException notFoundEx => (404, notFoundEx.Message, null),
            UnauthorizedException unauthorizedEx => (401, "Unauthorized", null),
            ForbiddenException forbiddenEx => (403, "Forbidden", null),
            BusinessLogicException businessEx => (422, businessEx.Message, null),
            _ => (500, "An error occurred while processing your request", null)
        };

        _logger.LogError(exception, 
            "Exception occurred: {ExceptionType} - {Message}", 
            exception.GetType().Name, 
            exception.Message);

        context.Response.StatusCode = statusCode;
        context.Response.ContentType = "application/json";

        var response = new ErrorResponse
        {
            Error = GetErrorType(statusCode),
            Message = message,
            Details = details,
            TraceId = context.TraceIdentifier
        };

        // 개발 환경에서만 상세 정보 포함
        if (_environment.IsDevelopment())
        {
            response.StackTrace = exception.StackTrace;
            response.InnerException = exception.InnerException?.Message;
        }

        await context.Response.WriteAsync(JsonSerializer.Serialize(response));
    }

    private static string GetErrorType(int statusCode) => statusCode switch
    {
        400 => "Bad Request",
        401 => "Unauthorized",
        403 => "Forbidden",
        404 => "Not Found",
        422 => "Unprocessable Entity",
        _ => "Internal Server Error"
    };
}

public class ErrorResponse
{
    public string Error { get; set; }
    public string Message { get; set; }
    public object Details { get; set; }
    public string TraceId { get; set; }
    public string StackTrace { get; set; }
    public string InnerException { get; set; }
}
```

### 4. 성능 모니터링 미들웨어 패턴

요청 처리 성능을 모니터링하는 패턴

```csharp
public class PerformanceMonitoringMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<PerformanceMonitoringMiddleware> _logger;
    private readonly IMetrics _metrics;

    public PerformanceMonitoringMiddleware(
        RequestDelegate next, 
        ILogger<PerformanceMonitoringMiddleware> logger,
        IMetrics metrics)
    {
        _next = next;
        _logger = logger;
        _metrics = metrics;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var stopwatch = Stopwatch.StartNew();
        var endpoint = GetEndpointName(context);

        try
        {
            await _next(context);
        }
        finally
        {
            stopwatch.Stop();
            
            var tags = new TagList
            {
                ["endpoint"] = endpoint,
                ["method"] = context.Request.Method,
                ["status_code"] = context.Response.StatusCode.ToString()
            };

            // 메트릭 기록
            _metrics.CreateHistogram<double>("http_request_duration_seconds")
                .Record(stopwatch.Elapsed.TotalSeconds, tags);

            _metrics.CreateCounter<long>("http_requests_total")
                .Add(1, tags);

            // 느린 요청 로깅
            if (stopwatch.ElapsedMilliseconds > 1000)
            {
                _logger.LogWarning(
                    "Slow request detected: {Method} {Path} took {ElapsedMs}ms",
                    context.Request.Method,
                    context.Request.Path,
                    stopwatch.ElapsedMilliseconds);
            }

            // 메모리 사용량 모니터링
            var memoryUsage = GC.GetTotalMemory(false);
            _metrics.CreateGauge<long>("memory_usage_bytes")
                .Record(memoryUsage, tags);
        }
    }

    private static string GetEndpointName(HttpContext context)
    {
        var endpoint = context.GetEndpoint();
        return endpoint?.DisplayName ?? context.Request.Path.ToString();
    }
}
```

### 5. 캐싱 미들웨어 패턴

응답 캐싱을 처리하는 패턴

```csharp
public class ResponseCachingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IDistributedCache _cache;
    private readonly ILogger<ResponseCachingMiddleware> _logger;

    public ResponseCachingMiddleware(
        RequestDelegate next, 
        IDistributedCache cache,
        ILogger<ResponseCachingMiddleware> logger)
    {
        _next = next;
        _cache = cache;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        // GET 요청만 캐싱
        if (!HttpMethods.IsGet(context.Request.Method))
        {
            await _next(context);
            return;
        }

        var cacheKey = GenerateCacheKey(context.Request);
        var cachedResponse = await _cache.GetStringAsync(cacheKey);

        if (!string.IsNullOrEmpty(cachedResponse))
        {
            // 캐시된 응답 반환
            _logger.LogDebug("Cache hit for key: {CacheKey}", cacheKey);
            await WriteCachedResponseAsync(context, cachedResponse);
            return;
        }

        // 응답 캐싱을 위한 스트림 래핑
        var originalBodyStream = context.Response.Body;
        using var responseBody = new MemoryStream();
        context.Response.Body = responseBody;

        try
        {
            await _next(context);

            // 성공적인 응답만 캐싱
            if (context.Response.StatusCode == 200)
            {
                responseBody.Seek(0, SeekOrigin.Begin);
                var response = await new StreamReader(responseBody).ReadToEndAsync();
                
                var cacheOptions = new DistributedCacheEntryOptions
                {
                    AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5)
                };

                await _cache.SetStringAsync(cacheKey, response, cacheOptions);
                _logger.LogDebug("Response cached for key: {CacheKey}", cacheKey);

                // 원본 응답 스트림에 복사
                responseBody.Seek(0, SeekOrigin.Begin);
                await responseBody.CopyToAsync(originalBodyStream);
            }
        }
        finally
        {
            context.Response.Body = originalBodyStream;
        }
    }

    private static string GenerateCacheKey(HttpRequest request)
    {
        var keyBuilder = new StringBuilder();
        keyBuilder.Append(request.Path);
        
        if (request.QueryString.HasValue)
        {
            keyBuilder.Append(request.QueryString);
        }

        return $"response_cache:{keyBuilder}";
    }

    private static async Task WriteCachedResponseAsync(HttpContext context, string cachedResponse)
    {
        context.Response.ContentType = "application/json";
        context.Response.Headers.Add("X-Cache", "HIT");
        await context.Response.WriteAsync(cachedResponse);
    }
}
```

## 미들웨어 등록 및 구성

### 미들웨어 등록 순서

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

// 미들웨어 등록 순서가 중요함
app.UseMiddleware<ExceptionHandlingMiddleware>();      // 1. 예외 처리 (최상위)
app.UseMiddleware<RequestLoggingMiddleware>();         // 2. 로깅
app.UseMiddleware<PerformanceMonitoringMiddleware>();  // 3. 성능 모니터링
app.UseMiddleware<CustomAuthenticationMiddleware>();   // 4. 인증/인가
app.UseMiddleware<ResponseCachingMiddleware>();        // 5. 캐싱

app.UseRouting();
app.MapControllers();

app.Run();
```

### 조건부 미들웨어 등록

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

// 개발 환경에서만 상세 로깅
if (app.Environment.IsDevelopment())
{
    app.UseMiddleware<DetailedLoggingMiddleware>();
}

// 프로덕션 환경에서만 캐싱
if (app.Environment.IsProduction())
{
    app.UseMiddleware<ResponseCachingMiddleware>();
}

app.Run();
```

### 미들웨어 확장 메서드

```csharp
public static class MiddlewareExtensions
{
    public static IApplicationBuilder UseRequestLogging(this IApplicationBuilder builder)
    {
        return builder.UseMiddleware<RequestLoggingMiddleware>();
    }

    public static IApplicationBuilder UseCustomAuthentication(this IApplicationBuilder builder)
    {
        return builder.UseMiddleware<CustomAuthenticationMiddleware>();
    }

    public static IApplicationBuilder UsePerformanceMonitoring(this IApplicationBuilder builder)
    {
        return builder.UseMiddleware<PerformanceMonitoringMiddleware>();
    }
}

// 사용법
app.UseRequestLogging();
app.UseCustomAuthentication();
app.UsePerformanceMonitoring();
```

## 미들웨어 AOP 모범 사례

### 1. 성능 최적화

```csharp
public class OptimizedMiddleware
{
    private readonly RequestDelegate _next;
    private static readonly ActivitySource ActivitySource = new("MyApp.Middleware");

    public async Task InvokeAsync(HttpContext context)
    {
        using var activity = ActivitySource.StartActivity("ProcessRequest");
        
        // 비동기 처리 최적화
        await _next(context).ConfigureAwait(false);
    }
}
```

### 2. 메모리 효율적인 구현

```csharp
public class MemoryEfficientMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ObjectPool<StringBuilder> _stringBuilderPool;

    public async Task InvokeAsync(HttpContext context)
    {
        var sb = _stringBuilderPool.Get();
        try
        {
            // StringBuilder 사용
            sb.Clear();
            sb.Append(context.Request.Method);
            sb.Append(' ');
            sb.Append(context.Request.Path);
            
            var logMessage = sb.ToString();
            // 로깅 처리
        }
        finally
        {
            _stringBuilderPool.Return(sb);
        }

        await _next(context);
    }
}
```

### 3. 테스트 가능한 설계

```csharp
[Test]
public async Task RequestLoggingMiddleware_ShouldLogRequestAndResponse()
{
    // Arrange
    var logger = new Mock<ILogger<RequestLoggingMiddleware>>();
    var middleware = new RequestLoggingMiddleware(
        next: (context) => Task.CompletedTask, 
        logger: logger.Object);
    
    var context = new DefaultHttpContext();
    context.Request.Method = "GET";
    context.Request.Path = "/api/users";

    // Act
    await middleware.InvokeAsync(context);

    // Assert
    logger.Verify(
        x => x.Log(
            LogLevel.Information,
            It.IsAny<EventId>(),
            It.Is<It.IsAnyType>((v, t) => v.ToString().Contains("GET /api/users")),
            It.IsAny<Exception>(),
            It.IsAny<Func<It.IsAnyType, Exception, string>>()),
        Times.AtLeastOnce);
}
```

## 주의사항

### 1. 미들웨어 순서

미들웨어의 등록 순서는 실행 순서를 결정하므로 신중하게 배치해야 한다.

### 2. 예외 전파

미들웨어에서 예외가 발생하면 파이프라인이 중단되므로 적절한 예외 처리가 필요하다.

### 3. 응답 스트림 조작

응답 스트림을 조작할 때는 원본 스트림을 반드시 복원해야 한다.

### 4. 성능 영향

모든 요청에 적용되므로 성능에 미치는 영향을 고려해야 한다.

## 참고자료

- [ASP.NET Core Middleware](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/middleware/)
- [Writing Custom Middleware](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/middleware/write)
- [Middleware Best Practices](https://docs.microsoft.com/en-us/aspnet/core/performance/performance-best-practices)
