# 전자정부 표준프레임워크 .NET 8 LTS 기반 개발 시작하기(Getting Started)

## 개요

본 가이드는 전자정부 표준프레임워크 .NET 버전 기반의 단순한 응용프로그램(HelloWorld)를 직접 실습해 봄으로써 빠른 시간 내에 전자정부 프레임워크의 기본 기능을 파악하기 위하여 제공한다.
본 가이드의 사용자는 C# 및 ASP.NET Core에 대한 기본적인 지식이 있다는 것을 가정하였다.

아래의 3가지 단계에 따라 순서대로 따라하기 방식으로 진행된다.

- 개발환경 설치 : 실습을 위한 개발환경을 구축한다.
- 프로젝트 생성 : 제공한 샘플 프로젝트를 이용하여 HelloWorld 응용 어플리케이션을 생성하고 실행해 본다.
- 자세히 들여다 보기 : 생성/실행한 프로젝트의 내부 소스코드를 학습하여 전자정부 표준 프레임워크 .NET 기반의 응용 어플리케이션 구현의 원리를 이해한다.

전자정부 표준 프레임워크 .NET 기반의 Hello World 응용 프로그램 개발 및 실행을 위한 구현도구의 환경정보는 다음과 같다.

| 항목 | 설명 | .NET 버전 | 비고 |
|------|------|----------|------|
| 개발환경 | Eclipse Theia (Latest) | .NET 8 LTS | OmniSharp C# Support 익스텐션 필요 |
| 실행환경 | ASP.NET Core 8.0 | .NET 8 LTS | Windows, macOS, Linux 지원 |

- [.NET 8 LTS] Long Term Support 버전으로 2026년 11월까지 지원
- [ASP.NET Core 8.0] 고성능 웹 프레임워크, 크로스 플랫폼 지원
- [C# 12] 최신 언어 기능 지원

(참고) .NET 8은 LTS(Long Term Support) 버전으로 안정적인 엔터프라이즈 개발에 적합합니다.

## Step 1. 개발환경 설치

전자정부 표준프레임워크 .NET에서 제공하는 개발 도구 및 HelloWorld 응용 어플리케이션을 위한 패키지를 이용하여 실습에 필요한 개발환경을 설치한다.

### .NET 8 SDK 설치

먼저 [.NET 8 SDK](https://dotnet.microsoft.com/download/dotnet/8.0)를 다운로드하여 설치한다.

설치 확인:

```bash
dotnet --version
```

### Eclipse Theia 설치

Eclipse Theia 기반의 개발환경을 설치한다.

#### 설치 방법

1. [Eclipse Theia](https://theia-ide.org/) 공식 사이트에서 다운로드

2. 또는 Docker를 이용한 설치:

   ```bash
   docker run -it --init -p 3000:3000 -v "$(pwd):/home/project:cached" theiaide/theia:next
   ```

### C# 개발 환경 설정

Theia에서 C# 개발을 위한 OmniSharp 기반 익스텐션을 설치한다.

#### 익스텐션 설치순서

- Theia를 실행한다.
- View > Extensions 메뉴를 선택한다.
- "C#" 검색하여 **C# for Visual Studio Code (powered by OmniSharp)** 익스텐션을 설치한다.
- 필요시 추가 익스텐션 설치:
  - .NET Install Tool
  - NuGet Package Manager

### NuGet 패키지 관리 설정

.NET 프로젝트의 종속성 관리를 위한 NuGet 설정을 구성한다.

#### 패키지 소스 설정

```bash
dotnet nuget add source https://api.nuget.org/v3/index.json -n nuget.org
```

## Step 2. 프로젝트 생성 및 실행

### 프로젝트 생성

.NET CLI를 사용하여 새로운 콘솔 애플리케이션 프로젝트를 생성한다.

#### 프로젝트 생성순서

- 터미널을 열고 작업 디렉토리로 이동한다.
- 다음 명령어로 새 프로젝트를 생성한다:

```bash
dotnet new console -n EgovHelloWorld
cd EgovHelloWorld
```

- 필요한 NuGet 패키지를 추가한다:

```bash
dotnet add package Microsoft.Extensions.DependencyInjection
dotnet add package Microsoft.Extensions.Logging
dotnet add package Microsoft.Extensions.Configuration
```

### 프로젝트 빌드

.NET CLI를 이용하여 프로젝트를 빌드한다.

#### 프로젝트 빌드순서

- 프로젝트 디렉토리에서 다음 명령어를 실행한다:

```bash
dotnet build
```

- 빌드 성공 메시지를 확인한다.

### HelloWorld 실행

#### HelloWorld 실행순서

- 프로젝트 디렉토리에서 다음 명령어를 실행한다:

```bash
dotnet run
```

- 콘솔에서 실행 결과를 확인한다.

### HelloWorld 테스트 실행

.NET 프로젝트는 xUnit Test Framework 기반의 Test Case를 포함할 수 있다.

#### 테스트 프로젝트 생성

```bash
dotnet new xunit -n EgovHelloWorld.Tests
cd EgovHelloWorld.Tests
dotnet add reference ../EgovHelloWorld/EgovHelloWorld.csproj
```

#### 테스트 실행순서

```bash
dotnet test
```

## Step 3. 자세히 들여다 보기

### 서비스 인터페이스 클래스

다음은 HelloWorld 서비스의 인터페이스 클래스인 IHelloWorldService이다.

```csharp
namespace EgovHelloWorld.Services
{
    public interface IHelloWorldService
    {
        string SayHello();
    }
}
```

### 서비스 구현 클래스

다음은 HelloWorld 서비스의 구현 클래스인 HelloWorldService이다. 서비스 인터페이스 클래스인 IHelloWorldService를 구현하고 있다.

```csharp
using EgovHelloWorld.Services;

namespace EgovHelloWorld.Services
{
    public class HelloWorldService : IHelloWorldService
    {
        private readonly string _name;

        public HelloWorldService(string name = "egov framework")
        {
            _name = name;
        }

        public string SayHello()
        {
            return $"Hello {_name}!!!";
        }
    }
}
```

### 의존성 주입 설정 파일

다음은 HelloWorld 서비스의 의존성 주입 설정이다. .NET의 내장 DI 컨테이너를 사용한다.

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "HelloWorld": {
    "Name": "egov framework"
  }
}
```

### 프로그램 진입점 클래스

다음은 HelloWorld 서비스를 실행하기 위한 메인 프로그램 클래스이다.

```csharp
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Configuration;
using EgovHelloWorld.Services;

namespace EgovHelloWorld
{
    class Program
    {
        static void Main(string[] args)
        {
            // 설정 구성
            var configuration = new ConfigurationBuilder()
                .AddJsonFile("appsettings.json", optional: true)
                .Build();

            // 서비스 컨테이너 구성
            var services = new ServiceCollection();
            services.AddLogging(builder => builder.AddConsole());
            services.AddScoped<IHelloWorldService>(provider => 
                new HelloWorldService(configuration["HelloWorld:Name"] ?? "egov framework"));

            var serviceProvider = services.BuildServiceProvider();

            // 서비스 실행
            var logger = serviceProvider.GetRequiredService<ILogger<Program>>();
            var helloWorldService = serviceProvider.GetRequiredService<IHelloWorldService>();

            var message = helloWorldService.SayHello();
            logger.LogInformation(message);
            Console.WriteLine(message);
        }
    }
}
```

### 테스트 클래스

다음은 HelloWorld 서비스 클래스를 테스트하기 위한 단위 테스트 클래스이다.

```csharp
using Xunit;
using EgovHelloWorld.Services;

namespace EgovHelloWorld.Tests
{
    public class HelloWorldServiceTest
    {
        [Fact]
        public void SayHello_ReturnsExpectedMessage()
        {
            // Arrange
            var service = new HelloWorldService("egov framework");

            // Act
            var result = service.SayHello();

            // Assert
            Assert.Equal("Hello egov framework!!!", result);
        }

        [Theory]
        [InlineData("test", "Hello test!!!")]
        [InlineData("world", "Hello world!!!")]
        public void SayHello_WithDifferentNames_ReturnsCorrectMessage(string name, string expected)
        {
            // Arrange
            var service = new HelloWorldService(name);

            // Act
            var result = service.SayHello();

            // Assert
            Assert.Equal(expected, result);
        }
    }
}
```

### 프로젝트 파일

.NET 프로젝트의 설정을 담고 있는 프로젝트 파일이다.

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.Extensions.Configuration" Version="8.0.0" />
    <PackageReference Include="Microsoft.Extensions.Configuration.Json" Version="8.0.0" />
    <PackageReference Include="Microsoft.Extensions.DependencyInjection" Version="8.0.0" />
    <PackageReference Include="Microsoft.Extensions.Logging" Version="8.0.0" />
    <PackageReference Include="Microsoft.Extensions.Logging.Console" Version="8.0.0" />
  </ItemGroup>

  <ItemGroup>
    <None Update="appsettings.json">
      <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
    </None>
  </ItemGroup>

</Project>
```
