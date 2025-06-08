# Container extension points

## 개요

.NET의 DI(Dependency Injection) 컨테이너는 확장성과 유연성을 고려하여 설계되었습니다. 일반적으로 애플리케이션 개발자가 DI 컨테이너의 기본 구현을 상속받아 직접 확장할 필요는 없습니다.  
.NET DI 컨테이너는 다양한 인터페이스와 확장 지점을 제공하여, 개발자가 서비스 등록, 인스턴스 생성, 라이프사이클 관리, 구성 바인딩 등 다양한 영역에서 동작을 커스터마이즈할 수 있습니다.

## 설명

### IServiceProvider/IServiceCollection을 통한 확장

.NET에서는 서비스 등록과 해제, 인스턴스 생성 로직을 커스터마이즈하기 위해 `IServiceCollection` 및 `IServiceProvider` 인터페이스를 활용합니다.  
서비스 등록 시, 팩토리 메서드, 데코레이터 패턴, 옵션 패턴 등 다양한 확장 기법을 사용할 수 있습니다.

```csharp
// 서비스 등록 시 팩토리 메서드 사용 예시
services.AddSingleton<IMyService>(provider => {
    var dependency = provider.GetRequiredService<IDependency>();
    return new MyService(dependency, "custom parameter");
});
```

### 객체 생성 및 초기화 커스터마이징

.NET DI 컨테이너는 객체 생성 및 초기화 시점에 커스텀 로직을 삽입할 수 있도록 지원합니다.  
대표적으로 다음과 같은 방법이 있습니다.

- **옵션 패턴 및 구성 바인딩**: `IOptions<T>`를 통해 외부 설정 파일(appsettings.json 등)과 POCO 클래스를 바인딩하고, 필요에 따라 커스텀 초기화 로직을 추가할 수 있습니다.
- **데코레이터 패턴**: 기존 서비스의 동작을 확장하거나 변경할 때 데코레이터를 등록할 수 있습니다.
- **서비스 팩토리**: 서비스 등록 시 람다(팩토리)로 인스턴스 생성 로직을 직접 지정할 수 있습니다.

```csharp
// 데코레이터 패턴 예시
services.AddTransient<IMyService, MyService>();
services.Decorate<IMyService, MyServiceDecorator>();
```

### 구성 값의 동적 변경 및 후처리

.NET에서는 구성 값의 동적 변경이나 후처리(후킹)를 위해 `IConfigureOptions<T>`, `IPostConfigureOptions<T>` 등의 인터페이스를 제공합니다.

```csharp
// 옵션 후처리 예시
services.PostConfigure<MyOptions>(options => {
    // options 값에 대한 추가 검증 또는 후처리
    if (string.IsNullOrEmpty(options.SomeProperty))
        options.SomeProperty = "DefaultValue";
});
```

### 고급 확장: IServiceProviderFactory 및 커스텀 컨테이너

특정 시나리오에서는 기본 DI 컨테이너 대신 Autofac, SimpleInjector 등 외부 컨테이너를 통합할 수 있습니다.  
이때 `IServiceProviderFactory<TContainerBuilder>`를 구현하여 .NET의 DI 파이프라인에 커스텀 컨테이너를 연결할 수 있습니다.

```csharp
// 예시: Autofac 통합
builder.Host.UseServiceProviderFactory(new AutofacServiceProviderFactory());
```

### Factory 패턴을 통한 복잡한 인스턴스 생성

복잡한 객체 생성이 필요한 경우, .NET에서는 Factory 패턴을 직접 구현하거나, DI 컨테이너의 팩토리 지원 기능을 사용할 수 있습니다.

```csharp
public interface IMyFactory
{
    MyComplexObject Create(string parameter);
}

public class MyFactory : IMyFactory
{
    private readonly IServiceProvider _provider;
    public MyFactory(IServiceProvider provider) => _provider = provider;

    public MyComplexObject Create(string parameter)
    {
        var dep = _provider.GetRequiredService<IDependency>();
        return new MyComplexObject(dep, parameter);
    }
}

// 등록
services.AddSingleton<IMyFactory, MyFactory>();
```

## 참고자료

- [.NET Dependency injection 기본 문서](https://learn.microsoft.com/dotnet/core/extensions/dependency-injection)
- [Options pattern in .NET](https://learn.microsoft.com/dotnet/core/extensions/options)
- [IServiceProviderFactory 설명](https://learn.microsoft.com/dotnet/core/extensions/dependency-injection-usage)
