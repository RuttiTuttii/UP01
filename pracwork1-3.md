Отличная задача. Я спроектировал архитектуру так, чтобы все три практические работы гармонично жили в одном приложении.
Мы будем использовать паттерн **MVVM** (через CommunityToolkit.Mvvm). Главное окно будет содержать TabControl (вкладки), где каждая вкладка — это отдельная практическая работа. Внутри вкладки ПР 21 мы реализуем вложенную навигацию (ContentControl + NavigationService).
### 📁 Полная структура проекта (HelloApp)
```text
HelloApp/
├── HelloApp.csproj
├── App.axaml                 # Настройки приложения и стили
├── App.axaml.cs              
├── ViewLocator.cs            # ПР 21: Локатор для автоматического поиска View по ViewModel
├── Program.cs                # Точка входа (стандартная, не меняем)
├── Models/
│   └── UserCredentials.cs    # ПР 20: Модель данных с валидацией
├── Services/
│   └── NavigationService.cs  # ПР 21: Сервис навигации
├── ViewModels/
│   ├── ViewModelBase.cs      # Базовый класс
│   ├── MainWindowViewModel.cs# Главная VM (содержит VM для вкладок)
│   ├── HomeViewModel.cs      # ПР 19: Логика первой вкладки
│   ├── ControlsViewModel.cs  # ПР 20: Логика второй вкладки
│   ├── NavDemoViewModel.cs   # ПР 21: Логика третьей вкладки (управляет навигацией)
│   ├── AuthorizationViewModel.cs 
│   └── RegistrationViewModel.cs
└── Views/
    ├── MainWindow.axaml      # Главное окно с TabControl
    ├── MainWindow.axaml.cs
    ├── HomeView.axaml        # ПР 19: Верстка базовых элементов и контейнеров
    ├── HomeView.axaml.cs
    ├── ControlsView.axaml    # ПР 20: Верстка элементов ввода и Slider
    ├── ControlsView.axaml.cs
    ├── NavDemoView.axaml     # ПР 21: Контейнер ContentControl для навигации
    ├── NavDemoView.axaml.cs
    ├── AuthorizationView.axaml
    ├── AuthorizationView.axaml.cs
    ├── RegistrationView.axaml
    └── RegistrationView.axaml.cs

```
### 1. Ядро приложения (Настройки и База)
**ViewLocator.cs** (Обязателен для работы ContentControl и привязки View к ViewModel)
```csharp
using Avalonia.Controls;
using Avalonia.Controls.Templates;
using HelloApp.ViewModels;
using System;

namespace HelloApp;

public class ViewLocator : IDataTemplate
{
    public Control? Build(object? data)
    {
        if (data is null) return null;
        var name = data.GetType().FullName!.Replace("ViewModel", "View", StringComparison.Ordinal);
        var type = Type.GetType(name);
        return type != null ? (Control)Activator.CreateInstance(type)! : new TextBlock { Text = "Not Found: " + name };
    }
    public bool Match(object? data) => data is ViewModelBase;
}

```
**ViewModels/ViewModelBase.cs**
```csharp
using CommunityToolkit.Mvvm.ComponentModel;

namespace HelloApp.ViewModels;

public class ViewModelBase : ObservableObject { }

```
**Services/NavigationService.cs**
```csharp
using System;
using System.Collections.Generic;
using HelloApp.ViewModels;

namespace HelloApp.Services;

public class NavigationService
{
    private readonly Stack<ViewModelBase> _history = new();
    private readonly Action<ViewModelBase> _onNavigate;

    public NavigationService(Action<ViewModelBase> onNavigate)
    {
        _onNavigate = onNavigate;
    }

    public void NavigateTo<T>(T viewModel, Action<T>? action = null) where T : ViewModelBase
    {
        action?.Invoke(viewModel);
        _history.Push(viewModel);
        _onNavigate(viewModel);
    }

    public void GoBack()
    {
        if (_history.Count > 1)
        {
            _history.Pop(); // Удаляем текущую
            _onNavigate(_history.Peek()); // Показываем предыдущую
        }
    }
}

```
### 2. Главное окно (Вкладки)
**ViewModels/MainWindowViewModel.cs**
```csharp
namespace HelloApp.ViewModels;

public partial class MainWindowViewModel : ViewModelBase
{
    public HomeViewModel HomeVM { get; } = new();
    public ControlsViewModel ControlsVM { get; } = new();
    public NavDemoViewModel NavDemoVM { get; } = new();
}

```
**Views/MainWindow.axaml**
```xml
<Window xmlns="https://github.com/avaloniaui"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:vm="using:HelloApp.ViewModels"
        xmlns:views="using:HelloApp.Views"
        x:Class="HelloApp.Views.MainWindow"
        x:DataType="vm:MainWindowViewModel"
        Icon="/Assets/avalonia-logo.ico"
        Title="HelloApp - 3 в 1" Width="800" Height="600">

    <TabControl>
        <TabItem Header="ПР 19: Интерфейс и Контейнеры">
            <views:HomeView DataContext="{Binding HomeVM}"/>
        </TabItem>
        
        <TabItem Header="ПР 20: Элементы ввода">
            <views:ControlsView DataContext="{Binding ControlsVM}"/>
        </TabItem>
        
        <TabItem Header="ПР 21: Навигация">
            <views:NavDemoView DataContext="{Binding NavDemoVM}"/>
        </TabItem>
    </TabControl>
</Window>

```
### 3. ПР 19: Интерфейс и контейнеры
**ViewModels/HomeViewModel.cs**
```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;

namespace HelloApp.ViewModels;

public partial class HomeViewModel : ViewModelBase
{
    [ObservableProperty]
    private string _text = "Hello Avalonia!";

    [RelayCommand]
    public void ChangeText()
    {
        Text = "Новый текст";
    }
}

```
**Views/HomeView.axaml**
```xml
<UserControl xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:vm="using:HelloApp.ViewModels"
             x:Class="HelloApp.Views.HomeView"
             x:DataType="vm:HomeViewModel">
    <ScrollViewer>
        <StackPanel Spacing="20" Margin="20">
            <StackPanel HorizontalAlignment="Center" Spacing="10">
                <TextBlock Text="{Binding Text}" Foreground="Green" FontSize="40" HorizontalAlignment="Center"/>
                <Button Command="{Binding ChangeTextCommand}" Content="Изменить текст" HorizontalAlignment="Center"/>
            </StackPanel>

            <Separator/>

            <TextBlock Text="DockPanel:" FontWeight="Bold"/>
            <DockPanel Height="150">
                <Rectangle Fill="Red" DockPanel.Dock="Top" Height="30"/>
                <Rectangle Fill="Blue" DockPanel.Dock="Left" Width="30"/>
                <Rectangle Fill="Green" DockPanel.Dock="Bottom" Height="30"/>
                <Rectangle Fill="Yellow" DockPanel.Dock="Right" Width="30"/>
                <Rectangle Fill="Orange"/>
            </DockPanel>

            <TextBlock Text="Grid:" FontWeight="Bold"/>
            <Grid ColumnDefinitions="*,2*" RowDefinitions="Auto,100">
                <Rectangle Fill="Purple" Grid.Row="0" Grid.Column="0" Height="50"/>
                <Rectangle Fill="Pink" Grid.Row="0" Grid.Column="1"/>
                <Rectangle Fill="Cyan" Grid.Row="1" Grid.Column="0"/>
                <Rectangle Fill="Gray" Grid.Row="1" Grid.Column="1"/>
            </Grid>

            <TextBlock Text="StackPanels (Hor &amp; Vert):" FontWeight="Bold"/>
            <StackPanel Orientation="Horizontal" Spacing="10">
                <TextBlock Text="Текст 1" FontSize="10"/>
                <TextBlock Text="Текст 2" FontSize="14"/>
                <TextBlock Text="Текст 3" FontSize="18"/>
                <TextBlock Text="Текст 4" FontSize="22"/>
            </StackPanel>
        </StackPanel>
    </ScrollViewer>
</UserControl>

```
### 4. ПР 20: Элементы ввода данных
**Models/UserCredentials.cs** (Отдельный файл для логики валидации)
```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using System.ComponentModel.DataAnnotations;

namespace HelloApp.Models;

public partial class UserCredentials : ObservableValidator
{
    [ObservableProperty]
    [Required(ErrorMessage = "Логин обязателен")]
    [RegularExpression(@"^[a-zA-Z0-9_]{3,20}$", ErrorMessage = "Латиница, цифры, _, 3-20 символов")]
    private string? _login;

    [ObservableProperty]
    [Required(ErrorMessage = "Пароль обязателен")]
    [MinLength(8, ErrorMessage = "Минимум 8 символов")]
    [RegularExpression(@"^(?=.*[a-z])(?=.*[A-Z])(?=.*\d).+$", ErrorMessage = "Нужны заглавные, строчные и цифры")]
    private string? _password;

    [ObservableProperty]
    [Required]
    [EmailAddress(ErrorMessage = "Неверный формат email")]
    private string? _email;
}

```
**ViewModels/ControlsViewModel.cs**
```csharp
using System;
using System.Collections.Generic;
using CommunityToolkit.Mvvm.ComponentModel;

namespace HelloApp.ViewModels;

public partial class ControlsViewModel : ViewModelBase
{
    [ObservableProperty]
    [NotifyPropertyChangedFor(nameof(Age))]
    private DateTimeOffset? _selectedDate = DateTimeOffset.Now;

    public int Age => SelectedDate.HasValue 
        ? (int)((DateTimeOffset.Now - SelectedDate.Value).TotalDays / 365.25) 
        : 0;

    [ObservableProperty]
    private string? _selectedLanguage;

    public List<string> Languages { get; } = new() { "C#", "Python", "Java", "C++" };
}

```
**Views/ControlsView.axaml**
```xml
<UserControl xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:vm="using:HelloApp.ViewModels"
             x:Class="HelloApp.Views.ControlsView"
             x:DataType="vm:ControlsViewModel">
    <ScrollViewer>
        <StackPanel Spacing="15" Margin="20">
            <TextBlock>
                <Span FontSize="18" Background="LightGray">
                    <Run Text="Красный " Foreground="Red" FontWeight="Bold"/>
                    <Run Text="Синий " Foreground="Blue" FontStyle="Italic"/>
                </Span>
                <LineBreak/>
                <Run Text="Зеленый " Foreground="Green" TextDecorations="Underline"/>
                <Run Text="Желтый " Foreground="Goldenrod"/>
                <InlineUIContainer BaselineAlignment="Baseline">
                    <Button Content="Встроенная кнопка" Padding="2"/>
                </InlineUIContainer>
            </TextBlock>

            <TextBox Watermark="Логин"/>
            <TextBox Watermark="Пароль" PasswordChar="*"/>
            <TextBox Watermark="Код (только цифры)" TextInputOptions.ContentType="Number"/>

            <CheckBox Content="Согласен с правилами"/>
            <StackPanel Orientation="Horizontal" Spacing="10">
                <RadioButton GroupName="G1" Content="Опция А"/>
                <RadioButton GroupName="G1" Content="Опция Б"/>
                <RadioButton GroupName="G2" Content="Группа 2 - В"/>
                <RadioButton GroupName="G2" Content="Группа 2 - Г"/>
            </StackPanel>

            <StackPanel Orientation="Horizontal" Spacing="10">
                <CalendarDatePicker SelectedDate="{Binding SelectedDate}"/>
                <TextBlock Text="{Binding Age, StringFormat='Возраст: {0}'}" VerticalAlignment="Center"/>
            </StackPanel>

            <ComboBox ItemsSource="{Binding Languages}" SelectedItem="{Binding SelectedLanguage}" PlaceholderText="Выберите язык"/>
            <TextBlock Text="{Binding SelectedLanguage, StringFormat='Вы выбрали: {0}'}"/>

            <Slider Name="OpacitySlider" Minimum="0" Maximum="1" Value="1"/>
            <Rectangle Fill="DarkBlue" Height="50" Opacity="{Binding #OpacitySlider.Value}"/>
            <ToggleSwitch OnContent="Включено" OffContent="Выключено"/>
        </StackPanel>
    </ScrollViewer>
</UserControl>

```
### 5. ПР 21: Навигация и Валидация
**ViewModels/NavDemoViewModel.cs**
```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using HelloApp.Services;

namespace HelloApp.ViewModels;

public partial class NavDemoViewModel : ViewModelBase
{
    public static NavigationService? Navigation { get; private set; }

    [ObservableProperty]
    private ViewModelBase _currentPage;

    public NavDemoViewModel()
    {
        // Инициализация сервиса. Делегат указывает, как менять текущую страницу
        Navigation = new NavigationService(vm => CurrentPage = vm);
        
        // Стартовая страница
        _currentPage = new AuthorizationViewModel();
        Navigation.NavigateTo(_currentPage);
    }

    [RelayCommand]
    public void GoToAuth() => Navigation?.NavigateTo(new AuthorizationViewModel());

    [RelayCommand]
    public void GoToReg() => Navigation?.NavigateTo(new RegistrationViewModel());

    [RelayCommand]
    public void GoBack() => Navigation?.GoBack();
}

```
**Views/NavDemoView.axaml**
```xml
<UserControl xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:vm="using:HelloApp.ViewModels"
             x:Class="HelloApp.Views.NavDemoView"
             x:DataType="vm:NavDemoViewModel">
    <DockPanel Margin="10">
        <StackPanel DockPanel.Dock="Top" Orientation="Horizontal" Spacing="10" Margin="0,0,0,10">
            <Button Command="{Binding GoBackCommand}" Content="&lt; Назад"/>
            <Button Command="{Binding GoToAuthCommand}" Content="Авторизация"/>
            <Button Command="{Binding GoToRegCommand}" Content="Регистрация (Валидация)"/>
        </StackPanel>
        
        <ContentControl Content="{Binding CurrentPage}" BorderBrush="Gray" BorderThickness="1" Padding="10"/>
    </DockPanel>
</UserControl>

```
**ViewModels/AuthorizationViewModel.cs**
```csharp
namespace HelloApp.ViewModels;
public class AuthorizationViewModel : ViewModelBase { } // Пустая для демонстрации

```
**Views/AuthorizationView.axaml**
```xml
<UserControl xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             x:Class="HelloApp.Views.AuthorizationView">
    <StackPanel Spacing="10" HorizontalAlignment="Center" VerticalAlignment="Center">
        <TextBlock Text="Страница авторизации" FontSize="20" FontWeight="Bold"/>
        <TextBox Watermark="Логин"/>
        <TextBox Watermark="Пароль" PasswordChar="*"/>
        <Button Content="Войти" HorizontalAlignment="Stretch" HorizontalContentAlignment="Center"/>
    </StackPanel>
</UserControl>

```
**ViewModels/RegistrationViewModel.cs** (Используем модель с валидацией)
```csharp
using HelloApp.Models;

namespace HelloApp.ViewModels;

public partial class RegistrationViewModel : ViewModelBase
{
    public UserCredentials Credentials { get; } = new();
}

```
**Views/RegistrationView.axaml** (Автоматическое отображение ошибок валидации)
```xml
<UserControl xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:vm="using:HelloApp.ViewModels"
             x:Class="HelloApp.Views.RegistrationView"
             x:DataType="vm:RegistrationViewModel">
    <StackPanel Spacing="10" Width="300" HorizontalAlignment="Center" VerticalAlignment="Center">
        <TextBlock Text="Регистрация (С валидацией)" FontSize="20" FontWeight="Bold"/>
        <TextBox Text="{Binding Credentials.Login}" Watermark="Логин (a-z, 0-9, _)"/>
        <TextBox Text="{Binding Credentials.Password}" Watermark="Пароль" PasswordChar="*"/>
        <TextBox Text="{Binding Credentials.Email}" Watermark="Email"/>
        <Button Content="Зарегистрироваться" HorizontalAlignment="Stretch" HorizontalContentAlignment="Center"/>
    </StackPanel>
</UserControl>

```
### ⚙️ Инструкция по сборке файлов "Code Behind" (.axaml.cs)
Для всех файлов .axaml их C# аналоги (.axaml.cs) остаются стандартными, создаваемыми IDE. Они выглядят одинаково, вам нужно только менять название класса. Пример для HomeView.axaml.cs:
```csharp
using Avalonia.Controls;

namespace HelloApp.Views;

public partial class HomeView : UserControl
{
    public HomeView()
    {
        InitializeComponent();
    }
}

```
Точно такой же код (с изменением имени класса) должен быть у ControlsView.axaml.cs, NavDemoView.axaml.cs, AuthorizationView.axaml.cs и RegistrationView.axaml.cs. Ошибок не будет.
