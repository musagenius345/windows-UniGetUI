# Theming System Guide: Light, Dark, and System Themes in WinUI 3

**A comprehensive guide to implementing dynamic theming in modern Windows apps**

> Learn how UniGetUI implements seamless theme switching with light, dark, and system-following modes using WinUI 3.

---

## Table of Contents

1. [Overview](#overview)
2. [Theme Architecture](#theme-architecture)
3. [Implementing Theme Support](#implementing-theme-support)
4. [Theme Resources](#theme-resources)
5. [Title Bar Theming](#title-bar-theming)
6. [Backdrop Materials](#backdrop-materials)
7. [Theme-Aware Components](#theme-aware-components)
8. [Settings UI](#settings-ui)
9. [Best Practices](#best-practices)
10. [Troubleshooting](#troubleshooting)

---

## Overview

### Why Theme Support Matters

Modern Windows users expect apps to:
- ✅ Support both light and dark modes
- ✅ Follow system theme automatically
- ✅ Remember user preferences
- ✅ Apply themes without restarting
- ✅ Theme all UI elements consistently

### UniGetUI's Approach

UniGetUI provides three theme options:
1. **Light** - Force light theme
2. **Dark** - Force dark theme
3. **Auto** - Follow system theme (default)

**Storage**: `PreferredTheme` setting with values: `"light"`, `"dark"`, or `"auto"`

---

## Theme Architecture

### Theme Flow Diagram

```
┌─────────────────────────────────────────────────────┐
│ User selects theme in Settings                      │
│ (ComboBox: Light / Dark / Auto)                    │
└─────────────────┬───────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────┐
│ Settings.SetValue(K.PreferredTheme, value)         │
│ Saves to: %LocalAppData%\UniGetUI\Configuration\   │
└─────────────────┬───────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────┐
│ MainWindow.ApplyTheme()                             │
│ - Reads PreferredTheme setting                      │
│ - Updates MainContentGrid.RequestedTheme           │
│ - Updates ThemeListener.CurrentTheme               │
│ - Customizes title bar colors                      │
└─────────────────┬───────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────┐
│ WinUI Theme System                                  │
│ - Applies theme resources                          │
│ - Updates all ThemeResource bindings               │
│ - Renders UI with appropriate colors               │
└─────────────────────────────────────────────────────┘
```

### Key Components

**File**: `/src/UniGetUI/MainWindow.xaml.cs`

```csharp
public void ApplyTheme()
{
    string preferredTheme = Settings.GetValue(Settings.K.PreferredTheme);

    if (preferredTheme == "dark")
    {
        MainApp.Instance.ThemeListener.CurrentTheme = ApplicationTheme.Dark;
        MainContentGrid.RequestedTheme = ElementTheme.Dark;
    }
    else if (preferredTheme == "light")
    {
        MainApp.Instance.ThemeListener.CurrentTheme = ApplicationTheme.Light;
        MainContentGrid.RequestedTheme = ElementTheme.Light;
    }
    else  // "auto" or empty
    {
        // Detect current system theme
        if (MainContentGrid.ActualTheme == ElementTheme.Dark)
        {
            MainApp.Instance.ThemeListener.CurrentTheme = ApplicationTheme.Dark;
        }
        else
        {
            MainApp.Instance.ThemeListener.CurrentTheme = ApplicationTheme.Light;
        }

        MainContentGrid.RequestedTheme = ElementTheme.Default;
    }

    // Customize title bar colors
    if (AppWindowTitleBar.IsCustomizationSupported())
    {
        if (MainApp.Instance.ThemeListener.CurrentTheme == ApplicationTheme.Light)
        {
            AppWindow.TitleBar.ButtonForegroundColor = Colors.Black;
        }
        else
        {
            AppWindow.TitleBar.ButtonForegroundColor = Colors.White;
        }
    }
}
```

---

## Implementing Theme Support

### Step 1: Set Up the Main Window

**MainWindow.xaml**:

```xml
<Window
    x:Class="MyApp.MainWindow"
    xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">

    <!-- Mica backdrop for modern look -->
    <Window.SystemBackdrop>
        <MicaBackdrop Kind="Base" />
    </Window.SystemBackdrop>

    <!-- Main content grid with theme binding -->
    <Grid x:Name="MainContentGrid">
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto" />
            <RowDefinition Height="*" />
        </Grid.RowDefinitions>

        <!-- Title bar -->
        <TitleBar
            x:Name="AppTitleBar"
            Title="My App"
            Grid.Row="0" />

        <!-- Content -->
        <Frame x:Name="ContentFrame" Grid.Row="1" />
    </Grid>
</Window>
```

**MainWindow.xaml.cs**:

```csharp
using Microsoft.UI;
using Microsoft.UI.Windowing;
using Microsoft.UI.Xaml;

namespace MyApp;

public sealed partial class MainWindow : Window
{
    public MainWindow()
    {
        InitializeComponent();

        // Set up title bar
        ExtendsContentIntoTitleBar = true;
        SetTitleBar(AppTitleBar);

        // Apply theme on startup
        ApplyTheme();
    }

    public void ApplyTheme()
    {
        // Get theme preference from settings
        string preferredTheme = Settings.GetValue(Settings.K.PreferredTheme);

        if (preferredTheme == "dark")
        {
            // Force dark theme
            MainContentGrid.RequestedTheme = ElementTheme.Dark;
        }
        else if (preferredTheme == "light")
        {
            // Force light theme
            MainContentGrid.RequestedTheme = ElementTheme.Light;
        }
        else
        {
            // Follow system theme (default)
            MainContentGrid.RequestedTheme = ElementTheme.Default;
        }

        // Update title bar colors
        UpdateTitleBarColors();
    }

    private void UpdateTitleBarColors()
    {
        if (!AppWindowTitleBar.IsCustomizationSupported())
            return;

        var currentTheme = MainContentGrid.ActualTheme;

        if (currentTheme == ElementTheme.Light)
        {
            // Light theme: dark buttons
            AppWindow.TitleBar.ButtonForegroundColor = Colors.Black;
        }
        else
        {
            // Dark theme: light buttons
            AppWindow.TitleBar.ButtonForegroundColor = Colors.White;
        }
    }
}
```

### Step 2: Create Settings Infrastructure

**Add to Settings Enum**:

```csharp
namespace MyApp.Core.SettingsEngine;

public static partial class Settings
{
    public enum K
    {
        // ... other settings
        PreferredTheme,
        // ... other settings
    }

    public static string ResolveKey(K key)
    {
        return key switch
        {
            K.PreferredTheme => "PreferredTheme",
            // ... other mappings
        };
    }
}
```

**Initialize Default Theme**:

```csharp
// In your initialization code
if (Settings.GetValue(Settings.K.PreferredTheme) == "")
{
    Settings.SetValue(Settings.K.PreferredTheme, "auto");
}
```

### Step 3: Create Theme Settings UI

**SettingsPage.xaml**:

```xml
<Page>
    <ScrollViewer>
        <StackPanel Spacing="4" Padding="16">
            <TextBlock
                Text="Appearance"
                FontWeight="SemiBold"
                Margin="4,32,4,8" />

            <SettingsCard
                Header="Application theme"
                Description="Choose how the app looks">
                <ComboBox
                    x:Name="ThemeSelector"
                    MinWidth="200"
                    SelectionChanged="ThemeSelector_SelectionChanged">
                    <ComboBoxItem Content="Light" Tag="light" />
                    <ComboBoxItem Content="Dark" Tag="dark" />
                    <ComboBoxItem Content="Use system setting" Tag="auto" IsSelected="True" />
                </ComboBox>
            </SettingsCard>
        </StackPanel>
    </ScrollViewer>
</Page>
```

**SettingsPage.xaml.cs**:

```csharp
public sealed partial class SettingsPage : Page
{
    public SettingsPage()
    {
        InitializeComponent();
        LoadCurrentTheme();
    }

    private void LoadCurrentTheme()
    {
        string currentTheme = Settings.GetValue(Settings.K.PreferredTheme);

        // Select the appropriate combobox item
        foreach (ComboBoxItem item in ThemeSelector.Items)
        {
            if (item.Tag?.ToString() == currentTheme)
            {
                ThemeSelector.SelectedItem = item;
                break;
            }
        }
    }

    private void ThemeSelector_SelectionChanged(object sender, SelectionChangedEventArgs e)
    {
        if (ThemeSelector.SelectedItem is ComboBoxItem item)
        {
            string themeValue = item.Tag?.ToString() ?? "auto";

            // Save preference
            Settings.SetValue(Settings.K.PreferredTheme, themeValue);

            // Apply immediately
            if (App.MainWindow != null)
            {
                App.MainWindow.ApplyTheme();
            }
        }
    }
}
```

---

## Theme Resources

### Using ThemeResource

WinUI 3 provides theme-aware resources that automatically adapt to theme changes.

**Always use `ThemeResource` for colors**:

```xml
<!-- ✅ Good - Adapts to theme -->
<Grid Background="{ThemeResource ApplicationPageBackgroundThemeBrush}">
    <TextBlock Foreground="{ThemeResource TextFillColorPrimaryBrush}" />
</Grid>

<!-- ❌ Bad - Hardcoded color -->
<Grid Background="White">
    <TextBlock Foreground="Black" />
</Grid>
```

### Common Theme Resources

#### Background Colors

```xml
<!-- Page backgrounds -->
<Grid Background="{ThemeResource ApplicationPageBackgroundThemeBrush}" />
<Grid Background="{ThemeResource LayerFillColorDefaultBrush}" />
<Grid Background="{ThemeResource SolidBackgroundFillColorBaseBrush}" />

<!-- Card backgrounds -->
<Border Background="{ThemeResource CardBackgroundFillColorDefaultBrush}" />
<Border Background="{ThemeResource CardBackgroundFillColorSecondaryBrush}" />

<!-- Acrylic effects -->
<Grid Background="{ThemeResource AcrylicBackgroundFillColorDefaultBrush}" />
```

#### Text Colors

```xml
<!-- Primary text -->
<TextBlock Foreground="{ThemeResource TextFillColorPrimaryBrush}" />

<!-- Secondary text -->
<TextBlock Foreground="{ThemeResource TextFillColorSecondaryBrush}" />

<!-- Tertiary/dimmed text -->
<TextBlock Foreground="{ThemeResource TextFillColorTertiaryBrush}" />

<!-- Disabled text -->
<TextBlock Foreground="{ThemeResource TextFillColorDisabledBrush}" />

<!-- Text on accent -->
<TextBlock Foreground="{ThemeResource TextOnAccentFillColorPrimaryBrush}" />
```

#### Border and Stroke Colors

```xml
<!-- Card borders -->
<Border BorderBrush="{ThemeResource CardStrokeColorDefaultBrush}" />

<!-- Dividers -->
<Border BorderBrush="{ThemeResource DividerStrokeColorDefaultBrush}" />

<!-- Control borders -->
<Border BorderBrush="{ThemeResource ControlStrokeColorDefaultBrush}" />
```

#### Interactive Colors

```xml
<!-- Buttons -->
<Button Background="{ThemeResource ButtonBackgroundPointerOver}" />

<!-- Accent -->
<Button Background="{ThemeResource AccentFillColorDefaultBrush}" />

<!-- Subtle -->
<Button Background="{ThemeResource SubtleFillColorSecondaryBrush}" />
```

### Custom Theme-Aware Resources

**App.xaml**:

```xml
<Application.Resources>
    <ResourceDictionary>
        <!-- Light theme colors -->
        <ResourceDictionary.ThemeDictionaries>
            <ResourceDictionary x:Key="Light">
                <Color x:Key="CustomAccentColor">#0078D4</Color>
                <SolidColorBrush x:Key="CustomAccentBrush" Color="{StaticResource CustomAccentColor}" />
            </ResourceDictionary>

            <!-- Dark theme colors -->
            <ResourceDictionary x:Key="Dark">
                <Color x:Key="CustomAccentColor">#60CDFF</Color>
                <SolidColorBrush x:Key="CustomAccentBrush" Color="{StaticResource CustomAccentColor}" />
            </ResourceDictionary>
        </ResourceDictionary.ThemeDictionaries>
    </ResourceDictionary>
</Application.Resources>
```

**Usage**:

```xml
<Border Background="{ThemeResource CustomAccentBrush}" />
```

---

## Title Bar Theming

### Custom Title Bar Colors

UniGetUI customizes title bar button colors to match the theme:

```csharp
private void UpdateTitleBarColors()
{
    if (!AppWindowTitleBar.IsCustomizationSupported())
    {
        Logger.Info("Title bar customization is not supported on this system");
        return;
    }

    var titleBar = AppWindow.TitleBar;
    var currentTheme = MainContentGrid.ActualTheme;

    if (currentTheme == ElementTheme.Light)
    {
        // Light theme
        titleBar.ButtonForegroundColor = Colors.Black;
        titleBar.ButtonHoverForegroundColor = Colors.Black;
        titleBar.ButtonPressedForegroundColor = Colors.Black;

        // Optional: customize backgrounds
        titleBar.ButtonBackgroundColor = Colors.Transparent;
        titleBar.ButtonHoverBackgroundColor = Color.FromArgb(20, 0, 0, 0);
        titleBar.ButtonPressedBackgroundColor = Color.FromArgb(30, 0, 0, 0);
    }
    else
    {
        // Dark theme
        titleBar.ButtonForegroundColor = Colors.White;
        titleBar.ButtonHoverForegroundColor = Colors.White;
        titleBar.ButtonPressedForegroundColor = Colors.White;

        titleBar.ButtonBackgroundColor = Colors.Transparent;
        titleBar.ButtonHoverBackgroundColor = Color.FromArgb(20, 255, 255, 255);
        titleBar.ButtonPressedBackgroundColor = Color.FromArgb(30, 255, 255, 255);
    }
}
```

### Title Bar with Content

```xml
<TitleBar
    x:Name="AppTitleBar"
    Title="My App"
    IsBackButtonVisible="False"
    IsPaneToggleButtonVisible="True">

    <!-- Custom content in title bar -->
    <TitleBar.Content>
        <AutoSuggestBox
            Width="400"
            Height="32"
            PlaceholderText="Search..."
            BorderBrush="{ThemeResource ButtonBorderBrush}" />
    </TitleBar.Content>
</TitleBar>
```

---

## Backdrop Materials

### Mica Backdrop

Mica provides a subtle, dynamic background material:

```xml
<Window>
    <Window.SystemBackdrop>
        <MicaBackdrop Kind="Base" />
    </Window.SystemBackdrop>
</Window>
```

**Mica Kinds**:
- `Base` - Standard Mica (recommended)
- `BaseAlt` - Alternative Mica variant

### Acrylic Backdrop

For a more transparent, blurred effect:

```xml
<Window>
    <Window.SystemBackdrop>
        <DesktopAcrylicBackdrop />
    </Window.SystemBackdrop>
</Window>
```

### Conditional Backdrop

```csharp
public MainWindow()
{
    InitializeComponent();

    // Use Mica if available, otherwise transparent
    if (MicaBackdrop.IsSupported())
    {
        SystemBackdrop = new MicaBackdrop() { Kind = MicaKind.Base };
    }
}
```

---

## Theme-Aware Components

### Adaptive Icons

Use `FontIcon` with theme-aware glyphs:

```xml
<!-- Icon that adapts to theme -->
<FontIcon
    FontFamily="{StaticResource SymbolThemeFontFamily}"
    Glyph="&#xE8B7;"
    Foreground="{ThemeResource TextFillColorPrimaryBrush}" />
```

### Conditional Visibility

Show/hide elements based on theme:

```xml
<VisualStateManager.VisualStateGroups>
    <VisualStateGroup x:Name="ThemeStates">
        <VisualState x:Name="Light">
            <VisualState.Setters>
                <Setter Target="LightOnlyImage.Visibility" Value="Visible" />
                <Setter Target="DarkOnlyImage.Visibility" Value="Collapsed" />
            </VisualState.Setters>
        </VisualState>
        <VisualState x:Name="Dark">
            <VisualState.Setters>
                <Setter Target="LightOnlyImage.Visibility" Value="Collapsed" />
                <Setter Target="DarkOnlyImage.Visibility" Value="Visible" />
            </VisualState.Setters>
        </VisualState>
    </VisualStateGroup>
</VisualStateManager.VisualStateGroups>
```

### Theme Change Detection

Listen for theme changes:

```csharp
public MyPage()
{
    InitializeComponent();

    // Listen for actual theme changes
    ActualThemeChanged += OnThemeChanged;
}

private void OnThemeChanged(FrameworkElement sender, object args)
{
    var newTheme = sender.ActualTheme;
    Logger.Info($"Theme changed to: {newTheme}");

    // Update custom UI elements
    UpdateCustomElements(newTheme);
}
```

---

## Settings UI

### UniGetUI's Theme Selector

**File**: `/src/UniGetUI/Pages/SettingsPages/GeneralPages/Interface_P.xaml`

```xml
<Page>
    <ScrollViewer>
        <StackPanel>
            <TranslatedTextBlock
                Text="Appearance"
                FontWeight="SemiBold"
                Margin="4,32,4,8" />

            <ComboboxCard
                x:Name="ThemeSelector"
                CornerRadius="8"
                SettingName="PreferredTheme"
                Text="Application theme:"
                ValueChanged="ThemeSelector_ValueChanged" />
        </StackPanel>
    </ScrollViewer>
</Page>
```

**Code-behind**:

```csharp
public Interface_P()
{
    InitializeComponent();

    // Initialize theme setting
    if (Settings.GetValue(Settings.K.PreferredTheme) == "")
    {
        Settings.SetValue(Settings.K.PreferredTheme, "auto");
    }

    // Populate theme options
    ThemeSelector.AddItem(CoreTools.AutoTranslated("Light"), "light");
    ThemeSelector.AddItem(CoreTools.AutoTranslated("Dark"), "dark");
    ThemeSelector.AddItem(CoreTools.AutoTranslated("Follow system color scheme"), "auto");
    ThemeSelector.ShowAddedItems();
}

private void ThemeSelector_ValueChanged(object sender, EventArgs e)
{
    // Apply theme immediately
    MainApp.Instance.MainWindow.ApplyTheme();
}
```

### Using CommunityToolkit SettingsCard

```xml
<Page xmlns:controls="using:CommunityToolkit.WinUI.Controls">
    <StackPanel>
        <controls:SettingsCard
            Header="Theme"
            Description="Select your preferred theme">
            <controls:SettingsCard.HeaderIcon>
                <FontIcon Glyph="&#xE771;" />
            </controls:SettingsCard.HeaderIcon>

            <ComboBox
                x:Name="ThemeComboBox"
                MinWidth="200"
                SelectionChanged="ThemeComboBox_SelectionChanged">
                <ComboBoxItem Content="Light" Tag="light" />
                <ComboBoxItem Content="Dark" Tag="dark" />
                <ComboBoxItem Content="System" Tag="auto" />
            </ComboBox>
        </controls:SettingsCard>
    </StackPanel>
</Page>
```

---

## Best Practices

### 1. Always Use ThemeResource for Colors

✅ **Do**:
```xml
<Grid Background="{ThemeResource ApplicationPageBackgroundThemeBrush}">
    <TextBlock Foreground="{ThemeResource TextFillColorPrimaryBrush}" />
</Grid>
```

❌ **Don't**:
```xml
<Grid Background="White">
    <TextBlock Foreground="Black" />
</Grid>
```

### 2. Initialize Theme on Startup

```csharp
public MainWindow()
{
    InitializeComponent();

    // IMPORTANT: Apply theme before showing window
    ApplyTheme();

    // Now show window
    Activate();
}
```

### 3. Handle System Theme Changes

When using "auto" mode, listen for system theme changes:

```csharp
public MainWindow()
{
    InitializeComponent();

    // Monitor for system theme changes
    MainContentGrid.ActualThemeChanged += (s, e) =>
    {
        if (Settings.GetValue(Settings.K.PreferredTheme) == "auto")
        {
            UpdateTitleBarColors();
            OnThemeChanged();
        }
    };
}
```

### 4. Update All Visual Elements

When theme changes, update:
- ✅ Title bar colors
- ✅ System tray icon (if applicable)
- ✅ Custom-drawn elements
- ✅ Chart colors
- ✅ Status bar colors

```csharp
public void ApplyTheme()
{
    // Apply theme to main grid
    UpdateMainGridTheme();

    // Update title bar
    UpdateTitleBarColors();

    // Update system tray
    UpdateSystemTrayIcon();

    // Update custom visuals
    UpdateCustomVisuals();
}
```

### 5. Provide Instant Feedback

Theme changes should be instant—no app restart required:

```csharp
private void ThemeSelector_SelectionChanged(object sender, SelectionChangedEventArgs e)
{
    // Save setting
    var theme = GetSelectedTheme();
    Settings.SetValue(Settings.K.PreferredTheme, theme);

    // Apply immediately
    MainWindow.ApplyTheme();

    // ❌ Don't do this:
    // ShowRestartDialog();
}
```

### 6. Test Both Themes

Always test your app in both themes:

```
Light Theme:
- All text readable?
- Icons visible?
- Borders distinguishable?
- Custom colors work?

Dark Theme:
- No eye-searing white?
- Contrast sufficient?
- Disabled states visible?
- Focus indicators clear?
```

### 7. Handle High Contrast Mode

```csharp
private void UpdateForHighContrast()
{
    var settings = new UISettings();
    var foreground = settings.GetColorValue(UIColorType.Foreground);
    var background = settings.GetColorValue(UIColorType.Background);

    // Check if high contrast
    bool isHighContrast =
        (foreground.R == 0 && foreground.G == 0 && foreground.B == 0 &&
         background.R == 255 && background.G == 255 && background.B == 255) ||
        (foreground.R == 255 && foreground.G == 255 && foreground.B == 255 &&
         background.R == 0 && background.G == 0 && background.B == 0);

    if (isHighContrast)
    {
        // Simplify UI, use system colors
        ApplyHighContrastMode();
    }
}
```

### 8. Document Custom Theme Resources

```xml
<!-- App.xaml -->
<Application.Resources>
    <!-- Custom theme-aware colors -->
    <ResourceDictionary.ThemeDictionaries>
        <ResourceDictionary x:Key="Light">
            <!-- Success color in light theme -->
            <Color x:Key="SuccessColor">#107C10</Color>
            <!-- Error color in light theme -->
            <Color x:Key="ErrorColor">#E81123</Color>
        </ResourceDictionary>

        <ResourceDictionary x:Key="Dark">
            <!-- Success color in dark theme (brighter) -->
            <Color x:Key="SuccessColor">#6CCB5F</Color>
            <!-- Error color in dark theme (brighter) -->
            <Color x:Key="ErrorColor">#FF99A4</Color>
        </ResourceDictionary>
    </ResourceDictionary.ThemeDictionaries>
</Application.Resources>
```

---

## Troubleshooting

### Issue: Theme Not Applying

**Problem**: Theme changes don't take effect

**Solution**:
```csharp
// ❌ Wrong - setting on Window
this.RequestedTheme = ElementTheme.Dark;

// ✅ Correct - setting on root Grid
MainContentGrid.RequestedTheme = ElementTheme.Dark;
```

### Issue: Title Bar Colors Not Updating

**Problem**: Title bar buttons stay wrong color

**Solution**:
```csharp
// Check if customization is supported
if (!AppWindowTitleBar.IsCustomizationSupported())
{
    Logger.Warn("Title bar customization not supported");
    return;
}

// Ensure you're using ActualTheme, not RequestedTheme
var currentTheme = MainContentGrid.ActualTheme;  // ✅
// Not: var currentTheme = MainContentGrid.RequestedTheme;  // ❌
```

### Issue: Custom Resources Not Changing

**Problem**: Custom colors don't adapt to theme

**Solution**: Use `ThemeResource` instead of `StaticResource`:

```xml
<!-- ❌ Won't update on theme change -->
<Border Background="{StaticResource CustomBrush}" />

<!-- ✅ Updates on theme change -->
<Border Background="{ThemeResource CustomBrush}" />
```

### Issue: Flickering on Theme Change

**Problem**: UI flickers when changing theme

**Solution**: Disable animations during theme change:

```csharp
public void ApplyTheme()
{
    // Disable implicit animations
    MainContentGrid.EnableImplicitAnimation = false;

    // Apply theme
    MainContentGrid.RequestedTheme = GetTheme();

    // Re-enable after short delay
    _ = Task.Delay(100).ContinueWith(_ =>
    {
        DispatcherQueue.TryEnqueue(() =>
        {
            MainContentGrid.EnableImplicitAnimation = true;
        });
    });
}
```

### Issue: Mica Not Working

**Problem**: Mica backdrop not showing

**Solution**:
```csharp
// Check system requirements
if (!MicaBackdrop.IsSupported())
{
    Logger.Info("Mica not supported, using fallback");
    // Use solid color fallback
    MainContentGrid.Background = new SolidColorBrush(
        Application.Current.RequestedTheme == ApplicationTheme.Dark
            ? Color.FromArgb(255, 32, 32, 32)
            : Color.FromArgb(255, 243, 243, 243)
    );
}
```

---

## Complete Example

### Full Implementation

**MainWindow.xaml**:

```xml
<Window
    x:Class="MyApp.MainWindow"
    xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">

    <Window.SystemBackdrop>
        <MicaBackdrop Kind="Base" />
    </Window.SystemBackdrop>

    <Grid x:Name="MainContentGrid">
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto" />
            <RowDefinition Height="*" />
        </Grid.RowDefinitions>

        <TitleBar x:Name="AppTitleBar" Title="My App" Grid.Row="0" />

        <Frame x:Name="ContentFrame" Grid.Row="1" />
    </Grid>
</Window>
```

**MainWindow.xaml.cs**:

```csharp
using Microsoft.UI;
using Microsoft.UI.Xaml;
using Microsoft.UI.Windowing;

namespace MyApp;

public sealed partial class MainWindow : Window
{
    public MainWindow()
    {
        InitializeComponent();

        // Set up window
        ExtendsContentIntoTitleBar = true;
        SetTitleBar(AppTitleBar);

        // Monitor theme changes
        MainContentGrid.ActualThemeChanged += OnActualThemeChanged;

        // Apply initial theme
        ApplyTheme();
    }

    public void ApplyTheme()
    {
        string preferredTheme = Settings.GetValue(Settings.K.PreferredTheme);

        if (preferredTheme == "dark")
        {
            MainContentGrid.RequestedTheme = ElementTheme.Dark;
        }
        else if (preferredTheme == "light")
        {
            MainContentGrid.RequestedTheme = ElementTheme.Light;
        }
        else
        {
            MainContentGrid.RequestedTheme = ElementTheme.Default;
        }

        UpdateTitleBarColors();
    }

    private void OnActualThemeChanged(FrameworkElement sender, object args)
    {
        UpdateTitleBarColors();
    }

    private void UpdateTitleBarColors()
    {
        if (!AppWindowTitleBar.IsCustomizationSupported())
            return;

        var currentTheme = MainContentGrid.ActualTheme;
        var titleBar = AppWindow.TitleBar;

        if (currentTheme == ElementTheme.Light)
        {
            titleBar.ButtonForegroundColor = Colors.Black;
        }
        else
        {
            titleBar.ButtonForegroundColor = Colors.White;
        }
    }
}
```

**SettingsPage.xaml**:

```xml
<Page xmlns:controls="using:CommunityToolkit.WinUI.Controls">
    <ScrollViewer>
        <StackPanel Spacing="4" Padding="16">
            <TextBlock
                Text="Appearance"
                Style="{StaticResource SettingsSectionHeaderTextBlockStyle}" />

            <controls:SettingsCard
                Header="Theme"
                Description="Select how the app looks">
                <ComboBox
                    x:Name="ThemeSelector"
                    MinWidth="200"
                    SelectionChanged="ThemeSelector_SelectionChanged">
                    <ComboBoxItem Content="Light" Tag="light" />
                    <ComboBoxItem Content="Dark" Tag="dark" />
                    <ComboBoxItem Content="Use system setting" Tag="auto" IsSelected="True" />
                </ComboBox>
            </controls:SettingsCard>
        </StackPanel>
    </ScrollViewer>
</Page>
```

**SettingsPage.xaml.cs**:

```csharp
public sealed partial class SettingsPage : Page
{
    public SettingsPage()
    {
        InitializeComponent();
        LoadCurrentTheme();
    }

    private void LoadCurrentTheme()
    {
        string currentTheme = Settings.GetValue(Settings.K.PreferredTheme);

        foreach (ComboBoxItem item in ThemeSelector.Items)
        {
            if (item.Tag?.ToString() == currentTheme)
            {
                ThemeSelector.SelectedItem = item;
                break;
            }
        }
    }

    private void ThemeSelector_SelectionChanged(object sender, SelectionChangedEventArgs e)
    {
        if (ThemeSelector.SelectedItem is ComboBoxItem item)
        {
            string theme = item.Tag?.ToString() ?? "auto";
            Settings.SetValue(Settings.K.PreferredTheme, theme);

            // Apply immediately
            ((MainWindow)App.MainWindow).ApplyTheme();
        }
    }
}
```

---

## Summary

UniGetUI demonstrates excellent theming implementation:

✅ **Three theme modes** - Light, Dark, and Auto
✅ **Instant switching** - No restart required
✅ **Complete coverage** - Title bar, backdrop, all UI elements
✅ **System integration** - Follows Windows theme automatically
✅ **Persistent** - Remembers user preference
✅ **Theme resources** - Uses WinUI 3 ThemeResource throughout
✅ **Modern materials** - Mica backdrop for depth

### Key Takeaways

1. **Use `ThemeResource`** for all colors
2. **Apply theme to root Grid**, not Window
3. **Update title bar colors** when theme changes
4. **Provide instant feedback** - no restart needed
5. **Support all three modes** - Light, Dark, Auto
6. **Test thoroughly** in both themes

### Files to Study

```
/src/UniGetUI/
├── MainWindow.xaml.cs              ⭐ ApplyTheme() method
├── MainWindow.xaml                 ⭐ Mica backdrop
├── Pages/SettingsPages/
│   └── GeneralPages/
│       ├── Interface_P.xaml        ⭐ Theme selector UI
│       └── Interface_P.xaml.cs     ⭐ Theme change handler
└── App.xaml.cs                     ⭐ ThemeListener initialization
```

---

**Happy theming! 🎨**

For questions or improvements, check the [UniGetUI repository](https://github.com/marticliment/UniGetUI).
