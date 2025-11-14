# План реализации UI редактора PAC правил

## 1. Обзор

UI редактор PAC правил - это приоритетная фича #1, которая значительно улучшит пользовательский опыт при управлении правилами прокси. Текущий подход (ручное редактирование текстового файла) неудобен и подвержен ошибкам.

## 2. Текущее состояние

### Существующая реализация
- Файл: `shadowsocks-csharp/Data/user-rule.txt`
- Формат: AdBlock Plus синтаксис
- Редактирование: Только через текстовый редактор
- Валидация: Отсутствует
- Группировка: Через комментарии

### Проблемы
- Нет визуального редактора
- Сложно управлять большим количеством правил
- Легко сделать синтаксическую ошибку
- Нет проверки дубликатов
- Неудобно организовывать правила по категориям

## 3. Цели и требования

### Функциональные требования

#### Обязательные
- [x] Визуальный редактор с деревом групп
- [x] Добавление правил из буфера обмена
- [x] Автоматический парсинг URL
- [x] Валидация правил в реальном времени
- [x] Проверка дубликатов
- [x] Drag & Drop между группами
- [x] Поиск и фильтрация
- [x] Автоформатирование
- [x] Обратная совместимость с user-rule.txt

#### Желательные
- [ ] Импорт из других источников (GFWList, и т.д.)
- [ ] Экспорт в различные форматы
- [ ] История изменений (undo/redo)
- [ ] Bulk операции (массовое редактирование)
- [ ] Статистика использования правил
- [ ] Предложения похожих правил

### Нефункциональные требования
- Производительность: Обработка 10000+ правил
- UI отзывчивость: < 100ms
- Обратная совместимость: 100%
- Простота использования: Интуитивный интерфейс

## 4. Архитектура

### 4.1 Структура данных

```
Models/
├── PacRule/
│   ├── PACRule.cs              # Базовая модель правила
│   ├── PACRuleGroup.cs         # Группа правил
│   ├── PACRuleType.cs          # Enum типов правил
│   └── PACRuleValidation.cs    # Валидация
Services/
├── PACRuleService.cs           # Бизнес-логика
├── PACRuleParser.cs            # Парсер AdBlock Plus
└── PACRuleFormatter.cs         # Форматирование
ViewModels/
├── PACRuleEditorViewModel.cs   # Главный ViewModel
├── PACRuleGroupViewModel.cs    # ViewModel для группы
└── PACRuleViewModel.cs         # ViewModel для правила
Views/
└── PACRuleEditorView.xaml      # UI
```

### 4.2 Модели данных

#### PACRule.cs

```csharp
namespace Shadowsocks.Models.PacRule
{
    /// <summary>
    /// Represents a single PAC rule
    /// </summary>
    public class PACRule : ReactiveObject, IEquatable<PACRule>
    {
        /// <summary>
        /// Unique identifier for the rule
        /// </summary>
        public Guid Id { get; set; } = Guid.NewGuid();

        /// <summary>
        /// Rule pattern (e.g., "||example.com^")
        /// </summary>
        [Reactive]
        public string Pattern { get; set; }

        /// <summary>
        /// Rule type (Domain, Regex, Keyword, etc.)
        /// </summary>
        [Reactive]
        public PACRuleType Type { get; set; }

        /// <summary>
        /// Optional comment/description
        /// </summary>
        [Reactive]
        public string Comment { get; set; }

        /// <summary>
        /// Is rule enabled
        /// </summary>
        [Reactive]
        public bool IsEnabled { get; set; } = true;

        /// <summary>
        /// Group this rule belongs to
        /// </summary>
        [Reactive]
        public Guid? GroupId { get; set; }

        /// <summary>
        /// When the rule was created
        /// </summary>
        public DateTime CreatedAt { get; set; } = DateTime.UtcNow;

        /// <summary>
        /// When the rule was last modified
        /// </summary>
        [Reactive]
        public DateTime ModifiedAt { get; set; } = DateTime.UtcNow;

        /// <summary>
        /// Optional tags for categorization
        /// </summary>
        public List<string> Tags { get; set; } = new();

        /// <summary>
        /// Validation errors
        /// </summary>
        [Reactive]
        public string ValidationError { get; set; }

        /// <summary>
        /// Is this rule valid
        /// </summary>
        public bool IsValid => string.IsNullOrEmpty(ValidationError);

        public bool Equals(PACRule other)
        {
            if (other == null) return false;
            return Pattern?.Trim() == other.Pattern?.Trim();
        }

        public override int GetHashCode()
        {
            return Pattern?.Trim()?.GetHashCode() ?? 0;
        }

        public override string ToString()
        {
            return Pattern;
        }
    }

    /// <summary>
    /// Types of PAC rules based on AdBlock Plus syntax
    /// </summary>
    public enum PACRuleType
    {
        /// <summary>
        /// Domain rule: ||example.com^
        /// </summary>
        Domain,

        /// <summary>
        /// Subdomain rule: ||example.com
        /// </summary>
        Subdomain,

        /// <summary>
        /// Keyword rule: keyword
        /// </summary>
        Keyword,

        /// <summary>
        /// Regex rule: /regex/
        /// </summary>
        Regex,

        /// <summary>
        /// Exact match: |https://example.com
        /// </summary>
        Exact,

        /// <summary>
        /// Exception rule: @@rule
        /// </summary>
        Exception,

        /// <summary>
        /// Comment: ! comment
        /// </summary>
        Comment
    }
}
```

#### PACRuleGroup.cs

```csharp
namespace Shadowsocks.Models.PacRule
{
    /// <summary>
    /// Represents a group of PAC rules
    /// </summary>
    public class PACRuleGroup : ReactiveObject
    {
        /// <summary>
        /// Unique identifier
        /// </summary>
        public Guid Id { get; set; } = Guid.NewGuid();

        /// <summary>
        /// Group name
        /// </summary>
        [Reactive]
        public string Name { get; set; }

        /// <summary>
        /// Group description
        /// </summary>
        [Reactive]
        public string Description { get; set; }

        /// <summary>
        /// Group color for UI
        /// </summary>
        [Reactive]
        public string Color { get; set; } = "#2196F3";

        /// <summary>
        /// Group icon
        /// </summary>
        [Reactive]
        public string Icon { get; set; }

        /// <summary>
        /// Is group enabled (enables/disables all rules)
        /// </summary>
        [Reactive]
        public bool IsEnabled { get; set; } = true;

        /// <summary>
        /// Is group expanded in UI
        /// </summary>
        [Reactive]
        public bool IsExpanded { get; set; } = true;

        /// <summary>
        /// Display order
        /// </summary>
        [Reactive]
        public int Order { get; set; }

        /// <summary>
        /// Rules in this group
        /// </summary>
        public ObservableCollection<PACRule> Rules { get; set; } = new();

        /// <summary>
        /// Creation timestamp
        /// </summary>
        public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    }
}
```

---

### 4.3 Services

#### IPACRuleService.cs

```csharp
namespace Shadowsocks.Services
{
    public interface IPACRuleService
    {
        /// <summary>
        /// Load rules from file
        /// </summary>
        Task<PACRuleCollection> LoadRulesAsync(CancellationToken ct = default);

        /// <summary>
        /// Save rules to file
        /// </summary>
        Task SaveRulesAsync(PACRuleCollection rules, CancellationToken ct = default);

        /// <summary>
        /// Add a new rule
        /// </summary>
        Task<PACRule> AddRuleAsync(PACRule rule, Guid? groupId = null);

        /// <summary>
        /// Update existing rule
        /// </summary>
        Task UpdateRuleAsync(PACRule rule);

        /// <summary>
        /// Delete a rule
        /// </summary>
        Task DeleteRuleAsync(Guid ruleId);

        /// <summary>
        /// Add a new group
        /// </summary>
        Task<PACRuleGroup> AddGroupAsync(PACRuleGroup group);

        /// <summary>
        /// Move rule to different group
        /// </summary>
        Task MoveRuleToGroupAsync(Guid ruleId, Guid? targetGroupId);

        /// <summary>
        /// Parse URL/domain and create rule
        /// </summary>
        PACRule ParseFromUrl(string input);

        /// <summary>
        /// Check for duplicate rules
        /// </summary>
        Task<bool> IsDuplicateAsync(PACRule rule);

        /// <summary>
        /// Search rules
        /// </summary>
        IObservable<List<PACRule>> Search(string query);

        /// <summary>
        /// Observable for rule changes
        /// </summary>
        IObservable<PACRuleCollection> RulesChanged { get; }
    }

    public class PACRuleCollection
    {
        public List<PACRuleGroup> Groups { get; set; } = new();
        public List<PACRule> UngroupedRules { get; set; } = new();
    }
}
```

#### PACRuleParser.cs

```csharp
namespace Shadowsocks.Services
{
    /// <summary>
    /// Parser for AdBlock Plus syntax
    /// </summary>
    public class PACRuleParser : IPACRuleParser
    {
        private readonly ILogger<PACRuleParser> _logger;

        public PACRuleParser(ILogger<PACRuleParser> logger)
        {
            _logger = logger;
        }

        /// <summary>
        /// Parse user-rule.txt file
        /// </summary>
        public async Task<PACRuleCollection> ParseFileAsync(string filePath, CancellationToken ct = default)
        {
            var collection = new PACRuleCollection();
            PACRuleGroup currentGroup = null;

            var lines = await File.ReadAllLinesAsync(filePath, ct);

            foreach (var line in lines)
            {
                var trimmed = line.Trim();

                // Skip empty lines
                if (string.IsNullOrWhiteSpace(trimmed))
                    continue;

                // Group detection: ! [GroupName]
                if (trimmed.StartsWith("! [") && trimmed.EndsWith("]"))
                {
                    var groupName = trimmed.Substring(3, trimmed.Length - 4);
                    currentGroup = new PACRuleGroup
                    {
                        Name = groupName,
                        Order = collection.Groups.Count
                    };
                    collection.Groups.Add(currentGroup);
                    continue;
                }

                // Parse rule
                var rule = ParseRule(trimmed);
                if (rule != null)
                {
                    if (currentGroup != null)
                    {
                        rule.GroupId = currentGroup.Id;
                        currentGroup.Rules.Add(rule);
                    }
                    else
                    {
                        collection.UngroupedRules.Add(rule);
                    }
                }
            }

            return collection;
        }

        /// <summary>
        /// Parse single rule line
        /// </summary>
        public PACRule ParseRule(string line)
        {
            if (string.IsNullOrWhiteSpace(line))
                return null;

            var rule = new PACRule();
            var trimmed = line.Trim();

            // Comment
            if (trimmed.StartsWith("!"))
            {
                rule.Type = PACRuleType.Comment;
                rule.Comment = trimmed.Substring(1).Trim();
                rule.Pattern = trimmed;
                return rule;
            }

            // Exception rule: @@pattern
            if (trimmed.StartsWith("@@"))
            {
                rule.Type = PACRuleType.Exception;
                rule.Pattern = trimmed;
                return rule;
            }

            // Regex rule: /pattern/
            if (trimmed.StartsWith("/") && trimmed.EndsWith("/"))
            {
                rule.Type = PACRuleType.Regex;
                rule.Pattern = trimmed;
                return rule;
            }

            // Domain rule: ||example.com^
            if (trimmed.StartsWith("||") && trimmed.EndsWith("^"))
            {
                rule.Type = PACRuleType.Domain;
                rule.Pattern = trimmed;
                return rule;
            }

            // Subdomain rule: ||example.com
            if (trimmed.StartsWith("||"))
            {
                rule.Type = PACRuleType.Subdomain;
                rule.Pattern = trimmed;
                return rule;
            }

            // Exact match: |http://example.com
            if (trimmed.StartsWith("|"))
            {
                rule.Type = PACRuleType.Exact;
                rule.Pattern = trimmed;
                return rule;
            }

            // Keyword (default)
            rule.Type = PACRuleType.Keyword;
            rule.Pattern = trimmed;

            return rule;
        }

        /// <summary>
        /// Parse URL and create appropriate rule
        /// </summary>
        public PACRule ParseFromUrl(string input)
        {
            if (string.IsNullOrWhiteSpace(input))
                return null;

            var trimmed = input.Trim();

            // Already a rule pattern
            if (trimmed.StartsWith("||") || trimmed.StartsWith("@@") ||
                trimmed.StartsWith("/") || trimmed.StartsWith("|"))
            {
                return ParseRule(trimmed);
            }

            // Try to parse as URL
            if (Uri.TryCreate(trimmed, UriKind.Absolute, out var uri))
            {
                // Create domain rule from URL
                return new PACRule
                {
                    Pattern = $"||{uri.Host}^",
                    Type = PACRuleType.Domain,
                    Comment = $"Auto-generated from {uri}"
                };
            }

            // Try as domain
            if (IsValidDomain(trimmed))
            {
                return new PACRule
                {
                    Pattern = $"||{trimmed}^",
                    Type = PACRuleType.Domain
                };
            }

            // Fallback to keyword
            return new PACRule
            {
                Pattern = trimmed,
                Type = PACRuleType.Keyword
            };
        }

        private bool IsValidDomain(string domain)
        {
            // Basic domain validation
            return Regex.IsMatch(domain, @"^([a-zA-Z0-9]([a-zA-Z0-9\-]{0,61}[a-zA-Z0-9])?\.)+[a-zA-Z]{2,}$");
        }
    }
}
```

#### PACRuleFormatter.cs

```csharp
namespace Shadowsocks.Services
{
    /// <summary>
    /// Formats PAC rules for saving to file
    /// </summary>
    public class PACRuleFormatter : IPACRuleFormatter
    {
        /// <summary>
        /// Format rules collection to user-rule.txt format
        /// </summary>
        public async Task<string> FormatAsync(PACRuleCollection collection)
        {
            var sb = new StringBuilder();

            // Header
            sb.AppendLine("! Put user rules line by line in this file.");
            sb.AppendLine("! See https://adblockplus.org/en/filter-cheatsheet");
            sb.AppendLine();

            // Ungrouped rules first
            if (collection.UngroupedRules.Any())
            {
                sb.AppendLine("! [Ungrouped]");
                foreach (var rule in collection.UngroupedRules.Where(r => r.IsEnabled))
                {
                    sb.AppendLine(FormatRule(rule));
                }
                sb.AppendLine();
            }

            // Grouped rules
            foreach (var group in collection.Groups.OrderBy(g => g.Order))
            {
                // Group header
                sb.AppendLine($"! [{group.Name}]");
                if (!string.IsNullOrEmpty(group.Description))
                {
                    sb.AppendLine($"! {group.Description}");
                }

                // Rules in group
                var enabledRules = group.Rules.Where(r => r.IsEnabled && group.IsEnabled);
                foreach (var rule in enabledRules)
                {
                    var formatted = FormatRule(rule);
                    sb.AppendLine(formatted);
                }

                sb.AppendLine();
            }

            return sb.ToString();
        }

        private string FormatRule(PACRule rule)
        {
            var result = rule.Pattern;

            // Add inline comment if exists
            if (!string.IsNullOrEmpty(rule.Comment) && rule.Type != PACRuleType.Comment)
            {
                result += $" ! {rule.Comment}";
            }

            return result;
        }
    }
}
```

#### PACRuleValidation.cs

```csharp
namespace Shadowsocks.Services
{
    /// <summary>
    /// Validates PAC rules
    /// </summary>
    public class PACRuleValidator : IPACRuleValidator
    {
        public ValidationResult Validate(PACRule rule)
        {
            if (rule == null)
                return ValidationResult.Error("Rule cannot be null");

            if (string.IsNullOrWhiteSpace(rule.Pattern))
                return ValidationResult.Error("Pattern cannot be empty");

            // Validate based on type
            switch (rule.Type)
            {
                case PACRuleType.Domain:
                case PACRuleType.Subdomain:
                    return ValidateDomainRule(rule.Pattern);

                case PACRuleType.Regex:
                    return ValidateRegexRule(rule.Pattern);

                case PACRuleType.Exact:
                    return ValidateExactRule(rule.Pattern);

                default:
                    return ValidationResult.Success();
            }
        }

        private ValidationResult ValidateDomainRule(string pattern)
        {
            // Remove ||  and ^ markers
            var domain = pattern.TrimStart('|').TrimEnd('^');

            if (string.IsNullOrWhiteSpace(domain))
                return ValidationResult.Error("Domain cannot be empty");

            // Basic domain validation
            if (!Regex.IsMatch(domain, @"^([a-zA-Z0-9*]([a-zA-Z0-9\-*]{0,61}[a-zA-Z0-9*])?\.)*[a-zA-Z*]{2,}$"))
                return ValidationResult.Error("Invalid domain format");

            return ValidationResult.Success();
        }

        private ValidationResult ValidateRegexRule(string pattern)
        {
            try
            {
                var regex = pattern.Trim('/');
                _ = new Regex(regex);
                return ValidationResult.Success();
            }
            catch (ArgumentException ex)
            {
                return ValidationResult.Error($"Invalid regex: {ex.Message}");
            }
        }

        private ValidationResult ValidateExactRule(string pattern)
        {
            var url = pattern.TrimStart('|');

            if (!Uri.TryCreate(url, UriKind.Absolute, out _))
                return ValidationResult.Error("Invalid URL format");

            return ValidationResult.Success();
        }
    }

    public class ValidationResult
    {
        public bool IsValid { get; set; }
        public string ErrorMessage { get; set; }

        public static ValidationResult Success() => new() { IsValid = true };
        public static ValidationResult Error(string message) => new() { IsValid = false, ErrorMessage = message };
    }
}
```

---

### 4.4 ViewModels

#### PACRuleEditorViewModel.cs

```csharp
namespace Shadowsocks.ViewModels
{
    public class PACRuleEditorViewModel : ReactiveObject
    {
        private readonly IPACRuleService _ruleService;
        private readonly ILogger<PACRuleEditorViewModel> _logger;
        private readonly ObservableAsPropertyHelper<List<PACRuleGroupViewModel>> _filteredGroups;

        public PACRuleEditorViewModel(
            IPACRuleService ruleService,
            ILogger<PACRuleEditorViewModel> logger)
        {
            _ruleService = ruleService;
            _logger = logger;

            // Commands
            LoadRulesCommand = ReactiveCommand.CreateFromTask(LoadRulesAsync);
            SaveRulesCommand = ReactiveCommand.CreateFromTask(SaveRulesAsync);
            AddRuleCommand = ReactiveCommand.Create(AddNewRule);
            AddFromClipboardCommand = ReactiveCommand.CreateFromTask(AddFromClipboardAsync);
            DeleteRuleCommand = ReactiveCommand.CreateFromTask<PACRuleViewModel>(DeleteRuleAsync);
            AddGroupCommand = ReactiveCommand.Create(AddNewGroup);
            AutoFormatCommand = ReactiveCommand.CreateFromTask(AutoFormatAsync);
            ValidateAllCommand = ReactiveCommand.CreateFromTask(ValidateAllAsync);

            // Search/Filter
            _filteredGroups = this
                .WhenAnyValue(x => x.SearchText)
                .Throttle(TimeSpan.FromMilliseconds(300))
                .Select(FilterGroups)
                .ToProperty(this, x => x.FilteredGroups);

            // Error handling
            LoadRulesCommand.ThrownExceptions.Subscribe(ex =>
            {
                _logger.LogError(ex, "Failed to load rules");
                ErrorMessage = $"Failed to load rules: {ex.Message}";
            });

            SaveRulesCommand.ThrownExceptions.Subscribe(ex =>
            {
                _logger.LogError(ex, "Failed to save rules");
                ErrorMessage = $"Failed to save rules: {ex.Message}";
            });

            // Auto-load on initialization
            LoadRulesCommand.Execute().Subscribe();
        }

        // Properties
        public ObservableCollection<PACRuleGroupViewModel> Groups { get; } = new();

        [Reactive]
        public string SearchText { get; set; }

        [Reactive]
        public PACRuleViewModel SelectedRule { get; set; }

        [Reactive]
        public string ErrorMessage { get; set; }

        [Reactive]
        public bool IsBusy { get; set; }

        public List<PACRuleGroupViewModel> FilteredGroups => _filteredGroups.Value;

        // Commands
        public ReactiveCommand<Unit, Unit> LoadRulesCommand { get; }
        public ReactiveCommand<Unit, Unit> SaveRulesCommand { get; }
        public ReactiveCommand<Unit, Unit> AddRuleCommand { get; }
        public ReactiveCommand<Unit, Unit> AddFromClipboardCommand { get; }
        public ReactiveCommand<PACRuleViewModel, Unit> DeleteRuleCommand { get; }
        public ReactiveCommand<Unit, Unit> AddGroupCommand { get; }
        public ReactiveCommand<Unit, Unit> AutoFormatCommand { get; }
        public ReactiveCommand<Unit, Unit> ValidateAllCommand { get; }

        // Methods
        private async Task LoadRulesAsync(CancellationToken ct)
        {
            IsBusy = true;
            try
            {
                var collection = await _ruleService.LoadRulesAsync(ct);

                Groups.Clear();

                foreach (var group in collection.Groups)
                {
                    var groupVm = new PACRuleGroupViewModel(group, _ruleService);
                    Groups.Add(groupVm);
                }

                _logger.LogInformation("Loaded {Count} groups", Groups.Count);
            }
            finally
            {
                IsBusy = false;
            }
        }

        private async Task SaveRulesAsync(CancellationToken ct)
        {
            IsBusy = true;
            try
            {
                var collection = new PACRuleCollection
                {
                    Groups = Groups.Select(g => g.Model).ToList()
                };

                await _ruleService.SaveRulesAsync(collection, ct);

                _logger.LogInformation("Saved {Count} groups", Groups.Count);
                ErrorMessage = "Rules saved successfully";
            }
            finally
            {
                IsBusy = false;
            }
        }

        private void AddNewRule()
        {
            var newRule = new PACRule
            {
                Pattern = "||example.com^",
                Type = PACRuleType.Domain
            };

            var ruleVm = new PACRuleViewModel(newRule, _ruleService);
            SelectedRule = ruleVm;

            // Add to first group or create ungrouped
            if (Groups.Any())
            {
                Groups.First().Rules.Add(ruleVm);
            }
        }

        private async Task AddFromClipboardAsync(CancellationToken ct)
        {
            try
            {
                var clipboardText = await Application.Current.Dispatcher.InvokeAsync(() =>
                    Clipboard.GetText());

                if (string.IsNullOrWhiteSpace(clipboardText))
                {
                    ErrorMessage = "Clipboard is empty";
                    return;
                }

                // Parse multiple lines
                var lines = clipboardText.Split('\n', StringSplitOptions.RemoveEmptyEntries);
                var addedCount = 0;

                foreach (var line in lines)
                {
                    var rule = _ruleService.ParseFromUrl(line.Trim());
                    if (rule != null)
                    {
                        // Check for duplicates
                        if (!await _ruleService.IsDuplicateAsync(rule))
                        {
                            var ruleVm = new PACRuleViewModel(rule, _ruleService);
                            Groups.First().Rules.Add(ruleVm);
                            addedCount++;
                        }
                    }
                }

                ErrorMessage = $"Added {addedCount} rules from clipboard";
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Failed to add from clipboard");
                ErrorMessage = $"Failed to add from clipboard: {ex.Message}";
            }
        }

        private async Task DeleteRuleAsync(PACRuleViewModel rule, CancellationToken ct)
        {
            await _ruleService.DeleteRuleAsync(rule.Model.Id);

            foreach (var group in Groups)
            {
                if (group.Rules.Contains(rule))
                {
                    group.Rules.Remove(rule);
                    break;
                }
            }
        }

        private void AddNewGroup()
        {
            var newGroup = new PACRuleGroup
            {
                Name = "New Group",
                Order = Groups.Count
            };

            var groupVm = new PACRuleGroupViewModel(newGroup, _ruleService);
            Groups.Add(groupVm);
        }

        private async Task AutoFormatAsync(CancellationToken ct)
        {
            // Implement auto-formatting logic
            // - Remove empty rules
            // - Sort rules alphabetically
            // - Remove duplicates
            // - Optimize patterns

            foreach (var group in Groups)
            {
                group.SortRules();
                await group.RemoveDuplicatesAsync();
            }

            ErrorMessage = "Auto-formatting completed";
        }

        private async Task ValidateAllAsync(CancellationToken ct)
        {
            var invalidCount = 0;

            foreach (var group in Groups)
            {
                foreach (var rule in group.Rules)
                {
                    await rule.ValidateAsync();
                    if (!rule.IsValid)
                        invalidCount++;
                }
            }

            ErrorMessage = invalidCount == 0
                ? "All rules are valid"
                : $"Found {invalidCount} invalid rules";
        }

        private List<PACRuleGroupViewModel> FilterGroups(string searchText)
        {
            if (string.IsNullOrWhiteSpace(searchText))
                return Groups.ToList();

            return Groups
                .Where(g => g.ContainsSearchText(searchText))
                .ToList();
        }
    }
}
```

---

### 4.5 Views (XAML)

#### PACRuleEditorView.xaml

```xml
<Window x:Class="Shadowsocks.Views.PACRuleEditorView"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
        xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
        xmlns:vm="clr-namespace:Shadowsocks.ViewModels"
        mc:Ignorable="d"
        Title="PAC Rules Editor" Height="600" Width="1000"
        d:DataContext="{d:DesignInstance Type=vm:PACRuleEditorViewModel}">

    <Grid>
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/> <!-- Toolbar -->
            <RowDefinition Height="*"/>    <!-- Main content -->
            <RowDefinition Height="Auto"/> <!-- Status bar -->
        </Grid.RowDefinitions>

        <!-- Toolbar -->
        <ToolBar Grid.Row="0" Padding="5">
            <Button Content="Load" Command="{Binding LoadRulesCommand}"/>
            <Button Content="Save" Command="{Binding SaveRulesCommand}"/>
            <Separator/>
            <Button Content="Add Rule" Command="{Binding AddRuleCommand}"/>
            <Button Content="Add from Clipboard" Command="{Binding AddFromClipboardCommand}"/>
            <Button Content="Add Group" Command="{Binding AddGroupCommand}"/>
            <Separator/>
            <Button Content="Auto-format" Command="{Binding AutoFormatCommand}"/>
            <Button Content="Validate All" Command="{Binding ValidateAllCommand}"/>
            <Separator/>
            <TextBox Width="200" Text="{Binding SearchText, UpdateSourceTrigger=PropertyChanged}"
                     VerticalContentAlignment="Center"/>
            <TextBlock Text="🔍" Margin="5,0" VerticalAlignment="Center"/>
        </ToolBar>

        <!-- Main Content: 3-panel layout -->
        <Grid Grid.Row="1">
            <Grid.ColumnDefinitions>
                <ColumnDefinition Width="250"/>    <!-- Groups panel -->
                <ColumnDefinition Width="Auto"/>   <!-- Splitter -->
                <ColumnDefinition Width="*"/>      <!-- Rules panel -->
                <ColumnDefinition Width="Auto"/>   <!-- Splitter -->
                <ColumnDefinition Width="300"/>    <!-- Details panel -->
            </Grid.ColumnDefinitions>

            <!-- LEFT PANEL: Groups -->
            <Border Grid.Column="0" BorderBrush="Gray" BorderThickness="0,0,1,0">
                <DockPanel>
                    <TextBlock DockPanel.Dock="Top" Text="Rule Groups"
                               FontWeight="Bold" Padding="10" Background="#F5F5F5"/>

                    <TreeView ItemsSource="{Binding FilteredGroups}"
                              BorderThickness="0">
                        <TreeView.Resources>
                            <HierarchicalDataTemplate DataType="{x:Type vm:PACRuleGroupViewModel}"
                                                      ItemsSource="{Binding Rules}">
                                <StackPanel Orientation="Horizontal">
                                    <CheckBox IsChecked="{Binding IsEnabled}" Margin="0,0,5,0"/>
                                    <Ellipse Width="10" Height="10" Fill="{Binding Color}" Margin="0,0,5,0"/>
                                    <TextBlock Text="{Binding Name}"/>
                                    <TextBlock Text="{Binding Rules.Count, StringFormat=' ({0})'}"
                                               Foreground="Gray" Margin="5,0,0,0"/>
                                </StackPanel>
                            </HierarchicalDataTemplate>

                            <DataTemplate DataType="{x:Type vm:PACRuleViewModel}">
                                <StackPanel Orientation="Horizontal">
                                    <CheckBox IsChecked="{Binding IsEnabled}" Margin="0,0,5,0"/>
                                    <TextBlock Text="{Binding Pattern}"/>
                                    <TextBlock Text="⚠" Foreground="Red" Margin="5,0,0,0"
                                               Visibility="{Binding IsValid, Converter={StaticResource InverseBoolToVisConverter}}"/>
                                </StackPanel>
                            </DataTemplate>
                        </TreeView.Resources>
                    </TreeView>
                </DockPanel>
            </Border>

            <GridSplitter Grid.Column="1" Width="5" HorizontalAlignment="Stretch"/>

            <!-- CENTER PANEL: Rules List -->
            <Border Grid.Column="2" Padding="10">
                <DockPanel>
                    <TextBlock DockPanel.Dock="Top" Text="Rules"
                               FontWeight="Bold" Margin="0,0,0,10"/>

                    <DataGrid ItemsSource="{Binding SelectedGroup.Rules}"
                              SelectedItem="{Binding SelectedRule}"
                              AutoGenerateColumns="False"
                              CanUserAddRows="False"
                              GridLinesVisibility="None"
                              HeadersVisibility="Column">
                        <DataGrid.Columns>
                            <DataGridCheckBoxColumn Header="✓" Binding="{Binding IsEnabled}" Width="30"/>
                            <DataGridTextColumn Header="Pattern" Binding="{Binding Pattern}" Width="*"/>
                            <DataGridTextColumn Header="Type" Binding="{Binding Type}" Width="100"/>
                            <DataGridTextColumn Header="Comment" Binding="{Binding Comment}" Width="150"/>
                            <DataGridTemplateColumn Header="Status" Width="50">
                                <DataGridTemplateColumn.CellTemplate>
                                    <DataTemplate>
                                        <TextBlock Text="{Binding IsValid, Converter={StaticResource BoolToStatusConverter}}"
                                                   Foreground="{Binding IsValid, Converter={StaticResource BoolToColorConverter}}"/>
                                    </DataTemplate>
                                </DataGridTemplateColumn.CellTemplate>
                            </DataGridTemplateColumn>
                        </DataGrid.Columns>
                    </DataGrid>
                </DockPanel>
            </Border>

            <GridSplitter Grid.Column="3" Width="5" HorizontalAlignment="Stretch"/>

            <!-- RIGHT PANEL: Rule Details -->
            <Border Grid.Column="4" Padding="10" Background="#FAFAFA">
                <DockPanel DataContext="{Binding SelectedRule}">
                    <TextBlock DockPanel.Dock="Top" Text="Rule Details"
                               FontWeight="Bold" Margin="0,0,0,20"/>

                    <StackPanel Spacing="10">
                        <TextBlock Text="Pattern:" FontWeight="SemiBold"/>
                        <TextBox Text="{Binding Pattern, UpdateSourceTrigger=PropertyChanged}"/>

                        <TextBlock Text="Type:" FontWeight="SemiBold" Margin="0,10,0,0"/>
                        <ComboBox ItemsSource="{Binding Source={StaticResource PACRuleTypes}}"
                                  SelectedItem="{Binding Type}"/>

                        <TextBlock Text="Comment:" FontWeight="SemiBold" Margin="0,10,0,0"/>
                        <TextBox Text="{Binding Comment, UpdateSourceTrigger=PropertyChanged}"
                                 TextWrapping="Wrap" AcceptsReturn="True" Height="60"/>

                        <CheckBox Content="Enabled" IsChecked="{Binding IsEnabled}" Margin="0,10,0,0"/>

                        <TextBlock Text="Tags:" FontWeight="SemiBold" Margin="0,10,0,0"/>
                        <TextBox Text="{Binding TagsString, UpdateSourceTrigger=PropertyChanged}"/>

                        <Border Background="#FFF3CD" Padding="10" Margin="0,20,0,0"
                                Visibility="{Binding ValidationError, Converter={StaticResource NullToVisConverter}}">
                            <StackPanel>
                                <TextBlock Text="⚠ Validation Error" FontWeight="Bold" Foreground="#856404"/>
                                <TextBlock Text="{Binding ValidationError}" Foreground="#856404" TextWrapping="Wrap"/>
                            </StackPanel>
                        </Border>

                        <StackPanel Orientation="Horizontal" Margin="0,20,0,0">
                            <TextBlock Text="Created:" Margin="0,0,5,0"/>
                            <TextBlock Text="{Binding CreatedAt, StringFormat='{}{0:yyyy-MM-dd HH:mm}'}"/>
                        </StackPanel>
                    </StackPanel>
                </DockPanel>
            </Border>
        </Grid>

        <!-- Status Bar -->
        <StatusBar Grid.Row="2">
            <StatusBarItem>
                <TextBlock Text="{Binding ErrorMessage}"/>
            </StatusBarItem>
            <StatusBarItem HorizontalAlignment="Right">
                <StackPanel Orientation="Horizontal">
                    <TextBlock Text="Total Rules: "/>
                    <TextBlock Text="{Binding TotalRulesCount}"/>
                    <Separator/>
                    <ProgressBar Width="100" Height="15" Visibility="{Binding IsBusy, Converter={StaticResource BoolToVisConverter}}"
                                 IsIndeterminate="True"/>
                </StackPanel>
            </StatusBarItem>
        </StatusBar>
    </Grid>
</Window>
```

---

## 5. План реализации

### Неделя 1-2: Модели и парсер
- [ ] Создать PACRule и PACRuleGroup модели
- [ ] Реализовать PACRuleParser
- [ ] Реализовать PACRuleFormatter
- [ ] Написать unit тесты для парсера
- [ ] Тестирование с реальным user-rule.txt

### Неделя 3-4: Сервисный слой
- [ ] Создать IPACRuleService интерфейс
- [ ] Реализовать PACRuleService
- [ ] Реализовать валидацию правил
- [ ] Добавить проверку дубликатов
- [ ] Написать интеграционные тесты

### Неделя 5-6: ViewModels
- [ ] Создать PACRuleEditorViewModel
- [ ] Создать PACRuleGroupViewModel
- [ ] Создать PACRuleViewModel
- [ ] Реализовать команды и логику
- [ ] Настроить ReactiveUI биндинги

### Неделя 7-8: UI
- [ ] Создать базовую разметку XAML
- [ ] Реализовать 3-панельный layout
- [ ] Добавить TreeView для групп
- [ ] Добавить DataGrid для правил
- [ ] Реализовать панель деталей

### Неделя 9: Drag & Drop
- [ ] Реализовать Drag & Drop между группами
- [ ] Добавить визуальную обратную связь
- [ ] Тестирование перемещения правил

### Неделя 10: Доп. функции
- [ ] Реализовать поиск и фильтрацию
- [ ] Добавить автоформатирование
- [ ] Добавить массовые операции
- [ ] Реализовать импорт/экспорт

### Неделя 11-12: Тестирование и доработка
- [ ] End-to-end тестирование
- [ ] Исправление багов
- [ ] Оптимизация производительности
- [ ] Документация
- [ ] Подготовка к релизу

---

## 6. Метрики успеха

- [ ] Может загрузить и отобразить 10000+ правил без лагов
- [ ] Поиск работает < 100ms
- [ ] Сохранение файла < 500ms
- [ ] Валидация в реальном времени < 50ms
- [ ] 100% обратная совместимость с user-rule.txt
- [ ] Покрытие тестами > 70%

---

**Версия**: 1.0
**Дата**: 2025-11-14
**Статус**: Ready for Implementation
**Приоритет**: #1 (Critical)
