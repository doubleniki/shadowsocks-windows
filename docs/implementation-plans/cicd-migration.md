# План миграции на GitHub Actions и улучшения CI/CD

## 1. Обзор

Текущий проект требует модернизации CI/CD процессов для автоматизации сборки, тестирования и релизов.

## 2. Текущее состояние

### Проблемы
- Отсутствие автоматизированной сборки на разных платформах
- Нет автоматического запуска тестов
- Ручной процесс создания релизов
- Нет проверки безопасности и качества кода

### Текущая инфраструктура
- `.github/workflows/release.yml` - базовый workflow для релизов
- Сборка только для x86 платформы

## 3. Цели миграции

### Основные
- [x] Полная автоматизация процесса сборки
- [ ] Автоматический запуск тестов на каждом PR
- [ ] Статический анализ кода
- [ ] Автоматическое управление зависимостями
- [ ] Автоматизированный процесс релиза

### Дополнительные
- [ ] Кэширование зависимостей для ускорения сборки
- [ ] Матричная сборка (x86/x64, Debug/Release)
- [ ] Артефакты для каждой сборки
- [ ] Уведомления о статусе сборки

## 4. Детальный план реализации

### 4.1 Базовый CI Workflow

**Файл**: `.github/workflows/ci.yml`

```yaml
name: CI

on:
  push:
    branches: [ main, develop, claude/** ]
  pull_request:
    branches: [ main, develop ]

jobs:
  build:
    name: Build and Test
    runs-on: windows-latest
    strategy:
      matrix:
        platform: [x86, x64]
        configuration: [Debug, Release]

    steps:
      - uses: actions/checkout@v4

      - name: Setup MSBuild
        uses: microsoft/setup-msbuild@v1.1

      - name: Setup NuGet
        uses: nuget/setup-nuget@v1

      - name: Cache NuGet packages
        uses: actions/cache@v3
        with:
          path: ~/.nuget/packages
          key: ${{ runner.os }}-nuget-${{ hashFiles('**/packages.config') }}
          restore-keys: |
            ${{ runner.os }}-nuget-

      - name: Restore NuGet packages
        run: nuget restore shadowsocks-windows.sln

      - name: Build solution
        run: msbuild shadowsocks-windows.sln /p:Configuration=${{ matrix.configuration }} /p:Platform=${{ matrix.platform }} /m

      - name: Run tests
        run: |
          $vsTestPath = (& "C:\Program Files (x86)\Microsoft Visual Studio\Installer\vswhere.exe" -latest -products * -requires Microsoft.VisualStudio.Workload.ManagedDesktop -property installationPath)
          & "$vsTestPath\Common7\IDE\CommonExtensions\Microsoft\TestWindow\vstest.console.exe" test\bin\${{ matrix.platform }}\${{ matrix.configuration }}\ShadowsocksTest.dll

      - name: Upload build artifacts
        uses: actions/upload-artifact@v3
        if: matrix.configuration == 'Release'
        with:
          name: shadowsocks-${{ matrix.platform }}-${{ matrix.configuration }}
          path: shadowsocks-csharp/bin/${{ matrix.platform }}/${{ matrix.configuration }}/
```

**Задачи**:
- [ ] Создать файл workflow
- [ ] Настроить матричную сборку
- [ ] Добавить кэширование NuGet пакетов
- [ ] Настроить запуск тестов
- [ ] Проверить работоспособность

**Срок**: 2-3 дня

---

### 4.2 CodeQL Security Scanning

**Файл**: `.github/workflows/codeql.yml`

```yaml
name: "CodeQL"

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]
  schedule:
    - cron: '0 0 * * 1' # Weekly on Monday

jobs:
  analyze:
    name: Analyze
    runs-on: windows-latest
    permissions:
      actions: read
      contents: read
      security-events: write

    strategy:
      fail-fast: false
      matrix:
        language: [ 'csharp' ]

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Initialize CodeQL
        uses: github/codeql-action/init@v3
        with:
          languages: ${{ matrix.language }}
          queries: +security-and-quality

      - name: Setup MSBuild
        uses: microsoft/setup-msbuild@v1.1

      - name: Setup NuGet
        uses: nuget/setup-nuget@v1

      - name: Restore packages
        run: nuget restore shadowsocks-windows.sln

      - name: Build
        run: msbuild shadowsocks-windows.sln /p:Configuration=Release /p:Platform=x86

      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v3
        with:
          category: "/language:${{matrix.language}}"
```

**Задачи**:
- [ ] Создать workflow для CodeQL
- [ ] Настроить расписание сканирования
- [ ] Включить security-and-quality правила
- [ ] Настроить уведомления о найденных уязвимостях
- [ ] Создать процесс для исправления уязвимостей

**Срок**: 1-2 дня

---

### 4.3 Dependabot Configuration

**Файл**: `.github/dependabot.yml`

```yaml
version: 2
updates:
  # NuGet packages
  - package-ecosystem: "nuget"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
    open-pull-requests-limit: 10
    reviewers:
      - "maintainer-username"
    labels:
      - "dependencies"
      - "nuget"
    commit-message:
      prefix: "chore"
      include: "scope"

  # GitHub Actions
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
    labels:
      - "dependencies"
      - "github-actions"
```

**Задачи**:
- [ ] Создать конфигурацию Dependabot
- [ ] Настроить автоматическое обновление NuGet пакетов
- [ ] Настроить автоматическое обновление GitHub Actions
- [ ] Создать процесс review для Dependabot PRs
- [ ] Настроить автомерж для minor/patch обновлений

**Срок**: 1 день

---

### 4.4 Code Quality Checks

**Файл**: `.github/workflows/code-quality.yml`

```yaml
name: Code Quality

on:
  pull_request:
    branches: [ main, develop ]

jobs:
  code-quality:
    name: Code Quality Analysis
    runs-on: windows-latest

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Setup .NET
        uses: actions/setup-dotnet@v3
        with:
          dotnet-version: '6.0.x'

      - name: Install StyleCop.Analyzers
        run: |
          # Add StyleCop.Analyzers to projects
          # This will be done in code refactoring phase

      - name: Setup MSBuild
        uses: microsoft/setup-msbuild@v1.1

      - name: Restore packages
        run: nuget restore shadowsocks-windows.sln

      - name: Build with analyzers
        run: msbuild shadowsocks-windows.sln /p:Configuration=Release /p:Platform=x86 /p:TreatWarningsAsErrors=false

      - name: Run Roslynator
        run: |
          dotnet tool install -g Roslynator.DotNet.Cli
          roslynator analyze shadowsocks-windows.sln
```

**Опционально**: Интеграция с SonarCloud

```yaml
      - name: SonarCloud Scan
        uses: SonarSource/sonarcloud-github-action@master
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

**Задачи**:
- [ ] Создать workflow для проверки качества кода
- [ ] Добавить StyleCop.Analyzers в проекты
- [ ] Настроить Roslynator
- [ ] (Опционально) Настроить SonarCloud
- [ ] Определить baseline для code quality метрик

**Срок**: 3-4 дня

---

### 4.5 Автоматический Release Workflow

**Файл**: `.github/workflows/release.yml` (обновленная версия)

```yaml
name: Release

on:
  push:
    tags:
      - 'v*.*.*'

jobs:
  build:
    name: Build Release
    runs-on: windows-latest
    strategy:
      matrix:
        platform: [x86, x64]

    steps:
      - uses: actions/checkout@v4

      - name: Setup MSBuild
        uses: microsoft/setup-msbuild@v1.1

      - name: Setup NuGet
        uses: nuget/setup-nuget@v1

      - name: Restore packages
        run: nuget restore shadowsocks-windows.sln

      - name: Build
        run: msbuild shadowsocks-windows.sln /p:Configuration=Release /p:Platform=${{ matrix.platform }} /m

      - name: Create package
        run: |
          $version = "${{ github.ref }}".Substring(11)
          7z a -tzip Shadowsocks-$version-${{ matrix.platform }}.zip ./shadowsocks-csharp/bin/${{ matrix.platform }}/Release/*

      - name: Upload artifact
        uses: actions/upload-artifact@v3
        with:
          name: release-${{ matrix.platform }}
          path: Shadowsocks-*.zip

  release:
    name: Create Release
    needs: build
    runs-on: ubuntu-latest
    permissions:
      contents: write

    steps:
      - uses: actions/checkout@v4

      - name: Download artifacts
        uses: actions/download-artifact@v3

      - name: Generate changelog
        id: changelog
        uses: mikepenz/release-changelog-builder-action@v4
        with:
          configuration: ".github/changelog-config.json"
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Create Release
        uses: softprops/action-gh-release@v1
        with:
          files: |
            release-x86/*.zip
            release-x64/*.zip
          body: ${{ steps.changelog.outputs.changelog }}
          draft: false
          prerelease: false
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

**Конфигурация changelog**: `.github/changelog-config.json`

```json
{
  "categories": [
    {
      "title": "## 🚀 Features",
      "labels": ["feature", "enhancement"]
    },
    {
      "title": "## 🐛 Bug Fixes",
      "labels": ["bug", "fix"]
    },
    {
      "title": "## 🔧 Maintenance",
      "labels": ["maintenance", "chore"]
    },
    {
      "title": "## 📦 Dependencies",
      "labels": ["dependencies"]
    }
  ],
  "template": "${{CHANGELOG}}\n\n**Full Changelog**: ${{RELEASE_DIFF}}",
  "pr_template": "- ${{TITLE}} (#${{NUMBER}})"
}
```

**Задачи**:
- [ ] Обновить release workflow
- [ ] Добавить поддержку x64 платформы
- [ ] Настроить автоматическую генерацию changelog
- [ ] Создать конфигурацию для changelog
- [ ] Добавить подпись релизов (опционально)
- [ ] Создать документацию по процессу релиза

**Срок**: 2-3 дня

---

### 4.6 PR и Issue Templates

**PR Template**: `.github/pull_request_template.md`

```markdown
## Описание
<!-- Краткое описание изменений -->

## Тип изменений
- [ ] Bug fix (исправление бага)
- [ ] New feature (новая функциональность)
- [ ] Breaking change (изменение, нарушающее обратную совместимость)
- [ ] Documentation update (обновление документации)
- [ ] Refactoring (рефакторинг без изменения функциональности)

## Чеклист
- [ ] Код следует style guide проекта
- [ ] Добавлены/обновлены unit тесты
- [ ] Все тесты проходят успешно
- [ ] Обновлена документация (если необходимо)
- [ ] Нет warnings при сборке
- [ ] Проверено на x86 и x64 платформах

## Тестирование
<!-- Как были протестированы изменения? -->

## Скриншоты (если применимо)

## Связанные issues
Closes #(issue number)
```

**Bug Report Template**: `.github/ISSUE_TEMPLATE/bug_report.yml`

```yaml
name: Bug Report
description: Сообщить о баге
title: "[Bug]: "
labels: ["bug", "triage"]
body:
  - type: markdown
    attributes:
      value: |
        Спасибо за сообщение о баге!

  - type: textarea
    id: description
    attributes:
      label: Описание бага
      description: Четкое описание проблемы
    validations:
      required: true

  - type: textarea
    id: reproduction
    attributes:
      label: Шаги воспроизведения
      description: Как воспроизвести баг?
      placeholder: |
        1. Открыть...
        2. Кликнуть на...
        3. Увидеть ошибку...
    validations:
      required: true

  - type: textarea
    id: expected
    attributes:
      label: Ожидаемое поведение
      description: Что должно было произойти?
    validations:
      required: true

  - type: input
    id: version
    attributes:
      label: Версия
      description: Какая версия Shadowsocks-Windows?
    validations:
      required: true

  - type: dropdown
    id: platform
    attributes:
      label: Платформа
      options:
        - x86
        - x64
    validations:
      required: true

  - type: textarea
    id: logs
    attributes:
      label: Логи
      description: Прикрепите логи если есть
      render: shell
```

**Feature Request Template**: `.github/ISSUE_TEMPLATE/feature_request.yml`

```yaml
name: Feature Request
description: Предложить новую функцию
title: "[Feature]: "
labels: ["enhancement"]
body:
  - type: textarea
    id: problem
    attributes:
      label: Описание проблемы
      description: Какую проблему решает эта функция?
    validations:
      required: true

  - type: textarea
    id: solution
    attributes:
      label: Предлагаемое решение
      description: Как должна работать новая функция?
    validations:
      required: true

  - type: textarea
    id: alternatives
    attributes:
      label: Альтернативы
      description: Рассматривали ли вы другие решения?
```

**Задачи**:
- [ ] Создать PR template
- [ ] Создать Bug Report template
- [ ] Создать Feature Request template
- [ ] Создать template для вопросов
- [ ] Добавить CONTRIBUTING.md

**Срок**: 1 день

---

## 5. Порядок внедрения

### Неделя 1
- [ ] Создать базовый CI workflow
- [ ] Настроить матричную сборку
- [ ] Добавить кэширование

### Неделя 2
- [ ] Настроить CodeQL
- [ ] Настроить Dependabot
- [ ] Создать PR/Issue templates

### Неделя 3
- [ ] Добавить code quality checks
- [ ] Настроить автоматические тесты
- [ ] Интегрировать StyleCop

### Неделя 4
- [ ] Обновить release workflow
- [ ] Настроить автоматический changelog
- [ ] Провести тестовый релиз

### Неделя 5
- [ ] Документация CI/CD процессов
- [ ] Оптимизация скорости сборки
- [ ] Обучение команды

---

## 6. Метрики успеха

### Производительность CI
- [ ] Время сборки < 10 минут
- [ ] Успешность сборок > 95%
- [ ] Покрытие тестами > 60%

### Безопасность
- [ ] 0 критических уязвимостей
- [ ] Все зависимости актуальны (< 6 месяцев старые)
- [ ] Еженедельное сканирование CodeQL

### Процесс
- [ ] Автоматический релиз работает
- [ ] Changelog генерируется автоматически
- [ ] PR проверяются автоматически

---

## 7. Риски

| Риск | Вероятность | Влияние | Митигация |
|------|-------------|---------|-----------|
| Долгое время сборки | Средняя | Высокое | Кэширование, параллельные jobs |
| Ложные срабатывания CodeQL | Высокая | Среднее | Baseline, исключения |
| Конфликты Dependabot | Средняя | Среднее | Автотесты, review процесс |
| Сложность миграции | Низкая | Среднее | Постепенное внедрение |

---

## 8. Ресурсы

### Документация
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [CodeQL Documentation](https://codeql.github.com/docs/)
- [Dependabot Documentation](https://docs.github.com/en/code-security/dependabot)

### Инструменты
- GitHub Actions (бесплатно для публичных репозиториев)
- CodeQL (бесплатно для публичных репозиториев)
- Dependabot (встроено в GitHub)
- StyleCop.Analyzers (open-source)
- Roslynator (open-source)

---

## 9. Следующие шаги

1. Обзор и утверждение плана
2. Создание веток для разработки
3. Начало с базового CI workflow
4. Постепенное добавление остальных компонентов
5. Документирование процессов

---

**Версия**: 1.0
**Дата**: 2025-11-14
**Статус**: Ready for Implementation
