# .NET 8 Razor 표현식

## 개요

.NET 8 LTS에서 사용되는 Razor 표현식은 강력하고 유연한 뷰 엔진 문법으로, ASP.NET Core의 핵심 기능 중 하나이다.  
Razor 표현식을 통해 C# 코드를 HTML과 원활하게 통합할 수 있으며, 모델 데이터에 직접 접근하여 동적인 웹 페이지를 생성할 수 있다.

### 역사적 배경

과거 .NET Framework 시절에는 Classic ASP 스타일의 `<% %>`, `<%= %>` 표현식이 사용되었으나, 현재는 모두 Razor 기반의 `@` 표현식으로 대체되었다. 이는 더 간결하고 가독성이 높으며, IntelliSense 지원도 우수하다.

## 설명

### 의존성 주입을 사용한 설정값 접근

ASP.NET Core의 의존성 주입 컨테이너를 통해 설정값이나 서비스에 접근할 수 있다.

* 다음은 **IConfiguration을 통한 설정값 접근** 예제이다.

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// 설정 서비스 등록
builder.Services.Configure<DatabaseOptions>(
    builder.Configuration.GetSection("Database"));

var app = builder.Build();
```

```csharp
// Models/DatabaseOptions.cs
public class DatabaseOptions
{
    public string ConnectionString { get; set; } = string.Empty;
    public string Provider { get; set; } = string.Empty;
    public string Username { get; set; } = string.Empty;
    public string Password { get; set; } = string.Empty;
}
```

* 다음은 **appsettings.json을 사용한 설정** 예제이다.

appsettings.json

```json
{
  "Database": {
    "ConnectionString": "Server=localhost;Database=EASYCOMPANY;",
    "Provider": "SqlServer",
    "Username": "tex",
    "Password": "texAdmin"
  }
}
```

* 다음은 **컨트롤러에서 설정값 주입** 예제이다.

```csharp
// Controllers/HomeController.cs
public class HomeController : Controller
{
    private readonly IOptions<DatabaseOptions> _dbOptions;
    
    public HomeController(IOptions<DatabaseOptions> dbOptions)
    {
        _dbOptions = dbOptions;
    }
    
    public IActionResult Index()
    {
        return View(_dbOptions.Value);
    }
}
```

### Razor 뷰에서 표현식 사용

Razor 뷰에서는 `@` 기호를 사용하여 C# 표현식을 작성할 수 있다. 과거 Classic ASP의 `<%= %>` 문법을 대체하는 현대적인 방식이다.

```html
@{
    ViewData["Title"] = "Home Page";
}
@model DatabaseOptions

<h1>데이터베이스 설정</h1>
```

`@` 표현식을 사용하여 모델 데이터에 직접 접근할 수 있다.

```html
<p>연결 문자열: @Model.ConnectionString</p>
<p>공급자: @Model.Provider</p>
```

메서드 호출도 가능하다.

```html
<p>현재 시간: @DateTime.Now.ToString("yyyy-MM-dd HH:mm:ss")</p>
```

### 포맷팅과 데이터 변환

.NET 8에서는 문자열 보간법과 포맷 문자열을 사용하여 데이터를 포맷할 수 있다.

```csharp
// Models/SampleViewModel.cs
public class SampleViewModel
{
    [DisplayFormat(DataFormatString = "{0:N0}")]
    public int Money { get; set; }
    
    public DateTime CreatedDate { get; set; }
}
```

```html
@model SampleViewModel

<!-- 숫자 포맷팅 -->
<p>잔액: @Model.Money.ToString("N0")</p>

<!-- 날짜 포맷팅 -->
<p>생성일: @Model.CreatedDate.ToString("yyyy년 MM월 dd일")</p>
```

위와 같이 적용하면 입력 값에 따라 3자리마다 쉼표(,)가 출력된다.

**입력값: 1234000**  
**출력값: 1,234,000**

직접 포맷 문자열을 사용할 수도 있다.

```html
<p>가격: @string.Format("{0:N0}", ViewBag.Price)</p>

<!-- 또는 문자열 보간법 사용 -->
<p>가격: @($"{ViewBag.Price:N0}")</p>
```

### Tag Helpers 사용

ASP.NET Core의 Tag Helpers를 사용하면 더욱 선언적인 방식으로 데이터를 표시할 수 있다.

```html
<!-- 기존 Razor 표현식 -->
<input type="text" value="@Model.Name" name="Name" />

<!-- Tag Helper 사용 -->
<input asp-for="Name" class="form-control" />
```

조건부 렌더링:

```html
@if (Model.Money > 0)
{
    <span class="text-success">@Model.Money.ToString("C")</span>
}
else
{
    <span class="text-danger">잔액 부족</span>
}
```

### XAML 참조 표현식 (클라이언트 애플리케이션)

SpEL과 비슷한 성격의 도메인 언어로 Razor 표현식 외에도 WPF, Avalonia 등의 클라이언트 애플리케이션에서 XAML의 참조 표현식을 사용한다. 이는 Razor와는 다른 방식이지만 비슷한 목적을 가진다.

#### WPF에서의 바인딩 표현식

```xml
<!-- MainWindow.xaml -->
<Window x:Class="SampleApp.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
    <Grid>
        <TextBlock Text="{Binding Name}" />
        <TextBlock Text="{Binding Money, StringFormat=N0}" />
        <TextBlock Text="{Binding CreatedDate, StringFormat=yyyy년 MM월 dd일}" />
    </Grid>
</Window>
```

```csharp
// ViewModels/MainViewModel.cs
public class MainViewModel : INotifyPropertyChanged
{
    private string _name = string.Empty;
    private int _money;
    private DateTime _createdDate = DateTime.Now;

    public string Name
    {
        get => _name;
        set { _name = value; OnPropertyChanged(); }
    }

    public int Money
    {
        get => _money;
        set { _money = value; OnPropertyChanged(); }
    }

    public DateTime CreatedDate
    {
        get => _createdDate;
        set { _createdDate = value; OnPropertyChanged(); }
    }

    public event PropertyChangedEventHandler? PropertyChanged;
    
    protected virtual void OnPropertyChanged([CallerMemberName] string? propertyName = null)
    {
        PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName));
    }
}
```

#### Avalonia에서의 바인딩 표현식

Avalonia는 크로스 플랫폼 XAML 프레임워크로, WPF와 유사한 문법을 사용한다.

```xml
<!-- Views/MainView.axaml -->
<UserControl xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             x:Class="SampleApp.Views.MainView">
    <StackPanel>
        <TextBlock Text="{Binding Name}" />
        <TextBlock Text="{Binding Money, StringFormat=\{0:N0\}}" />
        <TextBlock Text="{Binding #slider.Value}" />
        <Slider Name="slider" Minimum="0" Maximum="100" />
    </StackPanel>
</UserControl>
```

#### 리소스 참조와 정적 확장

XAML에서는 리소스나 정적 값을 참조하는 표현식도 사용할 수 있다.

```xml
<Window.Resources>
    <SolidColorBrush x:Key="PrimaryBrush" Color="Blue" />
    <sys:String x:Key="AppTitle">샘플 애플리케이션</sys:String>
</Window.Resources>

<Grid>
    <!-- 리소스 참조 -->
    <TextBlock Text="{StaticResource AppTitle}" 
               Foreground="{StaticResource PrimaryBrush}" />
    
    <!-- 정적 값 참조 -->
    <TextBlock Text="{x:Static sys:Environment.MachineName}" />
</Grid>
```

#### 마크업 확장과 컨버터

```csharp
// Converters/MoneyConverter.cs
public class MoneyConverter : IValueConverter
{
    public object Convert(object value, Type targetType, object parameter, CultureInfo culture)
    {
        if (value is int money)
            return money.ToString("N0") + "원";
        return value;
    }

    public object ConvertBack(object value, Type targetType, object parameter, CultureInfo culture)
    {
        throw new NotImplementedException();
    }
}
```

```xml
<Window.Resources>
    <local:MoneyConverter x:Key="MoneyConverter" />
</Window.Resources>

<TextBlock Text="{Binding Money, Converter={StaticResource MoneyConverter}}" />
```

**웹 vs 클라이언트 표현식 비교:**

| 구분 | 웹 (Razor) | 클라이언트 (XAML) |
|------|------------|-------------------|
| 문법 | `@Model.Property` | `{Binding Property}` |
| 실행 시점 | 서버 렌더링 시 | 런타임 데이터 바인딩 |
| 양방향 바인딩 | 불가능 | 가능 |
| 실시간 업데이트 | 불가능 | 가능 (INotifyPropertyChanged) |

## 참고자료

* [ASP.NET Core Razor 구문 참조](https://docs.microsoft.com/ko-kr/aspnet/core/mvc/views/razor)
* [.NET 8 문자열 보간법](https://docs.microsoft.com/ko-kr/dotnet/csharp/language-reference/tokens/interpolated)
* [ASP.NET Core 의존성 주입](https://docs.microsoft.com/ko-kr/aspnet/core/fundamentals/dependency-injection)
* [WPF 데이터 바인딩 개요](https://docs.microsoft.com/ko-kr/dotnet/desktop/wpf/data/)
* [Avalonia 바인딩 가이드](https://docs.avaloniaui.net/docs/data-binding/)
