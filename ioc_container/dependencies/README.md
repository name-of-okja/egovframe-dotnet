# Dependencies (종속성)

## 개요

일반적인 엔터프라이즈 애플리케이션은 단일 객체(또는 .NET 용어로 서비스)로만 이루어지지 않고, 간단한 애플리케이션도 최종 사용자에게 일관된 사용자 경험을 제공하기 위해 여러 객체가 함께 작동합니다. 이러한 객체들은 독립적으로 존재하며, .NET DI 컨테이너를 사용하여 각각의 서비스로 정의(등록)됩니다. 여기서는 독립적으로 정의된 여러 서비스들이 협업하여 목표를 달성하는 방법에 대해 설명합니다.

## 설명

### 종속성 주입(Injecting dependencies)

종속성 주입(Dependency Injection, DI)의 기본적인 원칙은 객체가 주로 생성자를 통해서만 자신의 종속성(필요로 하는 다른 서비스)을 정의(요청)한다는 것입니다.
그러면 DI 컨테이너는 서비스 객체를 생성할 때, 해당 서비스가 정의한 종속성을 자동으로 주입합니다. 이는 서비스가 스스로 필요한 객체를 생성하거나 찾는 등의 제어를 가지는 것과는 반대의 개념으로 제어의 역전(Inversion of Control, IoC)이라고 부릅니다.
.NET에서 종속성 주입의 가장 일반적인 방법은 생성자 주입(Constructor Injection)입니다. 속성 주입(Property Injection)도 가능하지만, 내장 DI 컨테이너에서는 직접적인 지원이 제한적이며 생성자 주입이 권장됩니다.

### Constructor Injection (생성자 주입)

생성자 기반 DI는 서비스가 필요로 하는 다른 서비스들을 생성자의 매개변수로 선언하면, DI 컨테이너가 해당 타입의 등록된 서비스 인스턴스를 자동으로 주입하는 방식입니다.

```csharp
// C# 서비스 클래스 예제
namespace MyProject.Services
{
    public interface ILogger
    {
        void Log(string message);
    }

    public class ConsoleLogger : ILogger
    {
        public void Log(string message)
        {
            Console.WriteLine($"[LOG]: {message}");
        }
    }

    public interface IMessageHandler
    {
        void ProcessMessage(string message);
    }

    public class MessageHandler : IMessageHandler
    {
        private readonly ILogger _logger;
        private readonly IConfiguration _configuration;

        // 생성자를 통해 ILogger와 IConfiguration 종속성을 주입받음
        public MessageHandler(ILogger logger, IConfiguration configuration)
        {
            _logger = logger;
            _configuration = configuration;
        }

        public void ProcessMessage(string message)
        {
            string greeting = _configuration["Greeting"] ?? "Hello";
            _logger.Log($"{greeting}, {message}!");
            // ... 메시지 처리 로직 ...
        }
    }
}
```

```csharp
// .NET 서비스 등록 (예: Program.cs 또는 Startup.cs의 ConfigureServices)
// builder.Services 또는 services (IServiceCollection)에 등록

// ILogger 서비스 등록
services.AddSingleton<ILogger, ConsoleLogger>();
// IMessageHandler 서비스 등록
services.AddTransient<IMessageHandler, MessageHandler>();

// IConfiguration은 일반적으로 프레임워크에 의해 이미 등록되어 있음
// 예: builder.Configuration 객체가 IConfiguration으로 사용 가능
```

위 예제에서 `MessageHandler`는 `ILogger`와 `IConfiguration` 타입의 서비스를 생성자를 통해 요청합니다. DI 컨테이너는 `MessageHandler`의 인스턴스를 생성할 때, 등록된 `ConsoleLogger` 인스턴스와 프레임워크가 제공하는 `IConfiguration` 인스턴스를 자동으로 주입합니다.

#### 생성자 인수 타입 매칭 (Constructor Argument Type Matching)

.NET DI 컨테이너는 생성자 매개변수의 **타입**을 기준으로 주입할 서비스를 결정합니다. 만약 동일 타입의 서비스가 여러 개 등록되어 있고 특정 구현을 선택해야 하는 경우, 추가적인 설정이나 팩토리 패턴이 필요할 수 있지만, 일반적인 경우는 타입으로 유일하게 식별됩니다.
기본형(primitive type)이나 문자열 같은 간단한 값은 보통 DI를 통해 직접 주입하기보다는, 옵션 패턴(`IOptions<T>`)을 사용하거나 팩토리 메서드 내에서 설정 값을 참조하여 주입합니다.

```csharp
public class MyServiceWithOptions
{
    private readonly int _retryAttempts;
    private readonly string _serviceUrl;

    public MyServiceWithOptions(IOptions<MyServiceOptionsConfig> options)
    {
        _retryAttempts = options.Value.RetryAttempts;
        _serviceUrl = options.Value.ServiceUrl;
    }
    // ...
}

public class MyServiceOptionsConfig
{
    public int RetryAttempts { get; set; }
    public string ServiceUrl { get; set; }
}

// appsettings.json
// {
//   "MyServiceOptionsConfig": {
//     "RetryAttempts": 3,
//     "ServiceUrl": "https://api.example.com"
//   }
// }

// 서비스 등록 (Program.cs)
// services.Configure<MyServiceOptionsConfig>(builder.Configuration.GetSection("MyServiceOptionsConfig"));
// services.AddTransient<MyServiceWithOptions>();
```

### Setter Injection (속성 주입)

속성(Setter) 기반 DI는 객체가 기본 생성자로 생성된 후, public 속성의 setter를 통해 종속성을 주입받는 방식입니다. .NET의 내장 DI 컨테이너는 생성자 주입을 기본으로 하며, 속성 주입을 위한 직접적인 메커니즘은 제공하지 않습니다. 속성 주입을 사용하려면 서비스가 활성화된 후 수동으로 속성을 설정하거나, `ActivatorUtilities.CreateInstance`를 사용할 때 속성을 채우는 등의 추가 작업이 필요하거나, Autofac과 같은 다른 DI 컨테이너를 사용해야 합니다.

```csharp
// 속성 주입을 사용하는 클래스 (내장 DI에서는 수동 설정 필요)
public class ServiceWithPropertyInjection
{
    public ILogger Logger { get; set; } // 종속성 속성

    private AnotherService _anotherService;

    // 생성자 주입과 함께 사용 가능
    public ServiceWithPropertyInjection(AnotherService anotherService)
    {
        _anotherService = anotherService;
    }

    public void DoWork()
    {
        Logger?.Log("Doing work in ServiceWithPropertyInjection.");
        _anotherService.PerformTask();
    }
}

// 내장 DI 사용 시 수동으로 속성 주입 (권장되지 않음)
// var service = serviceProvider.GetService<ServiceWithPropertyInjection>();
// if (service != null)
// {
//     service.Logger = serviceProvider.GetService<ILogger>();
// }
```

일반적으로 .NET에서는 필수 종속성은 생성자 주입을, 선택적 종속성은 메서드 주입이나 서비스 로케이터(주의해서 사용)를 통해 처리하는 것이 더 일반적입니다.

### 종속성 상세 설정 (Dependencies and configuration in detail)

#### 명확한 값 (Straight values - primitives, Strings, etc.)

앞서 언급했듯이, 간단한 값(primitive, string 등)은 일반적으로 생성자 매개변수로 직접 전달되거나, .NET의 옵션 패턴(`IOptions<T>`)을 통해 구성 파일에서 로드되어 주입됩니다.

```csharp
// 옵션 패턴 사용 예제 (appsettings.json 에서 설정)
public class MyConfigurableService
{
    private readonly MyOptions _options;

    public MyConfigurableService(IOptions<MyOptions> options)
    {
        _options = options.Value; // 실제 옵션 값에 접근
    }

    public void DisplayConfig()
    {
        Console.WriteLine($"Setting1: {_options.Setting1}, Setting2: {_options.Setting2}");
    }
}

public class MyOptions
{
    public string Setting1 { get; set; }
    public int Setting2 { get; set; }
}

// Program.cs 또는 Startup.cs 에서 옵션 설정
// services.Configure<MyOptions>(configuration.GetSection("MyApplication:MyOptionsSection"));
// services.AddTransient<MyConfigurableService>();
```

#### 다른 서비스 참조 (References to other services - collaborators)

DI 컨테이너는 등록된 서비스의 생성자 매개변수 타입을 보고, 해당 타입으로 등록된 다른 서비스를 자동으로 찾아 주입합니다.

```csharp
services.AddSingleton<IOrderRepository, SqlOrderRepository>();
services.AddTransient<IOrderService, OrderService>();

// OrderService 생성자: public OrderService(IOrderRepository orderRepository, ILogger logger)
// DI 컨테이너는 IOrderRepository로 SqlOrderRepository 인스턴스를,
// ILogger로 등록된 로거 인스턴스를 OrderService 생성 시 주입합니다.
```

#### Inner Services (내부 서비스 개념의 부재)

Spring의 "inner bean"과 같이 특정 서비스 내부에만 지역적으로 정의되는 서비스 개념은 .NET 내장 DI에 직접적으로 존재하지 않습니다. 모든 서비스는 `IServiceCollection`에 등록되며, 컨테이너에 의해 관리됩니다. 복잡한 객체 그래프는 DI 컨테이너가 종속성을 해결하면서 구성합니다. 팩토리 메서드를 사용하여 특정 서비스 인스턴스를 생성할 때 내부적으로 다른 객체를 생성하여 주입하는 것은 가능합니다.

#### Collections (컬렉션 주입)

동일한 서비스 인터페이스에 대해 여러 구현을 등록한 경우, DI 컨테이너는 해당 인터페이스의 `IEnumerable<T>`를 요청하는 생성자에 모든 등록된 구현의 인스턴스 컬렉션을 주입할 수 있습니다.

```csharp
public interface IMessageSink
{
    void Handle(string message);
}
public class EmailSink : IMessageSink { /* ... */ }
public class SmsSink : IMessageSink { /* ... */ }

// 서비스 등록
services.AddSingleton<IMessageSink, EmailSink>();
services.AddSingleton<IMessageSink, SmsSink>();

public class NotificationService
{
    private readonly IEnumerable<IMessageSink> _sinks;

    public NotificationService(IEnumerable<IMessageSink> sinks)
    {
        _sinks = sinks; // EmailSink와 SmsSink 인스턴스가 모두 주입됨
    }

    public void NotifyAll(string message)
    {
        foreach (var sink in _sinks)
        {
            sink.Handle(message);
        }
    }
}

services.AddTransient<NotificationService>();
```

#### Nulls (Null 값 처리)

.NET에서는 C#의 nullable 참조 타입 (`string?`, `IMyService?`)을 사용하여 종속성이 선택적임을 명시할 수 있습니다. DI 컨테이너가 해당 타입의 서비스를 찾지 못하면 `null`을 주입합니다 (단, `GetRequiredService<T>`와 달리 `GetService<T>` 동작 방식). 생성자에서 `null`을 허용하지 않는 타입(non-nullable reference type)의 매개변수에 대해 서비스를 찾지 못하면, 서비스 활성화 시 예외가 발생합니다.

```csharp
public class ServiceWithOptionalDependency
{
    private readonly IOptionalFeature? _optionalFeature;

    public ServiceWithOptionalDependency(IOptionalFeature? optionalFeature = null)
    {
        _optionalFeature = optionalFeature; // IOptionalFeature가 등록되지 않으면 null이 주입될 수 있음
    }

    public void TryFeature()
    {
        _optionalFeature?.PerformAction();
    }
}
// services.AddTransient<IOptionalFeature, OptionalFeatureImplementation>(); // 등록하거나 안 할 수 있음
```

#### XML 기반 설정의 간결한 표현 (Shortcuts for XML-based configuration)

.NET DI는 주로 C# 코드로 구성되므로, Spring XML 설정의 `p-namespace`와 같은 단축 표현은 직접적으로 적용되지 않습니다. C#의 객체 초기화자, 람다 식, 확장 메서드 등이 코드 기반 설정의 간결성을 제공합니다.

#### 복합 속성 이름 (Compound property names)

DI는 주로 객체 그래프의 루트에서 시작하여 종속성을 주입합니다. `foo.bar.baz`와 같은 복합 속성 경로를 DI 설정에서 직접 지정하는 방식은 일반적이지 않습니다. 대신, `foo` 서비스를 주입받고, `foo`가 내부적으로 `bar`를 관리하며, `bar`가 `baz`를 관리하는 형태로 객체 관계가 형성됩니다. 옵션 패턴을 사용할 때 구성 파일에서는 복합 경로(예: `Section:SubSection:Value`)를 사용할 수 있습니다.

### `depends-on` 사용 (Explicit Dependencies - 명시적 종속성)

.NET DI 컨테이너는 등록된 서비스 간의 종속성 그래프를 자동으로 분석하고 올바른 순서로 객체를 생성합니다. Spring의 `depends-on`과 같이 특정 서비스가 다른 서비스보다 먼저 초기화되어야 함을 명시적으로 선언하는 직접적인 기능은 내장 DI 컨테이너에 없습니다.
만약 특정 초기화 순서가 필요한 경우 (예: 정적 초기화 블록 실행 순서 제어), 다음과 같은 방법을 고려할 수 있습니다:

1. **`IHostedService`**: 백그라운드 작업이나 시작 시 초기화 로직을 위해 사용되며, 여러 `IHostedService` 간의 실행 순서는 등록 순서에 따릅니다.
2. **팩토리 메서드**: 서비스 등록 시 팩토리 메서드를 사용하여, 해당 서비스 생성 전에 필요한 다른 작업을 수행하도록 할 수 있습니다.
3. **수동 초기화 호출**: 애플리케이션 시작 부분에서 특정 모듈의 초기화 메서드를 명시적으로 호출합니다.

일반적으로는 DI 컨테이너가 종속성을 해결하도록 맡기는 것이 좋습니다.

### 늦은 객체화 (Lazily-instantiated services)

기본적으로 싱글톤 서비스는 처음 요청될 때 인스턴스화됩니다 (ASP.NET Core에서는 `IHostApplicationLifetime.ApplicationStarted` 이후에 요청될 때). 만약 서비스의 생성이 비용이 많이 들고 즉시 필요하지 않은 경우, `Lazy<T>`를 사용하여 지연 초기화를 구현할 수 있습니다.

DI 컨테이너에 `Lazy<T>`를 직접 등록하는 것이 아니라, `T`를 등록하고 `Lazy<T>`를 생성자에 주입받습니다. 컨테이너는 `Lazy<T>`를 주입할 때, `T`를 resolve할 수 있는 팩토리를 가진 `Lazy<T>` 인스턴스를 제공합니다. 실제 `T`의 인스턴스는 `Lazy<T>.Value` 속성에 처음 접근할 때 생성됩니다.

```csharp
public class ExpensiveService : IExpensiveService
{
    public ExpensiveService()
    {
        Console.WriteLine("ExpensiveService created!");
        // ... 무거운 초기화 작업 ...
    }
    public void Work() => Console.WriteLine("ExpensiveService working...");
}

public class ConsumerService
{
    private readonly Lazy<IExpensiveService> _lazyExpensiveService;

    public ConsumerService(Lazy<IExpensiveService> lazyExpensiveService)
    {
        _lazyExpensiveService = lazyExpensiveService;
        Console.WriteLine("ConsumerService created, ExpensiveService not yet (necessarily).");
    }

    public void DoWorkRequiringExpensiveService()
    {
        // _lazyExpensiveService.Value에 처음 접근할 때 ExpensiveService 인스턴스가 생성됨
        _lazyExpensiveService.Value.Work();
    }
}

// 서비스 등록
services.AddSingleton<IExpensiveService, ExpensiveService>(); // 또는 다른 수명
services.AddTransient<ConsumerService>();
```

`Lazy<T>`를 사용하면, `ConsumerService`가 생성될 때는 `ExpensiveService`가 즉시 생성되지 않고, `DoWorkRequiringExpensiveService` 메서드가 호출되어 `.Value`에 접근할 때 비로소 생성됩니다.

### 자동 연결 (Autowiring collaborators)

.NET DI 컨테이너의 핵심 기능은 종속성을 자동으로 해결하고 연결(autowire)하는 것입니다. 서비스가 생성자를 통해 다른 서비스를 요청하면, 컨테이너는 등록된 서비스 중에서 해당 타입과 일치하는 것을 찾아 자동으로 주입합니다. 이는 주로 **타입 기준(byType)** 으로 작동합니다.
Spring의 `byName`과 같은 이름 기반 자동 연결은 내장 DI 컨테이너의 기본 기능이 아닙니다. 특정 이름의 인스턴스를 구분해야 한다면, 팩토리 패턴이나 명명된 옵션 등을 사용해야 합니다.

기본적으로 생성자 매개변수 타입과 일치하는 등록된 서비스가 주입됩니다. 만약 일치하는 타입의 서비스가 없으면 예외가 발생합니다 (단, nullable 타입이나 기본값이 있는 매개변수는 예외).

### 특정 서비스를 자동 연결 대상에서 제외 (Excluding a service from autowiring)

서비스를 DI 컨테이너에 등록하지 않으면, 해당 서비스는 다른 서비스에 자동으로 주입될 수 없습니다. 만약 특정 구현을 내부적으로만 사용하고 DI를 통해 외부로 노출하고 싶지 않다면, 해당 구현을 컨테이너에 인터페이스로 등록하지 않거나, 접근 제한자(예: `internal`)를 사용하여 가시성을 제어할 수 있습니다.

### 종속성 검사 (Checking for dependencies)

.NET DI 컨테이너는 서비스 활성화 시점(즉, `serviceProvider.GetService<T>()` 또는 `serviceProvider.GetRequiredService<T>()` 호출 시)에 종속성 해결을 시도합니다. 만약 요청된 서비스나 그 하위 종속성 중 하나라도 컨테이너에 등록되어 있지 않거나 (non-nullable 타입의 경우) 해결할 수 없다면, 예외(보통 `InvalidOperationException`)가 발생합니다. 이를 통해 애플리케이션 시작 단계나 해당 기능이 처음 사용될 때 종속성 문제를 발견할 수 있습니다.
Spring의 `dependency-check` 속성과 같은 사전 검사 기능은 없지만, 애플리케이션 시작 시 주요 서비스들을 수동으로 resolve 해보는 테스트를 추가하여 유사한 효과를 얻을 수 있습니다.

### 메서드 주입 (Method Injection) 과 유사한 패턴

Spring의 "Lookup Method Injection"은 싱글톤 서비스가 매번 새로운 프로토타입(일시적) 서비스 인스턴스를 필요로 할 때 사용됩니다. .NET에서 유사한 요구사항을 해결하는 방법은 다음과 같습니다:

1. **`IServiceProvider` 직접 주입**: 싱글톤 서비스가 `IServiceProvider`를 주입받고, 필요할 때마다 `serviceProvider.GetRequiredService<IMyTransientService>()`를 호출하여 새로운 일시적 서비스 인스턴스를 가져옵니다. 이 방법은 서비스 로케이터 패턴의 안티패턴으로 간주될 수 있으므로 신중하게 사용해야 합니다.

    ```csharp
    public class SingletonManager
    {
        private readonly IServiceProvider _serviceProvider;

        public SingletonManager(IServiceProvider serviceProvider)
        {
            _serviceProvider = serviceProvider;
        }

        public void ProcessCommand()
        {
            // 매번 새로운 Command 인스턴스를 가져옴 (IMyCommand가 Transient로 등록된 경우)
            using (var scope = _serviceProvider.CreateScope()) // Transient라도 Scoped 서비스에 의존할 수 있으므로 스코프 관리
            {
                 var command = scope.ServiceProvider.GetRequiredService<IMyCommand>();
                 command.Execute();
            }
        }
    }
    // services.AddTransient<IMyCommand, MyCommand>();
    // services.AddSingleton<SingletonManager>();
    ```

2. **팩토리(Factory) 주입**: 일시적 서비스를 생성하는 팩토리를 정의하고, 싱글톤 서비스가 이 팩토리를 주입받도록 합니다.

    ```csharp
    // IMyCommand가 Transient로 등록되어 있다고 가정
    // public delegate IMyCommand MyCommandFactory();
    // services.AddTransient<IMyCommand, MyCommand>();
    // services.AddSingleton<MyCommandFactory>(sp => () => sp.GetRequiredService<IMyCommand>());

    // 또는 더 명시적인 팩토리 인터페이스
    public interface IMyCommandFactory
    {
        IMyCommand CreateCommand();
    }

    public class MyCommandFactory : IMyCommandFactory
    {
        private readonly IServiceProvider _serviceProvider;
        public MyCommandFactory(IServiceProvider serviceProvider) // 팩토리 자체도 DI를 활용 가능
        {
            _serviceProvider = serviceProvider;
        }
        public IMyCommand CreateCommand()
        {
            return _serviceProvider.GetRequiredService<IMyCommand>();
        }
    }
    // services.AddTransient<IMyCommand, MyCommand>();
    // services.AddSingleton<IMyCommandFactory, MyCommandFactory>();


    public class SingletonManagerWithFactory
    {
        private readonly IMyCommandFactory _commandFactory;

        public SingletonManagerWithFactory(IMyCommandFactory commandFactory)
        {
            _commandFactory = commandFactory;
        }

        public void ProcessCommand()
        {
            IMyCommand command = _commandFactory.CreateCommand();
            command.Execute();
        }
    }
    // services.AddSingleton<SingletonManagerWithFactory>();
    ```

    팩토리 패턴이 일반적으로 더 깔끔하고 테스트하기 쉬운 코드를 만듭니다. .NET에서는 CGLIB와 같은 동적 프록시 라이브러리를 사용하여 메서드를 재정의하는 방식의 메서드 주입은 내장 DI 기능이 아닙니다.

## 참고자료

* [.NET의 종속성 주입 (Microsoft Docs)](https://learn.microsoft.com/dotnet/core/extensions/dependency-injection)
* [ASP.NET Core의 종속성 주입 (Microsoft Docs)](https://learn.microsoft.com/aspnet/core/fundamentals/dependency-injection)
* [팩토리 기반 종속성 주입 (Microsoft Docs 가이드라인)](https://learn.microsoft.com/dotnet/architecture/modern-web-apps-azure/architectural-principles#dependency-injection)
