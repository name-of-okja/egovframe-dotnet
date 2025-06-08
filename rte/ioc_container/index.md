# IoC Container (.NET)

## 개요

프레임워크의 기본적인 기능인 Inversion of Control(IoC) Container 기능을 제공하는 서비스이다.
객체의 생성 시, 객체가 참조하고 있는 타 객체에 대한 종속성을 소스 코드 내부에서 직접 생성하는 것이 아닌, 소스 코드 외부의 설정(주로 코드 기반 설정)을 통해 컨테이너가 주입하도록 함으로써, 유연성 및 확장성을 향상시킨다.

### 주요 개념

#### Inversion of Control(IoC)

IoC는 Inversion of Control의 약자이다. 우리나라 말로 직역해 보면 "제어의 역전"이라고 할 수 있다. 제어의 역전 현상이 무엇인지 살펴본다.
기존에 .NET 기반으로 애플리케이션을 개발할 때 객체를 생성하고 서로 간의 의존 관계를 연결시키는 작업에 대한 제어권은 보통 개발되는 애플리케이션 코드에 있었다.
그러나, ASP.NET Core와 같은 프레임워크를 사용하는 경우, 프레임워크가 요청 처리 파이프라인, 컨트롤러 인스턴스화, 서비스 주입 등의 생명주기를 관리하게 된다.
이처럼 IoC에서 이야기하는 제어권의 역전이란 객체의 생성에서부터 생명주기의 관리까지 많은 객체에 대한 제어권이 개발자 코드에서 프레임워크 또는 컨테이너로 넘어가는 것을 의미한다.

다음은 제어 역전과 관련된 참고 자료들이다.

* [Martin Fowler](http://martinfowler.com)가 저술한 [Inversion of Control](http://martinfowler.com/bliki/InversionOfControl.html)
* [Inversion of Control 한글 번역](eGovframework/RTE/FDL/Ioc_Container/Inversion_of_Control.md) <!-- 이 링크는 일반적인 개념 설명으로 가정하고 유지합니다. -->

#### Dependency Injection

각 클래스 사이의 의존관계를 서비스 등록(Service Registration) 정보를 바탕으로 컨테이너가 자동적으로 연결해주는 것을 말한다.
컨테이너가 의존관계를 자동적으로 연결시켜주기 때문에 개발자들이 특정 구현에 직접 의존하는 코드를 작성할 필요가 줄어들고, 컨테이너가 제공하는 추상화에 의존하게 된다.
개발자들은 단지 애플리케이션의 시작 지점(예: `Program.cs`의 `WebApplication.CreateBuilder().Services` 또는 클래스 라이브러리의 `IServiceCollection` 확장 메서드)에서 서비스와 그 구현을 등록하기만 하면 된다.

다음은 의존성 주입과 관련된 참고 자료들이다.

* [Martin Fowler](http://martinfowler.com)가 저술한 [Inversion of Control Containers and the Dependency Injection pattern](http://martinfowler.com/articles/injection.html)
* [Dependency Injection in .NET](https://learn.microsoft.com/en-us/dotnet/core/extensions/dependency-injection) (Microsoft Docs)

### 사용된 .NET 기능

* IoC Container는 .NET에 내장된 [Microsoft.Extensions.DependencyInjection](https://learn.microsoft.com/en-us/dotnet/api/microsoft.extensions.dependencyinjection) 라이브러리를 주로 사용한다.

## 설명

본 IoC Container는 .NET의 `Microsoft.Extensions.DependencyInjection` 기능을 사용하는 것으로, 본 가이드 문서는 [Microsoft .NET Dependency Injection 문서](https://learn.microsoft.com/en-us/dotnet/core/extensions/dependency-injection)를 참조 및 요약한 것이다.
.NET DI 컨테이너에 대한 상세한 설명이 필요한 경우, Microsoft .NET 문서 원본 및 관련 API 문서를 참조한다.

### IoC Container in .NET

`Microsoft.Extensions.DependencyInjection` 네임스페이스는 .NET의 IoC 컨테이너의 기반을 제공한다. `IServiceCollection` 인터페이스는 서비스를 등록하기 위한 메커니즘을 제공하며, 이를 통해 구성된 `IServiceProvider` 인터페이스는 등록된 서비스를 해석(resolve)하고 애플리케이션의 여러 부분에 제공하는 역할을 한다.

일반적으로 ASP.NET Core 애플리케이션에서는 `WebApplicationBuilder.Services` (`IServiceCollection`의 인스턴스)를 사용하여 서비스를 등록한다. 다른 유형의 애플리케이션(예: 콘솔 앱, 워커 서비스)에서는 `Host.CreateDefaultBuilder().ConfigureServices(...)` 등을 통해 유사한 방식으로 서비스를 구성할 수 있다.

요약하면, `IServiceCollection`은 서비스 등록을 위한 명세를 제공하고, 이를 통해 생성된 `IServiceProvider`는 등록된 서비스를 실제로 해석하고 제공하는 역할을 한다. .NET DI 컨테이너는 생성자 주입(constructor injection)을 주로 사용하며, 서비스의 생명주기 관리(Singleton, Scoped, Transient) 기능을 제공한다.

본 문서는 .NET DI 컨테이너의 기본적인 원리와 사용 방법을 설명한다.

* [Basics](./basics/index.md)
  .NET DI 컨테이너의 기본적인 개념 및 사용 방법을 설명한다.
* [Attribute-based configuration](./annotation-based_configuration/index.md)
  .NET에서 특성(Attributes)을 사용한 의존성 주입과 생성자 주입을 권장하는 이유를 설명한다.
* [Assembly scanning for managed components](./classpath_scanning_for_managed_components/index.md)
  어셈블리 스캐닝을 통해 서비스를 자동으로 발견하고 등록하는 방법과 .NET의 명시적 등록 철학을 설명한다.
* [.NET 표준 의존성 주입 사용하기](./jsr_330_standard_annotations/index.md)
  Java의 JSR-330 표준과 달리 .NET에서 제공하는 표준 의존성 주입 방식을 설명한다.
* [Code-based configuration](./java-based_configuration/index.md)
  .NET에서 코드를 사용하여 의존성 주입 컨테이너를 구성하는 방법을 설명한다.
* [Environment Abstraction](./environment_abstraction/index.md)
  .NET에서 환경별 구성과 구성 시스템을 활용하는 방법을 설명한다.

## 참고자료

* [Dependency injection in .NET](https://learn.microsoft.com/en-us/dotnet/core/extensions/dependency-injection) (Microsoft Docs)
* [ASP.NET Core Dependency Injection](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/dependency-injection) (Microsoft Docs)
* [Inversion of Control](http://martinfowler.com/bliki/InversionOfControl.html)
* [Inversion of Control Containers and the Dependency Injection pattern](http://martinfowler.com/articles/injection.html)
* [최범균(Blog 자바캔(Java Can Do It))의 관련 번역글](http://javacan.tistory.com/entry/120) (원문은 Java 기반이나 DI 패턴 자체에 대한 이해에 도움이 될 수 있음)
