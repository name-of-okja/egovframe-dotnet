# .NET 서비스의 생명주기 사용자 정의

## 개요

.NET에서는 서비스 컨테이너가 관리하는 객체(서비스)의 생명주기와 상호 작용하기 위해 다양한 메커니즘을 사용할 수 있습니다. 예를 들어, `IHostedService` 인터페이스를 구현하여 애플리케이션 시작 및 종료 시점에 특정 작업을 수행하도록 할 수 있으며, `IDisposable` 또는 `IAsyncDisposable` 인터페이스를 구현하여 서비스가 소멸될 때 리소스를 정리하도록 할 수 있습니다. 초기화 로직은 주로 생성자에서 처리되거나, 필요한 경우 별도의 초기화 메서드를 통해 수행될 수 있습니다.

## 설명

### 생명주기 콜백 (Lifecycle callbacks)

.NET Core의 의존성 주입(DI) 컨테이너 및 호스팅 환경은 서비스의 생명주기 동안 특정 시점에 코드를 실행할 수 있는 여러 방법을 제공합니다.

#### 객체화 콜백 (Initialization callbacks)

.NET에서 서비스의 기본적인 초기화는 해당 서비스의 생성자에서 수행됩니다. 의존성 주입 컨테이너는 서비스가 요청될 때 필요한 의존성을 생성자에 주입하여 객체를 생성합니다.

```csharp
public class MyService
{
    private readonly IOtherService _otherService;

    // 생성자에서 의존성 주입 및 기본 초기화 수행
    public MyService(IOtherService otherService)
    {
        _otherService = otherService;
        // 추가적인 초기화 작업 수행 가능
        Console.WriteLine("MyService가 초기화되었습니다.");
    }

    public void DoWork()
    {
        Console.WriteLine("MyService 작업 수행 중...");
        _otherService.PerformAction();
    }
}

public interface IOtherService { void PerformAction(); }
public class OtherService : IOtherService 
{
    public void PerformAction() => Console.WriteLine("OtherService 작업 수행.");
}
```

호스팅 환경(예: ASP.NET Core, Generic Host)에서 백그라운드 작업이나 애플리케이션 시작 시 특별한 초기화가 필요한 서비스의 경우, `IHostedService` 인터페이스를 구현할 수 있습니다. `StartAsync` 메서드는 애플리케이션 시작 시 호출됩니다.

```csharp
public class MyHostedService : IHostedService
{
    public Task StartAsync(CancellationToken cancellationToken)
    {
        Console.WriteLine("MyHostedService가 시작되었습니다.");
        // 비동기 초기화 작업 수행
        return Task.CompletedTask;
    }

    public Task StopAsync(CancellationToken cancellationToken)
    {
        Console.WriteLine("MyHostedService가 중지되었습니다.");
        return Task.CompletedTask;
    }
}
```

서비스를 등록할 때 다음과 같이 설정합니다.

```csharp
// Program.cs 또는 Startup.cs
// builder.Services.AddSingleton<IOtherService, OtherService>();
// builder.Services.AddTransient<MyService>();
// builder.Services.AddHostedService<MyHostedService>();
```

#### 파괴 콜백 (Destruction callbacks)

.NET에서 서비스가 관리하는 리소스를 해제해야 할 때 `IDisposable` 또는 `IAsyncDisposable` 인터페이스를 구현합니다. DI 컨테이너는 서비스의 생명주기가 끝나면 (예: 스코프 종료 또는 컨테이너 폐기 시) `Dispose` 또는 `DisposeAsync` 메서드를 자동으로 호출합니다.

```csharp
public class MyDisposableService : IDisposable, IAsyncDisposable
{
    private bool _disposed = false;

    public void DoWork()
    {
        if (_disposed)
            throw new ObjectDisposedException(nameof(MyDisposableService));
        Console.WriteLine("MyDisposableService 작업 수행 중...");
    }

    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);
    }

    protected virtual void Dispose(bool disposing)
    {
        if (_disposed) return;

        if (disposing)
        {
            // 관리되는 리소스 해제
            Console.WriteLine("MyDisposableService (동기) 리소스 해제됨.");
        }
        // 관리되지 않는 리소스 해제 (필요한 경우)
        _disposed = true;
    }

    public async ValueTask DisposeAsync()
    {
        // 비동기 Dispose 로직
        // Dispose(false)는 여기서 호출하지 않도록 주의 (패턴에 따라 다름)
        Console.WriteLine("MyDisposableService (비동기) 리소스 해제 중...");
        // 예: await _someAsyncResource.DisposeAsync();
        Dispose(false); // 관리되지 않는 리소스는 여기서도 정리할 수 있으나, 패턴 주의
        GC.SuppressFinalize(this);
        _disposed = true; // 비동기 완료 후 상태 업데이트
        await Task.CompletedTask; // 실제 비동기 작업이 있다면 await
    }
}
```

`IHostedService`를 구현한 서비스의 경우, `StopAsync` 메서드가 애플리케이션 종료 시 호출되어 정리 작업을 수행할 기회를 제공합니다.

```csharp
public class MyHostedServiceWithCleanup : IHostedService, IDisposable
{
    public Task StartAsync(CancellationToken cancellationToken)
    {
        Console.WriteLine("MyHostedServiceWithCleanup 시작됨.");
        return Task.CompletedTask;
    }

    public Task StopAsync(CancellationToken cancellationToken)
    {
        Console.WriteLine("MyHostedServiceWithCleanup 중지됨. 정리 작업 수행.");
        // 예: 백그라운드 스레드 정상 종료 대기
        return Task.CompletedTask;
    }

    public void Dispose()
    {
        Console.WriteLine("MyHostedServiceWithCleanup Disposed.");
        // IHostedService가 IDisposable을 구현하면 호스트가 Dispose도 호출
    }
}
```

#### 생명주기 메커니즘 조합

.NET에서는 다음과 같은 주요 생명주기 관리 메커니즘이 있습니다:

1. **생성자**: 객체 생성 및 기본적인 의존성 주입 시점.
2. **`IHostedService`**:
    * `StartAsync`: 호스트 시작 시 서비스 초기화 및 장기 실행 작업 시작.
    * `StopAsync`: 호스트 종료 시 서비스의 정상적인 중지 및 정리.
3. **`IDisposable` / `IAsyncDisposable`**:
    * `Dispose()` / `DisposeAsync()`: 서비스가 DI 컨테이너에 의해 폐기될 때 리소스 정리.

호출 순서는 일반적으로 다음과 같습니다(서비스의 종류와 등록 방식에 따라 다를 수 있음):

* **초기화 시**:
    1. 생성자 호출 (의존성 주입 완료).
    2. `IHostedService.StartAsync()` (호스트에 의해 관리되는 서비스의 경우).
* **소멸 시**:
    1. `IHostedService.StopAsync()` (호스트에 의해 관리되는 서비스의 경우, 정상 종료 시).
    2. `IAsyncDisposable.DisposeAsync()` 또는 `IDisposable.Dispose()` (서비스가 해당 인터페이스를 구현하고 DI 컨테이너에 의해 관리될 때).
    3. 건너뛰지 않은 경우 소멸자 호출 (결정적이지 않음)

#### 시작 및 종료 콜백 (`IHostedService` 및 `BackgroundService`)

`IHostedService` 인터페이스는 애플리케이션의 시작과 종료 시점에 코드를 실행해야 하는 객체(일반적으로 백그라운드 작업 또는 장기 실행 서비스)를 위한 핵심 메커니즘입니다.

```csharp
public interface IHostedService
{
    Task StartAsync(CancellationToken cancellationToken);
    Task StopAsync(CancellationToken cancellationToken);
}
```

모든 .NET 관리형 객체가 `IHostedService`를 구현할 필요는 없지만, 호스트(예: Generic Host 또는 Web Host)는 등록된 모든 `IHostedService` 구현에 대해 시작 시 `StartAsync`를, 종료 시 `StopAsync`를 호출합니다.

`BackgroundService`는 `IHostedService`를 구현하는 추상 클래스로, 장기 실행 백그라운드 작업을 보다 쉽게 만들 수 있도록 `ExecuteAsync(CancellationToken stoppingToken)`라는 추상 메서드를 제공합니다.

```csharp
public abstract class BackgroundService : IHostedService, IDisposable
{
    // ... 내부 구현 ...
    protected abstract Task ExecuteAsync(CancellationToken stoppingToken);

    public virtual Task StartAsync(CancellationToken cancellationToken) { /* ... */ }
    public virtual Task StopAsync(CancellationToken cancellationToken) { /* ... */ }
    public virtual void Dispose() { /* ... */ }
}

// 사용 예시
public class MyBackgroundWorker : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        Console.WriteLine("MyBackgroundWorker 시작됨.");
        while (!stoppingToken.IsCancellationRequested)
        {
            Console.WriteLine("MyBackgroundWorker 작업 중...");
            await Task.Delay(TimeSpan.FromSeconds(5), stoppingToken);
        }
        Console.WriteLine("MyBackgroundWorker 중지됨.");
    }
}
```

## 참고자료

* [.NET의 의존성 주입](https://learn.microsoft.com/ko-kr/dotnet/core/extensions/dependency-injection)
* [.NET의 BackgroundService를 사용한 백그라운드 작업](https://learn.microsoft.com/ko-kr/dotnet/core/extensions/workers)
* [IHostedService 인터페이스](https://learn.microsoft.com/ko-kr/dotnet/api/microsoft.extensions.hosting.ihostedservice)
* [IDisposable 인터페이스](https://learn.microsoft.com/ko-kr/dotnet/api/system.idisposable)
* [IAsyncDisposable 인터페이스](https://learn.microsoft.com/ko-kr/dotnet/api/system.iasyncdisposable)
