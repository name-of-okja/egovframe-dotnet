# Basics

## 개요

.NET 환경에서 객체가 생성자 인수, 팩토리 메서드에 대한 인수 또는 객체 인스턴스가 생성되거나 팩토리 메서드에서 반환된 후 객체 인스턴스에 설정된 속성을 통해서만 종속성(함께 작업하는 다른 객체)을 정의하는 프로세스를 제어의 역전(Inversion of Control, IoC)이라고 합니다. 의존성 주입(Dependency Injection, DI)은 모듈 간의 의존성을 모듈의 외부 컨테이너(서비스 컨테이너)에서 주입시켜주는 기능으로 IoC의 핵심 구현 방식입니다.

## 설명

.NET에서 서비스(Service)는 애플리케이션을 구성하고, .NET의 내장 DI 컨테이너에 의해 관리되는 객체입니다. 간단히 말해 DI 컨테이너에 의해 객체화되고, 구성(조립)되며, 관리되는 객체를 의미합니다.
서비스들과 서비스들 간의 종속성은 DI 컨테이너에 서비스를 등록할 때 정의된 설정(주로 C# 코드)에 의해 결정됩니다.

### The Container (서비스 컨테이너)

`Microsoft.Extensions.DependencyInjection.IServiceProvider` 인터페이스는 .NET DI 컨테이너의 핵심 인터페이스로, 등록된 서비스들의 인스턴스를 제공(resolve)하는 역할을 합니다. 서비스 등록 자체는 `Microsoft.Extensions.DependencyInjection.IServiceCollection` 인터페이스를 통해 이루어집니다. DI 컨테이너는 객체를 생성하고, 객체 간의 종속성을 해결하여 주입하는 역할을 합니다.

### 설정 정보 (Service Registration)

.NET DI 컨테이너는 어떤 서비스를 어떻게 생성하고 관리할지에 대한 설정 정보를 필요로 합니다. 이 정보는 주로 애플리케이션 시작 코드(예: `Program.cs`의 `WebApplication.CreateBuilder()` 이후 또는 `Startup.cs`의 `ConfigureServices` 메서드)에서 C# 코드를 사용하여 `IServiceCollection`에 서비스를 등록하는 형태로 제공됩니다.

다음은 C# 코드를 사용하여 서비스를 등록하는 기본적인 예제입니다.

```csharp
// Program.cs 또는 Startup.cs 등에서
public void ConfigureServices(IServiceCollection services)
{
    // Singleton 수명의 서비스 등록
    services.AddSingleton<IMySingletonService, MySingletonService>();

    // Scoped 수명의 서비스 등록 (예: 웹 요청 범위)
    services.AddScoped<IMyScopedService, MyScopedService>();

    // Transient 수명의 서비스 등록 (요청할 때마다 새 인스턴스)
    services.AddTransient<IMyTransientService, MyTransientService>();

    // 기타 서비스 등록...
}
```

위 예제에서 `services.AddSingleton`, `services.AddScoped`, `services.AddTransient`와 같은 확장 메서드를 사용하여 서비스의 수명(lifetime)과 함께 서비스 인터페이스(또는 추상 클래스)와 해당 구현 클래스를 DI 컨테이너에 등록합니다.

### Container 객체화 (서비스 공급자 빌드)

ASP.NET Core 애플리케이션에서는 프레임워크가 `IServiceCollection`에 등록된 서비스들을 기반으로 `IServiceProvider`를 자동으로 빌드하고 관리합니다. 콘솔 애플리케이션 등 다른 환경에서는 `IServiceCollection`의 `BuildServiceProvider()` 메서드를 호출하여 수동으로 `IServiceProvider` 인스턴스를 생성할 수 있습니다.

```csharp
// 콘솔 애플리케이션 예시
var services = new ServiceCollection();

// 서비스 등록
services.AddTransient<IMyService, MyService>();
// ... 기타 서비스 등록

// 서비스 공급자 빌드
IServiceProvider serviceProvider = services.BuildServiceProvider();

// 이제 serviceProvider를 사용하여 서비스 인스턴스를 가져올 수 있습니다.
```

### Container 사용 방법 (서비스 사용)

`IServiceProvider`가 준비되면, `GetService<T>()` 또는 `GetRequiredService<T>()` 확장 메서드를 사용하여 등록된 서비스의 인스턴스를 가져올 수 있습니다.

```csharp
// IServiceProvider 인스턴스가 있다고 가정 (예: 생성자 주입, 또는 위에서 빌드한 serviceProvider)
IMyService myService = serviceProvider.GetService<IMyService>(); // 서비스가 없으면 null 반환
IMyAnotherService anotherService = serviceProvider.GetRequiredService<IMyAnotherService>(); // 서비스가 없으면 예외 발생

if (myService != null)
{
    myService.DoSomething();
}
```

일반적으로 애플리케이션 코드에서 직접 `IServiceProvider`를 사용하여 서비스를 가져오는 것(Service Locator 패턴)보다는, 필요한 서비스를 생성자를 통해 주입받는 방식(Constructor Injection)이 권장됩니다.

#### The Services (서비스)

.NET DI 컨테이너는 다수의 *서비스*들을 관리합니다. 컨테이너는 서비스 등록 정보를 사용하여 서비스 인스턴스를 생성하고 종속성을 해결합니다. 컨테이너에 등록되는 서비스 정의는 일반적으로 다음 정보를 포함합니다.

* **서비스 타입 (Service Type)**: 서비스의 계약을 정의하는 인터페이스 또는 추상 클래스입니다.
* **구현 타입 (Implementation Type)**: 서비스 타입의 구체적인 구현 클래스입니다.
* **수명 (Lifetime)**: 서비스 인스턴스가 생성되고 유지되는 방식을 정의합니다 (예: `Singleton`, `Scoped`, `Transient`). 자세한 내용은 [서비스 수명](#서비스-수명service-lifetimes) 섹션에서 다룹니다.
* **종속성 (Dependencies)**: 해당 서비스가 동작하기 위해 필요한 다른 서비스들입니다. 이러한 종속성은 주로 생성자 주입을 통해 컨테이너가 자동으로 해결합니다.
* **팩토리 메서드 (Factory Method)**: 경우에 따라 서비스 인스턴스를 생성하기 위한 커스텀 로직을 제공하는 팩토리 함수를 등록할 수 있습니다.

### 서비스 식별 (Service Identification)

.NET DI에서 서비스는 주로 해당 **타입**(인터페이스 또는 클래스)으로 식별됩니다. `GetService<T>()` 호출 시 제네릭 타입 `T`가 요청하는 서비스의 타입이 됩니다.

하나의 서비스 타입에 대해 여러 구현을 등록하고 구분해야 하는 경우(예: 이름 기반 서비스), 기본 DI 컨테이너만으로는 직접적인 "이름 지정" 기능이 제한적입니다. 이러한 시나리오를 위해서는 추가적인 패턴(예: 팩토리 패턴 사용, 조건부 등록)을 적용하거나, 서드파티 DI 컨테이너를 고려할 수 있습니다. 하지만 대부분의 경우 타입 기반 해결로 충분합니다.

### 서비스 별칭 또는 다중 인터페이스 등록 (Registering Multiple Interfaces / Forwarding)

.NET DI에는 Spring의 `<alias>`와 직접적으로 동일한 개념은 없지만, 유사한 목적을 달성하는 방법은 있습니다:

1. **동일 구현에 여러 인터페이스 등록**: 하나의 구현 클래스가 여러 인터페이스를 구현하는 경우, 각 인터페이스에 대해 동일한 구현 타입을 등록할 수 있습니다.

    ```csharp
    services.AddSingleton<IFirstInterface, MyImplementation>();
    services.AddSingleton<ISecondInterface, MyImplementation>(); // MyImplementation이 두 인터페이스 모두 구현
    ```

2. **서비스 전달(Forwarding)**: 특정 서비스 요청을 다른 등록된 서비스로 전달할 수 있습니다.

    ```csharp
    services.AddSingleton<IRealService, RealService>();
    services.AddSingleton<IAliasService>(provider => provider.GetRequiredService<IRealService>());
    ```

    이 경우 `IAliasService`를 요청하면 `IRealService`의 인스턴스가 반환됩니다.

### 서비스 등록 및 객체화 (Service Registration and Instantiation)

서비스는 DI 컨테이너에 등록될 때 정의된 수명 주기에 따라 객체화됩니다. 컨테이너는 서비스의 생성자에서 다른 등록된 서비스들을 자동으로 주입하여 종속성을 해결합니다.

다음은 다양한 서비스 등록 방법의 예입니다:

```csharp
// 타입으로 등록 (가장 일반적)
services.AddTransient<IMyService, MyService>();
services.AddScoped<IAnotherService, AnotherService>();
services.AddSingleton<ISharedService, SharedService>();

// 팩토리 함수를 사용하여 등록
services.AddSingleton<IMyComplexService>(provider =>
{
    var dependency = provider.GetRequiredService<IDependency>();
    return new MyComplexService(dependency, "some_configuration_value");
});

// 이미 생성된 인스턴스를 Singleton으로 등록
var preExistingInstance = new MyPreExistingService();
services.AddSingleton<IMyPreExistingService>(preExistingInstance);
```

생성자를 통한 의존성 주입이 가장 일반적이며 권장되는 방식입니다. 컨테이너는 등록된 서비스의 생성자를 분석하고, 필요한 매개변수 타입에 해당하는 다른 서비스들을 자동으로 찾아 주입합니다.

## 서비스 수명 (Service Lifetimes)

.NET DI 컨테이너는 다음과 같은 주요 서비스 수명을 지원합니다:

* **Transient**: `AddTransient`로 등록된 서비스는 요청될 때마다 새로운 인스턴스가 생성됩니다. 짧은 작업을 수행하고 상태를 유지할 필요가 없는 서비스에 적합합니다.
* **Scoped**: `AddScoped`로 등록된 서비스는 특정 범위(scope) 내에서 한 번 생성되고, 동일 범위 내의 모든 요청에 대해 동일한 인스턴스가 사용됩니다. ASP.NET Core에서는 각 웹 요청이 하나의 범위를 형성합니다. 웹 요청 처리 중 상태를 공유해야 하는 서비스에 유용합니다.
* **Singleton**: `AddSingleton`으로 등록된 서비스는 애플리케이션 수명 동안 단 하나의 인스턴스만 생성되며, 모든 요청에 대해 이 단일 인스턴스가 공유됩니다. 애플리케이션 전역에서 공유되어야 하는 상태나 설정, 무거운 객체 등에 적합합니다.

올바른 서비스 수명을 선택하는 것은 애플리케이션의 성능과 동작에 중요합니다.

## 참고자료

* [.NET의 종속성 주입 (Microsoft Docs)](https://learn.microsoft.com/dotnet/core/extensions/dependency-injection)
* [ASP.NET Core의 종속성 주입 (Microsoft Docs)](https://learn.microsoft.com/aspnet/core/fundamentals/dependency-injection)
