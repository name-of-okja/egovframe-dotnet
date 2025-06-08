# 표준프레임워크 실행환경 (.NET 8 LTS 기반) 소개

## 개요

표준프레임워크 실행환경은 응용SW의 구성기반이 되며 응용SW실행 시 필요한 기본 기능을 제공하는 환경을 의미합니다.
즉, .NET 기반 업무 구현을 목적으로 개발된 프로그램이 사용자가 의도하는 대로 정상적으로 실행될 수 있도록 지원하는 재사용 가능한 서버 실행 모듈, SW구조의 집합을 의미 합니다.

![실행환경02_1_batch.jpg](egovframework/실행환경02_1_batch.jpg) <!-- 이미지 경로는 예시로 유지합니다. -->

## 목표

.NET 표준프레임워크 실행환경은 ‘애플리케이션 서비스의 품질향상 및 정보화 투자 효율성 향상’을 위해 개발프레임워크 실행환경 표준을 정립하고, 개발프레임워크 표준 적용을 통한 응용 SW의 표준화 및 품질과 재사용성 향상을 목표로 합니다.
또한, 최신 웹 기술의 사용성과 편의성 증대를 위하여 기존 실행환경 기반 개발이 가능한 웹 기반의 표준패턴 및 가이드 코드를 제공합니다.

## 배경

현재 공공 및 기업 시스템은 유사한 기능을 가지는 다양한 종류 및 버전의 프레임워크를 개별 시스템 단위로 적용/관리하고 있으며, 이에 따라 다양한 문제점들이 발생하고 있습니다.
특정 기술 스택에 종속된 개발 프레임워크는 기술지원 없이는 응용 SW를 유지보수하기 어렵기 때문에 공급업체에 대한 의존성이 발생합니다.
복수개의 개발프레임워크가 적용된 사업의 경우, 개발프레임워크에 따라 개발표준 정의, 개발자수급, 교육시행 등 별도의 유지보수 체계를 갖추는 중복 투자가 발생하며, 개발프레임워크의 체계적인 관리절차의 미비로 동일 개발프레임워크라 하더라도 버전 관리에 어려움이 있습니다.

따라서 .NET 개발프레임워크의 표준화를 통하여

* 특정 공급업체의 개발프레임워크에 대한 기술 종속성을 배제하고,
* 프레임워크 표준화를 통해 응용 SW의 표준화와 품질 및 재사용성을 향상시키며,
* 개발 프레임워크의 유지 보수 단일화를 통한 투자 효율성을 높인다.
* 또한 최신 웹 표준 패턴 및 가이드 코드를 제시하여 프로그램 사용자의 편의성을 향상시킨다.

## 특징

표준프레임워크 실행환경은 .NET 기반 프레임워크에 대한 요구사항에 기반하고 기존 프로젝트에 적용된 프레임워크를 비교 분석하여 핵심 실행환경 프레임워크 서비스를 정의하였습니다.
표준프레임워크 실행환경은 서비스별 오픈 소스 소프트웨어 또는 .NET 기본 제공 기능을 활용하여 선정한 경량화된 표준프레임워크로서 업계 표준으로 널리 사용되는 ASP.NET Core 프레임워크를 기반으로 합니다.
연계 솔루션에 독립된 표준프레임워크 표준 연계 인터페이스를 정의하고 웹 API 기반 구현체를 제공하며, 최신 웹 UI 기술(예: Blazor, React, Angular, Vue 등)과의 통합을 지원합니다.

* 핵심 서비스 그룹 및 다수 서비스로 구성 - 표준프레임워크 실행환경은 화면처리, UX처리, 업무처리, 데이터처리, 연계통합, 배치처리, 공통기반 등의 핵심 서비스 그룹으로 구성되며, ASP.NET Core MVC/Razor Pages/Blazor, Web API, DI 컨테이너, Middleware(AOP 대안), Entity Framework Core, 인증/인가 등의 핵심 서비스를 제공합니다.
* .NET 프로젝트에 최적화된 오픈 소스 소프트웨어 및 .NET 기본 기능 활용 - 표준프레임워크 실행환경은 요구사항에 기반하여 서비스 별 오픈 소스 소프트웨어 또는 .NET SDK에 포함된 기능을 선정하였습니다.
* 경량화된 표준프레임워크로서 업계 표준인 ASP.NET Core 프레임워크를 적용 - .NET 표준을 준수하는 ASP.NET Core 프레임워크를 채택함으로써 특정 업체의 인프라나 DBMS에 독립적인 환경을 제공합니다.
* DI(Dependency Injection) 방식의 의존 관계 처리 - 표준프레임워크 실행환경은 .NET Core에 내장된 DI 컨테이너를 통해 개체나 컴포넌트 간의 의존성을 정의함으로써 변경용이성과 재사용성을 향상합니다.
* Middleware를 통한 AOP(Aspect Oriented Programming) 유사 기능 지원 - 표준프레임워크 실행환경은 ASP.NET Core의 Middleware 파이프라인을 통해 로깅, 트랜잭션, 예외처리와 같은 횡단 관심사를 분리하여 정의함으로써, 개발자가 비즈니스 업무 개발에 집중할 수 있도록 합니다.
* ASP.NET Core MVC/Razor Pages/Blazor 아키텍처 구조 제공 및 다양한 UI 클라이언트 연계 지원 - 표준프레임워크 실행환경은 ASP.NET Core를 기반으로 하며, 다양한 UI 클라이언트(웹, 모바일 웹 등) 연계를 위한 인터페이스(Web API)를 정의합니다.
* 표준프레임워크 표준 연계 인터페이스 정의 - 표준프레임워크 실행환경 표준 연계 인터페이스(Web API)를 정의함으로써 연계 솔루션에 대한 의존성을 배제하고 독립적인 어플리케이션 개발이 가능합니다.
* 최신 UX/UI 컴포넌트 및 프레임워크 활용 - 최신 웹 사용자 환경(UX/UI)에 대한 유연한 대응을 위해 Blazor 컴포넌트 또는 JavaScript 기반 UI 프레임워크(React, Angular, Vue 등)와 통합 가능한 컨트롤러 및 API 컴포넌트를 제공합니다.

## 실행환경 서비스 구성

.NET 표준프레임워크 실행환경은 주요 서비스 그룹으로 구성되며 다양한 서비스를 제공합니다. 실행환경 서비스 그룹 및 서비스는 아래 그림과 유사한 형태로 구성될 수 있습니다. (이미지는 기존 것 활용)

![civil_002_1_batch.jpg](egovframework/civil_002_1_batch.jpg) <!-- 이미지 경로는 예시로 유지합니다. -->

### 화면처리

화면처리 서비스그룹은 업무처리 서비스와 사용자간의 인터페이스를 담당하는 서비스로 사용자 화면 구성 및 사용자 입력 정보 검증 등의 기능을 지원합니다.

* Ajax Support: ASP.NET Core MVC, Razor Pages, Blazor는 기본적으로 비동기 통신 및 Ajax 패턴을 지원하며, JavaScript 라이브러리(예: htmx, jQuery)와 쉽게 통합됩니다.
* Internationalization: .NET은 리소스 파일(.resx) 및 관련 API를 통해 다국어 지원 기능을 기본으로 제공합니다.
* MVC/MVVM/Component-based: ASP.NET Core MVC, Razor Pages, Blazor 등 다양한 UI 패턴을 지원하여 사용자 화면을 개발할 수 있는 기반 구조를 제공합니다.
* Validation: DataAnnotations 및 FluentValidation과 같은 라이브러리를 통해 서버 측 및 클라이언트 측 유효성 검사 기능을 제공합니다.
* UI Adaptor: 화면 계층의 구현 방식에 따라 업무 로직 계층이 변경되는 것을 막기 위해, 업무처리 계층에서 사용할 데이터 모델(DTO)을 정의하고, 화면 계층에서 사용하는 입출력 파라미터를 해당 구현 방식에 맞게 변환해주는 기능을 제공합니다.

### UX처리

UX 처리 서비스는 최신 웹 기술의 사용성과 편의성 증대를 위하여 사용자 경험 기능을 제공하고 시각, 인터페이스, 효과 경험이 가능하도록 지원합니다.

* UX/UI Controller Component: Blazor 컴포넌트 모델 또는 JavaScript 기반 프레임워크(React, Angular, Vue 등)와 연동하여 사용자 환경(UX/UI)에 대한 유연한 대응을 지원합니다.
* HTML5 : 웹 페이지 구성 시 사용할 수 있는 마크업 언어로서 시맨틱 태그 및 디바이스 API 활용을 지원합니다.
* CSS3 : 최신 CSS 표준을 활용하여 반응형 디자인 및 다양한 시각 효과를 지원합니다.
* JavaScript/TypeScript Frameworks : ASP.NET Core는 React, Angular, Vue.js 등 주요 JavaScript/TypeScript 프레임워크와의 통합 템플릿 및 지원을 제공합니다.

### 업무처리

업무처리 서비스는 업무 프로그램의 업무 로직을 담당하는 서비스로 업무 흐름제어, 에러 처리 등의 기능을 제공합니다.

* Process Control: C# 클래스 및 서비스를 통해 비즈니스 로직과 업무 흐름을 구현하며, 필요시 워크플로우 엔진(예: Elsa Workflow)과 통합할 수 있습니다.
* Exception Handling: .NET의 강력한 예외 처리 메커니즘과 ASP.NET Core의 Middleware를 통해 표준화된 예외 처리 방법을 제공합니다.

### 데이터처리

데이터처리 서비스는 데이터베이스에 대한 연결 및 영속성 처리, 선언적인 트랜잭션 관리를 지원합니다.

* Data Access: Entity Framework Core (EF Core) 또는 Dapper와 같은 ORM/Micro-ORM을 통해 다양한 데이터베이스에 대한 추상화된 접근 방식을 제공합니다.
* Data Source: `appsettings.json` 파일 및 `DbContextOptions`를 통해 데이터베이스 연결 문자열을 관리하고, DI를 통해 `DbContext`를 주입받아 사용합니다.
* ORM: Entity Framework Core를 통해 객체 모델과 관계형 데이터베이스 간의 매핑(O/RM) 기능을 제공하여, SQL 중심이 아닌 객체 중심의 업무 로직 작성을 지원합니다.
* Transaction: `DbContext.Database.BeginTransactionAsync()` 또는 `TransactionScope`를 사용하여 데이터베이스 트랜잭션을 관리합니다.

### 연계통합

연계통합 레이어는 타 시스템과의 연동기능을 지원합니다.

* Naming Service: .NET 환경에서는 일반적으로 설정 파일(`appsettings.json`)이나 환경 변수, 서비스 디스커버리(마이크로서비스 환경)를 통해 서비스 위치를 확인합니다.
* Web Service: ASP.NET Core Web API를 사용하여 RESTful API 또는 gRPC 서비스를 쉽게 구축하고 노출시켜 타 시스템과의 연동을 지원합니다.
* Integration Service: 외부 시스템과의 연동을 위한 공통 인터페이스(API 클라이언트, 메시지 큐 연동 등)를 제공하여, 다양한 솔루션 연계 방식에 대한 표준 처리 방식을 제공합니다.

### 배치처리

배치처리 서비스는 일괄처리 업무 구현에 필요한 기능을 제공합니다.

* Batch Core: .NET에서는 Hangfire, Quartz.NET 또는 `BackgroundService`를 사용하여 배치 작업(Job) 및 스케줄링의 기본 설정 기능을 제공합니다.
* Batch Execution: 선택한 배치 프레임워크의 실행기(Runner) 및 작업 관리 기능을 사용하며, 로깅 및 모니터링 기능을 통합합니다.
* Batch Support: 배치 수행에 관한 다양한 부가 기능(재시도, 로깅, 알림 등) 및 확장 기능을 제공합니다.

### 공통기반

공통기반 서비스는 실행환경 서비스 간에 공통적으로 사용되는 기능을 제공합니다.

* AOP: ASP.NET Core Middleware 또는 PostSharp, Castle DynamicProxy (별도 라이브러리)와 같은 도구를 사용하여 관점 지향 프로그래밍(AOP)의 일부 기능을 구현하고 지원합니다.
* Cache: `IMemoryCache` (인메모리 캐시) 및 `IDistributedCache` (분산 캐시, 예: Redis, SQL Server) 인터페이스를 통해 캐싱 기능을 제공합니다.
* Compress/Decompress: `System.IO.Compression` 네임스페이스의 클래스들을 사용하여 데이터 압축 및 복원 기능을 제공합니다.
* Encryption/Decryption: `System.Security.Cryptography` 네임스페이스의 클래스들을 사용하여 데이터 암호화 및 복호화 기능을 제공합니다.
* Excel: EPPlus, ClosedXML과 같은 오픈소스 라이브러리를 사용하여 Excel 파일 포맷을 다루는 기능을 제공합니다.
* File Handling: `System.IO` 네임스페이스의 클래스들을 사용하여 파일 생성, 접근, 변경 기능을 제공합니다.
* File Upload/Download: ASP.NET Core에 내장된 기능을 사용하여 파일을 업로드 및 다운로드하는 기능을 제공합니다.
* FTP: `System.Net.FtpWebRequest` (레거시) 또는 FluentFTP, WinSCP .NET Assembly와 같은 라이브러리를 사용하여 FTP 클라이언트 기능을 제공합니다.
* ID Generation: `System.Guid.NewGuid()`를 통해 UUID(GUID)를 생성하거나, 데이터베이스 시퀀스, 또는 사용자 정의 ID 생성 전략을 사용합니다.
* IoC Container: .NET Core에 내장된 `Microsoft.Extensions.DependencyInjection`을 통해 IoC 컨테이너 기능을 기본으로 제공합니다.
* Logging: `Microsoft.Extensions.Logging` 프레임워크를 기반으로 하며, Serilog, NLog, log4net 등 다양한 로깅 프로바이더와 통합 가능합니다.
* Mail: MailKit 또는 `System.Net.Mail.SmtpClient` (플랫폼 지원에 따라 제한적일 수 있음)를 사용하여 이메일 송신 기능을 제공합니다.
* Marshalling/Unmarshalling: `System.Text.Json` (JSON), `System.Xml.Serialization` 또는 `System.Xml.Linq` (XML)를 사용하여 객체와 데이터 형식 간 변환 기능을 제공합니다.
* Object Pooling: `Microsoft.Extensions.ObjectPool`을 사용하여 객체 풀링 기능을 제공합니다.
* Property: `IConfiguration` 인터페이스 및 `appsettings.json`, 환경 변수 등을 통해 애플리케이션 설정을 관리합니다.
* Resource: .NET 리소스 파일(.resx) 및 `ResourceManager` 클래스를 통해 국제화(Internationalization) 및 현지화(Localization)를 지원합니다.
* Scheduling: Hangfire, Quartz.NET 또는 `BackgroundService`와 타이머를 조합하여 주기적/반복적 작업을 지원합니다.
* Server Security: ASP.NET Core Identity (사용자 인증 및 관리), JWT/OAuth2/OpenID Connect (API 보안 및 SSO) 등을 통해 서버 보안 기능을 제공합니다.
* String Util: `System.String` 클래스의 다양한 메소드 및 `System.Text.RegularExpressions` (정규 표현식) 등을 통해 문자열 처리 기능을 제공합니다.
* XML Manipulation: `System.Xml.Linq` (LINQ to XML) 또는 `System.Xml.XmlDocument`를 사용하여 XML 생성, 읽기, 쓰기 기능을 제공합니다.

## 실행환경 오픈소스 소프트웨어 및 .NET 기능 사용현황

실행환경의 서비스를 제공하기 위해 필요한 기반 오픈소스 소프트웨어 및 .NET SDK 기능을 도출하고, 평가 및 테스트를 통하여 서비스별 기술을 선정하였습니다.
선정 평가는 산업 표준, 라이선스, 기능 요건, 성숙도 및 확장 시 지원, 개발 환경 등 다양한 항목에 대한 종합적인 평가를 수행하였습니다.
표준프레임워크 실행환경 서비스는 선정된 오픈소스 소프트웨어 또는 .NET SDK 기능에 기반하여 재활용하거나 확장하여 구현되었으며, 일부 서비스는 특정 요구사항을 만족하는 오픈소스가 없는 경우 자체 구현될 수 있습니다.

| 서비스 그룹 | 서비스 | 오픈소스 소프트웨어 / .NET 기능 | 버전 | 확장 및 개발 |
|---|---|---|---|---|
| 화면처리 | Ajax Support | ASP.NET Core MVC/Razor Pages/Blazor, JavaScript 라이브러리 | .NET 8+ |  |
| | Internationalization | .NET Globalization/Localization (.resx) | .NET 8+ |  |
| | MVC/MVVM/Component | ASP.NET Core MVC/Razor Pages/Blazor | .NET 8+ |  |
| | Validation | DataAnnotations, FluentValidation | .NET 8+, 최신 |  |
| | UI Adaptor | 자체 구현 (DTO 패턴) | - | UI Adaptor 연동 매뉴얼 제공 |
| UX처리 | UX/UI Controller Component | Blazor, React/Angular/Vue (통합) | .NET 8+, 최신 |  |
| | HTML5 | 웹 표준 | - | HTML5 지원기능|
| | CSS3 | 웹 표준 | - | CSS3 지원기능|
| | JavaScript/TypeScript Frameworks | React, Angular, Vue.js 등 | 최신 | ASP.NET Core 통합 가이드 제공 |
| 업무처리 | Process Control | C#/.NET 기본 기능, Elsa Workflow (선택적) | .NET 8+, 최신 |  |
| | Exception Handling | .NET Exception Handling, ASP.NET Core Middleware | .NET 8+ | Exception 처리 확장 |
| 데이터처리 | Data Access | Entity Framework Core, Dapper | 최신 |  |
| | DataSource | `Microsoft.Extensions.Configuration`, `DbContext` | .NET 8+ |  |
| | ORM | Entity Framework Core | 최신 |  |
| | Transaction | EF Core Transactions, `TransactionScope` | .NET 8+ |  |
| 연계통합 | Naming Service Support | `IConfiguration`, Service Discovery (마이크로서비스) | .NET 8+ |  |
| | Integration Service | 자체 구현 (API 클라이언트, 메시지 큐 연동 등) | - | 표준 인터페이스 처리 기능 개발 |
| | Web Service Interface | ASP.NET Core Web API (REST, gRPC) | .NET 8+ | 표준 인터페이스 준수 API 개발 |
| 배치처리 | Batch Framework | Hangfire, Quartz.NET, `BackgroundService` | 최신 |  |
| 공통기반 | AOP (유사 기능) | ASP.NET Core Middleware, PostSharp (선택적) | .NET 8+, 최신 |  |
| | Cache | `IMemoryCache`, `IDistributedCache` (Redis 등 연동) | .NET 8+ |  |
| | Compress/Decompress | `System.IO.Compression` | .NET 8+ |  |
| | Encryption/Decryption | `System.Security.Cryptography` | .NET 8+ | 암호화 기능 확장 |
| | Excel | EPPlus, ClosedXML | 최신 | Excel 기능 확장 |
| | File Handling | `System.IO` | .NET 8+ | 파일 접근 기능 확장 |
| | File Upload/Download | ASP.NET Core 기본 기능 | .NET 8+ |  |
| | FTP | FluentFTP, WinSCP .NET Assembly | 최신 |  |
| | ID Generation | `System.Guid`, DB Sequence | .NET 8+ | 시스템 고유 ID 생성 기능 개발 |
| | IoC Container | `Microsoft.Extensions.DependencyInjection` | .NET 8+ |  |
| | Logging | `Microsoft.Extensions.Logging` (Serilog, NLog 등 연동) | .NET 8+, 최신 |  |
| | Mail | MailKit | 최신 |  |
| | Marshalling/Unmarshalling | `System.Text.Json`, `System.Xml.Linq` | .NET 8+ |  |
| | Object Pooling | `Microsoft.Extensions.ObjectPool` | .NET 8+ |  |
| | Property | `IConfiguration` (`appsettings.json`) | .NET 8+ |  |
| | Resource | .NET Resource Files (.resx) | .NET 8+ |  |
| | Scheduling | Hangfire, Quartz.NET, `BackgroundService` | 최신 |  |
| | Server Security | ASP.NET Core Identity, JWT/OAuth2 | .NET 8+, 최신 | 인증, 권한 관리 기능 확장 |
| | String Util | `System.String`, `System.Text.RegularExpressions` | .NET 8+ | 문자열 처리 기능 확장 |
| | XML Manipulation | `System.Xml.Linq`, `System.Xml.XmlDocument` | .NET 8+ | XML 처리 기능 확장 |
