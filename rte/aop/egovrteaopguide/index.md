# 실행환경 AOP 가이드라인

## 개요

전자정부 .NET 8 실행환경은 ASP.NET Core 미들웨어와 정적 소스 생성기(Source Generator)에 기반한 AOP 방법을 사용하며, 예외처리와 트랜잭션 처리에 적용하였다. 이러한 방법은 기존 어노테이션 기반 방법에 비해 횡단 관심사에 대한 설정관계를 파악하기 유리하고, 컴파일 타임 안전성을 제공한다.

## 설명

### 예외 처리

실행환경은 Repository에서 발생한 Exception을 받아 Service 계층에서 처리할 수 있다. 실행환경에서 추가로 제공하는 Exception은 다음과 같다.

- **EgovBizException**: 업무에서 Checked Exception인 경우에 공통으로 사용하는 Exception이다. 개발자가 특정한 오류에 대해서 throw하여 특정 메시지를 전달하고자 하는 경우에는 ProcessException() 메서드를 이용하도록 한다.
- **ExceptionTransferMiddleware**: 미들웨어 기능을 이용하여 Service 클래스에서 Exception이 발생한 경우에 ProcessAsync() 메서드에서 처리한다. 내부적으로 EgovBizException인지 시스템 예외인지 구분하여 처리한다.

#### 미들웨어 기반 예외 처리

예외 처리를 위한 미들웨어 구현:

```csharp
public class ExceptionHandlingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ExceptionHandlingMiddleware> _logger;
    private readonly IExceptionHandlerService _exceptionHandler;

    public ExceptionHandlingMiddleware(
        RequestDelegate next,
        ILogger<ExceptionHandlingMiddleware> logger,
        IExceptionHandlerService exceptionHandler)
    {
        _next = next;
        _logger = logger;
        _exceptionHandler = exceptionHandler;
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
        _logger.LogError(exception, "An unhandled exception occurred");

        var response = exception switch
        {
            EgovBizException bizEx => new
            {
                StatusCode = 400,
                Message = bizEx.Message,
                Details = bizEx.Details
            },
            ArgumentException argEx => new
            {
                StatusCode = 400,
                Message = "Invalid argument",
                Details = argEx.Message
            },
            PostgresException pgEx => new
            {
                StatusCode = 500,
                Message = "Database error",
                Details = pgEx.MessageText,
                SqlState = pgEx.SqlState
            },
            NpgsqlException npgEx => new
            {
                StatusCode = 500,
                Message = "Database connection error",
                Details = npgEx.Message
            },
            _ => new
            {
                StatusCode = 500,
                Message = "Internal server error",
                Details = "An unexpected error occurred"
            }
        };

        context.Response.StatusCode = response.StatusCode;
        context.Response.ContentType = "application/json";

        await context.Response.WriteAsync(JsonSerializer.Serialize(response));
    }
}
```

미들웨어 등록:

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddScoped<IExceptionHandlerService, DefaultExceptionHandlerService>();

var app = builder.Build();

app.UseMiddleware<ExceptionHandlingMiddleware>();
app.MapControllers();

app.Run();
```

#### 소스 생성기 기반 예외 처리

어트리뷰트 기반 예외 처리:

```csharp
[AttributeUsage(AttributeTargets.Method)]
public class ExceptionHandlingAttribute : Attribute
{
    public Type[] HandledExceptions { get; set; } = Array.Empty<Type>();
}

// 사용 예제
public class UserService : IUserService
{
    [ExceptionHandling(HandledExceptions = new[] { typeof(ArgumentException), typeof(InvalidOperationException) })]
    public async Task<User> GetUserAsync(int id)
    {
        if (id <= 0)
            throw new ArgumentException("User ID must be positive", nameof(id));
            
        return await _userRepository.GetByIdAsync(id);
    }
}
```

### 트랜잭션 처리

실행환경에서 트랜잭션 설정은 ASP.NET Core의 의존성 주입과 Entity Framework Core의 PostgreSQL 트랜잭션 관리를 활용한다.

#### Entity Framework Core PostgreSQL 트랜잭션 설정

```csharp
// Program.cs에서 PostgreSQL 트랜잭션 설정
builder.Services.AddDbContext<ApplicationDbContext>(options =>
{
    options.UseNpgsql(builder.Configuration.GetConnectionString("PostgreSqlConnection"), 
        npgsqlOptions =>
        {
            npgsqlOptions.EnableRetryOnFailure(
                maxRetryCount: 3,
                maxRetryDelay: TimeSpan.FromSeconds(5),
                errorCodesToAdd: null);
            npgsqlOptions.CommandTimeout(30);
        });
    
    // 개발 환경에서만 민감한 데이터 로깅 활성화
    if (builder.Environment.IsDevelopment())
    {
        options.EnableSensitiveDataLogging();
        options.EnableDetailedErrors();
    }
});

builder.Services.AddScoped<IUnitOfWork, UnitOfWork>();
builder.Services.AddScoped<ITransactionService, TransactionService>();
```

#### 미들웨어 기반 트랜잭션 관리

```csharp
public class TransactionMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<TransactionMiddleware> _logger;

    public TransactionMiddleware(RequestDelegate next, ILogger<TransactionMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context, ApplicationDbContext dbContext)
    {
        // POST, PUT, DELETE 요청에 대해서만 트랜잭션 적용
        if (IsTransactionRequired(context.Request.Method))
        {
            using var transaction = await dbContext.Database.BeginTransactionAsync(IsolationLevel.ReadCommitted);
            try
            {
                await _next(context);
                
                if (context.Response.StatusCode < 400)
                {
                    await transaction.CommitAsync();
                    _logger.LogDebug("PostgreSQL transaction committed successfully");
                }
                else
                {
                    await transaction.RollbackAsync();
                    _logger.LogDebug("PostgreSQL transaction rolled back due to error response");
                }
            }
            catch (PostgresException pgEx)
            {
                await transaction.RollbackAsync();
                _logger.LogError(pgEx, "PostgreSQL transaction rolled back due to database exception. SqlState: {SqlState}", pgEx.SqlState);
                throw;
            }
            catch (Exception ex)
            {
                await transaction.RollbackAsync();
                _logger.LogError(ex, "PostgreSQL transaction rolled back due to exception");
                throw;
            }
        }
        else
        {
            await _next(context);
        }
    }

    private static bool IsTransactionRequired(string method)
    {
        return method is "POST" or "PUT" or "DELETE" or "PATCH";
    }
}
```

#### 어트리뷰트 기반 트랜잭션 관리

```csharp
[AttributeUsage(AttributeTargets.Method)]
public class TransactionalAttribute : Attribute
{
    public IsolationLevel IsolationLevel { get; set; } = IsolationLevel.ReadCommitted;
    public bool ReadOnly { get; set; } = false;
    public int TimeoutSeconds { get; set; } = 30;
}

// 사용 예제
public class UserService : IUserService
{
    private readonly ApplicationDbContext _context;
    private readonly ILogger<UserService> _logger;

    public UserService(ApplicationDbContext context, ILogger<UserService> logger)
    {
        _context = context;
        _logger = logger;
    }

    [Transactional(IsolationLevel = IsolationLevel.Serializable)]
    public async Task<User> CreateUserAsync(CreateUserRequest request)
    {
        try
        {
            var user = new User
            {
                Id = Guid.NewGuid(),
                Name = request.Name,
                Email = request.Email,
                CreatedAt = DateTime.UtcNow
            };

            _context.Users.Add(user);
            await _context.SaveChangesAsync();
            
            _logger.LogInformation("User created successfully with ID: {UserId}", user.Id);
            return user;
        }
        catch (PostgresException pgEx) when (pgEx.SqlState == PostgresErrorCodes.UniqueViolation)
        {
            _logger.LogWarning("Duplicate email attempted: {Email}", request.Email);
            throw new EgovBizException("이미 존재하는 이메일입니다.", pgEx);
        }
    }

    [Transactional(ReadOnly = true, IsolationLevel = IsolationLevel.ReadCommitted)]
    public async Task<User> GetUserAsync(int id)
    {
        return await _context.Users
            .AsNoTracking()
            .FirstOrDefaultAsync(u => u.Id == id);
    }

    [Transactional(TimeoutSeconds = 60)]
    public async Task<List<User>> BatchUpdateUsersAsync(List<UpdateUserRequest> requests)
    {
        var updatedUsers = new List<User>();
        
        foreach (var request in requests)
        {
            var user = await _context.Users.FindAsync(request.Id);
            if (user != null)
            {
                user.Name = request.Name;
                user.Email = request.Email;
                user.UpdatedAt = DateTime.UtcNow;
                updatedUsers.Add(user);
            }
        }

        await _context.SaveChangesAsync();
        return updatedUsers;
    }
}
```

#### UnitOfWork 패턴을 통한 PostgreSQL 트랜잭션 관리

```csharp
public interface IUnitOfWork : IDisposable
{
    Task<int> SaveChangesAsync(CancellationToken cancellationToken = default);
    Task BeginTransactionAsync(IsolationLevel isolationLevel = IsolationLevel.ReadCommitted);
    Task CommitTransactionAsync();
    Task RollbackTransactionAsync();
    Task ExecuteInTransactionAsync(Func<Task> action, IsolationLevel isolationLevel = IsolationLevel.ReadCommitted);
}

public class UnitOfWork : IUnitOfWork
{
    private readonly ApplicationDbContext _context;
    private readonly ILogger<UnitOfWork> _logger;
    private IDbContextTransaction _transaction;

    public UnitOfWork(ApplicationDbContext context, ILogger<UnitOfWork> logger)
    {
        _context = context;
        _logger = logger;
    }

    public async Task BeginTransactionAsync(IsolationLevel isolationLevel = IsolationLevel.ReadCommitted)
    {
        _transaction = await _context.Database.BeginTransactionAsync(isolationLevel);
        _logger.LogDebug("PostgreSQL transaction started with isolation level: {IsolationLevel}", isolationLevel);
    }

    public async Task CommitTransactionAsync()
    {
        try
        {
            await _context.SaveChangesAsync();
            await _transaction?.CommitAsync();
            _logger.LogDebug("PostgreSQL transaction committed successfully");
        }
        catch (PostgresException pgEx)
        {
            _logger.LogError(pgEx, "PostgreSQL error during commit. SqlState: {SqlState}", pgEx.SqlState);
            await RollbackTransactionAsync();
            throw;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error during transaction commit");
            await RollbackTransactionAsync();
            throw;
        }
        finally
        {
            _transaction?.Dispose();
            _transaction = null;
        }
    }

    public async Task RollbackTransactionAsync()
    {
        try
        {
            await _transaction?.RollbackAsync();
            _logger.LogDebug("PostgreSQL transaction rolled back");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error during transaction rollback");
        }
        finally
        {
            _transaction?.Dispose();
            _transaction = null;
        }
    }

    public async Task ExecuteInTransactionAsync(Func<Task> action, IsolationLevel isolationLevel = IsolationLevel.ReadCommitted)
    {
        await BeginTransactionAsync(isolationLevel);
        try
        {
            await action();
            await CommitTransactionAsync();
        }
        catch
        {
            await RollbackTransactionAsync();
            throw;
        }
    }

    public async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
    {
        try
        {
            return await _context.SaveChangesAsync(cancellationToken);
        }
        catch (PostgresException pgEx)
        {
            _logger.LogError(pgEx, "PostgreSQL error during save changes. SqlState: {SqlState}", pgEx.SqlState);
            throw;
        }
    }

    public void Dispose()
    {
        _transaction?.Dispose();
        _context?.Dispose();
    }
}
```

### PostgreSQL 연결 설정

appsettings.json 설정:

```json
{
  "ConnectionStrings": {
    "PostgreSqlConnection": "Host=localhost;Port=5432;Database=egovframe_db;Username=egovframe_user;Password=your_password;Include Error Detail=true;Pooling=true;Minimum Pool Size=5;Maximum Pool Size=100;"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.EntityFrameworkCore.Database.Command": "Information"
    }
  }
}
```

### AOP 설정 통합

미들웨어와 서비스를 Program.cs에서 통합 설정:

```csharp
var builder = WebApplication.CreateBuilder(args);

// PostgreSQL 데이터베이스 설정
builder.Services.AddDbContext<ApplicationDbContext>(options =>
{
    options.UseNpgsql(builder.Configuration.GetConnectionString("PostgreSqlConnection"),
        npgsqlOptions =>
        {
            npgsqlOptions.EnableRetryOnFailure(
                maxRetryCount: 3,
                maxRetryDelay: TimeSpan.FromSeconds(5),
                errorCodesToAdd: null);
            npgsqlOptions.CommandTimeout(30);
            npgsqlOptions.MigrationsAssembly("EgovFrame.Data");
        });
    
    if (builder.Environment.IsDevelopment())
    {
        options.EnableSensitiveDataLogging();
        options.EnableDetailedErrors();
    }
});

// AOP 관련 서비스 등록
builder.Services.AddScoped<IUnitOfWork, UnitOfWork>();
builder.Services.AddScoped<IExceptionHandlerService, DefaultExceptionHandlerService>();

// 비즈니스 서비스 등록
builder.Services.AddScoped<IUserService, UserService>();

// PostgreSQL 헬스체크 추가
builder.Services.AddHealthChecks()
    .AddNpgSql(builder.Configuration.GetConnectionString("PostgreSqlConnection"));

var app = builder.Build();

// 데이터베이스 마이그레이션 (개발환경)
if (app.Environment.IsDevelopment())
{
    using var scope = app.Services.CreateScope();
    var context = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();
    await context.Database.MigrateAsync();
}

// AOP 미들웨어 등록 (순서 중요)
app.UseMiddleware<ExceptionHandlingMiddleware>();
app.UseMiddleware<TransactionMiddleware>();

app.MapControllers();
app.MapHealthChecks("/health");

app.Run();
```

### PostgreSQL 특화 성능 최적화 가이드라인

1. **연결 풀링**: 최소/최대 연결 수 설정으로 성능 최적화
2. **배치 처리**: `SaveChanges()` 호출 최소화로 네트워크 라운드트립 감소
3. **인덱스 활용**: PostgreSQL의 GIN, GiST 인덱스 적극 활용
4. **비동기 처리**: 모든 데이터베이스 작업에 async/await 패턴 사용
5. **읽기 전용 트랜잭션**: `AsNoTracking()`과 `ReadOnly` 트랜잭션 활용

## 참고자료

- [ASP.NET Core 미들웨어](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/middleware/)
- [Entity Framework Core PostgreSQL Provider](https://www.npgsql.org/efcore/)
- [Npgsql Documentation](https://www.npgsql.org/doc/)
- [PostgreSQL Error Codes](https://www.postgresql.org/docs/current/errcodes-appendix.html)
- [의존성 주입 가이드라인](../guidelines/dependency-injection.md)
