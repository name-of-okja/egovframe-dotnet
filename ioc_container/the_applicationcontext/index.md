# The Host and ServiceProvider

## 개요

.NET 8 LTS 애플리케이션에서는 Host가 DI, 구성, 로깅 및 호스팅 기능을 통합하는 핵심 구성 요소로 작동합니다.  
보통 `HostBuilder`를 사용하여 선언적으로 호스트를 구성하고, `IServiceProvider`를 통해 의존성을 관리합니다. 이는 Spring의 ApplicationContext 개념과 유사하며, 코드를 통해 직접 인스턴스를 생성하기보다는 선언적 구성을 활용합니다.

## 설명

.NET Host는 다음과 같은 기능을 제공합니다.

- **Configuration (구성)**: JSON 파일, 환경 변수, 명령줄 인수 등 다양한 소스로부터 설정 값을 로드하여 애플리케이션 전반에 제공합니다.
- **Dependency Injection (의존성 주입)**: `IServiceCollection`과 `IServiceProvider`를 사용하여 객체 생성과 수명 주기를 관리합니다.
- **Logging (로깅)**: 통합 로깅 추상화를 통해 Microsoft.Extensions.Logging 등의 프레임워크와 연동됩니다.
- **Hosting 및 수명 주기 관리**: `IHostedService`를 활용한 백그라운드 작업 실행과 함께, 애플리케이션의 시작, 정지, 재시작 이벤트를 관리합니다.
- **Minimal API 및 공통 진입점 패턴**: 최근 Minimal API의 도입으로 `IServiceContext`와 같은 핵심 요소와 함께, 로깅 및 구성 같은 주변 요소들이 Host에서 관리됩니다. 이 패턴은 ASP.NET Core뿐 아니라 모든 닷넷 기반 애플리케이션 (예: MAUI, Blazor, Aspire 등)의 표준화된 공통 진입점 패턴으로 자리잡고 있습니다.

### HostBuilder와 IServiceProvider 사용 예시

```csharp
var host = Host.CreateDefaultBuilder(args)
    .ConfigureServices((context, services) =>
    {
        // 서비스 등록 예시
        services.AddSingleton<IMyService, MyService>();
        // IHostedService를 이용한 백그라운드 작업 등록
        services.AddHostedService<MyBackgroundService>();
    })
    .Build();

await host.RunAsync();
```

### 국제화 및 리소스 접근

.NET에서는 리소스 파일(.resx)과 `IStringLocalizer<T>`를 사용하여 국제화를 지원합니다.

```csharp
public class MyService
{
    private readonly IStringLocalizer<MyService> _localizer;
    public MyService(IStringLocalizer<MyService> localizer)
    {
        _localizer = localizer;
    }
    
    public void DisplayMessage()
    {
        Console.WriteLine(_localizer["WelcomeMessage"]);
    }
}
```

### 이벤트 및 수명 주기 관리

호스트는 애플리케이션의 시작, 정지, 재시작 등의 수명 주기 이벤트를 관리합니다. `IHostedService`를 구현하여 초기화 및 정리 작업을 수행할 수 있으며, Host는 해당 작업이 완료될 때까지 기다립니다.

```csharp
public class MyBackgroundService : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            // 주기적인 작업 수행
            await Task.Delay(TimeSpan.FromSeconds(10), stoppingToken);
        }
    }
}
```

### 웹 애플리케이션의 호스트 구성

ASP.NET Core 애플리케이션에서는 `WebApplication.CreateBuilder()`를 사용하여 웹 서버, 미들웨어, 라우팅 등을 포함한 호스트를 구성합니다.

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers();
// ...existing code...
var app = builder.Build();
app.MapControllers();
app.Run();
```

## 참고자료

- [Spring Framework - Reference Document / 1.15 Additional capabilities of the ApplicationContext](https://docs.spring.io/spring-framework/docs/5.3.27/reference/html/core.html#context-introduction)
