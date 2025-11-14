# План миграции на .NET 6+

## 1. Обзор

Миграция с .NET Framework 4.8 на .NET 6+ (или .NET 8 LTS) позволит использовать современные возможности платформы, улучшить производительность и обеспечить долгосрочную поддержку.

## 2. Текущее состояние

### Текущая платформа
- **.NET Framework 4.8** (последняя версия .NET Framework)
- **WPF** для UI
- **Целевая платформа**: Windows only
- **Формат проекта**: Old-style .csproj

### Зависимости
```xml
<!-- Основные зависимости из текущего проекта -->
ReactiveUI (✅ совместим с .NET 6+)
Newtonsoft.Json (✅ совместим)
NLog (✅ совместим)
WPF (✅ доступен в .NET 6+)
```

## 3. Целевая платформа

### Рекомендация: .NET 8 LTS

**Преимущества .NET 8**:
- Long Term Support до ноября 2026
- Лучшая производительность (на 20-30% быстрее)
- Современные возможности C# 12
- Улучшенные WPF возможности
- AOT compilation (Native AOT)
- Меньший размер runtime

**Альтернатива: .NET 6 LTS** (если нужна большая стабильность)
- LTS до ноября 2024
- Более консервативный выбор
- Широкая поддержка

## 4. План миграции

### Фаза 1: Подготовка (1-2 недели)

#### 1.1 Аудит совместимости

```bash
# Использовать .NET Upgrade Assistant
dotnet tool install -g upgrade-assistant
dotnet tool install -g try-convert

# Проверить совместимость
upgrade-assistant analyze shadowsocks-csharp.csproj
```

**Проверить**:
- [ ] Все NuGet пакеты совместимы с .NET 6+
- [ ] API, используемые в коде, доступны в .NET 6+
- [ ] P/Invoke вызовы работают корректно
- [ ] Encryption библиотеки совместимы

#### 1.2 Создать ветку для миграции

```bash
git checkout -b feature/dotnet8-migration
```

---

### Фаза 2: Конвертация проекта (1 неделя)

#### 2.1 Конвертация .csproj файла

**Было** (старый формат):
```xml
<?xml version="1.0" encoding="utf-8"?>
<Project ToolsVersion="12.0" DefaultTargets="Build" xmlns="...">
  <Import Project="..." />
  <PropertyGroup>
    <TargetFrameworkVersion>v4.8</TargetFrameworkVersion>
  </PropertyGroup>
  <ItemGroup>
    <Reference Include="..." />
  </ItemGroup>
  <!-- Сотни строк... -->
</Project>
```

**Стало** (новый SDK-style):
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>WinExe</OutputType>
    <TargetFramework>net8.0-windows</TargetFramework>
    <UseWPF>true</UseWPF>
    <ApplicationIcon>shadowsocks.ico</ApplicationIcon>
    <Platforms>x86;x64</Platforms>
    <RuntimeIdentifiers>win-x86;win-x64</RuntimeIdentifiers>
    <SelfContained>false</SelfContained>
    <Nullable>enable</Nullable>
    <LangVersion>latest</LangVersion>
  </PropertyGroup>

  <ItemGroup>
    <!-- NuGet пакеты автоматически из packages.config -->
    <PackageReference Include="ReactiveUI.WPF" Version="20.1.1" />
    <PackageReference Include="Newtonsoft.Json" Version="13.0.3" />
    <PackageReference Include="NLog" Version="5.2.8" />
    <!-- ... остальные пакеты -->
  </ItemGroup>

  <ItemGroup>
    <!-- Embedded resources -->
    <EmbeddedResource Include="Data\*.txt" />
    <EmbeddedResource Include="Data\*.js" />
  </ItemGroup>
</Project>
```

**Команды для автоматической конвертации**:
```bash
# Автоматическая конвертация
try-convert -w shadowsocks-csharp.csproj

# Или вручную создать новый файл
```

#### 2.2 Миграция packages.config → PackageReference

```bash
# Автоматическая миграция
dotnet migrate-packagereference shadowsocks-csharp.csproj
```

#### 2.3 Обновление зависимостей

```xml
<ItemGroup>
  <!-- Обновленные версии для .NET 8 -->
  <PackageReference Include="ReactiveUI.WPF" Version="20.1.1" />
  <PackageReference Include="ReactiveUI.Validation" Version="4.0.9" />
  <PackageReference Include="Newtonsoft.Json" Version="13.0.3" />
  <PackageReference Include="NLog" Version="5.2.8" />
  <PackageReference Include="NLog.Extensions.Logging" Version="5.3.5" />
  <PackageReference Include="Microsoft.Extensions.DependencyInjection" Version="8.0.0" />
  <PackageReference Include="Microsoft.Extensions.Hosting" Version="8.0.0" />
  <PackageReference Include="System.Reactive" Version="6.0.0" />

  <!-- WPF доступен из SDK, не нужны отдельные ссылки -->
</ItemGroup>
```

---

### Фаза 3: Исправление кода (2-3 недели)

#### 3.1 Изменения в API

**1. WebClient → HttpClient**

```csharp
// Старый код (.NET Framework)
using (var client = new WebClient())
{
    var data = client.DownloadString(url);
}

// Новый код (.NET 8)
using var client = new HttpClient();
var data = await client.GetStringAsync(url);
```

**2. ConfigurationManager**

```csharp
// Было
using System.Configuration;
var setting = ConfigurationManager.AppSettings["key"];

// Стало (использовать appsettings.json + IConfiguration)
using Microsoft.Extensions.Configuration;
var setting = _configuration["key"];
```

**3. BinaryFormatter (удален в .NET 8)**

```csharp
// Было
var formatter = new BinaryFormatter();
formatter.Serialize(stream, obj);

// Стало (использовать JSON или другие сериализаторы)
var json = JsonConvert.SerializeObject(obj);
await File.WriteAllTextAsync(path, json);
```

#### 3.2 Nullable Reference Types

Включить nullable reference types:

```xml
<PropertyGroup>
  <Nullable>enable</Nullable>
</PropertyGroup>
```

Исправить предупреждения:
```csharp
// Было
public string Name { get; set; }

// Стало
public string? Name { get; set; }  // Nullable
// или
public string Name { get; set; } = string.Empty;  // Non-null
```

#### 3.3 File-scoped namespaces (C# 10+)

```csharp
// Было
namespace Shadowsocks.Controller
{
    public class ShadowsocksController
    {
        // ...
    }
}

// Стало
namespace Shadowsocks.Controller;

public class ShadowsocksController
{
    // ...
}
```

#### 3.4 Global using directives

Создать `GlobalUsings.cs`:
```csharp
global using System;
global using System.Collections.Generic;
global using System.Linq;
global using System.Threading.Tasks;
global using System.Reactive;
global using System.Reactive.Linq;
global using ReactiveUI;
global using ReactiveUI.Fody.Helpers;
global using Newtonsoft.Json;
global using NLog;
```

---

### Фаза 4: Тестирование (2 недели)

#### 4.1 Unit тесты

```bash
# Запустить все тесты
dotnet test

# С покрытием
dotnet test /p:CollectCoverage=true
```

#### 4.2 Integration тесты

- [ ] Проверить запуск приложения
- [ ] Проверить загрузку конфигурации
- [ ] Проверить подключение к серверам
- [ ] Проверить PAC функциональность
- [ ] Проверить системный прокси

#### 4.3 Performance тесты

```csharp
using BenchmarkDotNet.Attributes;

[MemoryDiagnoser]
public class ConfigurationBenchmark
{
    [Benchmark]
    public async Task LoadConfiguration_NetFramework()
    {
        // Старая реализация
    }

    [Benchmark]
    public async Task LoadConfiguration_Net8()
    {
        // Новая реализация
    }
}
```

---

### Фаза 5: Packaging и Deployment (1 неделя)

#### 5.1 Self-contained deployment

```xml
<PropertyGroup>
  <SelfContained>true</SelfContained>
  <PublishSingleFile>true</PublishSingleFile>
  <IncludeNativeLibrariesForSelfExtract>true</IncludeNativeLibrariesForSelfExtract>
  <PublishReadyToRun>true</PublishReadyToRun>
</PropertyGroup>
```

Команды публикации:
```bash
# x86
dotnet publish -c Release -r win-x86 --self-contained

# x64
dotnet publish -c Release -r win-x64 --self-contained

# Single file
dotnet publish -c Release -r win-x64 --self-contained /p:PublishSingleFile=true
```

#### 5.2 Размер приложения

**Оптимизация размера**:
```xml
<PropertyGroup>
  <!-- Trim unused assemblies -->
  <PublishTrimmed>true</PublishTrimmed>
  <TrimMode>link</TrimMode>

  <!-- Compress assemblies -->
  <EnableCompressionInSingleFile>true</EnableCompressionInSingleFile>
</PropertyGroup>
```

#### 5.3 Native AOT (опционально)

Для максимальной производительности:
```xml
<PropertyGroup>
  <PublishAot>true</PublishAot>
</PropertyGroup>
```

**Ограничения AOT**:
- Reflection ограничен
- Некоторые библиотеки не совместимы
- Требует дополнительной настройки

---

## 5. Порядок внедрения

### Месяц 1: Подготовка
- Недели 1-2: Аудит совместимости, создание плана

### Месяц 2: Конвертация
- Недели 3-4: Конвертация проектов, обновление зависимостей

### Месяц 3: Исправление кода
- Недели 5-7: Исправление несовместимостей API
- Неделя 8: Включение nullable reference types

### Месяц 4: Тестирование
- Недели 9-10: Unit и integration тесты
- Неделя 11: Performance тесты
- Неделя 12: Beta тестирование

### Месяц 5: Release
- Неделя 13: Packaging и documentation
- Неделя 14: Release

---

## 6. Риски и митигация

| Риск | Вероятность | Влияние | Митигация |
|------|-------------|---------|-----------|
| Несовместимость API | Средняя | Высокое | Тщательный аудит, поэтапная миграция |
| Проблемы с P/Invoke | Низкая | Среднее | Тестирование на разных платформах |
| Увеличение размера | Высокая | Низкое | Trimming, compression |
| Breaking changes для пользователей | Низкая | Высокое | Миграция конфигураций, backward compatibility |

---

## 7. Преимущества после миграции

### Производительность
- 20-30% улучшение производительности
- Меньшее потребление памяти
- Быстрый старт приложения

### Разработка
- Современный C# (12)
- Лучшая tooling поддержка
- Hot reload в Visual Studio

### Безопасность
- Регулярные security обновления
- Современные криптографические алгоритмы
- Улучшенная защита от уязвимостей

### Кросс-платформенность (будущее)
- Потенциал для Linux/macOS поддержки
- Единая кодовая база

---

## 8. Метрики успеха

- [ ] Все тесты проходят успешно
- [ ] Производительность не хуже, чем на .NET Framework
- [ ] Размер приложения < 150MB (self-contained)
- [ ] Время запуска < 2 секунд
- [ ] 100% функциональная совместимость
- [ ] 0 критических багов

---

**Версия**: 1.0
**Дата**: 2025-11-14
**Статус**: Ready for Planning
**Приоритет**: Medium
