# План модернизации пользовательского интерфейса

## 1. Обзор

Обновление UI до современного дизайна улучшит пользовательский опыт и сделает приложение более привлекательным и интуитивным.

## 2. Текущее состояние

### Проблемы текущего UI
- Устаревший дизайн (Windows Forms style)
- Нет темной темы
- Ограниченная анимация
- Не адаптивный дизайн
- Отсутствие визуального feedback

### Сильные стороны
- ✅ Функциональность присутствует
- ✅ System tray integration
- ✅ Локализация поддерживается

## 3. Варианты дизайн-систем

### Вариант 1: Material Design (Рекомендуется) ⭐

**Библиотека**: [MaterialDesignInXamlToolkit](https://github.com/MaterialDesignInXAML/MaterialDesignInXamlToolkit)

**Преимущества**:
- Современный, узнаваемый дизайн
- Богатая библиотека компонентов
- Активная поддержка
- Хорошая документация
- Темная/светлая темы из коробки
- Анимации и transitions

**Недостатки**:
- Может быть "слишком Google-style"
- Больший размер библиотеки

**Пример**:
```xml
<materialDesign:Card Padding="32" Margin="16">
    <StackPanel>
        <TextBlock Style="{StaticResource MaterialDesignHeadline5TextBlock}">
            Server Configuration
        </TextBlock>
        <materialDesign:ColorZone Mode="Accent" Padding="8" Margin="0,16,0,0">
            <StackPanel Orientation="Horizontal">
                <materialDesign:PackIcon Kind="Server" />
                <TextBlock Text="Active Server" Margin="8,0,0,0"/>
            </StackPanel>
        </materialDesign:ColorZone>
    </StackPanel>
</materialDesign:Card>
```

---

### Вариант 2: Fluent Design (Windows 11 Style)

**Библиотека**: [ModernWpf](https://github.com/Kinnara/ModernWpf) или [WPF UI](https://github.com/lepoco/wpfui)

**Преимущества**:
- Нативный Windows 11 вид
- Acrylic эффекты
- Легковесная библиотека
- Хорошая интеграция с Windows

**Недостатки**:
- Меньше компонентов чем Material Design
- Зависимость от Windows версии для некоторых эффектов

**Пример (WPF UI)**:
```xml
<ui:FluentWindow>
    <ui:NavigationView>
        <ui:NavigationView.MenuItems>
            <ui:NavigationViewItem Content="Servers" Icon="{ui:SymbolIcon Server24}" />
            <ui:NavigationViewItem Content="Settings" Icon="{ui:SymbolIcon Settings24}" />
        </ui:NavigationView.MenuItems>
    </ui:NavigationView>
</ui:FluentWindow>
```

---

### Вариант 3: Custom Design

**Преимущества**:
- Полный контроль над дизайном
- Уникальный стиль
- Минимальные зависимости

**Недостатки**:
- Требует больше времени на разработку
- Нужен дизайнер
- Сложнее поддерживать

---

## 4. Рекомендуемое решение: Material Design

### 4.1 Установка

```xml
<PackageReference Include="MaterialDesignThemes" Version="5.0.0" />
<PackageReference Include="MaterialDesignColors" Version="3.0.0" />
```

### 4.2 Базовая настройка

**App.xaml**:
```xml
<Application x:Class="Shadowsocks.App"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:materialDesign="http://materialdesigninxaml.net/winfx/xaml/themes">
    <Application.Resources>
        <ResourceDictionary>
            <ResourceDictionary.MergedDictionaries>
                <!-- Material Design -->
                <materialDesign:BundledTheme BaseTheme="Dark"
                                            PrimaryColor="Blue"
                                            SecondaryColor="Lime" />
                <ResourceDictionary Source="pack://application:,,,/MaterialDesignThemes.Wpf;component/Themes/MaterialDesignTheme.Defaults.xaml" />

                <!-- Custom styles -->
                <ResourceDictionary Source="Styles/CustomStyles.xaml"/>
            </ResourceDictionary.MergedDictionaries>
        </ResourceDictionary>
    </Application.Resources>
</Application>
```

### 4.3 Компоненты UI

#### MainWindow с Navigation Drawer

```xml
<Window x:Class="Shadowsocks.Views.MainWindow"
        xmlns:materialDesign="http://materialdesigninxaml.net/winfx/xaml/themes"
        TextElement.Foreground="{DynamicResource MaterialDesignBody}"
        Background="{DynamicResource MaterialDesignPaper}"
        FontFamily="{materialDesign:MaterialDesignFont}"
        Title="Shadowsocks" Height="600" Width="1000">

    <materialDesign:DialogHost>
        <materialDesign:DrawerHost IsLeftDrawerOpen="{Binding IsMenuOpen}">
            <!-- Left Menu -->
            <materialDesign:DrawerHost.LeftDrawerContent>
                <DockPanel MinWidth="220">
                    <!-- Header -->
                    <StackPanel DockPanel.Dock="Top" Margin="16">
                        <Image Source="/Resources/logo.png" Height="48" />
                        <TextBlock Text="Shadowsocks"
                                   Style="{StaticResource MaterialDesignHeadline6TextBlock}"
                                   HorizontalAlignment="Center" Margin="0,8,0,0"/>
                    </StackPanel>

                    <!-- Menu Items -->
                    <ListBox>
                        <ListBoxItem>
                            <StackPanel Orientation="Horizontal">
                                <materialDesign:PackIcon Kind="Server" Margin="16,0"/>
                                <TextBlock Text="Servers" VerticalAlignment="Center"/>
                            </StackPanel>
                        </ListBoxItem>
                        <ListBoxItem>
                            <StackPanel Orientation="Horizontal">
                                <materialDesign:PackIcon Kind="Shield" Margin="16,0"/>
                                <TextBlock Text="PAC Rules" VerticalAlignment="Center"/>
                            </StackPanel>
                        </ListBoxItem>
                        <ListBoxItem>
                            <StackPanel Orientation="Horizontal">
                                <materialDesign:PackIcon Kind="Settings" Margin="16,0"/>
                                <TextBlock Text="Settings" VerticalAlignment="Center"/>
                            </StackPanel>
                        </ListBoxItem>
                    </ListBox>
                </DockPanel>
            </materialDesign:DrawerHost.LeftDrawerContent>

            <!-- Main Content -->
            <DockPanel>
                <!-- Top Bar -->
                <materialDesign:ColorZone DockPanel.Dock="Top"
                                         Mode="PrimaryMid"
                                         Padding="16">
                    <DockPanel>
                        <Button DockPanel.Dock="Left"
                                Style="{StaticResource MaterialDesignIconButton}"
                                Command="{Binding ToggleMenuCommand}">
                            <materialDesign:PackIcon Kind="Menu"/>
                        </Button>

                        <TextBlock Text="{Binding CurrentPageTitle}"
                                   VerticalAlignment="Center"
                                   Margin="16,0,0,0"
                                   Style="{StaticResource MaterialDesignHeadline6TextBlock}"/>

                        <StackPanel DockPanel.Dock="Right" Orientation="Horizontal">
                            <!-- Connection Status -->
                            <Border Background="{DynamicResource PrimaryHueMidBrush}"
                                    CornerRadius="12" Padding="12,4">
                                <StackPanel Orientation="Horizontal">
                                    <materialDesign:PackIcon Kind="Circle"
                                                            Foreground="{Binding StatusColor}"/>
                                    <TextBlock Text="{Binding ConnectionStatus}"
                                              Margin="8,0,0,0"/>
                                </StackPanel>
                            </Border>

                            <!-- Theme Toggle -->
                            <ToggleButton Style="{StaticResource MaterialDesignSwitchToggleButton}"
                                         IsChecked="{Binding IsDarkTheme}"
                                         Margin="16,0,0,0"
                                         ToolTip="Toggle Dark Mode">
                                <materialDesign:PackIcon Kind="Brightness6"/>
                            </ToggleButton>
                        </StackPanel>
                    </DockPanel>
                </materialDesign:ColorZone>

                <!-- Page Content -->
                <ContentControl Content="{Binding CurrentPage}" Margin="16"/>
            </DockPanel>
        </materialDesign:DrawerHost>
    </materialDesign:DialogHost>
</Window>
```

#### Server Card Component

```xml
<materialDesign:Card Margin="8" Padding="0" UniformCornerRadius="8">
    <Grid>
        <Grid.ColumnDefinitions>
            <ColumnDefinition Width="Auto"/>
            <ColumnDefinition Width="*"/>
            <ColumnDefinition Width="Auto"/>
        </Grid.ColumnDefinitions>

        <!-- Server Icon/Status -->
        <Border Grid.Column="0" Width="80" Background="{DynamicResource PrimaryHueLightBrush}">
            <StackPanel VerticalAlignment="Center" HorizontalAlignment="Center">
                <materialDesign:PackIcon Kind="Server" Width="32" Height="32"/>
                <materialDesign:PackIcon Kind="Check" Foreground="LimeGreen"
                                        Visibility="{Binding IsConnected, Converter={StaticResource BoolToVisConverter}}"/>
            </StackPanel>
        </Border>

        <!-- Server Info -->
        <StackPanel Grid.Column="1" Margin="16">
            <TextBlock Text="{Binding ServerName}"
                       Style="{StaticResource MaterialDesignHeadline6TextBlock}"/>
            <TextBlock Text="{Binding ServerAddress}"
                       Style="{StaticResource MaterialDesignBody2TextBlock}"
                       Foreground="{DynamicResource MaterialDesignBodyLight}"/>

            <!-- Stats -->
            <StackPanel Orientation="Horizontal" Margin="0,8,0,0">
                <materialDesign:Chip Content="{Binding Latency, StringFormat={}{0}ms}"
                                    IconBackground="Transparent" IconForeground="White">
                    <materialDesign:Chip.Icon>
                        <materialDesign:PackIcon Kind="Timer"/>
                    </materialDesign:Chip.Icon>
                </materialDesign:Chip>

                <materialDesign:Chip Content="{Binding Method}" Margin="8,0,0,0"/>
            </StackPanel>
        </StackPanel>

        <!-- Actions -->
        <StackPanel Grid.Column="2" Orientation="Horizontal" Margin="16">
            <Button Style="{StaticResource MaterialDesignIconButton}"
                    Command="{Binding ConnectCommand}"
                    ToolTip="Connect">
                <materialDesign:PackIcon Kind="Play"/>
            </Button>
            <Button Style="{StaticResource MaterialDesignIconButton}"
                    Command="{Binding EditCommand}"
                    ToolTip="Edit">
                <materialDesign:PackIcon Kind="Pencil"/>
            </Button>
            <Button Style="{StaticResource MaterialDesignIconButton}"
                    Command="{Binding DeleteCommand}"
                    ToolTip="Delete">
                <materialDesign:PackIcon Kind="Delete"/>
            </Button>
        </StackPanel>
    </Grid>
</materialDesign:Card>
```

#### Settings Page with Categories

```xml
<ScrollViewer>
    <StackPanel Margin="16">
        <!-- General Settings -->
        <TextBlock Text="General"
                   Style="{StaticResource MaterialDesignHeadline5TextBlock}"
                   Margin="0,0,0,16"/>

        <materialDesign:Card Padding="16" Margin="0,0,0,16">
            <StackPanel>
                <Grid>
                    <Grid.ColumnDefinitions>
                        <ColumnDefinition Width="*"/>
                        <ColumnDefinition Width="Auto"/>
                    </Grid.ColumnDefinitions>

                    <StackPanel Grid.Column="0">
                        <TextBlock Text="Start on system startup"/>
                        <TextBlock Text="Launch Shadowsocks when Windows starts"
                                   Style="{StaticResource MaterialDesignBody2TextBlock}"
                                   Foreground="{DynamicResource MaterialDesignBodyLight}"/>
                    </StackPanel>

                    <ToggleButton Grid.Column="1"
                                  Style="{StaticResource MaterialDesignSwitchToggleButton}"
                                  IsChecked="{Binding StartOnBoot}"/>
                </Grid>

                <Separator Margin="0,16"/>

                <Grid>
                    <Grid.ColumnDefinitions>
                        <ColumnDefinition Width="*"/>
                        <ColumnDefinition Width="Auto"/>
                    </Grid.ColumnDefinitions>

                    <StackPanel Grid.Column="0">
                        <TextBlock Text="System Proxy"/>
                        <TextBlock Text="Automatically set system proxy settings"
                                   Style="{StaticResource MaterialDesignBody2TextBlock}"
                                   Foreground="{DynamicResource MaterialDesignBodyLight}"/>
                    </StackPanel>

                    <ToggleButton Grid.Column="1"
                                  Style="{StaticResource MaterialDesignSwitchToggleButton}"
                                  IsChecked="{Binding EnableSystemProxy}"/>
                </Grid>
            </StackPanel>
        </materialDesign:Card>

        <!-- Network Settings -->
        <TextBlock Text="Network"
                   Style="{StaticResource MaterialDesignHeadline5TextBlock}"
                   Margin="0,16,0,16"/>

        <materialDesign:Card Padding="16">
            <StackPanel>
                <TextBlock Text="Local Port"/>
                <TextBox Text="{Binding LocalPort}"
                         materialDesign:HintAssist.Hint="Port number"
                         Style="{StaticResource MaterialDesignOutlinedTextBox}"
                         Margin="0,8,0,0"/>

                <TextBlock Text="Proxy Mode" Margin="0,16,0,8"/>
                <ComboBox ItemsSource="{Binding ProxyModes}"
                         SelectedItem="{Binding SelectedProxyMode}"
                         Style="{StaticResource MaterialDesignOutlinedComboBox}">
                    <ComboBox.ItemTemplate>
                        <DataTemplate>
                            <StackPanel Orientation="Horizontal">
                                <materialDesign:PackIcon Kind="{Binding Icon}" Margin="0,0,8,0"/>
                                <TextBlock Text="{Binding Name}"/>
                            </StackPanel>
                        </DataTemplate>
                    </ComboBox.ItemTemplate>
                </ComboBox>
            </StackPanel>
        </materialDesign:Card>
    </StackPanel>
</ScrollViewer>
```

---

## 5. Ключевые возможности

### 5.1 Темная/Светлая темы

```csharp
public class ThemeService : IThemeService
{
    private readonly PaletteHelper _paletteHelper = new();

    public void ToggleTheme()
    {
        var theme = _paletteHelper.GetTheme();
        var baseTheme = theme.GetBaseTheme() == BaseTheme.Dark
            ? BaseTheme.Light
            : BaseTheme.Dark;

        theme.SetBaseTheme(baseTheme);
        _paletteHelper.SetTheme(theme);
    }

    public void SetPrimaryColor(Color color)
    {
        var theme = _paletteHelper.GetTheme();
        theme.SetPrimaryColor(color);
        _paletteHelper.SetTheme(theme);
    }
}
```

### 5.2 Анимации и Transitions

```xml
<!-- Fade in animation -->
<materialDesign:Card>
    <materialDesign:Card.Triggers>
        <EventTrigger RoutedEvent="Loaded">
            <BeginStoryboard>
                <Storyboard>
                    <DoubleAnimation Storyboard.TargetProperty="Opacity"
                                   From="0" To="1" Duration="0:0:0.3"/>
                </Storyboard>
            </BeginStoryboard>
        </EventTrigger>
    </materialDesign:Card.Triggers>
</materialDesign:Card>

<!-- Transition between pages -->
<materialDesign:Transitioner SelectedIndex="{Binding CurrentPageIndex}">
    <materialDesign:TransitionerSlide>
        <local:ServersPage/>
    </materialDesign:TransitionerSlide>
    <materialDesign:TransitionerSlide>
        <local:SettingsPage/>
    </materialDesign:TransitionerSlide>
</materialDesign:Transitioner>
```

### 5.3 Диалоги

```csharp
// Confirmation dialog
var result = await DialogHost.Show(new ConfirmationDialog
{
    Message = "Are you sure you want to delete this server?"
}, "RootDialog");

if (result is bool confirmed && confirmed)
{
    await DeleteServerAsync(server);
}

// Progress dialog
await DialogHost.Show(new ProgressDialog
{
    Message = "Connecting to server..."
}, "RootDialog", async (sender, args) =>
{
    await ConnectToServerAsync(server);
    args.Session.Close();
});
```

### 5.4 Snackbar уведомления

```csharp
var messageQueue = new SnackbarMessageQueue(TimeSpan.FromSeconds(3));

messageQueue.Enqueue(
    "Server connected successfully",
    "UNDO",
    () => DisconnectFromServer());
```

---

## 6. План реализации

### Неделя 1-2: Подготовка
- [ ] Установить Material Design библиотеку
- [ ] Создать базовую тему
- [ ] Настроить цветовую схему
- [ ] Создать шаблоны компонентов

### Неделя 3-4: Main Window
- [ ] Создать новый MainWindow с Navigation Drawer
- [ ] Реализовать навигацию между страницами
- [ ] Добавить статус бар
- [ ] Добавить переключатель темы

### Неделя 5-6: Server Management
- [ ] Создать Server Card компонент
- [ ] Реализовать список серверов
- [ ] Добавить диалог редактирования сервера
- [ ] Добавить анимации

### Неделя 7-8: Settings и другие страницы
- [ ] Создать страницу настроек
- [ ] Создать страницу логов
- [ ] Создать about страницу
- [ ] Добавить help/documentation

### Неделя 9-10: Доработка и полировка
- [ ] Анимации и transitions
- [ ] Responsive design
- [ ] Accessibility improvements
- [ ] Локализация UI строк

---

## 7. Метрики успеха

- [ ] Современный дизайн соответствует Material Design guidelines
- [ ] Темная и светлая темы работают корректно
- [ ] Плавные анимации (60 FPS)
- [ ] Отзывчивый UI
- [ ] Положительные отзывы пользователей

---

**Версия**: 1.0
**Дата**: 2025-11-14
**Статус**: Ready for Implementation
**Приоритет**: Medium
