# Complete Guide to Tables and Lists in WinUI 3

## Table of Contents
1. [Introduction](#introduction)
2. [Core Concepts](#core-concepts)
3. [Architecture Overview](#architecture-overview)
4. [View Modes and DataTemplates](#view-modes-and-datatemplates)
5. [ItemsView and Layouts](#itemsview-and-layouts)
6. [Data Binding with Wrappers](#data-binding-with-wrappers)
7. [Sorting and Filtering](#sorting-and-filtering)
8. [Complete Working Example](#complete-working-example)
9. [Advanced Patterns](#advanced-patterns)
10. [Performance Optimization](#performance-optimization)
11. [Best Practices](#best-practices)

---

## Introduction

This guide explains how UniGetUI implements its sophisticated table and list UI system. You'll learn how to create flexible, high-performance lists with multiple view modes, sorting, filtering, and virtual scrolling.

**What You'll Learn:**
- Creating DataTemplates for different view modes (List, Grid, Icons)
- Using ItemsView with various layouts
- Implementing wrapper classes for data binding
- Building sortable and filterable collections
- Managing view mode switching
- Performance optimization techniques

**Prerequisites:**
- Basic WinUI 3 and XAML knowledge
- Understanding of MVVM patterns
- Familiarity with ObservableCollection

---

## Core Concepts

### The UniGetUI Approach

UniGetUI doesn't use traditional table controls. Instead, it uses a sophisticated system built on:

1. **Multiple DataTemplates**: One for each view mode (List, Grid, Icons)
2. **ItemsView Control**: Flexible container that supports different layouts
3. **Wrapper Pattern**: PackageWrapper wraps data models for UI binding
4. **Observable Collections**: Custom sortable collections for dynamic updates
5. **Filter Pipeline**: Multi-stage filtering with caching

### Key Components

```
┌─────────────────────────────────────────────┐
│          AbstractPackagesPage               │
│  ┌───────────────────────────────────────┐  │
│  │      View Mode Selector               │  │
│  │   (List / Grid / Icons)               │  │
│  └───────────────────────────────────────┘  │
│                    ↓                         │
│  ┌───────────────────────────────────────┐  │
│  │      SwitchPresenter                  │  │
│  │  ┌─────────┬─────────┬─────────┐     │  │
│  │  │ListView │GridView │IconsView│     │  │
│  │  └─────────┴─────────┴─────────┘     │  │
│  └───────────────────────────────────────┘  │
│                    ↓                         │
│  ┌───────────────────────────────────────┐  │
│  │   FilteredPackages (Observable)       │  │
│  │   ┌───────────────────────────┐       │  │
│  │   │ PackageWrapper            │       │  │
│  │   │ PackageWrapper            │       │  │
│  │   │ PackageWrapper            │       │  │
│  │   └───────────────────────────┘       │  │
│  └───────────────────────────────────────┘  │
└─────────────────────────────────────────────┘
```

---

## Architecture Overview

### File Structure

Understanding how UniGetUI organizes its table/list code:

```
src/UniGetUI/
├── Pages/SoftwarePages/
│   ├── AbstractPackagesPage.xaml          # Main UI definition
│   ├── AbstractPackagesPage.xaml.cs       # Logic, filtering, sorting
│   └── DiscoverPage.xaml                  # Inherits AbstractPackagesPage
├── Controls/
│   ├── PackageWrapper.cs                  # Data wrapper for binding
│   ├── PackageItemContainer.cs            # Custom item container
│   └── ObservablePackageCollection.cs     # Sortable collection
└── Helpers/
    └── FilterHelpers.cs                   # Search/filter utilities
```

### Data Flow

```
Raw Data (IPackage)
        ↓
PackageWrapper (adds UI properties)
        ↓
WrappedPackages (all items)
        ↓
FilterPackages() (search + source filter)
        ↓
FilteredPackages (visible items)
        ↓
ObservablePackageCollection.Sort()
        ↓
ItemsView (displays with DataTemplate)
```

---

## View Modes and DataTemplates

### Overview

UniGetUI provides three view modes, each with its own DataTemplate:

1. **List View** - Compact table-like rows with multiple columns
2. **Grid View** - Card-based grid with icon and key info
3. **Icons View** - Large icon tiles for visual browsing

### List View DataTemplate

The list view displays data in a table-like format using a Grid with 6 columns:

**File:** `AbstractPackagesPage.xaml` (lines 130-202)

```xml
<DataTemplate x:Key="PackageTemplate_List" x:DataType="pkgClasses:PackageWrapper">
    <widgets:PackageItemContainer
        AutomationProperties.Name="{x:Bind Package.AutomationName}"
        DoubleTapped="{x:Bind PackageItemContainer_DoubleTapped}"
        Package="{x:Bind Package}"
        PreviewKeyDown="{x:Bind PackageItemContainer_PreviewKeyDown}"
        RightTapped="{x:Bind PackageItemContainer_RightTapped}"
        Wrapper="{x:Bind Self}">

        <Grid Padding="12,3,8,3" ColumnSpacing="4" Opacity="{x:Bind ListedOpacity, Mode=OneWay}">
            <Grid.ColumnDefinitions>
                <ColumnDefinition Width="24" />                              <!-- Checkbox -->
                <ColumnDefinition Width="*" MinWidth="125" />                <!-- Icon + Name -->
                <ColumnDefinition Width="*" MinWidth="125" />                <!-- Package ID -->
                <ColumnDefinition Width="*" MaxWidth="150" />                <!-- Current Version -->
                <ColumnDefinition Width="*" MaxWidth="{x:Bind NewVersionLabelWidth}" /> <!-- New Version -->
                <ColumnDefinition Width="*" MaxWidth="175" />                <!-- Source -->
            </Grid.ColumnDefinitions>

            <!-- Checkbox Column -->
            <CheckBox
                Grid.Column="0"
                Margin="0,0,0,0"
                VerticalAlignment="Center"
                IsChecked="{x:Bind IsChecked, Mode=TwoWay}" />

            <!-- Icon + Name Column -->
            <StackPanel Grid.Column="1" Orientation="Horizontal" Spacing="8">
                <!-- Package Icon (default or custom) -->
                <TextBlock
                    widgets:IconBuilder.Icon="Package"
                    FontSize="24"
                    VerticalAlignment="Center"
                    Visibility="{x:Bind ShowDefaultPackageIcon, Mode=OneWay}" />

                <Image
                    Width="24"
                    Height="24"
                    VerticalAlignment="Center"
                    Source="{x:Bind MainIconSource, Mode=OneWay}"
                    Visibility="{x:Bind ShowCustomPackageIcon, Mode=OneWay}" />

                <!-- Package Name -->
                <TextBlock
                    Text="{x:Bind Package.Name}"
                    FontSize="14"
                    VerticalAlignment="Center"
                    TextTrimming="CharacterEllipsis"
                    ToolTipService.ToolTip="{x:Bind ListedNameTooltip, Mode=OneWay}" />
            </StackPanel>

            <!-- Package ID Column -->
            <TextBlock
                Grid.Column="2"
                Text="{x:Bind Package.Id}"
                FontSize="13"
                Opacity="0.8"
                VerticalAlignment="Center"
                TextTrimming="CharacterEllipsis" />

            <!-- Current Version Column -->
            <TextBlock
                Grid.Column="3"
                Text="{x:Bind Package.VersionString}"
                FontSize="13"
                VerticalAlignment="Center" />

            <!-- New Version Column (Updates page only) -->
            <StackPanel
                Grid.Column="4"
                Orientation="Horizontal"
                Spacing="4"
                VerticalAlignment="Center">
                <TextBlock
                    widgets:IconBuilder.Icon="Forward"
                    FontSize="16"
                    Visibility="{x:Bind NewVersionIconWidth, Mode=OneWay}" />
                <TextBlock
                    Text="{x:Bind Package.NewVersionString}"
                    FontSize="13"
                    FontWeight="SemiBold" />
            </StackPanel>

            <!-- Source Column -->
            <TextBlock
                Grid.Column="5"
                Text="{x:Bind Package.Source.AsString_DisplayName}"
                FontSize="13"
                Opacity="0.7"
                VerticalAlignment="Center" />
        </Grid>
    </widgets:PackageItemContainer>
</DataTemplate>
```

**Key Features:**
- **Responsive Columns**: Uses `*` width with min/max constraints
- **x:Bind**: Compiled bindings for better performance
- **Mode=OneWay/TwoWay**: Optimized binding modes
- **Custom Widgets**: PackageItemContainer for item-level events

### Grid View DataTemplate

The grid view displays items as cards in a uniform grid:

**File:** `AbstractPackagesPage.xaml` (lines 204-320)

```xml
<DataTemplate x:Key="PackageTemplate_Grid" x:DataType="pkgClasses:PackageWrapper">
    <widgets:PackageItemContainer
        AutomationProperties.Name="{x:Bind Package.AutomationName}"
        Background="{ThemeResource ControlFillColorDefaultBrush}"
        CornerRadius="4"
        DoubleTapped="{x:Bind PackageItemContainer_DoubleTapped}"
        Package="{x:Bind Package}"
        PreviewKeyDown="{x:Bind PackageItemContainer_PreviewKeyDown}"
        RightTapped="{x:Bind PackageItemContainer_RightTapped}"
        Wrapper="{x:Bind Self}">

        <Grid Padding="4" HorizontalAlignment="Stretch" ColumnSpacing="4" Opacity="{x:Bind ListedOpacity, Mode=OneWay}">
            <Grid.RowDefinitions>
                <RowDefinition Height="48" />
            </Grid.RowDefinitions>
            <Grid.ColumnDefinitions>
                <ColumnDefinition Width="48" />   <!-- Icon -->
                <ColumnDefinition Width="*" />    <!-- Info -->
                <ColumnDefinition Width="22" />   <!-- Checkbox + Context Menu -->
            </Grid.ColumnDefinitions>

            <!-- Package Icon -->
            <Image
                Grid.Column="0"
                Width="44"
                Height="44"
                HorizontalAlignment="Center"
                VerticalAlignment="Center"
                Source="{x:Bind MainIconSource, Mode=OneWay}"
                Visibility="{x:Bind ShowCustomPackageIcon, Mode=OneWay}" />

            <!-- Package Info -->
            <StackPanel Grid.Column="1" VerticalAlignment="Center">
                <!-- Name -->
                <TextBlock
                    Text="{x:Bind Package.Name}"
                    FontSize="14"
                    FontWeight="SemiBold"
                    TextTrimming="CharacterEllipsis" />

                <!-- ID -->
                <TextBlock
                    Text="{x:Bind Package.Id}"
                    FontSize="11"
                    Opacity="0.8"
                    TextTrimming="CharacterEllipsis" />

                <!-- Version -->
                <TextBlock
                    Text="{x:Bind VersionComboString, Mode=OneWay}"
                    FontSize="11"
                    Opacity="0.5" />
            </StackPanel>

            <!-- Checkbox -->
            <CheckBox
                Grid.Column="2"
                VerticalAlignment="Top"
                IsChecked="{x:Bind IsChecked, Mode=TwoWay}" />

            <!-- Context Menu Button -->
            <Button
                Grid.Column="2"
                Width="22"
                Height="22"
                VerticalAlignment="Bottom"
                Background="Transparent"
                BorderThickness="0"
                Click="{x:Bind RightClick}">
                <TextBlock widgets:IconBuilder.Glyph="&#xE712;" FontSize="18" />
            </Button>
        </Grid>
    </widgets:PackageItemContainer>
</DataTemplate>
```

### Icons View DataTemplate

The icons view displays large, visually-prominent tiles:

**File:** `AbstractPackagesPage.xaml` (lines 322-439)

```xml
<DataTemplate x:Key="PackageTemplate_Icons" x:DataType="pkgClasses:PackageWrapper">
    <widgets:PackageItemContainer
        AutomationProperties.Name="{x:Bind Package.AutomationName}"
        Background="{ThemeResource ControlFillColorDefaultBrush}"
        CornerRadius="4"
        DoubleTapped="{x:Bind PackageItemContainer_DoubleTapped}"
        Package="{x:Bind Package}"
        PreviewKeyDown="{x:Bind PackageItemContainer_PreviewKeyDown}"
        RightTapped="{x:Bind PackageItemContainer_RightTapped}"
        Wrapper="{x:Bind Self}">

        <Grid Padding="4" Opacity="{x:Bind ListedOpacity, Mode=OneWay}">
            <Grid.RowDefinitions>
                <RowDefinition Height="22" />    <!-- Checkbox -->
                <RowDefinition Height="60" />    <!-- Icon -->
                <RowDefinition Height="30" />    <!-- Name -->
                <RowDefinition Height="15" />    <!-- Version -->
            </Grid.RowDefinitions>
            <Grid.ColumnDefinitions>
                <ColumnDefinition Width="120" />
            </Grid.ColumnDefinitions>

            <!-- Large Icon -->
            <Image
                Grid.Row="1"
                Width="64"
                Height="64"
                HorizontalAlignment="Center"
                VerticalAlignment="Center"
                Source="{x:Bind MainIconSource, Mode=OneWay}"
                Visibility="{x:Bind ShowCustomPackageIcon, Mode=OneWay}" />

            <!-- Checkbox -->
            <CheckBox
                Grid.Row="0"
                HorizontalAlignment="Left"
                VerticalAlignment="Top"
                IsChecked="{x:Bind IsChecked, Mode=TwoWay}" />

            <!-- Context Menu Button -->
            <Button
                Grid.Row="0"
                Width="22"
                Height="22"
                HorizontalAlignment="Right"
                VerticalAlignment="Top"
                Background="Transparent"
                BorderThickness="0"
                Click="{x:Bind RightClick}">
                <TextBlock widgets:IconBuilder.Glyph="&#xE712;" FontSize="18" />
            </Button>

            <!-- Package Name -->
            <TextBlock
                Grid.Row="2"
                MaxWidth="120"
                Text="{x:Bind Package.Name}"
                FontSize="12"
                FontWeight="SemiBold"
                HorizontalAlignment="Center"
                TextWrapping="Wrap"
                HorizontalTextAlignment="Center" />

            <!-- Version -->
            <TextBlock
                Grid.Row="3"
                Text="{x:Bind VersionComboString, Mode=OneWay}"
                FontSize="11"
                Opacity="0.5"
                HorizontalAlignment="Center" />
        </Grid>
    </widgets:PackageItemContainer>
</DataTemplate>
```

---

## ItemsView and Layouts

### Layout Definitions

Each view mode uses a different layout strategy:

**File:** `AbstractPackagesPage.xaml` (lines 441-458)

```xml
<!-- List Layout: Vertical stack -->
<StackLayout x:Key="Layout_List" Spacing="3" />

<!-- Grid Layout: Responsive grid with minimum sizes -->
<UniformGridLayout
    x:Key="Layout_Grid"
    ItemsStretch="Fill"
    MinColumnSpacing="8"
    MinItemHeight="56"
    MinItemWidth="275"
    MinRowSpacing="8" />

<!-- Icons Layout: Responsive grid for large tiles -->
<UniformGridLayout
    x:Key="Layout_Icons"
    ItemsJustification="Start"
    MinColumnSpacing="8"
    MinItemHeight="134"
    MinItemWidth="128"
    MinRowSpacing="8" />
```

**Layout Characteristics:**

| Layout | Type | Best For | Features |
|--------|------|----------|----------|
| List | StackLayout | Detailed data, many columns | Compact, table-like |
| Grid | UniformGridLayout | Moderate detail, cards | Responsive, 275px min width |
| Icons | UniformGridLayout | Visual browsing | Large icons, 128px min width |

### View Mode Switching

UniGetUI uses CommunityToolkit's `SwitchPresenter` to switch between ItemsView controls:

**File:** `AbstractPackagesPage.xaml` (lines 1060-1100)

```xml
<Toolkit:SwitchPresenter
    HorizontalAlignment="Stretch"
    VerticalAlignment="Stretch"
    TargetType="x:Int32"
    Value="{x:Bind ViewModeSelector.SelectedIndex, Mode=OneWay}">

    <!-- Case 0: List View -->
    <Toolkit:Case IsDefault="True" Value="0">
        <ItemsView
            x:Name="PackageList_List"
            Padding="4,0"
            HorizontalAlignment="Stretch"
            VerticalAlignment="Stretch"
            x:FieldModifier="protected"
            CanReorderItems="False"
            IsItemInvokedEnabled="True"
            ItemTemplate="{StaticResource PackageTemplate_List}"
            ItemsSource="{x:Bind FilteredPackages}"
            Layout="{StaticResource Layout_List}" />
    </Toolkit:Case>

    <!-- Case 1: Grid View -->
    <Toolkit:Case Value="1">
        <ItemsView
            x:Name="PackageList_Grid"
            Padding="4,0"
            HorizontalAlignment="Stretch"
            VerticalAlignment="Stretch"
            x:FieldModifier="protected"
            CanReorderItems="False"
            IsItemInvokedEnabled="True"
            ItemTemplate="{StaticResource PackageTemplate_Grid}"
            ItemsSource="{x:Bind FilteredPackages}"
            Layout="{StaticResource Layout_Grid}" />
    </Toolkit:Case>

    <!-- Case 2: Icons View -->
    <Toolkit:Case Value="2">
        <ItemsView
            x:Name="PackageList_Icons"
            Padding="4,0"
            HorizontalAlignment="Stretch"
            VerticalAlignment="Stretch"
            x:FieldModifier="protected"
            CanReorderItems="False"
            IsItemInvokedEnabled="True"
            ItemTemplate="{StaticResource PackageTemplate_Icons}"
            ItemsSource="{x:Bind FilteredPackages}"
            Layout="{StaticResource Layout_Icons}" />
    </Toolkit:Case>
</Toolkit:SwitchPresenter>
```

### View Mode Selector UI

**File:** `AbstractPackagesPage.xaml` (lines 563-586)

```xml
<StackPanel Orientation="Horizontal" Spacing="4">
    <widgets:TranslatedTextBlock VerticalAlignment="Center" Text="View mode:" />

    <!-- Segmented control for view selection -->
    <Toolkit:Segmented
        x:Name="ViewModeSelector"
        SelectionChanged="ViewModeSelector_SelectionChanged"
        SelectionMode="Single">

        <!-- List View -->
        <Toolkit:SegmentedItem x:Name="Selector_List">
            <Toolkit:SegmentedItem.Icon>
                <FontIcon Glyph="&#xE8FD;"/>  <!-- List icon -->
            </Toolkit:SegmentedItem.Icon>
        </Toolkit:SegmentedItem>

        <!-- Grid View -->
        <Toolkit:SegmentedItem x:Name="Selector_Grid">
            <Toolkit:SegmentedItem.Icon>
                <FontIcon Glyph="&#xF168;"/>  <!-- Grid icon -->
            </Toolkit:SegmentedItem.Icon>
        </Toolkit:SegmentedItem>

        <!-- Icons View -->
        <Toolkit:SegmentedItem x:Name="Selector_Icons">
            <Toolkit:SegmentedItem.Icon>
                <FontIcon Glyph="&#xF0E2;"/>  <!-- Icons icon -->
            </Toolkit:SegmentedItem.Icon>
        </Toolkit:SegmentedItem>
    </Toolkit:Segmented>
</StackPanel>
```

### Code-Behind: View Mode Management

**File:** `AbstractPackagesPage.xaml.cs` (lines 157-165, 237-245)

```csharp
// Property to get the currently active ItemsView
protected ItemsView CurrentPackageList
{
    get => (ViewModeSelector.SelectedIndex switch
    {
        1 => PackageList_Grid,
        2 => PackageList_Icons,
        _ => PackageList_List
    });
}

// Constructor: Load saved view mode from settings
public AbstractPackagesPage(PackagesPageData data)
{
    // ... other initialization ...

    // Load saved view mode preference
    int viewMode = Settings.GetDictionaryItem<string, int>(
        Settings.K.PackageListViewMode,
        PAGE_NAME
    );

    // Validate and set view mode
    if (viewMode < 0 || viewMode >= ViewModeSelector.Items.Count)
        viewMode = 0;

    ViewModeSelector.SelectedIndex = viewMode;

    // Set tooltips for view mode buttons
    ToolTipService.SetToolTip(Selector_List, CoreTools.Translate("List"));
    ToolTipService.SetToolTip(Selector_Grid, CoreTools.Translate("Grid"));
    ToolTipService.SetToolTip(Selector_Icons, CoreTools.Translate("Icons"));
}

// Save view mode when changed
private void ViewModeSelector_SelectionChanged(object sender, SelectionChangedEventArgs e)
{
    Settings.SetDictionaryItem(
        Settings.K.PackageListViewMode,
        PAGE_NAME,
        ViewModeSelector.SelectedIndex
    );
}
```

---

## Data Binding with Wrappers

### The Wrapper Pattern

UniGetUI uses the **Wrapper Pattern** to separate data models from UI concerns. This provides:

✓ UI-specific properties (opacity, tooltip, icon state)
✓ Event forwarding to parent page
✓ Property change notifications
✓ Icon caching and lazy loading
✓ Computed properties for display

### PackageWrapper Class

**File:** `PackageWrapper.cs` (lines 1-100)

```csharp
using System.ComponentModel;
using Microsoft.UI.Xaml.Media;
using UniGetUI.PackageEngine.Interfaces;

namespace UniGetUI.PackageEngine.PackageClasses
{
    /// <summary>
    /// A wrapper for packages to be able to show in ItemCollections
    /// </summary>
    public partial class PackageWrapper : IIndexableListItem, INotifyPropertyChanged, IDisposable
    {
        private static readonly ConcurrentDictionary<long, Uri?> CachedPackageIcons = new();

        // Core package reference
        public IPackage Package { get; private set; }
        public PackageWrapper Self { get; private set; }

        // UI State Properties
        public bool IsChecked
        {
            get => Package.IsChecked;
            set
            {
                Package.IsChecked = value;
                PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(nameof(IsChecked)));
                _page.UpdatePackageCount();
            }
        }

        public float ListedOpacity { get; set; } = 1.0f;
        public string ListedNameTooltip { get; set; } = "";
        public readonly string ExtendedTooltip = "";

        // Icon Management
        public bool IconWasLoaded { get; set; }
        public bool ShowCustomPackageIcon { get; set; }
        public bool ShowDefaultPackageIcon { get; set; } = true;
        public ImageSource? MainIconSource { get; set; }

        public Uri? PackageIcon
        {
            set
            {
                CachedPackageIcons[Package.GetHash()] = value;
                UpdatePackageIcon();
            }
        }

        // Version Display (handles both regular and update scenarios)
        public string VersionComboString { get; set; }

        // Dynamic column widths for Updates page
        public int NewVersionLabelWidth { get => Package.IsUpgradable ? 125 : 0; }
        public int NewVersionIconWidth { get => Package.IsUpgradable ? 24 : 0; }

        // INotifyPropertyChanged implementation
        public event PropertyChangedEventHandler? PropertyChanged;

        // Reference to parent page for event handling
        private readonly AbstractPackagesPage _page;

        public PackageWrapper(IPackage package, AbstractPackagesPage page)
        {
            Package = package;
            Self = this;
            _page = page;

            // Initialize version display string
            VersionComboString = package.IsUpgradable
                ? $"{package.VersionString} -> {package.NewVersionString}"
                : package.VersionString;

            // Create extended tooltip
            if(package.Name.ToLower() != package.Id.ToLower())
                ExtendedTooltip = $"{package.Name} ({package.Id} from {package.Source.AsString_DisplayName})";
            else
                ExtendedTooltip = $"{package.Name} (from {package.Source.AsString_DisplayName})";

            // Subscribe to package changes
            Package.PropertyChanged += Package_PropertyChanged;

            // Load icon
            UpdatePackageIcon();
        }

        // Event Forwarding to Parent Page
        public void PackageItemContainer_DoubleTapped(object sender, DoubleTappedRoutedEventArgs e)
            => _page.PackageItemContainer_DoubleTapped(sender, e);

        public void PackageItemContainer_PreviewKeyDown(object sender, KeyRoutedEventArgs e)
            => _page.PackageItemContainer_PreviewKeyDown(sender, e);

        public void PackageItemContainer_RightTapped(object sender, RightTappedRoutedEventArgs e)
            => _page.PackageItemContainer_RightTapped(sender, e);

        public async Task RightClick()
        {
            await _page.ShowContextMenu(this);
        }

        private void Package_PropertyChanged(object? sender, PropertyChangedEventArgs e)
        {
            // Forward property changes to UI
            if (e.PropertyName == nameof(Package.IsChecked))
            {
                PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(nameof(IsChecked)));
            }
            else if (e.PropertyName == nameof(Package.NewVersionString))
            {
                VersionComboString = $"{Package.VersionString} -> {Package.NewVersionString}";
                PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(nameof(VersionComboString)));
            }
        }

        // Icon update logic (simplified)
        private void UpdatePackageIcon()
        {
            if (CachedPackageIcons.TryGetValue(Package.GetHash(), out Uri? iconUri) && iconUri != null)
            {
                MainIconSource = new BitmapImage(iconUri);
                ShowCustomPackageIcon = true;
                ShowDefaultPackageIcon = false;
                IconWasLoaded = true;
            }
        }

        public void Dispose()
        {
            Package.PropertyChanged -= Package_PropertyChanged;
        }
    }
}
```

**Key Wrapper Features:**

1. **Icon Caching**: Static dictionary prevents redundant icon loads
2. **Lazy Loading**: Icons loaded asynchronously after initial render
3. **Computed Properties**: `VersionComboString` formats version display
4. **Event Forwarding**: UI events routed to parent page
5. **Change Notifications**: INotifyPropertyChanged updates UI automatically

### PackageItemContainer Control

**File:** `PackageItemContainer.cs`

```csharp
using Microsoft.UI.Xaml.Controls;
using UniGetUI.PackageEngine.Interfaces;
using UniGetUI.PackageEngine.PackageClasses;

namespace UniGetUI.Interface.Widgets
{
    /// <summary>
    /// Custom item container that holds package references
    /// </summary>
    public partial class PackageItemContainer : ItemContainer
    {
        public IPackage? Package { get; set; }
        public PackageWrapper Wrapper { get; set; } = null!;
    }
}
```

This simple wrapper around `ItemContainer` provides:
- Strongly-typed access to Package data
- Reference to the wrapper for UI operations
- Integration with ItemsView selection system

---

## Sorting and Filtering

### ObservablePackageCollection

UniGetUI uses a custom sortable collection that extends `ObservableCollection`:

**File:** `ObservablePackageCollection.cs` (lines 1-131)

```csharp
using UniGetUI.Core.Classes;
using UniGetUI.PackageEngine.Interfaces;

namespace UniGetUI.PackageEngine.PackageClasses
{
    /// <summary>
    /// A special ObservableCollection designed to work with Package objects
    /// </summary>
    public partial class ObservablePackageCollection : SortableObservableCollection<PackageWrapper>
    {
        public enum Sorter
        {
            Checked,
            Name,
            Id,
            Version,
            NewVersion,
            Source,
        }

        public Sorter CurrentSorter { get; private set; }

        public ObservablePackageCollection()
        {
            CurrentSorter = Sorter.Name;
            SortingSelector = x => x.Package.Name;
        }

        /// <summary>
        /// Replaces collection contents efficiently
        /// </summary>
        public void FromRange(IReadOnlyList<PackageWrapper> packages)
        {
            BlockSorting = true;  // Prevent sorting on each Add()

            Clear();
            foreach (var w in packages)
                Add(w);

            BlockSorting = false;
            Sort();  // Sort once after all items added
        }

        /// <summary>
        /// Sets the property with which to sort the collection
        /// </summary>
        public void SetSorter(Sorter field)
        {
            CurrentSorter = field;
            switch (field)
            {
                case Sorter.Checked:
                    SortingSelector = x => x.Package.IsChecked;
                    break;

                case Sorter.Name:
                    SortingSelector = x => x.Package.Name;
                    break;

                case Sorter.Id:
                    SortingSelector = x => x.Package.Id;
                    break;

                case Sorter.Version:
                    SortingSelector = x => x.Package.NormalizedVersion;
                    break;

                case Sorter.NewVersion:
                    SortingSelector = x => x.Package.NormalizedNewVersion;
                    break;

                case Sorter.Source:
                    SortingSelector = x => x.Package.Source.AsString_DisplayName;
                    break;
            }
        }

        /// <summary>
        /// Returns checked packages only
        /// </summary>
        public List<IPackage> GetCheckedPackages()
        {
            List<IPackage> packages = [];
            foreach (PackageWrapper wrapper in this)
            {
                if (wrapper.Package.IsChecked)
                {
                    packages.Add(wrapper.Package);
                }
            }
            return packages;
        }

        /// <summary>
        /// Mark all packages as checked
        /// </summary>
        public void SelectAll()
        {
            foreach (PackageWrapper wrapper in this)
            {
                wrapper.IsChecked = true;
            }
        }

        /// <summary>
        /// Mark all packages as unchecked
        /// </summary>
        public void ClearSelection()
        {
            foreach (PackageWrapper wrapper in this)
            {
                wrapper.IsChecked = false;
            }
        }
    }
}
```

### Sorting UI

**File:** `AbstractPackagesPage.xaml` (lines 538-561)

```xml
<StackPanel Orientation="Horizontal" Spacing="4">
    <widgets:TranslatedTextBlock VerticalAlignment="Center" Text="Order by:" />

    <DropDownButton x:Name="OrderByButton">
        <DropDownButton.Flyout>
            <widgets:BetterMenu Placement="Bottom">
                <!-- Sort Field Options -->
                <widgets:BetterToggleMenuItem x:Name="OrderByName_Menu" Text="Name" />
                <widgets:BetterToggleMenuItem x:Name="OrderById_Menu" Text="Id" />
                <widgets:BetterToggleMenuItem x:Name="OrderByVer_Menu" Text="Version" />
                <widgets:BetterToggleMenuItem
                    x:Name="OrderByNewVer_Menu"
                    Text="New version"
                    Visibility="{x:Bind RoleIsUpdateLike}" />
                <widgets:BetterToggleMenuItem x:Name="OrderBySrc_Menu" Text="Source" />

                <MenuFlyoutSeparator />

                <!-- Sort Direction Options -->
                <widgets:BetterToggleMenuItem x:Name="OrderAsc_Menu" Text="Ascendant" />
                <widgets:BetterToggleMenuItem x:Name="OrderDesc_Menu" Text="Descendant" />
            </widgets:BetterMenu>
        </DropDownButton.Flyout>
    </DropDownButton>
</StackPanel>
```

### Sorting Code-Behind

**File:** `AbstractPackagesPage.xaml.cs` (lines 960-975)

```csharp
/// <summary>
/// Changes how the packages are sorted
/// </summary>
public void SortPackagesBy(ObservablePackageCollection.Sorter sorter)
{
    // Toggle direction if clicking same sorter
    if(sorter == FilteredPackages.CurrentSorter)
        FilteredPackages.Descending = !FilteredPackages.Descending;

    FilteredPackages.SetSorter(sorter);
    FilteredPackages.Sort();
    UpdateSortingMenu();
}

public void SortPackagesBy(bool ascendent)
{
    FilteredPackages.Descending = !ascendent;
    FilteredPackages.Sort();
    UpdateSortingMenu();
}
```

### Filtering Pipeline

UniGetUI uses a two-stage filtering system:

**Stage 1: Search Query Filter** (with caching)
**Stage 2: Source Filter**

**File:** `AbstractPackagesPage.xaml.cs` (lines 800-891)

```csharp
public void FilterPackages(bool forceQueryUpdate = false)
{
    var previousSelection = CurrentPackageList.SelectedItem as PackageWrapper;

    // Stage 1: Build source filter list
    List<IManagerSource> visibleSources = [];
    List<IPackageManager> visibleManagers = [];

    if (SourcesTreeView.SelectedNodes.Count > 0)
    {
        foreach (TreeViewNode node in SourcesTreeView.SelectedNodes)
        {
            if (NodesForSources.Values.Contains(node))
            {
                visibleSources.Add(NodesForSources.First(x => x.Value == node).Key);
            }
            else if (RootNodeForManager.Values.Contains(node))
            {
                IPackageManager manager = RootNodeForManager.First(x => x.Value == node).Key;
                visibleManagers.Add(manager);

                if (manager.Capabilities.SupportsCustomSources)
                {
                    foreach (IManagerSource source in manager.SourcesHelper.Factory.GetAvailableSources())
                        if (!visibleSources.Contains(source))
                            visibleSources.Add(source);
                }
            }
        }
    }

    // Stage 2: Search query filter (with caching)
    if (forceQueryUpdate || LastQueryResult is null)
    {
        // Build filter pipeline
        List<Func<string, string>> appliedFilters = [];
        if (UpperLowerCaseCheckbox.IsChecked is false)
            appliedFilters.Add(FilterHelpers.NormalizeCase);
        if (IgnoreSpecialCharsCheckbox.IsChecked is true)
            appliedFilters.Add(FilterHelpers.NormalizeSpecialCharacters);

        // Apply filters to search query
        string treatedQuery = QueryBlock.Text.Trim();
        foreach (var filter in appliedFilters)
            treatedQuery = filter(treatedQuery);

        // Execute search based on selected mode
        if (QueryIdRadio.IsChecked is true)
            LastQueryResult = WrappedPackages.Where(wrapper =>
                FilterHelpers.IdContains(wrapper.Package, treatedQuery, appliedFilters));
        else if (QueryNameRadio.IsChecked is true)
            LastQueryResult = WrappedPackages.Where(wrapper =>
                FilterHelpers.NameContains(wrapper.Package, treatedQuery, appliedFilters));
        else if (QueryBothRadio.IsChecked is true)
            LastQueryResult = WrappedPackages.Where(wrapper =>
                FilterHelpers.NameOrIdContains(wrapper.Package, treatedQuery, appliedFilters));
        else if (QueryExactMatch.IsChecked == true)
            LastQueryResult = WrappedPackages.Where(wrapper =>
                FilterHelpers.NameOrIdExactMatch(wrapper.Package, treatedQuery, appliedFilters));
        else // QuerySimilarResultsRadio == true
            LastQueryResult = WrappedPackages;
    }

    // Stage 3: Apply source filter to cached query results
    List<PackageWrapper> matchingList_selectedSources = [];

    foreach (var match in LastQueryResult)
    {
        if (visibleSources.Contains(match.Package.Source) ||
            (!match.Package.Manager.Capabilities.SupportsCustomSources &&
             visibleManagers.Contains(match.Package.Manager)))
        {
            matchingList_selectedSources.Add(match);
        }
    }

    // Update visible collection (triggers UI update)
    FilteredPackages.FromRange(matchingList_selectedSources);
    UpdatePackageCount();

    // Restore previous selection if still visible
    if (previousSelection is not null)
    {
        for (int i = 0; i < FilteredPackages.Count; i++)
        {
            if (FilteredPackages[i].Package.Equals(previousSelection.Package))
            {
                SelectAndScrollTo(i, false);
                break;
            }
        }
    }

    // Load icons for newly visible packages
    if (!Settings.Get(Settings.K.DisableIconsOnPackageLists))
        _ = LoadIconsForNewPackages();
}
```

**Filtering Optimizations:**

1. **Query Result Caching**: `LastQueryResult` stores search results
2. **Incremental Filtering**: Source filter applied to cached results
3. **Selection Preservation**: Maintains selection after filtering
4. **Lazy Icon Loading**: Icons loaded only for visible items

---

## Complete Working Example

Let's build a complete task manager app using UniGetUI's table/list patterns:

### Project Structure

```
TaskManagerApp/
├── Models/
│   └── Task.cs
├── Controls/
│   ├── TaskWrapper.cs
│   ├── TaskItemContainer.cs
│   └── ObservableTaskCollection.cs
├── Pages/
│   ├── TasksPage.xaml
│   └── TasksPage.xaml.cs
└── Helpers/
    └── TaskFilterHelpers.cs
```

### Step 1: Create the Data Model

**File:** `Models/Task.cs`

```csharp
using System.ComponentModel;

namespace TaskManagerApp.Models
{
    public class Task : INotifyPropertyChanged
    {
        private bool _isCompleted;
        private string _title = "";
        private string _description = "";
        private DateTime _dueDate;
        private string _priority = "Medium";
        private string _category = "General";

        public string Id { get; set; } = Guid.NewGuid().ToString();

        public string Title
        {
            get => _title;
            set { _title = value; OnPropertyChanged(nameof(Title)); }
        }

        public string Description
        {
            get => _description;
            set { _description = value; OnPropertyChanged(nameof(Description)); }
        }

        public DateTime DueDate
        {
            get => _dueDate;
            set { _dueDate = value; OnPropertyChanged(nameof(DueDate)); }
        }

        public string Priority
        {
            get => _priority;
            set { _priority = value; OnPropertyChanged(nameof(Priority)); }
        }

        public string Category
        {
            get => _category;
            set { _category = value; OnPropertyChanged(nameof(Category)); }
        }

        public bool IsCompleted
        {
            get => _isCompleted;
            set { _isCompleted = value; OnPropertyChanged(nameof(IsCompleted)); }
        }

        public string DueDateString => DueDate.ToString("MMM dd, yyyy");
        public string AutomationName => $"{Title} - {Priority} priority, due {DueDateString}";

        public event PropertyChangedEventHandler? PropertyChanged;

        protected void OnPropertyChanged(string propertyName)
        {
            PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName));
        }
    }
}
```

### Step 2: Create the Wrapper

**File:** `Controls/TaskWrapper.cs`

```csharp
using System.ComponentModel;
using TaskManagerApp.Models;

namespace TaskManagerApp.Controls
{
    public class TaskWrapper : INotifyPropertyChanged
    {
        public Models.Task Task { get; private set; }
        public TaskWrapper Self { get; private set; }

        public bool IsChecked
        {
            get => Task.IsCompleted;
            set
            {
                Task.IsCompleted = value;
                PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(nameof(IsChecked)));
            }
        }

        public string PriorityIcon => Task.Priority switch
        {
            "High" => "&#xE7BA;",    // ⚠ Warning
            "Medium" => "&#xE946;",  // ◉ Circle
            "Low" => "&#xE91F;",     // ↓ Down arrow
            _ => "&#xE946;"
        };

        public string PriorityColor => Task.Priority switch
        {
            "High" => "#FF4444",
            "Medium" => "#FFA500",
            "Low" => "#4CAF50",
            _ => "#888888"
        };

        public string DaysUntilDue
        {
            get
            {
                var days = (Task.DueDate - DateTime.Now).Days;
                return days switch
                {
                    < 0 => $"Overdue by {Math.Abs(days)} days",
                    0 => "Due today",
                    1 => "Due tomorrow",
                    _ => $"Due in {days} days"
                };
            }
        }

        public event PropertyChangedEventHandler? PropertyChanged;

        public TaskWrapper(Models.Task task)
        {
            Task = task;
            Self = this;
            Task.PropertyChanged += Task_PropertyChanged;
        }

        private void Task_PropertyChanged(object? sender, PropertyChangedEventArgs e)
        {
            if (e.PropertyName == nameof(Task.IsCompleted))
            {
                PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(nameof(IsChecked)));
            }
            else if (e.PropertyName == nameof(Task.Priority))
            {
                PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(nameof(PriorityIcon)));
                PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(nameof(PriorityColor)));
            }
        }
    }
}
```

### Step 3: Create Observable Collection

**File:** `Controls/ObservableTaskCollection.cs`

```csharp
using System.Collections.ObjectModel;

namespace TaskManagerApp.Controls
{
    public class ObservableTaskCollection : ObservableCollection<TaskWrapper>
    {
        public enum Sorter
        {
            Title,
            DueDate,
            Priority,
            Category,
            Completed
        }

        public Sorter CurrentSorter { get; private set; } = Sorter.DueDate;
        public bool Descending { get; set; } = false;

        public void SetSorter(Sorter field)
        {
            CurrentSorter = field;
            Sort();
        }

        public void Sort()
        {
            var sorted = CurrentSorter switch
            {
                Sorter.Title => this.OrderBy(x => x.Task.Title),
                Sorter.DueDate => this.OrderBy(x => x.Task.DueDate),
                Sorter.Priority => this.OrderBy(x => x.Task.Priority),
                Sorter.Category => this.OrderBy(x => x.Task.Category),
                Sorter.Completed => this.OrderBy(x => x.Task.IsCompleted),
                _ => this.OrderBy(x => x.Task.DueDate)
            };

            var finalList = Descending ? sorted.Reverse().ToList() : sorted.ToList();

            for (int i = 0; i < finalList.Count; i++)
            {
                var item = finalList[i];
                int currentIndex = IndexOf(item);
                if (currentIndex != i)
                    Move(currentIndex, i);
            }
        }

        public void FromRange(IReadOnlyList<TaskWrapper> tasks)
        {
            Clear();
            foreach (var task in tasks)
                Add(task);
            Sort();
        }

        public List<Models.Task> GetCompletedTasks()
        {
            return this.Where(w => w.Task.IsCompleted).Select(w => w.Task).ToList();
        }

        public void SelectAll()
        {
            foreach (var wrapper in this)
                wrapper.IsChecked = true;
        }

        public void ClearSelection()
        {
            foreach (var wrapper in this)
                wrapper.IsChecked = false;
        }
    }
}
```

### Step 4: Create TaskItemContainer

**File:** `Controls/TaskItemContainer.cs`

```csharp
using Microsoft.UI.Xaml.Controls;
using TaskManagerApp.Models;

namespace TaskManagerApp.Controls
{
    public partial class TaskItemContainer : ItemContainer
    {
        public Models.Task? Task { get; set; }
        public TaskWrapper Wrapper { get; set; } = null!;
    }
}
```

### Step 5: Create the XAML Page

**File:** `Pages/TasksPage.xaml`

```xml
<Page
    x:Class="TaskManagerApp.Pages.TasksPage"
    xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
    xmlns:controls="using:TaskManagerApp.Controls"
    xmlns:toolkit="using:CommunityToolkit.WinUI.Controls">

    <Page.Resources>
        <!-- List View DataTemplate -->
        <DataTemplate x:Key="TaskTemplate_List" x:DataType="controls:TaskWrapper">
            <controls:TaskItemContainer
                AutomationProperties.Name="{x:Bind Task.AutomationName}"
                Task="{x:Bind Task}"
                Wrapper="{x:Bind Self}">

                <Grid Padding="12,8" ColumnSpacing="8">
                    <Grid.ColumnDefinitions>
                        <ColumnDefinition Width="32" />                  <!-- Checkbox -->
                        <ColumnDefinition Width="24" />                  <!-- Priority Icon -->
                        <ColumnDefinition Width="2*" MinWidth="150" />   <!-- Title -->
                        <ColumnDefinition Width="3*" MinWidth="200" />   <!-- Description -->
                        <ColumnDefinition Width="*" MaxWidth="120" />    <!-- Due Date -->
                        <ColumnDefinition Width="*" MaxWidth="100" />    <!-- Category -->
                    </Grid.ColumnDefinitions>

                    <!-- Checkbox -->
                    <CheckBox
                        Grid.Column="0"
                        IsChecked="{x:Bind IsChecked, Mode=TwoWay}"
                        VerticalAlignment="Center" />

                    <!-- Priority Icon -->
                    <TextBlock
                        Grid.Column="1"
                        FontFamily="Segoe MDL2 Assets"
                        Text="{x:Bind PriorityIcon}"
                        Foreground="{x:Bind PriorityColor}"
                        FontSize="16"
                        VerticalAlignment="Center" />

                    <!-- Title -->
                    <TextBlock
                        Grid.Column="2"
                        Text="{x:Bind Task.Title}"
                        FontSize="14"
                        FontWeight="SemiBold"
                        VerticalAlignment="Center"
                        TextTrimming="CharacterEllipsis" />

                    <!-- Description -->
                    <TextBlock
                        Grid.Column="3"
                        Text="{x:Bind Task.Description}"
                        FontSize="13"
                        Opacity="0.8"
                        VerticalAlignment="Center"
                        TextTrimming="CharacterEllipsis" />

                    <!-- Due Date -->
                    <StackPanel Grid.Column="4" VerticalAlignment="Center">
                        <TextBlock
                            Text="{x:Bind Task.DueDateString}"
                            FontSize="13"
                            FontWeight="Medium" />
                        <TextBlock
                            Text="{x:Bind DaysUntilDue}"
                            FontSize="11"
                            Opacity="0.6" />
                    </StackPanel>

                    <!-- Category -->
                    <Border
                        Grid.Column="5"
                        Background="{ThemeResource AccentFillColorDefaultBrush}"
                        CornerRadius="4"
                        Padding="8,4"
                        VerticalAlignment="Center">
                        <TextBlock
                            Text="{x:Bind Task.Category}"
                            FontSize="12"
                            HorizontalAlignment="Center" />
                    </Border>
                </Grid>
            </controls:TaskItemContainer>
        </DataTemplate>

        <!-- Grid View DataTemplate -->
        <DataTemplate x:Key="TaskTemplate_Grid" x:DataType="controls:TaskWrapper">
            <controls:TaskItemContainer
                AutomationProperties.Name="{x:Bind Task.AutomationName}"
                Background="{ThemeResource CardBackgroundFillColorDefaultBrush}"
                CornerRadius="8"
                Task="{x:Bind Task}"
                Wrapper="{x:Bind Self}">

                <Grid Padding="16">
                    <Grid.RowDefinitions>
                        <RowDefinition Height="Auto" />
                        <RowDefinition Height="Auto" />
                        <RowDefinition Height="*" />
                        <RowDefinition Height="Auto" />
                    </Grid.RowDefinitions>

                    <!-- Header: Checkbox + Priority -->
                    <Grid Grid.Row="0">
                        <CheckBox
                            IsChecked="{x:Bind IsChecked, Mode=TwoWay}"
                            HorizontalAlignment="Left" />
                        <TextBlock
                            FontFamily="Segoe MDL2 Assets"
                            Text="{x:Bind PriorityIcon}"
                            Foreground="{x:Bind PriorityColor}"
                            FontSize="20"
                            HorizontalAlignment="Right" />
                    </Grid>

                    <!-- Title -->
                    <TextBlock
                        Grid.Row="1"
                        Margin="0,12,0,8"
                        Text="{x:Bind Task.Title}"
                        FontSize="16"
                        FontWeight="Bold"
                        TextWrapping="Wrap" />

                    <!-- Description -->
                    <TextBlock
                        Grid.Row="2"
                        Text="{x:Bind Task.Description}"
                        FontSize="13"
                        Opacity="0.8"
                        TextWrapping="Wrap"
                        MaxLines="3"
                        TextTrimming="CharacterEllipsis" />

                    <!-- Footer: Due Date + Category -->
                    <StackPanel Grid.Row="3" Margin="0,12,0,0" Spacing="8">
                        <TextBlock
                            Text="{x:Bind DaysUntilDue}"
                            FontSize="12"
                            FontWeight="SemiBold" />
                        <Border
                            Background="{ThemeResource AccentFillColorDefaultBrush}"
                            CornerRadius="4"
                            Padding="8,4"
                            HorizontalAlignment="Left">
                            <TextBlock
                                Text="{x:Bind Task.Category}"
                                FontSize="11" />
                        </Border>
                    </StackPanel>
                </Grid>
            </controls:TaskItemContainer>
        </DataTemplate>

        <!-- Layouts -->
        <StackLayout x:Key="Layout_List" Spacing="2" />
        <UniformGridLayout
            x:Key="Layout_Grid"
            ItemsStretch="Fill"
            MinColumnSpacing="12"
            MinItemHeight="180"
            MinItemWidth="280"
            MinRowSpacing="12" />
    </Page.Resources>

    <Grid>
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto" />
            <RowDefinition Height="*" />
        </Grid.RowDefinitions>

        <!-- Header -->
        <Grid Grid.Row="0" Padding="24" ColumnSpacing="12">
            <Grid.ColumnDefinitions>
                <ColumnDefinition Width="*" />
                <ColumnDefinition Width="Auto" />
                <ColumnDefinition Width="Auto" />
            </Grid.ColumnDefinitions>

            <!-- Search Box -->
            <TextBox
                x:Name="SearchBox"
                Grid.Column="0"
                PlaceholderText="Search tasks..."
                TextChanged="SearchBox_TextChanged" />

            <!-- Sort Button -->
            <DropDownButton Grid.Column="1" Content="Sort">
                <DropDownButton.Flyout>
                    <MenuFlyout>
                        <MenuFlyoutItem Text="Sort by Title" Click="SortByTitle_Click" />
                        <MenuFlyoutItem Text="Sort by Due Date" Click="SortByDueDate_Click" />
                        <MenuFlyoutItem Text="Sort by Priority" Click="SortByPriority_Click" />
                        <MenuFlyoutItem Text="Sort by Category" Click="SortByCategory_Click" />
                        <MenuFlyoutSeparator />
                        <ToggleFlyoutItem x:Name="DescendingToggle" Text="Descending" Click="ToggleDescending_Click" />
                    </MenuFlyout>
                </DropDownButton.Flyout>
            </DropDownButton>

            <!-- View Mode Selector -->
            <toolkit:Segmented
                x:Name="ViewModeSelector"
                Grid.Column="2"
                SelectionChanged="ViewModeSelector_SelectionChanged"
                SelectionMode="Single">
                <toolkit:SegmentedItem>
                    <toolkit:SegmentedItem.Icon>
                        <FontIcon Glyph="&#xE8FD;" />
                    </toolkit:SegmentedItem.Icon>
                </toolkit:SegmentedItem>
                <toolkit:SegmentedItem>
                    <toolkit:SegmentedItem.Icon>
                        <FontIcon Glyph="&#xF168;" />
                    </toolkit:SegmentedItem.Icon>
                </toolkit:SegmentedItem>
            </toolkit:Segmented>
        </Grid>

        <!-- Content Area with View Switcher -->
        <toolkit:SwitchPresenter
            Grid.Row="1"
            TargetType="x:Int32"
            Value="{x:Bind ViewModeSelector.SelectedIndex, Mode=OneWay}">

            <!-- List View -->
            <toolkit:Case IsDefault="True" Value="0">
                <ItemsView
                    x:Name="TaskList_List"
                    Padding="24,0"
                    ItemTemplate="{StaticResource TaskTemplate_List}"
                    ItemsSource="{x:Bind FilteredTasks}"
                    Layout="{StaticResource Layout_List}" />
            </toolkit:Case>

            <!-- Grid View -->
            <toolkit:Case Value="1">
                <ItemsView
                    x:Name="TaskList_Grid"
                    Padding="24,0"
                    ItemTemplate="{StaticResource TaskTemplate_Grid}"
                    ItemsSource="{x:Bind FilteredTasks}"
                    Layout="{StaticResource Layout_Grid}" />
            </toolkit:Case>
        </toolkit:SwitchPresenter>
    </Grid>
</Page>
```

### Step 6: Create Code-Behind

**File:** `Pages/TasksPage.xaml.cs`

```csharp
using Microsoft.UI.Xaml;
using Microsoft.UI.Xaml.Controls;
using System.Collections.ObjectModel;
using TaskManagerApp.Controls;
using TaskManagerApp.Models;

namespace TaskManagerApp.Pages
{
    public sealed partial class TasksPage : Page
    {
        public ObservableTaskCollection FilteredTasks { get; } = new();
        private readonly ObservableCollection<TaskWrapper> AllTasks = new();

        public TasksPage()
        {
            this.InitializeComponent();
            LoadSampleData();
            FilterTasks();
        }

        private void LoadSampleData()
        {
            var tasks = new[]
            {
                new Models.Task
                {
                    Title = "Finish project proposal",
                    Description = "Complete the Q4 project proposal for management review",
                    DueDate = DateTime.Now.AddDays(2),
                    Priority = "High",
                    Category = "Work"
                },
                new Models.Task
                {
                    Title = "Buy groceries",
                    Description = "Milk, eggs, bread, vegetables",
                    DueDate = DateTime.Now.AddDays(1),
                    Priority = "Medium",
                    Category = "Personal"
                },
                new Models.Task
                {
                    Title = "Review pull requests",
                    Description = "Review pending PRs in GitHub repository",
                    DueDate = DateTime.Now.AddHours(4),
                    Priority = "High",
                    Category = "Work"
                },
                new Models.Task
                {
                    Title = "Call dentist",
                    Description = "Schedule annual checkup appointment",
                    DueDate = DateTime.Now.AddDays(7),
                    Priority = "Low",
                    Category = "Health"
                },
                new Models.Task
                {
                    Title = "Update documentation",
                    Description = "Update API documentation with new endpoints",
                    DueDate = DateTime.Now.AddDays(5),
                    Priority = "Medium",
                    Category = "Work"
                }
            };

            foreach (var task in tasks)
            {
                AllTasks.Add(new TaskWrapper(task));
            }
        }

        private void FilterTasks()
        {
            string query = SearchBox.Text.ToLower().Trim();

            var filtered = string.IsNullOrEmpty(query)
                ? AllTasks.ToList()
                : AllTasks.Where(t =>
                    t.Task.Title.ToLower().Contains(query) ||
                    t.Task.Description.ToLower().Contains(query) ||
                    t.Task.Category.ToLower().Contains(query)
                  ).ToList();

            FilteredTasks.FromRange(filtered);
        }

        private void SearchBox_TextChanged(object sender, TextChangedEventArgs e)
        {
            FilterTasks();
        }

        private void SortByTitle_Click(object sender, RoutedEventArgs e)
        {
            FilteredTasks.SetSorter(ObservableTaskCollection.Sorter.Title);
        }

        private void SortByDueDate_Click(object sender, RoutedEventArgs e)
        {
            FilteredTasks.SetSorter(ObservableTaskCollection.Sorter.DueDate);
        }

        private void SortByPriority_Click(object sender, RoutedEventArgs e)
        {
            FilteredTasks.SetSorter(ObservableTaskCollection.Sorter.Priority);
        }

        private void SortByCategory_Click(object sender, RoutedEventArgs e)
        {
            FilteredTasks.SetSorter(ObservableTaskCollection.Sorter.Category);
        }

        private void ToggleDescending_Click(object sender, RoutedEventArgs e)
        {
            FilteredTasks.Descending = DescendingToggle.IsChecked;
            FilteredTasks.Sort();
        }

        private void ViewModeSelector_SelectionChanged(object sender, SelectionChangedEventArgs e)
        {
            // View mode automatically switches via SwitchPresenter binding
        }
    }
}
```

### Running the Example

This complete example demonstrates:

✓ **Multiple View Modes**: List and Grid layouts
✓ **Wrapper Pattern**: TaskWrapper adds UI-specific properties
✓ **Sortable Collection**: Sort by any field, ascending/descending
✓ **Filtering**: Real-time search across multiple fields
✓ **Data Binding**: x:Bind for performance
✓ **Responsive Design**: Layouts adapt to window size

---

## Advanced Patterns

### Lazy Icon Loading

UniGetUI loads icons asynchronously to avoid blocking the UI:

**File:** `AbstractPackagesPage.xaml.cs`

```csharp
private async Task LoadIconsForNewPackages()
{
    const int ICON_LOAD_BATCH_SIZE = 10;

    foreach (PackageWrapper wrapper in FilteredPackages)
    {
        if (wrapper.IconWasLoaded) continue;

        try
        {
            // Load icon asynchronously
            Uri? iconUri = await wrapper.Package.GetIconUrl();
            if (iconUri != null)
            {
                wrapper.PackageIcon = iconUri;
            }
        }
        catch (Exception ex)
        {
            Logger.Warn($"Failed to load icon for {wrapper.Package.Id}: {ex.Message}");
        }

        // Load in batches to prevent UI freezing
        if (FilteredPackages.IndexOf(wrapper) % ICON_LOAD_BATCH_SIZE == 0)
        {
            await Task.Delay(10); // Give UI thread time to breathe
        }
    }
}
```

**Key Techniques:**
- Batch processing (10 items at a time)
- `await Task.Delay()` to yield to UI thread
- Icon caching prevents redundant loads
- Graceful fallback if icon load fails

### Virtual Scrolling

`ItemsView` automatically provides virtual scrolling, but you can optimize further:

```csharp
// Force ItemsView to redraw after filtering
private async Task ForceRedrawByScroll()
{
    var scrollViewer = FindDescendant<ScrollViewer>(CurrentPackageList);
    if (scrollViewer != null)
    {
        // Scroll slightly to trigger virtualization refresh
        scrollViewer.ChangeView(null, scrollViewer.VerticalOffset + 1, null, true);
        await Task.Delay(10);
        scrollViewer.ChangeView(null, scrollViewer.VerticalOffset - 1, null, true);
    }
}
```

### Selection Preservation

Maintain user selection across filter/sort operations:

```csharp
public void FilterPackages(bool forceQueryUpdate = false)
{
    // Store current selection
    var previousSelection = CurrentPackageList.SelectedItem as PackageWrapper;

    // ... perform filtering ...

    // Restore selection if item still visible
    if (previousSelection is not null)
    {
        for (int i = 0; i < FilteredPackages.Count; i++)
        {
            if (FilteredPackages[i].Package.Equals(previousSelection.Package))
            {
                SelectAndScrollTo(i, smooth: false);
                break;
            }
        }
    }
}

private void SelectAndScrollTo(int index, bool smooth)
{
    CurrentPackageList.SelectedIndex = index;
    CurrentPackageList.StartBringItemIntoView(index, new BringIntoViewOptions
    {
        VerticalAlignmentRatio = 0.5,
        AnimationDesired = smooth
    });
}
```

### Multi-Level Filtering with TreeView

UniGetUI uses a TreeView for source filtering:

```xml
<TreeView
    Name="SourcesTreeView"
    SelectionMode="Multiple"
    SelectionChanged="SourcesTreeView_SelectionChanged">
    <!-- Dynamically populated with package sources -->
</TreeView>
```

```csharp
// Build source tree programmatically
private void BuildSourceTree()
{
    foreach (var manager in UsedManagers)
    {
        var managerNode = new TreeViewNode
        {
            Content = manager.DisplayName,
            IsExpanded = true
        };

        if (manager.Capabilities.SupportsCustomSources)
        {
            foreach (var source in manager.SourcesHelper.Factory.GetAvailableSources())
            {
                var sourceNode = new TreeViewNode
                {
                    Content = source.DisplayName
                };
                managerNode.Children.Add(sourceNode);
                NodesForSources[source] = sourceNode;
            }
        }

        SourcesTreeView.RootNodes.Add(managerNode);
        RootNodeForManager[manager] = managerNode;
    }
}
```

---

## Performance Optimization

### 1. Use x:Bind Instead of Binding

```xml
<!-- ✗ BAD: Classic binding (runtime evaluated) -->
<TextBlock Text="{Binding Package.Name}" />

<!-- ✓ GOOD: Compiled binding (compile-time) -->
<TextBlock Text="{x:Bind Package.Name}" />
```

**Performance Impact:**
- x:Bind is up to **3x faster** than classic Binding
- Compile-time type checking prevents runtime errors
- Reduced memory overhead

### 2. Specify Binding Modes

```xml
<!-- ✗ BAD: Unnecessary two-way binding -->
<TextBlock Text="{x:Bind Package.Name, Mode=TwoWay}" />

<!-- ✓ GOOD: One-time for static data -->
<TextBlock Text="{x:Bind Package.Id, Mode=OneTime}" />

<!-- ✓ GOOD: One-way for read-only dynamic data -->
<TextBlock Text="{x:Bind Package.Version, Mode=OneWay}" />

<!-- ✓ GOOD: Two-way only for editable controls -->
<CheckBox IsChecked="{x:Bind IsSelected, Mode=TwoWay}" />
```

### 3. Batch Collection Updates

```csharp
// ✗ BAD: Triggers UI update for each item
foreach (var item in newItems)
{
    FilteredPackages.Add(item);
}

// ✓ GOOD: Block notifications, update once
public void FromRange(IReadOnlyList<TaskWrapper> items)
{
    BlockSorting = true;  // Prevent sort on each Add()

    Clear();
    foreach (var item in items)
        Add(item);

    BlockSorting = false;
    Sort();  // Sort once at the end
}
```

### 4. Cache Expensive Computations

```csharp
public class TaskWrapper
{
    private static readonly ConcurrentDictionary<long, Uri?> IconCache = new();

    public Uri? PackageIcon
    {
        set
        {
            IconCache[Task.GetHashCode()] = value;
            UpdateIcon();
        }
        get
        {
            if (IconCache.TryGetValue(Task.GetHashCode(), out Uri? uri))
                return uri;
            return null;
        }
    }
}
```

### 5. Debounce Search Input

```csharp
private DispatcherTimer _searchDebounceTimer;

private void SearchBox_TextChanged(object sender, TextChangedEventArgs e)
{
    _searchDebounceTimer?.Stop();
    _searchDebounceTimer = new DispatcherTimer
    {
        Interval = TimeSpan.FromMilliseconds(300)
    };
    _searchDebounceTimer.Tick += (s, args) =>
    {
        _searchDebounceTimer.Stop();
        FilterTasks();
    };
    _searchDebounceTimer.Start();
}
```

### 6. Virtualize Large Lists

`ItemsView` automatically virtualizes, but ensure:

```xml
<!-- ✓ GOOD: Enable item recycling -->
<ItemsView
    ItemsSource="{x:Bind FilteredTasks}"
    ItemTemplate="{StaticResource TaskTemplate_List}"
    Layout="{StaticResource Layout_List}" />
```

**Virtualization Tips:**
- Keep DataTemplates lightweight
- Avoid complex animations in templates
- Use simple layouts (Grid > Canvas)
- Minimize nested ScrollViewers

---

## Best Practices

### ✓ DO

1. **Use Wrappers for UI Logic**
   ```csharp
   // Wrapper handles UI concerns
   public class TaskWrapper {
       public string StatusColor => Task.IsComplete ? "Green" : "Red";
       public string DaysRemaining => CalculateDaysRemaining();
   }
   ```

2. **Separate Data and Presentation**
   ```
   Models/Task.cs         → Pure data
   Controls/TaskWrapper.cs → UI properties
   Pages/TasksPage.xaml    → Presentation
   ```

3. **Cache Filter Results**
   ```csharp
   private IEnumerable<TaskWrapper>? _lastQueryResult;

   public void FilterTasks(bool forceUpdate = false)
   {
       if (!forceUpdate && _lastQueryResult != null)
           return _lastQueryResult;

       _lastQueryResult = AllTasks.Where(/* filter */);
   }
   ```

4. **Use Static Resources**
   ```xml
   <!-- Define once, use many times -->
   <Page.Resources>
       <DataTemplate x:Key="TaskTemplate">...</DataTemplate>
       <StackLayout x:Key="ListLayout" />
   </Page.Resources>
   ```

5. **Implement INotifyPropertyChanged**
   ```csharp
   public class TaskWrapper : INotifyPropertyChanged
   {
       private bool _isChecked;
       public bool IsChecked
       {
           get => _isChecked;
           set
           {
               _isChecked = value;
               PropertyChanged?.Invoke(this,
                   new PropertyChangedEventArgs(nameof(IsChecked)));
           }
       }
   }
   ```

### ✗ DON'T

1. **Don't Use Binding for Static Data**
   ```xml
   <!-- ✗ BAD -->
   <TextBlock Text="{x:Bind Task.Id, Mode=OneWay}" />

   <!-- ✓ GOOD -->
   <TextBlock Text="{x:Bind Task.Id, Mode=OneTime}" />
   ```

2. **Don't Update Collections on UI Thread**
   ```csharp
   // ✗ BAD
   private void LoadTasks()
   {
       foreach (var task in database.GetTasks())
           AllTasks.Add(new TaskWrapper(task));
   }

   // ✓ GOOD
   private async Task LoadTasks()
   {
       var tasks = await Task.Run(() => database.GetTasks());
       await Dispatcher.RunAsync(CoreDispatcherPriority.Normal, () =>
       {
           foreach (var task in tasks)
               AllTasks.Add(new TaskWrapper(task));
       });
   }
   ```

3. **Don't Nest ScrollViewers**
   ```xml
   <!-- ✗ BAD: Breaks virtualization -->
   <ScrollViewer>
       <ItemsView />
   </ScrollViewer>

   <!-- ✓ GOOD: ItemsView has built-in scrolling -->
   <ItemsView />
   ```

4. **Don't Use Complex DataTemplates**
   ```xml
   <!-- ✗ BAD: Heavy template -->
   <DataTemplate>
       <Grid>
           <Image Source="{x:Bind LargeImage}" />
           <MediaElement Source="{x:Bind VideoPreview}" />
           <WebView Source="{x:Bind EmbeddedPage}" />
       </Grid>
   </DataTemplate>

   <!-- ✓ GOOD: Lightweight template -->
   <DataTemplate>
       <Grid>
           <TextBlock Text="{x:Bind Title}" />
           <TextBlock Text="{x:Bind Description}" />
       </Grid>
   </DataTemplate>
   ```

5. **Don't Ignore Memory Management**
   ```csharp
   // ✓ GOOD: Dispose of wrappers
   public class TaskWrapper : IDisposable
   {
       public void Dispose()
       {
           Task.PropertyChanged -= Task_PropertyChanged;
       }
   }

   // Clean up when page unloads
   private void Page_Unloaded(object sender, RoutedEventArgs e)
   {
       foreach (var wrapper in AllTasks)
           wrapper.Dispose();
       AllTasks.Clear();
   }
   ```

---

## Summary

You've learned how UniGetUI builds sophisticated table/list UIs:

1. **Multiple View Modes**: List, Grid, and Icons layouts using DataTemplates
2. **ItemsView Control**: Flexible container with pluggable layouts
3. **Wrapper Pattern**: Separate UI concerns from data models
4. **Sortable Collections**: Custom ObservableCollection with sorting logic
5. **Advanced Filtering**: Multi-stage pipeline with caching
6. **Performance Optimization**: x:Bind, virtualization, lazy loading
7. **Best Practices**: Separation of concerns, proper binding modes

### Key Takeaways

✓ Use **DataTemplates** for different view modes
✓ Implement **Wrapper classes** for UI-specific properties
✓ Leverage **ObservableCollection** for automatic UI updates
✓ Cache **filter results** to avoid redundant processing
✓ Use **x:Bind** for better performance
✓ Load **icons lazily** to prevent UI blocking
✓ Preserve **user selection** across operations
✓ Implement **virtual scrolling** for large datasets

### Further Reading

- [ItemsView Documentation (Microsoft)](https://learn.microsoft.com/en-us/windows/windows-app-sdk/api/winrt/microsoft.ui.xaml.controls.itemsview)
- [Data binding overview (Microsoft)](https://learn.microsoft.com/en-us/windows/uwp/data-binding/data-binding-quickstart)
- [ObservableCollection Class (Microsoft)](https://learn.microsoft.com/en-us/dotnet/api/system.collections.objectmodel.observablecollection-1)
- [WinUI 3 Performance Best Practices](https://learn.microsoft.com/en-us/windows/apps/develop/performance/)

---

**Created:** 2025-11-07
**Based on:** UniGetUI v3.1.0
**Author:** Claude (Anthropic)
**License:** Same as UniGetUI project
