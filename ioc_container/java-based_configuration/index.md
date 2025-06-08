# Code-based configuration

## 개요

.NET 8 LTS에서 코드를 사용하여 의존성 주입 컨테이너를 구성하는 방법에 대해 알아본다. .NET은 Spring의 `@Configuration`과 `@Bean` 어노테이션과 달리 **명시적 코드 기반 설정**을 기본 철학으로 하며, `Program.cs`와 확장 메서드를 통해 서비스를 등록한다.

## 설명

### 기본 개념: Program.cs와 확장 메서드

.NET에서는 Spring의 `@Configuration` 클래스 대신 `Program.cs` 파일과 `IServiceCollection` 확장 메서드를 사용하여 서비스를 구성한다.

Spring의 `@Bean` 메서드와 같은 역할을 하는 것은 `IServiceCollection.Add*` 메서드들이며, 이들은 .NET DI 컨테이너가 관리할 서비스를 등록하는데 사용된다.

.NET의 서비스 구성은 **명시적이고 컴파일 타임에 안전한** 방식으로 이루어진다.

기본적인 서비스 구성은 다음과 같이 할 수 있다:

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// 서비스 등록 (Spring의 @Bean과 유사한 역할)
builder.Services.AddScoped<IMyService, MyServiceImpl>();

var app = builder.Build();
```

이는 다음 Spring XML 구성과 동일한 역할을 한다:

```xml
<beans>
    <bean id="myService" class="com.acme.services.MyServiceImpl"/>
</beans>
```

### .NET DI 컨테이너를 이용한 애플리케이션 구성

.NET의 `IServiceCollection`은 다양한 방식으로 서비스를 등록할 수 있으며, 인터페이스와 구현체, 팩토리 메서드, 인스턴스 등을 받아들일 수 있다.

서비스가 등록되면 해당 서비스는 DI 컨테이너에 의해 관리되고, 다른 서비스에서 생성자 주입을 통해 사용할 수 있다.

#### 간단한 구성

.NET에서는 별도의 XML 구성 없이 코드만으로 전체 애플리케이션을 구성할 수 있다:

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// 서비스 등록
builder.Services.AddScoped<IMyService, MyServiceImpl>();

var app = builder.Build();

// 애플리케이션 실행
app.Run();

// 서비스 사용 예시 (컨트롤러나 다른 서비스에서)
public class SomeController : ControllerBase
{
    private readonly IMyService _myService;

    public SomeController(IMyService myService)
    {
        _myService = myService;
    }

    public IActionResult DoStuff()
    {
        _myService.DoStuff();
        return Ok();
    }
}
```

#### 프로그래밍 방식으로 서비스 등록

.NET에서는 조건부 등록이나 동적 등록을 쉽게 할 수 있다:

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// 환경에 따른 조건부 등록
if (builder.Environment.IsDevelopment())
{
    builder.Services.AddScoped<IMyService, MockMyService>();
}
else
{
    builder.Services.AddScoped<IMyService, MyServiceImpl>();
}

// 구성에 따른 동적 등록
var useCache = builder.Configuration.GetValue<bool>("UseCache");
if (useCache)
{
    builder.Services.AddSingleton<ICacheService, RedisCacheService>();
}

var app = builder.Build();
```

#### 확장 메서드를 통한 서비스 그룹 등록

복잡한 서비스 구성은 확장 메서드로 모듈화할 수 있다:

```csharp
// ServiceCollectionExtensions.cs
public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddBusinessServices(this IServiceCollection services)
    {
        services.AddScoped<ITransferService, TransferServiceImpl>();
        services.AddScoped<IAccountService, AccountServiceImpl>();
        return services;
    }
}

// Program.cs에서 사용
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddBusinessServices(); // 구성요소 그룹 등록
var app = builder.Build();
```

#### ASP.NET Core에서 웹 애플리케이션 지원

.NET의 ASP.NET Core는 기본적으로 DI 컨테이너를 지원하며, 별도 설정 없이 사용 가능하다:

```csharp
// Program.cs - ASP.NET Core 애플리케이션
var builder = WebApplication.CreateBuilder(args);

// 컨트롤러 서비스 추가
builder.Services.AddControllers();

// 비즈니스 서비스 추가
builder.Services.AddScoped<IMovieService, MovieService>();

// Entity Framework 추가
builder.Services.AddDbContext<MovieDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

var app = builder.Build();

// 미들웨어 파이프라인 구성
app.UseRouting();
app.MapControllers();

app.Run();
```

### 팩토리 메서드를 통한 서비스 등록

.NET에서는 Spring의 `@Bean` 메서드와 유사하게 팩토리 메서드를 통해 서비스를 등록할 수 있다:

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// 팩토리 메서드를 통한 서비스 등록 (Spring의 @Bean과 유사)
builder.Services.AddScoped<ITransferService>(provider =>
{
    var accountRepository = provider.GetRequiredService<IAccountRepository>();
    return new TransferServiceImpl(accountRepository);
});

var app = builder.Build();
```

#### 서비스 의존성

서비스 등록 시 의존성은 생성자 주입을 통해 자동으로 해결된다:

```csharp
// 인터페이스 정의
public interface ITransferService
{
    void Transfer(decimal amount, string fromAccount, string toAccount);
}

public interface IAccountRepository
{
    void UpdateAccount(string accountId, decimal amount);
}

// 구현체
public class TransferServiceImpl : ITransferService
{
    private readonly IAccountRepository _accountRepository;

    // 생성자 주입 - 의존성 자동 해결
    public TransferServiceImpl(IAccountRepository accountRepository)
    {
        _accountRepository = accountRepository;
    }

    public void Transfer(decimal amount, string fromAccount, string toAccount)
    {
        // 구현 로직
    }
}

// 서비스 등록
builder.Services.AddScoped<IAccountRepository, AccountRepositoryImpl>();
builder.Services.AddScoped<ITransferService, TransferServiceImpl>();
```

#### 서비스 생명주기 지정

.NET에서는 세 가지 기본 생명주기를 제공한다:

```csharp
public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddMyServices(this IServiceCollection services)
    {
        // Singleton - 애플리케이션 수명 동안 단일 인스턴스
        services.AddSingleton<ICacheService, RedisCacheService>();
        
        // Scoped - 요청당 하나의 인스턴스 (웹 앱에서)
        services.AddScoped<ITransferService, TransferServiceImpl>();
        
        // Transient - 매번 새 인스턴스
        services.AddTransient<IEmailService, EmailService>();
        
        return services;
    }
}
```

### 구성 클래스 패턴

.NET에서는 Spring의 `@Configuration` 클래스와 유사한 패턴을 확장 메서드로 구현할 수 있다:

```csharp
// AppConfiguration.cs - Spring의 @Configuration과 유사한 역할
public static class AppConfiguration
{
    public static IServiceCollection AddAppServices(this IServiceCollection services)
    {
        // Bean 간 의존성을 가진 서비스 등록
        services.AddScoped<IBeanOne>(provider =>
        {
            var beanTwo = provider.GetRequiredService<IBeanTwo>();
            return new BeanOne(beanTwo);
        });

        services.AddScoped<IBeanTwo, BeanTwo>();
        
        return services;
    }
}
```

### 모듈화된 구성

.NET에서는 여러 구성 모듈을 조합할 수 있다:

#### 구성 모듈 분리

```csharp
// DataConfiguration.cs
public static class DataConfiguration
{
    public static IServiceCollection AddDataServices(this IServiceCollection services, 
        IConfiguration configuration)
    {
        services.AddDbContext<AppDbContext>(options =>
            options.UseSqlServer(configuration.GetConnectionString("DefaultConnection")));
        
        services.AddScoped<IAccountRepository, EntityFrameworkAccountRepository>();
        
        return services;
    }
}

// BusinessConfiguration.cs  
public static class BusinessConfiguration
{
    public static IServiceCollection AddBusinessServices(this IServiceCollection services)
    {
        services.AddScoped<ITransferService, TransferServiceImpl>();
        services.AddScoped<IAccountService, AccountServiceImpl>();
        
        return services;
    }
}

// InfrastructureConfiguration.cs
public static class InfrastructureConfiguration
{
    public static IServiceCollection AddInfrastructureServices(this IServiceCollection services,
        IConfiguration configuration)
    {
        services.AddSingleton<IEmailService, SendGridEmailService>();
        services.AddSingleton<ICacheService, RedisCacheService>();
        
        return services;
    }
}
```

#### 통합 구성

```csharp
// Program.cs - 모든 구성 모듈을 조합
var builder = WebApplication.CreateBuilder(args);

// 모듈별 서비스 등록
builder.Services.AddDataServices(builder.Configuration);
builder.Services.AddBusinessServices();
builder.Services.AddInfrastructureServices(builder.Configuration);

var app = builder.Build();
```

#### 구성 간 의존성 관리

다른 모듈의 서비스에 의존하는 경우도 자동으로 해결된다:

```csharp
// 여러 모듈에 걸친 의존성 예시
public class SystemTestConfiguration
{
    public static IServiceCollection AddSystemTestServices(this IServiceCollection services)
    {
        // DataSource 등록
        services.AddSingleton<IDataSource, InMemoryDataSource>();
        
        // Repository 등록 (DataSource에 의존)
        services.AddScoped<IAccountRepository>(provider =>
        {
            var dataSource = provider.GetRequiredService<IDataSource>();
            return new JdbcAccountRepository(dataSource);
        });
        
        // Service 등록 (Repository에 의존)
        services.AddScoped<ITransferService>(provider =>
        {
            var accountRepository = provider.GetRequiredService<IAccountRepository>();
            return new TransferServiceImpl(accountRepository);
        });
        
        return services;
    }
}

// 사용
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddSystemTestServices();
var app = builder.Build();

// 서비스 사용
using var scope = app.Services.CreateScope();
var transferService = scope.ServiceProvider.GetRequiredService<ITransferService>();
transferService.Transfer(100.00m, "A123", "C456");
```

### .NET 구성의 장점

1. **컴파일 타임 안전성**: 모든 의존성이 컴파일 시점에 검증됨
2. **명시적 등록**: 모든 서비스 등록이 코드로 명시되어 추적 용이
3. **모듈화**: 확장 메서드를 통한 깔끔한 구성 분리
4. **조건부 등록**: 환경이나 구성에 따른 유연한 서비스 등록
5. **성능**: Native AOT 지원으로 빠른 시작과 적은 메모리 사용

## 참고자료

- [.NET Dependency injection](https://learn.microsoft.com/dotnet/core/extensions/dependency-injection)
- [ASP.NET Core dependency injection](https://learn.microsoft.com/aspnet/core/fundamentals/dependency-injection)
- [.NET Configuration](https://learn.microsoft.com/dotnet/core/extensions/configuration)
- [Service lifetimes](https://learn.microsoft.com/dotnet/core/extensions/dependency-injection#service-lifetimes)
