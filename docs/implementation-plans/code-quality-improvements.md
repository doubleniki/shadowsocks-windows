# План улучшения качества кода

## 1. Обзор

Данный документ описывает детальный план по улучшению качества кода проекта Shadowsocks-Windows через внедрение современных практик разработки на .NET.

## 2. Текущее состояние

### Проблемы
- Синхронные I/O операции блокируют UI поток
- Использование `Thread.Sleep` вместо асинхронных таймеров
- Service Locator паттерн усложняет тестирование
- Недостаточная обработка исключений
- Низкое покрытие тестами
- Устаревшие паттерны кода

### Сильные стороны
- ✅ Уже используется ReactiveUI
- ✅ MVVM архитектура частично реализована
- ✅ Структурированная организация кода

## 3. Цели

### Основные
- [ ] Миграция на async/await для всех I/O операций
- [ ] Внедрение Dependency Injection
- [ ] Улучшение обработки исключений
- [ ] Использование ReactiveUI для управления состоянием
- [ ] Увеличение покрытия тестами до 60%+

### Дополнительные
- [ ] Рефакторинг legacy кода
- [ ] Улучшение производительности
- [ ] Снижение технического долга

---

## 4. Детальный план

### 4.1 Миграция на Async/Await

#### Текущие проблемы

Поиск синхронных I/O операций:
```bash
# Файловые операции
File.ReadAllText()
File.WriteAllText()
File.ReadAllBytes()

# Сетевые операции
WebClient.DownloadString()
HttpClient синхронные методы
Stream.Read() без async

# Database/Config operations
Синхронное чтение конфигурации
```

#### План миграции

**Фаза 1: Идентификация (1 неделя)**

```csharp
// Плохо ❌
public void LoadConfiguration()
{
    var json = File.ReadAllText(configPath);
    var config = JsonConvert.DeserializeObject<Configuration>(json);
    ApplyConfiguration(config);
}

// Хорошо ✅
public async Task LoadConfigurationAsync(CancellationToken cancellationToken = default)
{
    var json = await File.ReadAllTextAsync(configPath, cancellationToken);
    var config = JsonConvert.DeserializeObject<Configuration>(json);
    await ApplyConfigurationAsync(config, cancellationToken);
}
```

**Фаза 2: Приоритетные файлы (2-3 недели)**

Порядок миграции:
1. **Controller/FileManager.cs** - файловые операции
   ```csharp
   // До
   public static void Save(Configuration config)
   {
       var json = JsonConvert.SerializeObject(config, Formatting.Indented);
       File.WriteAllText(CONFIG_FILE, json);
   }

   // После
   public static async Task SaveAsync(Configuration config, CancellationToken ct = default)
   {
       var json = JsonConvert.SerializeObject(config, Formatting.Indented);
       await File.WriteAllTextAsync(CONFIG_FILE, json, ct);
   }
   ```

2. **Controller/Service/UpdateChecker.cs** - HTTP запросы
   ```csharp
   // До
   using (var client = new WebClient())
   {
       var response = client.DownloadString(updateUrl);
       // ...
   }

   // После
   using (var client = new HttpClient())
   {
       var response = await client.GetStringAsync(updateUrl, cancellationToken);
       // ...
   }
   ```

3. **Controller/Service/GeositeUpdater.cs** - загрузка данных
4. **Controller/Service/OnlineConfigResolver.cs** - онлайн конфигурации
5. **Proxy/HttpProxy.cs, Socks5Proxy.cs** - прокси операции

**Фаза 3: UI интеграция (1-2 недели)**

```csharp
// ViewModel пример
public class MainViewModel : ReactiveObject
{
    private readonly IConfigurationService _configService;

    // ReactiveCommand автоматически обрабатывает async
    public ReactiveCommand<Unit, Unit> LoadConfigCommand { get; }

    public MainViewModel(IConfigurationService configService)
    {
        _configService = configService;

        LoadConfigCommand = ReactiveCommand.CreateFromTask(
            async ct => await _configService.LoadConfigurationAsync(ct),
            outputScheduler: RxApp.MainThreadScheduler);

        // Обработка ошибок
        LoadConfigCommand.ThrownExceptions
            .Subscribe(ex => this.Log().Error(ex, "Failed to load configuration"));
    }
}
```

**Задачи**:
- [ ] Провести аудит всех синхронных I/O операций
- [ ] Создать список приоритетных файлов
- [ ] Мигрировать файловые операции
- [ ] Мигрировать сетевые операции
- [ ] Обновить ViewModels для использования async команд
- [ ] Добавить CancellationToken поддержку
- [ ] Обновить тесты

**Срок**: 4-6 недель

---

### 4.2 Внедрение Dependency Injection

#### Текущий подход (Service Locator)

```csharp
// Плохо ❌
public class ShadowsocksController
{
    private Configuration _config;

    public ShadowsocksController()
    {
        // Прямые зависимости
        _config = Configuration.Load();
        fileManager = new FileManager();
        updateChecker = new UpdateChecker(this);
    }
}
```

#### Новый подход (DI)

**Шаг 1: Установка пакетов**

```xml
<PackageReference Include="Microsoft.Extensions.DependencyInjection" Version="8.0.0" />
<PackageReference Include="Microsoft.Extensions.Hosting" Version="8.0.0" />
<PackageReference Include="Microsoft.Extensions.Logging.NLog" Version="5.3.5" />
```

**Шаг 2: Создание интерфейсов**

```csharp
// Interfaces/IConfigurationService.cs
public interface IConfigurationService
{
    Task<Configuration> LoadAsync(CancellationToken ct = default);
    Task SaveAsync(Configuration config, CancellationToken ct = default);
    IObservable<Configuration> ConfigurationChanged { get; }
}

// Interfaces/IUpdateChecker.cs
public interface IUpdateChecker
{
    Task<UpdateInfo> CheckForUpdatesAsync(CancellationToken ct = default);
}

// Interfaces/IShadowsocksController.cs
public interface IShadowsocksController
{
    Task StartAsync(CancellationToken ct = default);
    Task StopAsync(CancellationToken ct = default);
    Configuration CurrentConfiguration { get; }
}
```

**Шаг 3: Регистрация сервисов**

```csharp
// ServiceCollectionExtensions.cs
public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddShadowsocksServices(
        this IServiceCollection services)
    {
        // Core services
        services.AddSingleton<IConfigurationService, ConfigurationService>();
        services.AddSingleton<IShadowsocksController, ShadowsocksController>();

        // Feature services
        services.AddSingleton<IUpdateChecker, UpdateChecker>();
        services.AddSingleton<IGeositeUpdater, GeositeUpdater>();
        services.AddSingleton<IPACServer, PACServer>();
        services.AddSingleton<IPrivoxyRunner, PrivoxyRunner>();

        // ViewModels
        services.AddTransient<MainViewModel>();
        services.AddTransient<ServerViewModel>();
        services.AddTransient<ForwardProxyViewModel>();
        services.AddTransient<HotkeysViewModel>();

        return services;
    }
}

// Program.cs
public static class Program
{
    [STAThread]
    public static void Main(string[] args)
    {
        var host = Host.CreateDefaultBuilder(args)
            .ConfigureServices((context, services) =>
            {
                services.AddShadowsocksServices();

                // Logging
                services.AddLogging(logging =>
                {
                    logging.ClearProviders();
                    logging.AddNLog();
                });
            })
            .Build();

        // Start application
        var app = new App(host.Services);
        app.InitializeComponent();
        app.Run();
    }
}
```

**Шаг 4: Обновление классов**

```csharp
// Services/ConfigurationService.cs
public class ConfigurationService : IConfigurationService
{
    private readonly ILogger<ConfigurationService> _logger;
    private readonly string _configPath;
    private readonly Subject<Configuration> _configChanged;

    public IObservable<Configuration> ConfigurationChanged => _configChanged;

    public ConfigurationService(ILogger<ConfigurationService> logger)
    {
        _logger = logger;
        _configPath = Path.Combine(AppDomain.CurrentDomain.BaseDirectory, "config.json");
        _configChanged = new Subject<Configuration>();
    }

    public async Task<Configuration> LoadAsync(CancellationToken ct = default)
    {
        try
        {
            _logger.LogInformation("Loading configuration from {Path}", _configPath);

            if (!File.Exists(_configPath))
            {
                var defaultConfig = Configuration.CreateDefault();
                await SaveAsync(defaultConfig, ct);
                return defaultConfig;
            }

            var json = await File.ReadAllTextAsync(_configPath, ct);
            var config = JsonConvert.DeserializeObject<Configuration>(json);

            return config ?? Configuration.CreateDefault();
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to load configuration");
            throw;
        }
    }

    public async Task SaveAsync(Configuration config, CancellationToken ct = default)
    {
        try
        {
            _logger.LogInformation("Saving configuration to {Path}", _configPath);

            var json = JsonConvert.SerializeObject(config, Formatting.Indented);
            await File.WriteAllTextAsync(_configPath, json, ct);

            _configChanged.OnNext(config);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to save configuration");
            throw;
        }
    }
}

// Controller/ShadowsocksController.cs
public class ShadowsocksController : IShadowsocksController
{
    private readonly IConfigurationService _configService;
    private readonly IUpdateChecker _updateChecker;
    private readonly ILogger<ShadowsocksController> _logger;

    public Configuration CurrentConfiguration { get; private set; }

    public ShadowsocksController(
        IConfigurationService configService,
        IUpdateChecker updateChecker,
        ILogger<ShadowsocksController> logger)
    {
        _configService = configService;
        _updateChecker = updateChecker;
        _logger = logger;
    }

    public async Task StartAsync(CancellationToken ct = default)
    {
        _logger.LogInformation("Starting Shadowsocks controller");

        CurrentConfiguration = await _configService.LoadAsync(ct);

        // Subscribe to configuration changes
        _configService.ConfigurationChanged
            .Subscribe(config =>
            {
                CurrentConfiguration = config;
                // Reload services with new config
            });

        // Start services...
    }
}
```

**Задачи**:
- [ ] Установить Microsoft.Extensions.DependencyInjection
- [ ] Создать интерфейсы для всех сервисов
- [ ] Реализовать ServiceCollectionExtensions
- [ ] Обновить Program.cs для использования DI
- [ ] Мигрировать ViewModels на DI
- [ ] Обновить тесты с использованием моков
- [ ] Удалить Service Locator код

**Срок**: 3-4 недели

---

### 4.3 Улучшение обработки исключений

#### Текущие проблемы

```csharp
// Проблема 1: Проглатывание исключений ❌
try
{
    SomeOperation();
}
catch { }

// Проблема 2: Общий catch без логирования ❌
try
{
    SomeOperation();
}
catch (Exception)
{
    return false;
}

// Проблема 3: Нет контекста ошибки ❌
throw new Exception("Failed");
```

#### Решения

**1. Централизованная обработка ошибок**

```csharp
// Utilities/ErrorHandling/GlobalExceptionHandler.cs
public static class GlobalExceptionHandler
{
    private static ILogger _logger;

    public static void Initialize(ILogger logger)
    {
        _logger = logger;

        AppDomain.CurrentDomain.UnhandledException += OnUnhandledException;
        TaskScheduler.UnobservedTaskException += OnUnobservedTaskException;
        Application.Current.DispatcherUnhandledException += OnDispatcherUnhandledException;
    }

    private static void OnUnhandledException(object sender, UnhandledExceptionEventArgs e)
    {
        LogException(e.ExceptionObject as Exception, "Unhandled exception");

        if (e.IsTerminating)
        {
            MessageBox.Show(
                "A critical error occurred. The application will now close.",
                "Critical Error",
                MessageBoxButton.OK,
                MessageBoxImage.Error);
        }
    }

    private static void OnUnobservedTaskException(object sender, UnobservedTaskExceptionEventArgs e)
    {
        LogException(e.Exception, "Unobserved task exception");
        e.SetObserved(); // Prevent process termination
    }

    private static void OnDispatcherUnhandledException(object sender, DispatcherUnhandledExceptionEventArgs e)
    {
        LogException(e.Exception, "Dispatcher unhandled exception");

        MessageBox.Show(
            $"An error occurred: {e.Exception.Message}\n\nSee logs for details.",
            "Error",
            MessageBoxButton.OK,
            MessageBoxImage.Error);

        e.Handled = true;
    }

    private static void LogException(Exception ex, string context)
    {
        _logger.Error(ex, "{Context}: {Message}", context, ex?.Message);
    }
}
```

**2. Кастомные исключения**

```csharp
// Exceptions/ShadowsocksException.cs
public class ShadowsocksException : Exception
{
    public string ErrorCode { get; }

    public ShadowsocksException(string message, string errorCode = null)
        : base(message)
    {
        ErrorCode = errorCode;
    }

    public ShadowsocksException(string message, Exception innerException, string errorCode = null)
        : base(message, innerException)
    {
        ErrorCode = errorCode;
    }
}

// Exceptions/ConfigurationException.cs
public class ConfigurationException : ShadowsocksException
{
    public string ConfigPath { get; }

    public ConfigurationException(string message, string configPath)
        : base(message, "CONFIG_ERROR")
    {
        ConfigPath = configPath;
    }
}

// Exceptions/NetworkException.cs
public class NetworkException : ShadowsocksException
{
    public string ServerAddress { get; }

    public NetworkException(string message, string serverAddress, Exception innerException = null)
        : base(message, innerException, "NETWORK_ERROR")
    {
        ServerAddress = serverAddress;
    }
}
```

**3. Graceful Degradation**

```csharp
// Services/ConfigurationService.cs (улучшенная версия)
public async Task<Configuration> LoadAsync(CancellationToken ct = default)
{
    try
    {
        return await LoadFromFileAsync(ct);
    }
    catch (FileNotFoundException ex)
    {
        _logger.LogWarning(ex, "Configuration file not found, creating default");
        return await CreateDefaultConfigAsync(ct);
    }
    catch (JsonException ex)
    {
        _logger.LogError(ex, "Invalid configuration format, backing up and creating new");

        // Backup corrupted file
        await BackupCorruptedConfigAsync(ct);

        return await CreateDefaultConfigAsync(ct);
    }
    catch (UnauthorizedAccessException ex)
    {
        _logger.LogError(ex, "Access denied to configuration file");
        throw new ConfigurationException(
            "Cannot access configuration file. Check permissions.",
            _configPath);
    }
}

private async Task BackupCorruptedConfigAsync(CancellationToken ct)
{
    var backupPath = $"{_configPath}.corrupted.{DateTime.Now:yyyyMMddHHmmss}";
    await File.CopyAsync(_configPath, backupPath, ct);
    _logger.LogInformation("Backed up corrupted config to {Path}", backupPath);
}
```

**4. ReactiveUI Integration**

```csharp
// ViewModels/BaseViewModel.cs
public abstract class BaseViewModel : ReactiveObject
{
    protected ILogger Logger { get; }

    protected BaseViewModel(ILogger logger)
    {
        Logger = logger;
    }

    protected ReactiveCommand<TParam, TResult> CreateCommand<TParam, TResult>(
        Func<TParam, CancellationToken, Task<TResult>> execute,
        IObservable<bool> canExecute = null)
    {
        var command = ReactiveCommand.CreateFromTask(execute, canExecute);

        // Centralized error handling
        command.ThrownExceptions
            .Subscribe(ex =>
            {
                Logger.LogError(ex, "Command execution failed");
                ShowErrorMessage(ex);
            });

        return command;
    }

    protected virtual void ShowErrorMessage(Exception ex)
    {
        var message = ex switch
        {
            ConfigurationException cfg => $"Configuration error: {cfg.Message}",
            NetworkException net => $"Network error: {net.Message}",
            _ => $"An error occurred: {ex.Message}"
        };

        MessageBox.Show(message, "Error", MessageBoxButton.OK, MessageBoxImage.Error);
    }
}
```

**Задачи**:
- [ ] Создать GlobalExceptionHandler
- [ ] Создать кастомные исключения
- [ ] Обновить все catch блоки с логированием
- [ ] Реализовать graceful degradation
- [ ] Интегрировать с ReactiveUI
- [ ] Добавить telemetry/crash reporting (опционально)

**Срок**: 2-3 недели

---

### 4.4 Замена Thread.Sleep на ReactiveUI

#### Текущие проблемы

```csharp
// Плохо ❌
while (retrying)
{
    try
    {
        Connect();
        retrying = false;
    }
    catch
    {
        Thread.Sleep(1000); // Блокирует поток!
    }
}
```

#### Решения

**1. Retry с ReactiveUI**

```csharp
// Хорошо ✅
Observable.FromAsync(ConnectAsync)
    .Retry(3)
    .Delay(TimeSpan.FromSeconds(1))
    .Subscribe(
        result => _logger.LogInformation("Connected"),
        error => _logger.LogError(error, "Connection failed"));

// Или с экспоненциальным backoff
Observable.FromAsync(ConnectAsync)
    .RetryWithBackoff(
        retryCount: 5,
        delay: TimeSpan.FromSeconds(1),
        backoffStrategy: RetryStrategy.Exponential)
    .Subscribe(/* ... */);
```

**2. Throttle/Debounce для UI**

```csharp
// ViewModel
public class ServerViewModel : BaseViewModel
{
    private readonly ObservableAsPropertyHelper<List<Server>> _filteredServers;

    [Reactive]
    public string SearchText { get; set; }

    public IObservable<List<Server>> FilteredServers => _filteredServers;

    public ServerViewModel(IServerService serverService)
    {
        // Debounce search input - ждем 300ms после последнего ввода
        _filteredServers = this.WhenAnyValue(x => x.SearchText)
            .Throttle(TimeSpan.FromMilliseconds(300))
            .Select(searchText => serverService.Search(searchText))
            .ToProperty(this, x => x.FilteredServers);
    }
}
```

**3. Периодические задачи**

```csharp
// Плохо ❌
while (true)
{
    CheckForUpdates();
    Thread.Sleep(3600000); // 1 hour
}

// Хорошо ✅
Observable.Interval(TimeSpan.FromHours(1))
    .SelectMany(_ => Observable.FromAsync(ct => CheckForUpdatesAsync(ct)))
    .Subscribe(
        updateInfo => ProcessUpdate(updateInfo),
        error => _logger.LogError(error, "Update check failed"));

// Или с Timer
var updateCheckTimer = Observable.Timer(
    dueTime: TimeSpan.Zero,
    period: TimeSpan.FromHours(1))
    .SelectMany(_ => Observable.FromAsync(CheckForUpdatesAsync))
    .Publish();

_subscriptions.Add(updateCheckTimer.Connect());
```

**4. Cancellation Token Integration**

```csharp
public class UpdateChecker : IUpdateChecker, IDisposable
{
    private readonly CancellationTokenSource _cts = new();
    private readonly CompositeDisposable _subscriptions = new();

    public void StartPeriodicCheck()
    {
        Observable.Interval(TimeSpan.FromHours(1))
            .TakeUntil(_ => _cts.Token.IsCancellationRequested)
            .SelectMany(_ => Observable.FromAsync(ct => CheckForUpdatesAsync(ct)))
            .Subscribe(/* ... */)
            .DisposeWith(_subscriptions);
    }

    public void Dispose()
    {
        _cts.Cancel();
        _cts.Dispose();
        _subscriptions.Dispose();
    }
}
```

**Задачи**:
- [ ] Найти все использования Thread.Sleep
- [ ] Заменить на Observable.Timer/Interval
- [ ] Реализовать retry с backoff
- [ ] Добавить throttle/debounce для UI
- [ ] Обеспечить правильную отмену подписок

**Срок**: 1-2 недели

---

### 4.5 Увеличение покрытия тестами

#### Текущее состояние

```bash
# Проверить текущее покрытие
dotnet test /p:CollectCoverage=true /p:CoverletOutputFormat=opencover
```

#### Стратегия тестирования

**1. Unit тесты для сервисов**

```csharp
// Test/Services/ConfigurationServiceTests.cs
public class ConfigurationServiceTests
{
    private readonly Mock<ILogger<ConfigurationService>> _loggerMock;
    private readonly string _testConfigPath;

    public ConfigurationServiceTests()
    {
        _loggerMock = new Mock<ILogger<ConfigurationService>>();
        _testConfigPath = Path.Combine(Path.GetTempPath(), "test-config.json");
    }

    [Fact]
    public async Task LoadAsync_ConfigFileExists_ReturnsConfiguration()
    {
        // Arrange
        var expectedConfig = new Configuration { /* ... */ };
        var json = JsonConvert.SerializeObject(expectedConfig);
        await File.WriteAllTextAsync(_testConfigPath, json);

        var service = new ConfigurationService(_loggerMock.Object, _testConfigPath);

        // Act
        var result = await service.LoadAsync();

        // Assert
        result.Should().BeEquivalentTo(expectedConfig);
    }

    [Fact]
    public async Task LoadAsync_ConfigFileMissing_ReturnsDefault()
    {
        // Arrange
        if (File.Exists(_testConfigPath))
            File.Delete(_testConfigPath);

        var service = new ConfigurationService(_loggerMock.Object, _testConfigPath);

        // Act
        var result = await service.LoadAsync();

        // Assert
        result.Should().NotBeNull();
        result.Should().BeOfType<Configuration>();
    }

    [Fact]
    public async Task SaveAsync_ValidConfiguration_SavesAndNotifies()
    {
        // Arrange
        var service = new ConfigurationService(_loggerMock.Object, _testConfigPath);
        var config = new Configuration { /* ... */ };
        Configuration notifiedConfig = null;

        service.ConfigurationChanged.Subscribe(c => notifiedConfig = c);

        // Act
        await service.SaveAsync(config);

        // Assert
        File.Exists(_testConfigPath).Should().BeTrue();
        notifiedConfig.Should().BeEquivalentTo(config);
    }
}
```

**2. Integration тесты**

```csharp
// Test/Integration/ShadowsocksControllerTests.cs
public class ShadowsocksControllerTests : IAsyncLifetime
{
    private ServiceProvider _serviceProvider;
    private IShadowsocksController _controller;

    public async Task InitializeAsync()
    {
        var services = new ServiceCollection();
        services.AddShadowsocksServices();
        services.AddLogging();

        _serviceProvider = services.BuildServiceProvider();
        _controller = _serviceProvider.GetRequiredService<IShadowsocksController>();
    }

    [Fact]
    public async Task StartAsync_ValidConfiguration_StartsSuccessfully()
    {
        // Act
        await _controller.StartAsync();

        // Assert
        _controller.CurrentConfiguration.Should().NotBeNull();
    }

    public async Task DisposeAsync()
    {
        await _serviceProvider.DisposeAsync();
    }
}
```

**3. ViewModel тесты**

```csharp
// Test/ViewModels/ServerViewModelTests.cs
public class ServerViewModelTests
{
    private readonly Mock<IServerService> _serverServiceMock;

    public ServerViewModelTests()
    {
        _serverServiceMock = new Mock<IServerService>();
    }

    [Fact]
    public async Task LoadServersCommand_WhenExecuted_LoadsServers()
    {
        // Arrange
        var expectedServers = new List<Server> { /* ... */ };
        _serverServiceMock
            .Setup(s => s.GetAllAsync(It.IsAny<CancellationToken>()))
            .ReturnsAsync(expectedServers);

        var viewModel = new ServerViewModel(_serverServiceMock.Object);

        // Act
        await viewModel.LoadServersCommand.Execute();

        // Assert
        viewModel.Servers.Should().BeEquivalentTo(expectedServers);
    }

    [Fact]
    public void SearchText_WhenChanged_FiltersServers()
    {
        // Arrange
        var servers = new List<Server>
        {
            new Server { Name = "Server 1" },
            new Server { Name = "Test Server" },
            new Server { Name = "Server 3" }
        };

        _serverServiceMock
            .Setup(s => s.Search(It.IsAny<string>()))
            .Returns<string>(text => servers.Where(s => s.Name.Contains(text)).ToList());

        var viewModel = new ServerViewModel(_serverServiceMock.Object);

        // Act
        viewModel.SearchText = "Test";

        // Assert - после debounce
        Thread.Sleep(400); // Wait for throttle
        viewModel.FilteredServers.Should().HaveCount(1);
    }
}
```

**4. Test Coverage Tools**

```xml
<!-- Add to test project -->
<PackageReference Include="coverlet.collector" Version="6.0.0" />
<PackageReference Include="ReportGenerator" Version="5.2.0" />
```

```bash
# Run tests with coverage
dotnet test /p:CollectCoverage=true /p:CoverletOutputFormat=cobertura

# Generate HTML report
reportgenerator -reports:coverage.cobertura.xml -targetdir:coverage-report
```

**Задачи**:
- [ ] Настроить тестовую инфраструктуру
- [ ] Написать unit тесты для всех сервисов
- [ ] Написать integration тесты
- [ ] Написать тесты для ViewModels
- [ ] Настроить coverage reporting
- [ ] Интегрировать в CI pipeline
- [ ] Достичь 60%+ покрытия

**Срок**: 4-6 недель (параллельно с другими задачами)

---

## 5. Порядок внедрения

### Месяц 1: Подготовка
- [ ] Настройка DI контейнера
- [ ] Создание интерфейсов
- [ ] Миграция критических сервисов на DI

### Месяц 2: Async миграция
- [ ] Миграция файловых операций
- [ ] Миграция сетевых операций
- [ ] Обновление ViewModels

### Месяц 3: Обработка ошибок
- [ ] Централизованная обработка исключений
- [ ] Кастомные исключения
- [ ] Graceful degradation

### Месяц 4: Тестирование
- [ ] Написание unit тестов
- [ ] Написание integration тестов
- [ ] Достижение целевого покрытия

---

## 6. Метрики успеха

### Качество
- [ ] 0 блокирующих операций в UI потоке
- [ ] Все публичные методы имеют async версии
- [ ] 100% сервисов используют DI
- [ ] Покрытие тестами > 60%

### Производительность
- [ ] Время запуска < 2 секунд
- [ ] UI отзывчивость < 100ms
- [ ] Использование памяти < 100MB

### Поддерживаемость
- [ ] Все исключения логируются
- [ ] Нет использования Thread.Sleep
- [ ] Code maintainability index > 75

---

**Версия**: 1.0
**Дата**: 2025-11-14
**Статус**: Ready for Implementation
