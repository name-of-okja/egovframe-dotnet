# Service definition inheritance

## 개요

.NET에서 서비스 정의는 많은 양의 설정 정보를 포함하고 있습니다. .NET 환경에서는 Spring의 Bean 정의 상속과 유사한 개념을 다양한 방식으로 구현할 수 있습니다. 확장 또는 파생된 서비스 정의는 기본 서비스 정의로부터 설정 정보를 상속받아 사용할 수 있으며, 필요에 따라 상속받은 설정 정보를 재정의하거나 새로운 정보를 추가할 수 있습니다.

## 설명

.NET 8 이상 환경에서는 Spring의 XML 기반 빈 상속 대신 주로 다음과 같은 방식으로 서비스 정의 상속과 유사한 패턴을 구현합니다:

### 1. 클래스 상속을 사용한 설정 모델링

C# 클래스를 사용하여 설정을 정의할 때, 상속 관계를 통해 공통 속성을 공유할 수 있습니다:

```csharp
// 기본 설정 클래스
public class BaseSettings
{
    public string Name { get; set; } = "parent";
    public int Age { get; set; } = 1;
}

// 파생 설정 클래스
public class DerivedSettings : BaseSettings
{
    public DerivedSettings() 
    {
        // 생성자에서 부모 속성 재정의
        Name = "override";
        // Age는 부모에서 상속받은 1 값을 그대로 사용
    }
    
    // 추가 속성 정의
    public string AdditionalProperty { get; set; }
}
```

### 2. 계층적 구성 파일 사용

appsettings.json과 같은 구성 파일을 환경별로 계층화하여 설정 상속 효과를 얻을 수 있습니다:

```json
// appsettings.json (기본 설정)
{
  "ServiceSettings": {
    "Name": "parent",
    "Age": 1
  }
}

// appsettings.Development.json (개발 환경 특화 설정)
{
  "ServiceSettings": {
    "Name": "override"
    // Age 속성은 상속됨
  }
}
```

### 3. 의존성 주입 등록 시 옵션 패턴 활용

서비스 등록 시 옵션 패턴을 활용하여 설정 상속과 유사한 효과를 구현할 수 있습니다:

```csharp
// 프로그램 시작 시 (Program.cs)
builder.Services.AddOptions<BaseSettings>()
    .Configure(options => {
        options.Name = "parent";
        options.Age = 1;
    });

builder.Services.AddOptions<DerivedSettings>()
    .Configure<IOptions<BaseSettings>>((derived, baseOptions) => {
        // 기본 설정에서 속성 복사 (상속 효과)
        var baseSettings = baseOptions.Value;
        derived.Age = baseSettings.Age; // 기본값 상속
        
        // 특정 속성은 재정의
        derived.Name = "override";
        
        // 추가 속성 설정
        derived.AdditionalProperty = "additional";
    });
```

### 4. 서비스 등록 시 데코레이터 패턴

서비스 등록 시 데코레이터 패턴을 사용하여 기존 서비스의 동작을 확장할 수도 있습니다:

```csharp
// 기본 서비스 인터페이스
public interface IBaseService
{
    string GetName();
    int GetAge();
}

// 기본 서비스 구현
public class BaseService : IBaseService
{
    public string GetName() => "parent";
    public int GetAge() => 1;
}

// 데코레이터를 통한 서비스 확장 (상속과 유사한 효과)
public class DerivedService : IBaseService
{
    private readonly IBaseService _baseService;
    
    public DerivedService(IBaseService baseService)
    {
        _baseService = baseService;
    }
    
    public string GetName() => "override"; // 재정의
    public int GetAge() => _baseService.GetAge(); // 기본 동작 활용
}

// 서비스 등록
services.AddSingleton<BaseService>();
services.AddSingleton<IBaseService>(sp => 
    new DerivedService(sp.GetRequiredService<BaseService>()));
```

.NET의 서비스 정의에서 기본 서비스나 설정의 속성이 파생된 서비스나 설정에 명시되어 있지 않을 경우, 기본 서비스나 설정의 값이 사용됩니다. 파생된 서비스나 설정에서 속성이 명시적으로 정의되어 있는 경우, 해당 값이 기본 서비스나 설정의 값을 재정의합니다.

## 참고자료

* [.NET Configuration](https://learn.microsoft.com/dotnet/core/extensions/configuration)
* [Options pattern in .NET](https://learn.microsoft.com/dotnet/core/extensions/options)
* [Dependency injection in .NET](https://learn.microsoft.com/dotnet/core/extensions/dependency-injection)
