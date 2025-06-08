# Resource 서비스

## 개요

.NET 8 LTS 기반 실행환경에서 리소스를 활용한 메시지 제공 서비스를 살펴본다. 메시지 제공 서비스는 미리 정의된 파일에서 메시지를 읽어들인 후, 오류 발생 시 또는 안내 메시지를 제공하기 위해 키값에 해당하는 메시지를 가져오는 기능을 제공한다.

.NET 8에서는 다음과 같은 방식으로 리소스 메시지 서비스를 구현할 수 있다:

1. **기본 리소스 로컬라이제이션**: `IStringLocalizer`와 `.resx` 파일 활용
2. **미들웨어 기반 다국어 지원**: HTTP 파이프라인에서 문화권 설정
3. **소스 생성기 기반 타입 안전 메시지**: 컴파일 시점 메시지 생성

## 설명

### 기본 메시지 서비스

.NET 8의 내장 로컬라이제이션 기능을 활용한 기본 메시지 서비스 구현 방법이다.

#### 프로젝트 설정

```xml
<!-- 프로젝트 파일에 필요한 패키지 추가 -->
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.Extensions.Localization" Version="8.0.0" />
    <PackageReference Include="Microsoft.Extensions.Localization.Abstractions" Version="8.0.0" />
  </ItemGroup>
</Project>
```

#### 서비스 등록 및 구성

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// 로컬라이제이션 서비스 등록
builder.Services.AddLocalization(options => options.ResourcesPath = "Resources");

// 지원할 문화권 설정
builder.Services.Configure<RequestLocalizationOptions>(options =>
{
    var supportedCultures = new[] { "ko", "en", "ja" };
    options.SetDefaultCulture(supportedCultures[0])
        .AddSupportedCultures(supportedCultures)
        .AddSupportedUICultures(supportedCultures);
});

var app = builder.Build();

// 요청 로컬라이제이션 미들웨어 추가
app.UseRequestLocalization();

app.MapControllers();
app.Run();
```

#### 리소스 파일 구성

```
Resources/
├── Messages.resx              // 기본 언어 (한국어)
├── Messages.en.resx           // 영어
└── Messages.ja.resx           // 일본어
```

**Messages.resx (기본 - 한국어)**

```xml
<?xml version="1.0" encoding="utf-8"?>
<root>
  <data name="resource.basic.msg1" xml:space="preserve">
    <value>메시지1</value>
  </data>
  <data name="resource.basic.msg3" xml:space="preserve">
    <value>메시지 {0} {1}</value>
  </data>
  <data name="error.validation.required" xml:space="preserve">
    <value>필수 입력 항목입니다.</value>
  </data>
</root>
```

**Messages.en.resx (영어)**

```xml
<?xml version="1.0" encoding="utf-8"?>
<root>
  <data name="resource.basic.msg1" xml:space="preserve">
    <value>message1</value>
  </data>
  <data name="resource.basic.msg3" xml:space="preserve">
    <value>message {0} {1}</value>
  </data>
  <data name="error.validation.required" xml:space="preserve">
    <value>This field is required.</value>
  </data>
</root>
```

#### 기본 사용 예제

```csharp
// MessageService.cs
using Microsoft.Extensions.Localization;

public class MessageService
{
    private readonly IStringLocalizer<Messages> _localizer;

    public MessageService(IStringLocalizer<Messages> localizer)
    {
        _localizer = localizer;
    }

    public string GetMessage(string key)
    {
        return _localizer[key];
    }

    public string GetMessage(string key, params object[] arguments)
    {
        return _localizer[key, arguments];
    }
}

// Controllers에서 사용
[ApiController]
[Route("api/[controller]")]
public class MessagesController : ControllerBase
{
    private readonly IStringLocalizer<Messages> _localizer;

    public MessagesController(IStringLocalizer<Messages> localizer)
    {
        _localizer = localizer;
    }

    [HttpGet("basic")]
    public IActionResult GetBasicMessage()
    {
        var message = _localizer["resource.basic.msg1"];
        return Ok(new { message = message.ToString() });
    }

    [HttpGet("parameter")]
    public IActionResult GetParameterMessage()
    {
        var message = _localizer["resource.basic.msg3", "1", 2];
        return Ok(new { message = message.ToString() });
    }
}
```

### 미들웨어 기반 다국어 지원

HTTP 요청 파이프라인에서 문화권을 자동으로 설정하고 메시지를 제공하는 미들웨어 구현이다.

#### 문화권 설정 미들웨어

```csharp
// CultureMiddleware.cs
using System.Globalization;

public class CultureMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<CultureMiddleware> _logger;

    public CultureMiddleware(RequestDelegate next, ILogger<CultureMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        // 헤더에서 언어 설정 추출
        var culture = GetCultureFromRequest(context.Request);
        
        if (!string.IsNullOrEmpty(culture))
        {
            try
            {
                var cultureInfo = new CultureInfo(culture);
                CultureInfo.CurrentCulture = cultureInfo;
                CultureInfo.CurrentUICulture = cultureInfo;
                
                _logger.LogDebug("Culture set to: {Culture}", culture);
            }
            catch (CultureNotFoundException ex)
            {
                _logger.LogWarning(ex, "Invalid culture requested: {Culture}", culture);
            }
        }

        await _next(context);
    }

    private static string GetCultureFromRequest(HttpRequest request)
    {
        // 1. 쿼리 파라미터에서 확인
        if (request.Query.ContainsKey("culture"))
        {
            return request.Query["culture"].ToString();
        }

        // 2. Accept-Language 헤더에서 확인
        var acceptLanguage = request.Headers["Accept-Language"].ToString();
        if (!string.IsNullOrEmpty(acceptLanguage))
        {
            var firstLanguage = acceptLanguage.Split(',').FirstOrDefault()?.Split(';').FirstOrDefault();
            return firstLanguage?.Trim();
        }

        // 3. 기본값 반환
        return "ko";
    }
}

// 미들웨어 등록 확장 메서드
public static class CultureMiddlewareExtensions
{
    public static IApplicationBuilder UseCultureMiddleware(this IApplicationBuilder builder)
    {
        return builder.UseMiddleware<CultureMiddleware>();
    }
}
```

#### 메시지 제공 미들웨어

```csharp
// MessageMiddleware.cs
public class MessageMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IStringLocalizer<Messages> _localizer;
    private readonly ILogger<MessageMiddleware> _logger;

    public MessageMiddleware(
        RequestDelegate next, 
        IStringLocalizer<Messages> localizer,
        ILogger<MessageMiddleware> logger)
    {
        _next = next;
        _localizer = localizer;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        // HttpContext에 메시지 서비스 추가
        context.Items["MessageService"] = new MessageService(_localizer);

        try
        {
            await _next(context);
        }
        catch (Exception ex)
        {
            await HandleExceptionAsync(context, ex);
        }
    }

    private async Task HandleExceptionAsync(HttpContext context, Exception exception)
    {
        var response = context.Response;
        response.ContentType = "application/json";

        var errorMessage = exception switch
        {
            ArgumentNullException => _localizer["error.validation.required"],
            ArgumentException => _localizer["error.validation.invalid"],
            FileNotFoundException => _localizer["error.file.notfound"],
            UnauthorizedAccessException => _localizer["error.auth.unauthorized"],
            _ => _localizer["error.general.unexpected"]
        };

        response.StatusCode = GetStatusCode(exception);

        var result = JsonSerializer.Serialize(new
        {
            error = errorMessage.ToString(),
            traceId = context.TraceIdentifier
        });

        await response.WriteAsync(result);
    }

    private static int GetStatusCode(Exception exception) => exception switch
    {
        ArgumentNullException or ArgumentException => 400,
        UnauthorizedAccessException => 401,
        FileNotFoundException => 404,
        _ => 500
    };
}
```

#### 미들웨어 등록

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// 서비스 등록
builder.Services.AddLocalization(options => options.ResourcesPath = "Resources");
builder.Services.AddScoped<MessageService>();

var app = builder.Build();

// 미들웨어 파이프라인 구성
app.UseCultureMiddleware();        // 문화권 설정
app.UseMiddleware<MessageMiddleware>(); // 메시지 서비스 제공

app.UseRequestLocalization();
app.MapControllers();

app.Run();
```

### 소스 생성기 기반 타입 안전 메시지

컴파일 시점에 타입 안전한 메시지 클래스를 생성하는 소스 생성기 구현이다.

#### 메시지 정의 어트리뷰트

```csharp
// MessageAttributes.cs
[AttributeUsage(AttributeTargets.Class)]
public class GenerateMessagesAttribute : Attribute
{
    public string ResourcePath { get; set; } = "Resources";
    public string ClassName { get; set; } = "GeneratedMessages";
}

[AttributeUsage(AttributeTargets.Property)]
public class MessageKeyAttribute : Attribute
{
    public string Key { get; }
    public string DefaultValue { get; set; } = "";

    public MessageKeyAttribute(string key)
    {
        Key = key;
    }
}
```

#### 메시지 정의 클래스

```csharp
// MessageDefinitions.cs
[GenerateMessages(ResourcePath = "Resources", ClassName = "AppMessages")]
public partial class MessageDefinitions
{
    [MessageKey("resource.basic.msg1", DefaultValue = "기본 메시지")]
    public static string BasicMessage => throw new NotImplementedException();

    [MessageKey("resource.basic.msg3", DefaultValue = "메시지 {0} {1}")]
    public static string ParameterMessage => throw new NotImplementedException();

    [MessageKey("error.validation.required", DefaultValue = "필수 입력 항목입니다.")]
    public static string ValidationRequired => throw new NotImplementedException();

    [MessageKey("error.auth.unauthorized", DefaultValue = "권한이 없습니다.")]
    public static string Unauthorized => throw new NotImplementedException();
}
```

#### 소스 생성기 구현

```csharp
// MessageSourceGenerator.cs
[Generator]
public class MessageSourceGenerator : IIncrementalGenerator
{
    public void Initialize(IncrementalGeneratorInitializationContext context)
    {
        var classDeclarations = context.SyntaxProvider
            .ForAttributeWithMetadataName(
                "GenerateMessagesAttribute",
                predicate: static (node, _) => node is ClassDeclarationSyntax,
                transform: static (context, _) => GetMessageClassInfo(context))
            .Where(static m => m is not null);

        context.RegisterSourceOutput(classDeclarations, static (spc, source) => 
        {
            if (source != null)
            {
                var sourceCode = GenerateMessageClass(source);
                spc.AddSource($"{source.ClassName}.g.cs", sourceCode);
            }
        });
    }

    private static MessageClassInfo? GetMessageClassInfo(GeneratorAttributeSyntaxContext context)
    {
        var classDeclaration = (ClassDeclarationSyntax)context.TargetNode;
        var classSymbol = (INamedTypeSymbol)context.TargetSymbol;

        var generateAttribute = context.Attributes.FirstOrDefault();
        if (generateAttribute == null) return null;

        var resourcePath = generateAttribute.NamedArguments
            .FirstOrDefault(x => x.Key == "ResourcePath").Value.Value?.ToString() ?? "Resources";
        var className = generateAttribute.NamedArguments
            .FirstOrDefault(x => x.Key == "ClassName").Value.Value?.ToString() ?? "GeneratedMessages";

        var messageKeys = new List<MessageKeyInfo>();

        foreach (var member in classSymbol.GetMembers().OfType<IPropertySymbol>())
        {
            var messageKeyAttr = member.GetAttributes()
                .FirstOrDefault(attr => attr.AttributeClass?.Name == "MessageKeyAttribute");

            if (messageKeyAttr != null && messageKeyAttr.ConstructorArguments.Length > 0)
            {
                var key = messageKeyAttr.ConstructorArguments[0].Value?.ToString() ?? "";
                var defaultValue = messageKeyAttr.NamedArguments
                    .FirstOrDefault(x => x.Key == "DefaultValue").Value.Value?.ToString() ?? "";

                messageKeys.Add(new MessageKeyInfo
                {
                    PropertyName = member.Name,
                    Key = key,
                    DefaultValue = defaultValue
                });
            }
        }

        return new MessageClassInfo
        {
            ClassName = className,
            Namespace = classSymbol.ContainingNamespace.ToDisplayString(),
            ResourcePath = resourcePath,
            MessageKeys = messageKeys
        };
    }

    private static string GenerateMessageClass(MessageClassInfo info)
    {
        var sb = new StringBuilder();

        sb.AppendLine("using Microsoft.Extensions.Localization;");
        sb.AppendLine("using Microsoft.Extensions.DependencyInjection;");
        sb.AppendLine();

        if (!string.IsNullOrEmpty(info.Namespace))
        {
            sb.AppendLine($"namespace {info.Namespace};");
            sb.AppendLine();
        }

        sb.AppendLine($"public static class {info.ClassName}");
        sb.AppendLine("{");
        sb.AppendLine("    private static IStringLocalizer? _localizer;");
        sb.AppendLine();
        sb.AppendLine("    public static void Initialize(IServiceProvider serviceProvider)");
        sb.AppendLine("    {");
        sb.AppendLine("        _localizer = serviceProvider.GetService<IStringLocalizer<MessageDefinitions>>();");
        sb.AppendLine("    }");
        sb.AppendLine();

        foreach (var messageKey in info.MessageKeys)
        {
            sb.AppendLine($"    public static string {messageKey.PropertyName}");
            sb.AppendLine("    {");
            sb.AppendLine($"        get => _localizer?[\"{messageKey.Key}\"] ?? \"{messageKey.DefaultValue}\";");
            sb.AppendLine("    }");
            sb.AppendLine();

            // 매개변수가 있는 메시지용 오버로드
            if (messageKey.DefaultValue.Contains("{0}"))
            {
                sb.AppendLine($"    public static string {messageKey.PropertyName}(params object[] args)");
                sb.AppendLine("    {");
                sb.AppendLine($"        return _localizer?[\"{messageKey.Key}\", args] ?? string.Format(\"{messageKey.DefaultValue}\", args);");
                sb.AppendLine("    }");
                sb.AppendLine();
            }
        }

        sb.AppendLine("}");

        return sb.ToString();
    }

    private class MessageClassInfo
    {
        public string ClassName { get; set; } = "";
        public string Namespace { get; set; } = "";
        public string ResourcePath { get; set; } = "";
        public List<MessageKeyInfo> MessageKeys { get; set; } = new();
    }

    private class MessageKeyInfo
    {
        public string PropertyName { get; set; } = "";
        public string Key { get; set; } = "";
        public string DefaultValue { get; set; } = "";
    }
}
```

#### 사용 예제

```csharp
// Program.cs에서 초기화
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddLocalization();

var app = builder.Build();

// 생성된 메시지 클래스 초기화
AppMessages.Initialize(app.Services);

// 컨트롤러에서 사용
[ApiController]
[Route("api/[controller]")]
public class MessagesController : ControllerBase
{
    [HttpGet("typed")]
    public IActionResult GetTypedMessage()
    {
        // 컴파일 시점에 타입 안전성 보장
        var basicMessage = AppMessages.BasicMessage;
        var parameterMessage = AppMessages.ParameterMessage("1", 2);
        
        return Ok(new 
        { 
            basic = basicMessage,
            parameter = parameterMessage
        });
    }

    [HttpGet("validation")]
    public IActionResult GetValidationMessage()
    {
        // IntelliSense 지원으로 개발 생산성 향상
        var requiredMessage = AppMessages.ValidationRequired;
        var unauthorizedMessage = AppMessages.Unauthorized;
        
        return Ok(new 
        { 
            required = requiredMessage,
            unauthorized = unauthorizedMessage
        });
    }
}
```

### 통합 사용 예제

위의 세 가지 방식을 통합하여 사용하는 완전한 예제이다.

```csharp
// MessageController.cs - 모든 방식을 통합한 컨트롤러
[ApiController]
[Route("api/[controller]")]
public class MessageController : ControllerBase
{
    private readonly IStringLocalizer<Messages> _localizer;
    private readonly MessageService _messageService;

    public MessageController(
        IStringLocalizer<Messages> localizer,
        MessageService messageService)
    {
        _localizer = localizer;
        _messageService = messageService;
    }

    [HttpGet("basic/{key}")]
    public IActionResult GetBasicMessage(string key)
    {
        // 1. 직접 IStringLocalizer 사용
        var directMessage = _localizer[key];
        
        // 2. MessageService 사용
        var serviceMessage = _messageService.GetMessage(key);
        
        // 3. 생성된 타입 안전 메시지 사용
        var typedMessage = key switch
        {
            "resource.basic.msg1" => AppMessages.BasicMessage,
            "error.validation.required" => AppMessages.ValidationRequired,
            _ => "Unknown message key"
        };

        return Ok(new
        {
            direct = directMessage.ToString(),
            service = serviceMessage,
            typed = typedMessage,
            culture = CultureInfo.CurrentUICulture.Name
        });
    }

    [HttpGet("parameter")]
    public IActionResult GetParameterMessage([FromQuery] string param1, [FromQuery] int param2)
    {
        // 매개변수가 있는 메시지 처리
        var serviceMessage = _messageService.GetMessage("resource.basic.msg3", param1, param2);
        var typedMessage = AppMessages.ParameterMessage(param1, param2);

        return Ok(new
        {
            service = serviceMessage,
            typed = typedMessage
        });
    }

    [HttpPost("culture")]
    public IActionResult SetCulture([FromBody] SetCultureRequest request)
    {
        try
        {
            var culture = new CultureInfo(request.Culture);
            CultureInfo.CurrentCulture = culture;
            CultureInfo.CurrentUICulture = culture;

            return Ok(new 
            { 
                message = _localizer["culture.changed"],
                newCulture = culture.Name 
            });
        }
        catch (CultureNotFoundException)
        {
            return BadRequest(new 
            { 
                error = _localizer["error.culture.invalid"],
                requestedCulture = request.Culture 
            });
        }
    }
}

public class SetCultureRequest
{
    public string Culture { get; set; } = "";
}
```

### 성능 최적화

.NET 8에서 리소스 메시지 서비스의 성능을 최적화하는 방법이다.

#### 메모리 캐싱

```csharp
// CachedMessageService.cs
public class CachedMessageService : IMessageService
{
    private readonly IStringLocalizer _localizer;
    private readonly IMemoryCache _cache;
    private readonly ILogger<CachedMessageService> _logger;
    private static readonly TimeSpan CacheExpiration = TimeSpan.FromHours(1);

    public CachedMessageService(
        IStringLocalizer localizer,
        IMemoryCache cache,
        ILogger<CachedMessageService> logger)
    {
        _localizer = localizer;
        _cache = cache;
        _logger = logger;
    }

    public string GetMessage(string key, CultureInfo? culture = null)
    {
        var cacheKey = $"message:{key}:{culture?.Name ?? CultureInfo.CurrentUICulture.Name}";
        
        return _cache.GetOrCreate(cacheKey, entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = CacheExpiration;
            
            var originalCulture = CultureInfo.CurrentUICulture;
            try
            {
                if (culture != null)
                    CultureInfo.CurrentUICulture = culture;
                
                var message = _localizer[key].ToString();
                _logger.LogDebug("Message cached: {Key} = {Message}", key, message);
                return message;
            }
            finally
            {
                CultureInfo.CurrentUICulture = originalCulture;
            }
        });
    }
}
```

#### AOT 호환 최적화

```csharp
// AotOptimizedMessageService.cs
[JsonSerializable(typeof(MessageResponse))]
[JsonSerializable(typeof(Dictionary<string, string>))]
public partial class MessageJsonContext : JsonSerializerContext
{
}

public class AotOptimizedMessageService
{
    private readonly IStringLocalizer _localizer;
    private static readonly JsonSerializerOptions JsonOptions = new()
    {
        TypeInfoResolver = MessageJsonContext.Default
    };

    public string SerializeMessages(Dictionary<string, string> messages)
    {
        // AOT 호환 JSON 직렬화
        return JsonSerializer.Serialize(messages, MessageJsonContext.Default.DictionaryStringString);
    }

    public MessageResponse CreateResponse(string key, string message)
    {
        return new MessageResponse { Key = key, Message = message };
    }
}

public class MessageResponse
{
    public string Key { get; set; } = "";
    public string Message { get; set; } = "";
}
```

## 참고자료

- [ASP.NET Core Localization](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/localization)
- [.NET Globalization and ICU](https://docs.microsoft.com/en-us/dotnet/core/extensions/globalization-icu)
- [Source Generators](https://docs.microsoft.com/en-us/dotnet/csharp/roslyn-sdk/source-generators-overview)
- [.NET 8 Performance Improvements](https://devblogs.microsoft.com/dotnet/performance-improvements-in-net-8/)
