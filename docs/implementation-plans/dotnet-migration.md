# План миграции на .NET (Современная платформа)

## 1. Обзор

Миграция с .NET Framework 4.8 на современную .NET платформу (рекомендуется .NET 10 LTS) позволит использовать современные возможности платформы, улучшить производительность и обеспечить долгосрочную поддержку.

## 2. Текущее состояние

### Текущая платформа
- **.NET Framework 4.8** (последняя версия .NET Framework)
- **WPF** для UI
- **Целевая платформа**: Windows only
- **Формат проекта**: Old-style .csproj

### Зависимости
```xml
<!-- Основные зависимости из текущего проекта -->
ReactiveUI (✅ совместим с .NET 8+)
Newtonsoft.Json (✅ совместим)
NLog (✅ совместим)
WPF (✅ доступен в .NET 8+)
```

## 3. Целевая платформа

### Обзор доступных версий

**Текущая дата**: Ноябрь 2025
**Предполагаемый срок миграции**: 5 месяцев (до апреля 2026)

| Версия | Тип | Релиз | End of Support | Статус |
|--------|-----|-------|----------------|--------|
| .NET 6 | LTS | Nov 2021 | **Nov 2024** | ❌ EOL - не рекомендуется |
| .NET 8 | LTS | Nov 2023 | **Nov 2026** | ✅ Текущий LTS |
| .NET 9 | STS | Nov 2024 | **May 2026** | ⚠️ Short-term support |
| .NET 10 | LTS | Nov 2025 (ожидается) | **Nov 2028** | 🔮 Будущий LTS |

---

### Рекомендация: Выбор целевой платформы

#### Вариант A: .NET 8 LTS (Рекомендуется для немедленного старта) ⭐

**Преимущества:**
- ✅ Стабильная LTS версия с проверенной производительностью
- ✅ Широкая экосистема и поддержка сообщества
- ✅ Значительное улучшение производительности (20-30% быстрее .NET Framework)
- ✅ Современные возможности C# 12
- ✅ Улучшенные WPF возможности
- ✅ AOT compilation (Native AOT)
- ✅ Меньший размер runtime

**Недостатки:**
- ⚠️ Поддержка до **ноября 2026** (только ~1.5 года после завершения миграции)
- ⚠️ Потребуется планировать следующую миграцию на .NET 10 в 2026-2027

**Временная шкала:**

```text
Сегодня (Nov 2025)  →  Миграция завершена (Apr 2026)  →  EOL .NET 8 (Nov 2026)
       |                        |                              |
       └── 5 месяцев ──────────┘────── 7 месяцев поддержки ──┘

                          ↓ Рекомендуется планировать миграцию
                            на .NET 10 в середине 2026
```

**Рекомендация для этого варианта:**
- Начать миграцию немедленно
- Параллельно планировать переход на .NET 10 LTS (релиз Nov 2025)
- Оценить .NET 10 в Q2 2026, мигрировать в Q3-Q4 2026
- **Общая стоимость**: Две миграции за 1.5 года

---

#### Вариант B: Дождаться .NET 10 LTS (Рекомендуется для долгосрочной стабильности) 🎯

**Преимущества:**
- ✅ Поддержка до **ноября 2028** (2.5 года после завершения миграции)
- ✅ Одна миграция вместо двух (экономия ресурсов)
- ✅ Все улучшения .NET 8 + новые возможности .NET 10
- ✅ C# 13
- ✅ Более длительный период стабильности

**Недостатки:**
- ⚠️ Релиз ожидается в **ноябре 2025** (в текущем месяце или недавно выпущен)
- ⚠️ Начальные версии могут иметь меньше материалов и сообщества
- ⚠️ Возможны breaking changes, требующие дополнительного времени
- ⚠️ Небольшая задержка в получении преимуществ современной платформы (если релиз еще не произошел)

**Временная шкала:**

```text
Сегодня (Nov 2025)  →  .NET 10 релиз  →  Миграция завершена  →  EOL .NET 10
       |                 (Nov 2025)         (Apr-May 2026)         (Nov 2028)
       |                     |                    |                     |
       └── Ожидание (1 мес) ┘── Миграция 5 мес ─┘─── 2.5 года ────────┘
                                                     поддержки
```

**Рекомендация для этого варианта:**
- Проверить статус релиза .NET 10 LTS (ноябрь 2025)
- Если уже выпущен - начать оценку немедленно
- Начать миграцию в декабре 2025 - январе 2026
- Завершить к маю-июню 2026
- **Общая стоимость**: Одна миграция с долгим периодом стабильности

---

#### Вариант C: .NET 9 STS (НЕ рекомендуется)

**Характеристики:**
- Standard Term Support (18 месяцев)
- EOL: **май 2026** (через ~6 месяцев после завершения миграции)
- Новые возможности, но короткая поддержка

**Вывод:** ❌ Не рекомендуется из-за очень короткого окна поддержки после миграции.

---

#### Вариант D: .NET 6 LTS (Устаревший)

> **ВАЖНО**: .NET 6 LTS достиг **End of Support в ноябре 2024**. Версия больше не получает security updates и багфиксы.

**Историческая справка:**
- Релиз: Ноябрь 2021
- End of Support: Ноябрь 2024
- Статус: ❌ EOL (End of Life)

**Рекомендация:** ❌ НЕ использовать для новых проектов или миграций.

---

### Итоговая рекомендация

#### Для проекта Shadowsocks-Windows рекомендуется Вариант B: .NET 10 LTS

**Обоснование:**

1. **Экономия ресурсов:**
   - Одна миграция вместо двух экономит ~3-4 недели работы команды
   - Меньше риска регрессий и breaking changes
   - Упрощает планирование и коммуникацию

2. **Долгосрочная стабильность:**
   - 2.5 года поддержки vs 1.5 года (.NET 8)
   - Меньше срочных обновлений в будущем
   - Более предсказуемый maintenance cycle

3. **Технические преимущества:**
   - Все улучшения .NET 8 сохраняются
   - Дополнительные оптимизации .NET 10
   - C# 13 возможности

4. **Низкий риск ожидания:**
   - Минимальная или нулевая задержка (релиз ожидается в ноябре 2025)
   - .NET 10 RC/Preview уже доступны для тестирования совместимости
   - Microsoft имеет хороший track record с LTS релизами

**План действий:**
1. **Ноябрь 2025**: Проверка релиза .NET 10, тестирование совместимости
2. **Декабрь 2025**: Начало миграции
3. **Январь-апрель 2026**: Активная миграция
4. **Май 2026**: Релиз на .NET 10
5. **2026-2028**: Стабильная работа без необходимости миграции

**Альтернатива:** Если требуется срочный старт миграции (критические уязвимости, блокеры):
- Выбрать .NET 8
- Запланировать техническое окно для миграции на .NET 10 в Q3-Q4 2026

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
- [ ] Все NuGet пакеты совместимы с целевой версией .NET (8 или 10)
- [ ] API, используемые в коде, доступны в целевой версии
- [ ] P/Invoke вызовы работают корректно
- [ ] Encryption библиотеки совместимы

#### 1.2 Создать ветку для миграции

```bash
# Для .NET 10 LTS (рекомендуется)
git checkout -b feature/dotnet10-migration

# Или для .NET 8 LTS (если выбран вариант A)
# git checkout -b feature/dotnet8-migration
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
    <!-- Используйте net10.0-windows для .NET 10 LTS или net8.0-windows для .NET 8 -->
    <TargetFramework>net10.0-windows</TargetFramework>
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

##### 1. WebClient → HttpClient

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

##### 2. ConfigurationManager

```csharp
// Было
using System.Configuration;
var setting = ConfigurationManager.AppSettings["key"];

// Стало (использовать appsettings.json + IConfiguration)
using Microsoft.Extensions.Configuration;
var setting = _configuration["key"];
```

##### 3. BinaryFormatter (удален в .NET 8)

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
    [Benchmark(Baseline = true)]
    public async Task LoadConfiguration_NetFramework()
    {
        // Старая реализация (.NET Framework 4.8)
    }

    [Benchmark]
    public async Task LoadConfiguration_ModernNet()
    {
        // Новая реализация (.NET 8/10)
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
