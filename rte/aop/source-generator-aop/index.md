# 소스 생성기 AOP 구현

## 개요

.NET 소스 생성기(Source Generator)를 활용한 AOP 구현 방법을 다룬다. 소스 생성기는 컴파일 시점에 코드를 자동 생성하여 런타임 오버헤드 없이 AOP를 구현할 수 있는 강력한 기능이다.

## 소스 생성기 AOP 기본 개념

### 컴파일 시점 코드 생성

소스 생성기는 컴파일 과정에서 추가 소스 코드를 생성하여 어셈블리에 포함시킨다.

```
소스 코드 → 구문 분석 → 소스 생성기 실행 → 추가 코드 생성 → 컴파일 → 어셈블리
```

### 장점

1. **제로 런타임 오버헤드**: 컴파일 시점에 코드가 생성되므로 런타임 성능 영향 없음
2. **AOT 호환성**: Native AOT 컴파일과 완벽 호환
3. **강타입 지원**: 컴파일 타임에 타입 안전성 보장
4. **디버깅 지원**: 생성된 코드를 직접 디버깅 가능

## 기본 소스 생성기 구현

### 1. 소스 생성기 프로젝트 설정

```xml
<!-- SourceGenerator.csproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>netstandard2.0</TargetFramework>
    <LangVersion>latest</LangVersion>
    <IncludeBuildOutput>false</IncludeBuildOutput>
    <GeneratePackageOnBuild>true</GeneratePackageOnBuild>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.CodeAnalysis.Analyzers" Version="3.3.4" PrivateAssets="all" />
    <PackageReference Include="Microsoft.CodeAnalysis.CSharp" Version="4.5.0" PrivateAssets="all" />
  </ItemGroup>
</Project>
```

### 2. 기본 어트리뷰트 정의

```csharp
// Attributes.cs
using System;

[AttributeUsage(AttributeTargets.Method)]
public class LogExecutionTimeAttribute : Attribute
{
    public LogLevel LogLevel { get; set; } = LogLevel.Information;
    public bool IncludeParameters { get; set; } = false;
    public bool IncludeReturnValue { get; set; } = false;
}

[AttributeUsage(AttributeTargets.Method)]
public class CacheResultAttribute : Attribute
{
    public int ExpirationMinutes { get; set; } = 5;
    public string KeyPrefix { get; set; } = "";
}

[AttributeUsage(AttributeTargets.Method)]
public class ValidateInputAttribute : Attribute
{
    public bool ValidateNotNull { get; set; } = true;
    public bool ValidateRange { get; set; } = false;
}

[AttributeUsage(AttributeTargets.Method)]
public class RetryAttribute : Attribute
{
    public int MaxAttempts { get; set; } = 3;
    public int DelayMilliseconds { get; set; } = 1000;
    public Type[] RetriableExceptions { get; set; } = new Type[0];
}

public enum LogLevel
{
    Trace = 0,
    Debug = 1,
    Information = 2,
    Warning = 3,
    Error = 4,
    Critical = 5
}
```

### 3. 메인 소스 생성기

```csharp
// AopSourceGenerator.cs
using Microsoft.CodeAnalysis;
using Microsoft.CodeAnalysis.CSharp;
using Microsoft.CodeAnalysis.CSharp.Syntax;
using System.Collections.Generic;
using System.Linq;
using System.Text;

[Generator]
public class AopSourceGenerator : IIncrementalGenerator
{
    public void Initialize(IncrementalGeneratorInitializationContext context)
    {
        // 부분 클래스에서 AOP 어트리뷰트가 있는 메서드 찾기
        var classDeclarations = context.SyntaxProvider
            .CreateSyntaxProvider(
                predicate: static (s, _) => IsSyntaxTargetForGeneration(s),
                transform: static (ctx, _) => GetSemanticTargetForGeneration(ctx))
            .Where(static m => m is not null);

        // 컴파일과 클래스 선언 결합
        var compilationAndClasses = context.CompilationProvider.Combine(classDeclarations.Collect());

        // 소스 생성
        context.RegisterSourceOutput(compilationAndClasses,
            static (spc, source) => Execute(source.Left, source.Right, spc));
    }

    private static bool IsSyntaxTargetForGeneration(SyntaxNode node)
    {
        return node is ClassDeclarationSyntax classDeclaration &&
               classDeclaration.Modifiers.Any(SyntaxKind.PartialKeyword) &&
               classDeclaration.Members.OfType<MethodDeclarationSyntax>()
                   .Any(method => method.AttributeLists.Count > 0);
    }

    private static ClassDeclarationSyntax? GetSemanticTargetForGeneration(GeneratorSyntaxContext context)
    {
        var classDeclaration = (ClassDeclarationSyntax)context.Node;
        
        // AOP 어트리뷰트가 있는 메서드가 있는지 확인
        var hasAopMethods = classDeclaration.Members.OfType<MethodDeclarationSyntax>()
            .Any(method => HasAopAttributes(method, context.SemanticModel));

        return hasAopMethods ? classDeclaration : null;
    }

    private static bool HasAopAttributes(MethodDeclarationSyntax method, SemanticModel semanticModel)
    {
        var aopAttributeNames = new[] 
        { 
            "LogExecutionTimeAttribute", 
            "CacheResultAttribute", 
            "ValidateInputAttribute",
            "RetryAttribute"
        };

        return method.AttributeLists
            .SelectMany(list => list.Attributes)
            .Any(attr =>
            {
                var symbolInfo = semanticModel.GetSymbolInfo(attr);
                var typeName = symbolInfo.Symbol?.ContainingType?.Name;
                return aopAttributeNames.Contains(typeName);
            });
    }

    private static void Execute(Compilation compilation, 
        ImmutableArray<ClassDeclarationSyntax> classes, 
        SourceProductionContext context)
    {
        if (classes.IsDefaultOrEmpty)
            return;

        foreach (var classDeclaration in classes.Distinct())
        {
            var semanticModel = compilation.GetSemanticModel(classDeclaration.SyntaxTree);
            var classSymbol = semanticModel.GetDeclaredSymbol(classDeclaration);
            
            if (classSymbol == null)
                continue;

            var sourceCode = GenerateAopImplementation(classDeclaration, semanticModel);
            
            if (!string.IsNullOrEmpty(sourceCode))
            {
                context.AddSource($"{classSymbol.Name}.Aop.g.cs", sourceCode);
            }
        }
    }

    private static string GenerateAopImplementation(ClassDeclarationSyntax classDeclaration, 
        SemanticModel semanticModel)
    {
        var namespaceName = GetNamespace(classDeclaration);
        var className = classDeclaration.Identifier.ValueText;
        
        var stringBuilder = new StringBuilder();
        
        // 네임스페이스와 using 문
        stringBuilder.AppendLine("using System;");
        stringBuilder.AppendLine("using System.Threading.Tasks;");
        stringBuilder.AppendLine("using System.Diagnostics;");
        stringBuilder.AppendLine("using Microsoft.Extensions.Logging;");
        stringBuilder.AppendLine("using Microsoft.Extensions.Caching.Memory;");
        stringBuilder.AppendLine();

        if (!string.IsNullOrEmpty(namespaceName))
        {
            stringBuilder.AppendLine($"namespace {namespaceName};");
            stringBuilder.AppendLine();
        }

        // 부분 클래스 시작
        stringBuilder.AppendLine($"public partial class {className}");
        stringBuilder.AppendLine("{");

        // AOP가 적용된 메서드들 처리
        var methods = classDeclaration.Members.OfType<MethodDeclarationSyntax>()
            .Where(method => HasAopAttributes(method, semanticModel));

        foreach (var method in methods)
        {
            GenerateAopMethod(stringBuilder, method, semanticModel);
        }

        stringBuilder.AppendLine("}");

        return stringBuilder.ToString();
    }

    private static void GenerateAopMethod(StringBuilder stringBuilder, 
        MethodDeclarationSyntax method, 
        SemanticModel semanticModel)
    {
        var methodName = method.Identifier.ValueText;
        var coreMethodName = $"{methodName}Core";
        var returnType = method.ReturnType.ToString();
        var parameters = string.Join(", ", method.ParameterList.Parameters.Select(p => $"{p.Type} {p.Identifier}"));
        var arguments = string.Join(", ", method.ParameterList.Parameters.Select(p => p.Identifier.ValueText));

        // 어트리뷰트 분석
        var attributes = AnalyzeMethodAttributes(method, semanticModel);

        // AOP 래퍼 메서드 생성
        stringBuilder.AppendLine($"    public {returnType} {methodName}({parameters})");
        stringBuilder.AppendLine("    {");

        // 로깅 시작
        if (attributes.HasLogging)
        {
            stringBuilder.AppendLine($"        var stopwatch = Stopwatch.StartNew();");
            stringBuilder.AppendLine($"        _logger?.LogInformation(\"Executing method {methodName}\");");
        }

        // 입력 검증
        if (attributes.HasValidation)
        {
            GenerateValidationCode(stringBuilder, method);
        }

        // 캐싱 확인
        if (attributes.HasCaching)
        {
            GenerateCacheCheckCode(stringBuilder, method, attributes.CacheKey);
        }

        // 재시도 로직
        if (attributes.HasRetry)
        {
            GenerateRetryCode(stringBuilder, coreMethodName, arguments, attributes.MaxAttempts);
        }
        else
        {
            // 일반 메서드 호출
            if (returnType == "void")
            {
                stringBuilder.AppendLine($"        {coreMethodName}({arguments});");
            }
            else if (returnType.StartsWith("Task"))
            {
                stringBuilder.AppendLine($"        var result = await {coreMethodName}({arguments});");
            }
            else
            {
                stringBuilder.AppendLine($"        var result = {coreMethodName}({arguments});");
            }
        }

        // 캐싱 저장
        if (attributes.HasCaching && returnType != "void")
        {
            GenerateCacheStoreCode(stringBuilder, attributes.CacheKey, attributes.CacheExpiration);
        }

        // 로깅 종료
        if (attributes.HasLogging)
        {
            stringBuilder.AppendLine("        stopwatch.Stop();");
            stringBuilder.AppendLine($"        _logger?.LogInformation(\"Method {methodName} executed in {{ElapsedMs}}ms\", stopwatch.ElapsedMilliseconds);");
        }

        // 반환
        if (returnType != "void")
        {
            stringBuilder.AppendLine("        return result;");
        }

        stringBuilder.AppendLine("    }");
        stringBuilder.AppendLine();

        // 원본 메서드를 Core 메서드로 변경
        GenerateCoreMethod(stringBuilder, method, coreMethodName);
    }

    private static void GenerateValidationCode(StringBuilder stringBuilder, MethodDeclarationSyntax method)
    {
        foreach (var parameter in method.ParameterList.Parameters)
        {
            var paramName = parameter.Identifier.ValueText;
            var paramType = parameter.Type?.ToString();

            if (paramType != null && !paramType.Contains("?") && !IsValueType(paramType))
            {
                stringBuilder.AppendLine($"        if ({paramName} == null)");
                stringBuilder.AppendLine($"            throw new ArgumentNullException(nameof({paramName}));");
            }
        }
    }

    private static void GenerateCacheCheckCode(StringBuilder stringBuilder, MethodDeclarationSyntax method, string cacheKey)
    {
        var returnType = method.ReturnType.ToString();
        
        stringBuilder.AppendLine($"        var cacheKey = \"{cacheKey}\";");
        stringBuilder.AppendLine($"        if (_cache?.TryGetValue(cacheKey, out {returnType} cachedResult) == true)");
        stringBuilder.AppendLine("        {");
        stringBuilder.AppendLine("            _logger?.LogDebug(\"Cache hit for key: {CacheKey}\", cacheKey);");
        stringBuilder.AppendLine("            return cachedResult;");
        stringBuilder.AppendLine("        }");
    }

    private static void GenerateCacheStoreCode(StringBuilder stringBuilder, string cacheKey, int expirationMinutes)
    {
        stringBuilder.AppendLine($"        _cache?.Set(cacheKey, result, TimeSpan.FromMinutes({expirationMinutes}));");
        stringBuilder.AppendLine("        _logger?.LogDebug(\"Result cached for key: {CacheKey}\", cacheKey);");
    }

    private static void GenerateRetryCode(StringBuilder stringBuilder, string coreMethodName, string arguments, int maxAttempts)
    {
        stringBuilder.AppendLine($"        for (int attempt = 1; attempt <= {maxAttempts}; attempt++)");
        stringBuilder.AppendLine("        {");
        stringBuilder.AppendLine("            try");
        stringBuilder.AppendLine("            {");
        stringBuilder.AppendLine($"                var result = {coreMethodName}({arguments});");
        stringBuilder.AppendLine("                return result;");
        stringBuilder.AppendLine("            }");
        stringBuilder.AppendLine($"            catch (Exception ex) when (attempt < {maxAttempts})");
        stringBuilder.AppendLine("            {");
        stringBuilder.AppendLine("                _logger?.LogWarning(ex, \"Attempt {Attempt} failed, retrying...\", attempt);");
        stringBuilder.AppendLine("                await Task.Delay(TimeSpan.FromSeconds(Math.Pow(2, attempt)));");
        stringBuilder.AppendLine("            }");
        stringBuilder.AppendLine("        }");
    }

    private static void GenerateCoreMethod(StringBuilder stringBuilder, MethodDeclarationSyntax method, string coreMethodName)
    {
        var modifiers = method.Modifiers.ToString().Replace("public", "private");
        var returnType = method.ReturnType.ToString();
        var parameters = string.Join(", ", method.ParameterList.Parameters.Select(p => $"{p.Type} {p.Identifier}"));

        stringBuilder.AppendLine($"    {modifiers} {returnType} {coreMethodName}({parameters})");
        stringBuilder.AppendLine("    {");
        stringBuilder.AppendLine("        // TODO: 원본 메서드 구현을 여기로 이동");
        stringBuilder.AppendLine("        throw new NotImplementedException(\"Core method implementation required\");");
        stringBuilder.AppendLine("    }");
        stringBuilder.AppendLine();
    }

    private static string GetNamespace(ClassDeclarationSyntax classDeclaration)
    {
        var namespaceDeclaration = classDeclaration.Ancestors().OfType<NamespaceDeclarationSyntax>().FirstOrDefault();
        return namespaceDeclaration?.Name?.ToString() ?? "";
    }

    private static bool IsValueType(string typeName)
    {
        var valueTypes = new[] { "int", "long", "float", "double", "decimal", "bool", "char", "byte", "short" };
        return valueTypes.Contains(typeName);
    }

    private static MethodAttributes AnalyzeMethodAttributes(MethodDeclarationSyntax method, SemanticModel semanticModel)
    {
        var attributes = new MethodAttributes();

        foreach (var attributeList in method.AttributeLists)
        {
            foreach (var attribute in attributeList.Attributes)
            {
                var symbolInfo = semanticModel.GetSymbolInfo(attribute);
                var attributeName = symbolInfo.Symbol?.ContainingType?.Name;

                switch (attributeName)
                {
                    case "LogExecutionTimeAttribute":
                        attributes.HasLogging = true;
                        break;
                    case "CacheResultAttribute":
                        attributes.HasCaching = true;
                        attributes.CacheKey = $"{method.Identifier.ValueText}_cache";
                        attributes.CacheExpiration = GetAttributeParameter(attribute, "ExpirationMinutes", 5);
                        break;
                    case "ValidateInputAttribute":
                        attributes.HasValidation = true;
                        break;
                    case "RetryAttribute":
                        attributes.HasRetry = true;
                        attributes.MaxAttempts = GetAttributeParameter(attribute, "MaxAttempts", 3);
                        break;
                }
            }
        }

        return attributes;
    }

    private static int GetAttributeParameter(AttributeSyntax attribute, string parameterName, int defaultValue)
    {
        // 간단한 구현 - 실제로는 더 정교한 파싱이 필요
        return defaultValue;
    }

    private class MethodAttributes
    {
        public bool HasLogging { get; set; }
        public bool HasCaching { get; set; }
        public bool HasValidation { get; set; }
        public bool HasRetry { get; set; }
        public string CacheKey { get; set; } = "";
        public int CacheExpiration { get; set; } = 5;
        public int MaxAttempts { get; set; } = 3;
    }
}
```

## 사용 예제

### 1. 비즈니스 서비스 클래스

```csharp
// UserService.cs
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Caching.Memory;

public partial class UserService : IUserService
{
    private readonly IUserRepository _repository;
    private readonly ILogger<UserService> _logger;
    private readonly IMemoryCache _cache;

    public UserService(IUserRepository repository, 
        ILogger<UserService> logger, 
        IMemoryCache cache)
    {
        _repository = repository;
        _logger = logger;
        _cache = cache;
    }

    [LogExecutionTime]
    [CacheResult(ExpirationMinutes = 10)]
    [ValidateInput]
    public async Task<User> GetUserAsync(int id)
    {
        // 이 부분은 GetUserAsyncCore로 이동됨
        return await _repository.GetByIdAsync(id);
    }

    [LogExecutionTime]
    [Retry(MaxAttempts = 3)]
    public async Task<bool> UpdateUserAsync(User user)
    {
        // 이 부분은 UpdateUserAsyncCore로 이동됨
        return await _repository.UpdateAsync(user);
    }

    [LogExecutionTime]
    [ValidateInput]
    public async Task<List<User>> SearchUsersAsync(string searchTerm)
    {
        // 이 부분은 SearchUsersAsyncCore로 이동됨
        return await _repository.SearchAsync(searchTerm);
    }
}
```

### 2. 생성된 코드 예제

```csharp
// UserService.Aop.g.cs (자동 생성됨)
using System;
using System.Threading.Tasks;
using System.Diagnostics;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Caching.Memory;

public partial class UserService
{
    public async Task<User> GetUserAsync(int id)
    {
        var stopwatch = Stopwatch.StartNew();
        _logger?.LogInformation("Executing method GetUserAsync");
        
        // 입력 검증
        if (id <= 0)
            throw new ArgumentException("Invalid user ID", nameof(id));
        
        // 캐시 확인
        var cacheKey = "GetUserAsync_cache";
        if (_cache?.TryGetValue(cacheKey, out User cachedResult) == true)
        {
            _logger?.LogDebug("Cache hit for key: {CacheKey}", cacheKey);
            return cachedResult;
        }
        
        // 핵심 로직 실행
        var result = await GetUserAsyncCore(id);
        
        // 캐싱 저장
        _cache?.Set(cacheKey, result, TimeSpan.FromMinutes(10));
        _logger?.LogDebug("Result cached for key: {CacheKey}", cacheKey);
        
        // 로깅 종료
        stopwatch.Stop();
        _logger?.LogInformation("Method GetUserAsync executed in {ElapsedMs}ms", stopwatch.ElapsedMilliseconds);
        
        return result;
    }

    private async Task<User> GetUserAsyncCore(int id)
    {
        return await _repository.GetByIdAsync(id);
    }

    public async Task<bool> UpdateUserAsync(User user)
    {
        var stopwatch = Stopwatch.StartNew();
        _logger?.LogInformation("Executing method UpdateUserAsync");
        
        // 입력 검증
        if (user == null)
            throw new ArgumentNullException(nameof(user));
        
        // 재시도 로직
        for (int attempt = 1; attempt <= 3; attempt++)
        {
            try
            {
                var result = await UpdateUserAsyncCore(user);
                
                stopwatch.Stop();
                _logger?.LogInformation("Method UpdateUserAsync executed in {ElapsedMs}ms", stopwatch.ElapsedMilliseconds);
                
                return result;
            }
            catch (Exception ex) when (attempt < 3)
            {
                _logger?.LogWarning(ex, "Attempt {Attempt} failed, retrying...", attempt);
                await Task.Delay(TimeSpan.FromSeconds(Math.Pow(2, attempt)));
            }
        }
        
        throw new InvalidOperationException("All retry attempts failed");
    }

    private async Task<bool> UpdateUserAsyncCore(User user)
    {
        return await _repository.UpdateAsync(user);
    }
}
```

## 고급 소스 생성기 기능

### 1. 인크리멘털 생성기

```csharp
[Generator]
public class IncrementalAopGenerator : IIncrementalGenerator
{
    public void Initialize(IncrementalGeneratorInitializationContext context)
    {
        // 컴파일 전체를 다시 분석하지 않고 변경된 부분만 처리
        var methodsWithAttributes = context.SyntaxProvider
            .ForAttributeWithMetadataName(
                "LogExecutionTimeAttribute",
                predicate: static (node, _) => node is MethodDeclarationSyntax,
                transform: static (context, _) => GetMethodInfo(context))
            .Where(static m => m is not null);

        context.RegisterSourceOutput(methodsWithAttributes, static (spc, method) => 
        {
            if (method != null)
            {
                var source = GenerateAopCode(method);
                spc.AddSource($"{method.ClassName}.{method.MethodName}.g.cs", source);
            }
        });
    }
}
```

### 2. 진단 정보 제공

```csharp
public class AopAnalyzer : DiagnosticAnalyzer
{
    public static readonly DiagnosticDescriptor MissingPartialModifier = new DiagnosticDescriptor(
        "AOP001",
        "Class with AOP attributes must be partial",
        "Class '{0}' has AOP attributes but is not marked as partial",
        "Usage",
        DiagnosticSeverity.Error,
        isEnabledByDefault: true);

    public override ImmutableArray<DiagnosticDescriptor> SupportedDiagnostics =>
        ImmutableArray.Create(MissingPartialModifier);

    public override void Initialize(AnalysisContext context)
    {
        context.ConfigureGeneratedCodeAnalysis(GeneratedCodeAnalysisFlags.None);
        context.EnableConcurrentExecution();
        context.RegisterSyntaxNodeAction(AnalyzeClassDeclaration, SyntaxKind.ClassDeclaration);
    }

    private static void AnalyzeClassDeclaration(SyntaxNodeAnalysisContext context)
    {
        var classDeclaration = (ClassDeclarationSyntax)context.Node;
        
        if (HasAopAttributes(classDeclaration) && !IsPartialClass(classDeclaration))
        {
            var diagnostic = Diagnostic.Create(
                MissingPartialModifier,
                classDeclaration.Identifier.GetLocation(),
                classDeclaration.Identifier.ValueText);
            
            context.ReportDiagnostic(diagnostic);
        }
    }
}
```

### 3. 설정 파일 지원

```json
// aop.config.json
{
    "logging": {
        "defaultLogLevel": "Information",
        "includeParameters": false,
        "includeReturnValue": false
    },
    "caching": {
        "defaultExpirationMinutes": 5,
        "keyPrefix": "app_cache"
    },
    "retry": {
        "defaultMaxAttempts": 3,
        "defaultDelayMilliseconds": 1000,
        "retriableExceptions": [
            "System.TimeoutException",
            "System.Net.Http.HttpRequestException"
        ]
    }
}
```

## 성능 최적화

### 1. 컴파일 시간 최적화

```csharp
// 캐싱을 통한 중복 생성 방지
private static readonly ConcurrentDictionary<string, string> GeneratedCodeCache = new();

private static string GenerateAopImplementation(ClassDeclarationSyntax classDeclaration)
{
    var key = GetCacheKey(classDeclaration);
    return GeneratedCodeCache.GetOrAdd(key, _ => GenerateCodeInternal(classDeclaration));
}
```

### 2. 메모리 최적화

```csharp
// StringBuilder 풀링
private static readonly ObjectPool<StringBuilder> StringBuilderPool = 
    new ObjectPool<StringBuilder>(() => new StringBuilder(), sb => sb.Clear());

private static string GenerateCode()
{
    var sb = StringBuilderPool.Rent();
    try
    {
        // 코드 생성
        return sb.ToString();
    }
    finally
    {
        StringBuilderPool.Return(sb);
    }
}
```

## 테스트 및 디버깅

### 1. 소스 생성기 단위 테스트

```csharp
[Test]
public void Generator_ShouldGenerateLoggingCode()
{
    // Arrange
    var source = @"
public partial class TestService
{
    [LogExecutionTime]
    public void TestMethod() { }
}";

    // Act
    var result = TestHelper.RunGenerator(source);

    // Assert
    Assert.Contains("Stopwatch.StartNew()", result.GeneratedSources[0]);
    Assert.Contains("_logger?.LogInformation", result.GeneratedSources[0]);
}
```

### 2. 생성된 코드 확인

```xml
<!-- 프로젝트 파일에 추가 -->
<PropertyGroup>
  <EmitCompilerGeneratedFiles>true</EmitCompilerGeneratedFiles>
  <CompilerGeneratedFilesOutputPath>Generated</CompilerGeneratedFilesOutputPath>
</PropertyGroup>
```

## 배포 및 패키징

### 1. NuGet 패키지 생성

```xml
<!-- AopSourceGenerator.csproj -->
<PropertyGroup>
  <PackageId>MyCompany.Aop.SourceGenerator</PackageId>
  <PackageVersion>1.0.0</PackageVersion>
  <Authors>MyCompany</Authors>
  <Description>AOP Source Generator for .NET 8</Description>
  <DevelopmentDependency>true</DevelopmentDependency>
  <IncludeBuildOutput>false</IncludeBuildOutput>
</PropertyGroup>

<ItemGroup>
  <None Include="tools\install.ps1" Pack="true" PackagePath="tools\install.ps1" />
  <None Include="tools\uninstall.ps1" Pack="true" PackagePath="tools\uninstall.ps1" />
</ItemGroup>
```

### 2. 프로젝트에서 사용

```xml
<!-- 클라이언트 프로젝트 -->
<ItemGroup>
  <PackageReference Include="MyCompany.Aop.SourceGenerator" Version="1.0.0">
    <PrivateAssets>all</PrivateAssets>
    <IncludeAssets>runtime; build; native; contentfiles; analyzers</IncludeAssets>
  </PackageReference>
</ItemGroup>
```

## 모범 사례

### 1. 명확한 어트리뷰트 설계

```csharp
// 좋은 예: 명확하고 구체적
[LogMethodExecution(LogLevel.Information, IncludeParameters = true)]
[CacheResult(Duration = "00:05:00", KeyTemplate = "user_{id}")]

// 나쁜 예: 모호함
[Log]
[Cache]
```

### 2. 에러 처리

```csharp
private static void Execute(GeneratorExecutionContext context)
{
    try
    {
        // 생성 로직
    }
    catch (Exception ex)
    {
        var diagnostic = Diagnostic.Create(
            new DiagnosticDescriptor(
                "SG001",
                "Source generation failed",
                "Failed to generate AOP code: {0}",
                "SourceGenerator",
                DiagnosticSeverity.Error,
                true),
            Location.None,
            ex.Message);
        
        context.ReportDiagnostic(diagnostic);
    }
}
```

### 3. 호환성 고려

```csharp
#if NET8_0_OR_GREATER
    // .NET 8 특화 기능
#else
    // 이전 버전 호환 코드
#endif
```

## 참고자료

- [Source Generators in .NET](https://docs.microsoft.com/en-us/dotnet/csharp/roslyn-sdk/source-generators-overview)
- [Incremental Generators](https://github.com/dotnet/roslyn/blob/main/docs/features/incremental-generators.md)
- [Source Generator Cookbook](https://github.com/dotnet/roslyn/blob/main/docs/features/source-generators.cookbook.md)
- [Source Generator Samples](https://github.com/dotnet/roslyn-sdk/tree/main/samples/CSharp/SourceGenerators)
