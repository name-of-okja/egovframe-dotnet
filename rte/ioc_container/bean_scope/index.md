# 서비스 수명 (Service Lifetime)

## 개요

## 설명

서비스 등록은 실제 서비스 객체를 생성하는 방식을 정의하는 것입니다. 클래스와 마찬가지로 하나의 서비스 등록에 해당하는 다수의 객체가 생성될 수 있습니다.

서비스 등록을 통해 객체에 다양한 종속성 및 설정값을 주입할 수 있을 뿐 아니라, 객체의 수명(lifetime)을 정의할 수 있습니다. .NET DI 컨테이너는 주로 세 가지 기본 수명을 제공합니다.

| 수명 (Lifetime)                       | 설명                                                                                                                                                                                             |
| :-------------------------------------- | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [Singleton]    | 하나의 서비스 등록에 대해서 DI 컨테이너 내에 단 하나의 공유 객체만 존재합니다. 애플리케이션 전체 수명 동안 동일한 인스턴스가 사용됩니다.                                                                               |
| [Transient]    | 하나의 서비스 등록에 대해서 요청할 때마다 새로운 객체가 생성됩니다.                                                                                                                                      |
| [Scoped]          | 특정 범위(scope) 내에서 서비스가 요청될 때 한 번 생성되며, 동일 범위 내에서는 같은 인스턴스가 사용됩니다. ASP.NET Core 애플리케이션에서 각 웹 요청은 하나의 범위를 형성합니다.                                                              |

### The Singleton Lifetime (싱글톤 수명)

서비스가 싱글톤(Singleton)으로 등록된 경우, DI 컨테이너는 해당 서비스에 대해 단 하나의 공유 인스턴스만 관리하고 반환합니다. 애플리케이션이 시작될 때 처음 요청 시 생성되어 애플리케이션이 종료될 때까지 동일한 인스턴스가 사용됩니다.

.NET에서 싱글톤 수명으로 서비스를 등록하는 방법은 다음과 같습니다:

```csharp
// IServiceCollection에 서비스 등록 (예: Program.cs 또는 Startup.cs)
services.AddSingleton<IMyService, MyService>();

// 또는 이미 생성된 인스턴스를 싱글톤으로 등록
var myServiceInstance = new MyService();
services.AddSingleton<IMyService>(myServiceInstance);
```

### The Transient Lifetime (일시적 수명)

일시적(Transient) 수명으로 등록된 서비스는 서비스 컨테이너로부터 요청될 때마다 새로운 인스턴스가 생성됩니다. 상태를 유지하지 않는 가벼운 서비스에 적합합니다.

.NET에서 일시적 수명으로 서비스를 등록하는 방법은 다음과 같습니다:

```csharp
// IServiceCollection에 서비스 등록
services.AddTransient<IMyService, MyService>();
```

일시적 수명으로 등록된 서비스는 `IDisposable`을 구현하는 경우, 해당 서비스가 생성된 범위(scope)가 해제될 때 DI 컨테이너에 의해 Dispose됩니다. 최상위 컨테이너에서 직접 resolve된 경우 애플리케이션 종료 시 또는 컨테이너가 Dispose될 때 함께 Dispose됩니다.

### 수명이 짧은 서비스에 종속적인 싱글톤 서비스 (Singleton services with shorter-lived dependencies)

싱글톤 서비스가 일시적(Transient) 또는 범위 지정(Scoped) 서비스를 직접 주입받는 경우 주의해야 합니다. 싱글톤 서비스는 애플리케이션 시작 시 한 번 생성되므로, 이때 주입받은 일시적 또는 범위 지정 서비스 인스턴스를 계속 사용하게 됩니다. 이는 의도치 않은 동작(예: 범위 지정 서비스가 요청 범위를 넘어 살아남는 것처럼 보이는 현상)을 유발할 수 있으며, 이를 "Captive Dependency" 문제라고 합니다.

이 문제를 해결하기 위해서는 싱글톤 서비스 내에서 필요할 때마다 `IServiceProvider`를 주입받아 수동으로 짧은 수명의 서비스를 요청하거나, 팩토리 패턴을 사용하여 짧은 수명의 서비스 인스턴스를 생성해야 합니다.

```csharp
public class MySingletonService
{
    private readonly IServiceProvider _serviceProvider;

    public MySingletonService(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }

    public void DoSomething()
    {
        // 필요할 때마다 Scoped 또는 Transient 서비스의 새 인스턴스를 가져옴
        using (var scope = _serviceProvider.CreateScope())
        {
            var transientService = scope.ServiceProvider.GetRequiredService<IMyTransientService>();
            transientService.Process();
        }
    }
}
```

### The Scoped Lifetime (범위 지정 수명)

범위 지정(Scoped) 수명으로 등록된 서비스는 특정 요청 범위(scope) 내에서 한 번 생성되며, 동일 범위 내에서 해당 서비스를 여러 번 요청하더라도 항상 동일한 인스턴스가 반환됩니다. 다른 요청 범위에서는 새로운 인스턴스가 생성됩니다.

ASP.NET Core 애플리케이션에서 각 웹 요청은 하나의 독립된 서비스 범위를 생성합니다. 따라서 범위 지정 서비스는 특정 HTTP 요청의 컨텍스트 내에서 상태를 공유하는 데 유용합니다 (예: Entity Framework의 `DbContext`).

.NET에서 범위 지정 수명으로 서비스를 등록하는 방법은 다음과 같습니다:

```csharp
// IServiceCollection에 서비스 등록
services.AddScoped<IMyScopedService, MyScopedService>();
```

#### 웹 애플리케이션에서의 범위 지정 수명

ASP.NET Core 애플리케이션에서는 프레임워크가 각 HTTP 요청에 대한 서비스 범위를 자동으로 생성하고 관리합니다. 따라서 별도의 설정 없이 `AddScoped`로 등록된 서비스를 컨트롤러나 다른 서비스에 주입받아 사용할 수 있습니다. 요청 처리가 완료되면 해당 범위와 함께 범위 지정 서비스 인스턴스도 해제(Dispose)됩니다.

.NET에는 Spring Framework의 `session` 또는 `globalSession`과 직접적으로 일치하는 내장 서비스 수명은 없습니다.

* **HTTP 세션과 유사한 동작**: 사용자별 세션 데이터를 저장하려면 ASP.NET Core의 세션 상태 관리 기능을 사용해야 합니다. 서비스 자체의 수명을 HTTP 세션에 직접 바인딩하는 것은 표준 DI 기능이 아닙니다. 필요하다면 `IHttpContextAccessor`를 통해 현재 `HttpContext`에 접근하여 세션 데이터를 읽거나 쓰는 서비스를 만들 수 있으며, 이 서비스 자체는 `Scoped` 또는 `Transient`로 등록될 수 있습니다.
* **Portlet 환경의 Global Session**: 이는 .NET 웹 애플리케이션의 일반적인 시나리오가 아니며, 직접적인 대응 기능은 없습니다. 유사한 전역적 공유가 필요하다면 `Singleton` 서비스를 사용하거나, 특정 요구사항에 맞는 커스텀 솔루션을 구현해야 합니다.

#### 범위 지정 서비스에 대한 종속성 (Scoped services as dependencies)

싱글톤 서비스가 범위 지정(Scoped) 서비스를 직접 주입받으면 위에서 언급한 "Captive Dependency" 문제가 발생합니다. 범위 지정 서비스는 요청마다 달라져야 하지만, 싱글톤 서비스는 애플리케이션 시작 시 한 번만 생성되어 동일한 범위 지정 인스턴스를 계속 참조하게 됩니다.

이를 해결하기 위해 싱글톤 서비스는 `IServiceProvider`를 주입받아 각 작업 단위마다 새로운 스코프를 생성하고 그 스코프로부터 범위 지정 서비스를 가져와 사용해야 합니다.

```csharp
public class MySingletonUserService
{
    private readonly IServiceProvider _serviceProvider;

    public MySingletonUserService(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }

    public UserPreferences GetUserPreferencesForCurrentUser()
    {
        // 각 호출에 대해 새 스코프를 만들어 Scoped 서비스를 가져옴
        // 실제 사용자 컨텍스트를 가져오는 로직은 IHttpContextAccessor 등을 통해 구현 필요
        using (var scope = _serviceProvider.CreateScope())
        {
            // IUserPreferences가 Scoped로 등록되어 있다고 가정
            var userPreferences = scope.ServiceProvider.GetRequiredService<IUserPreferences>();
            // ... userPreferences를 사용하여 작업 ...
            return userPreferences;
        }
    }
}

// 서비스 등록 예시
// services.AddScoped<IUserPreferences, UserPreferences>();
// services.AddSingleton<MySingletonUserService>();
```

ASP.NET Core 환경에서는 `IHttpContextAccessor`를 사용하여 현재 요청의 `HttpContext`에 접근하고, 이를 통해 요청 범위의 `IServiceProvider`를 얻어 범위 지정 서비스를 안전하게 resolve할 수도 있습니다. 그러나 `IHttpContextAccessor`의 사용은 신중해야 하며, 가능한 경우 생성자 주입을 통해 의존성을 명확히 하는 것이 좋습니다.

## 참고자료

* [.NET의 종속성 주입 - 서비스 수명 (Microsoft Docs)](https://learn.microsoft.com/dotnet/core/extensions/dependency-injection#service-lifetimes)
* [ASP.NET Core의 종속성 주입 (Microsoft Docs)](https://learn.microsoft.com/aspnet/core/fundamentals/dependency-injection)
