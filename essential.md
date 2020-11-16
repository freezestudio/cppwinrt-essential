# `C++/WinRT` 要点

项目链接 <https://github.com/microsoft/cppwinrt>

术语:

名称|说明
-|-
`WinRT`| `Windows` 运行时
`API`| `Windows` 运行时 `API`
`ABI`| `Windows` 运行时二进制接口
`C++/WinRT`| `Windows` 运行时的投影(接口和类型)
投影类型| `C++/WinRT` 是 `WinRT` 类型(和接口)的包装, 相当于智能指针
实现类型| 实现一个 `Windows` 运行时类型. 可以使用多种语言来实现一个 `WinRT` 类型, 这里使用 `C++/WinRT` 来实现一个 `WinRT` 类型. 不能直接分配一个实现类型, 可以使用 `winrt::make` 获取(实例或接口), 可以使用 `winrt::make_self` 获取一个 `winrt::com_ptr` 包装的实现类型, 可以使用 `winrt::get_self` 从接口返回实现类型.

## `winrt::hstring` 字符串类型

```c++
// 1. 可以从 `L"..."`, `L"..."sv`, `std::wstring` 转换为 `winrt::hstring`
// 2. `winrt::hstring` 有 `operator std::wstring_view` 和 `hstring.c_str()` 用于转换为 `std::wstring`
// 3. `winrt::to_hstring(scalar)` 对于模板仅有 `std::wstring_view` 的特化
```

格式化字符串时, 使用 `std::wostringstream`

## 标准 `C++` 类型和 `C++/WinRT`

* 可以使用标准 `C++` 类型调用 `WinRT`
* 传递标准字符串
* 传递初始化列表 `std::initializer_list`
* 传递标准容器

### 标准初始化列表

```cpp
// DataWriter.WriteBytes 签名:
void WriteBytes(std::Array<byte> const & value);

// 调用时(自动转换过程如下)
// 1. WriteBytes 接受 winrt::array_view<uint8_t> 类型的参数
// 2. 传入的实参为初始化列表类型, 通过 winrt::array_view<T>(std::initializer_list<T>) 构造函数转换
DataWriter.WriteBytes({97,98,99});
```

```cpp
// StorageItemContentProperties.RetrievePropertiesAsync 签名:
IAsyncOperation<IMap<winrt::hstring, IInspectable>> StorageItemContentProperties::RetrievePropertiesAsync(IIterable<winrt::hstring> propertiesToRetrieve) const;

// 调用时(转换过程如下)
// 1. RetrievePropertiesAsync 接受 winrt::param::async_iterable<winrt::hstring> 类型的参数
// 2. 从 std::initializer_list 构建 std::vector
// 3. C++/WinRT 透明地(并且不会引入副本)将 std::vector 转换为 WinRT 集合
IAsyncAction retrieve_properties_async(StorageFile const& storageFile)
{
    auto properties{ co_await storageFile.Properties().RetrievePropertiesAsync({ L"System.ItemUrl" }) };
}
```

### 标准数组和 `std::vector`

当 `C++/WinRT` 投影类型的方法需要传入 `winrt::array_view` 类型时, 可以直接传递标准数组或 `std::vector`:

```cpp

// 从 std::array 构造
template <typename C, size_type N>
winrt::array_view(std::array<C, N>& value) noexcept;

// 从 std::vector 构造
template <typename C> 
winrt::array_view(std::vector<C>& vectorValue) noexcept;
```

### 原始数组和指针

当传入参数是原始数组或指针时, 需要显式转换为 `winrt::array_view`:

```cpp
using namespace winrt;

// ...

byte theRawArray[]{ 99, 98, 97 };
array_view<byte const> fromRawArray{ theRawArray };
dataWriter.WriteBytes(fromRawArray); // the winrt::array_view is passed to WriteBytes.

array_view<byte const> fromRange{ theArray.data(), theArray.data() + 2 }; // just the first two elements.
dataWriter.WriteBytes(fromRange); // the winrt::array_view is passed to WriteBytes.
```

### IVector<T> 和标准迭代构造

可以使用 `C++` 基于范围的 `for` 或者 `std::for_each` 等迭代 `IVector<T>`:

```cpp
// SyndicationFeed.Items 签名:
IVector<SyndicationItem> Items();

// 调用
// main.cpp 中

#include "pch.h"
#include <winrt/Windows.Web.Syndication.h>
#include <iostream>

using namespace winrt;
using namespace Windows::Web::Syndication;

void PrintFeed(SyndicationFeed const& syndicationFeed)
{
    for (SyndicationItem const& syndicationItem : syndicationFeed.Items())
    {
        std::wcout << syndicationItem.Title().Text().c_str() << std::endl;
    }
}
```

## 标量值装箱到 `IInspectable`

* `winrt::box_value(Scale)` e.g. box_value(123) box_value(L"hello")
* `winrt::box_value(WinRT)` e.g. box_value(event_handler)
* 不能装箱常规 `C++` 类型 (struct, class)

`Slider` 控件在鼠标拖动滑杆时会引发 `ValueChanged` 事件改变滑动值, 有时候我们不希望频繁地处理改变值, 而是希望当拖动结束时来处理最终值, 但是 `Slider` 控件不引发拖动结束时的鼠标释放事件. 为了捕获此事件，我们需要以代码的方式手动指定再次引发。

在适当的位置手动注册鼠标释放事件, 比如在构造函数中, 或者页面装载完成时:

```cpp
// 签名:
void AddHandler(Windows::UI::Xaml::RoutedEvent const& routedEvent, Windows::Foundation::IInspectable const& handler, bool handledEventsToo);

// 鼠标释放事件为:
Windows::UI::Xaml::RoutedEvent Windows::UI::Xaml::UIElement::PointerReleasedEvent();
// 事件处理器为:
Windows::UI::Xaml::Input::PointerEventHandler;

// AddHandler(event, handler, bool) 中, handler 要求为 IInspectable 类型, 
// 但是 Windows::UI::Xaml::Input::PointerEventHandler 并没有从 IInspectable 接口派生, 
// 为了适应 AddHandler 方法的要求, 需要使用 winrt::box_value 包装 handler, 
// 即: winrt::box_value<Windows::UI::Xaml::Input::PointerEventHandler>(handler)

// 因此, Slider 控件鼠标释放事件手动注册如下:

// 添加事件处理器, 并再次启用该事件
auto pointerReleasedEvent = UIElement::PointerReleasedEvent();
auto handler = Input::PointerEventHandler(this, &MainPage::UsmSlider_PointerReleased);
UsmSlider().AddHandler(pointerReleasedEvent, winrt::box_value(handler), true);
```

### 确定装箱值的类型

```cpp
float pi = 3.14f;
auto piInspectable = winrt::box_value(pi);
auto piPropertyValue = piInspectable.as<winrt::Windows::Foundation::IPropertyValue>();
WINRT_ASSERT(piPropertyValue.Type() == winrt::Windows::Foundation::PropertyType::Single);
```

## 使用 `C++/WinRT` 操作运行时 `API`

1. `Windows` 运行时 `api` 大多位于 `Windows::..`. 和 `Microsoft::...` 命名空间
`C++/WinRT` 生成对应的等效友好投影类型(位于 `winrt/` 文件夹下)
例如: `Windows::Foundation::Uri` 投影到 `winrt::Windows::Foundation::Uri` 位于 `winrt/Windows.Fuondation.h` 文件中

  ```cpp
  // main.cpp
  #include <winrt/Windows.Foundation.h>
  
  using namespace winrt;
  using namespace Windows::Foundation;
  
  int main()
  {
      winrt::init_apartment();

      // 投影类型: 其值可以视为 WinRT 的实例
      // 以下为 winrt::Windows::Foundation::Uri 投影类型
      // 类型值 contosoUri, combinedUri 可以视为 Windows::Foundation::Uri 的实例

      Uri contosoUri{ L"http://www.contoso.com" };
      Uri combinedUri = contosoUri.CombineUri(L"products");
  }
  ```

  > __说明__
  >* 投影类型 -> `WinRT` 类型的包装器
  >* 投影接口 -> `WinRT` 接口的包装器

2. 通过对象、接口或 `ABI` 访问成员

  ```cpp
  Uri contosoUri{ L"http://www.contoso.com" };
  WINRT_ASSERT(contosoUri.ToString() == L"http://www.contoso.com/"); // QueryInterface is called at this point.
  ```

  投影类型(可以看做智能指针) `contosoUri` 通过投影操作调用运行时类型(`WinRT`)对应的方法 `ToString()`, 实际上, `contosoUir.ToString()` 调用是通过一次性查询调用了运行时接口 `IStringable` 上的方法.

  相当于:

  ```cpp
  // ...
  IStringable stringable = contosoUri; // One-off QueryInterface.
  WINRT_ASSERT(stringable.ToString() == L"http://www.contoso.com/");
  ```

  如果希望直接调用 `ABI` 级别的成员, 如下:

  ```cpp
  #include <unknwn.h>

  // abi 接口
  #include <Windows.Foundation.h>
  // 投影
  #include <winrt/Windows.Foundation.h>

  using namespace winrt::Windows::Foundation;
  
  int main()
  {
      winrt::init_apartment();

      // 投影
      Uri contosoUri{ L"http://www.contoso.com" };
      
      // 投影调用
      int port{ contosoUri.Port() }; // Access the Port "property" accessor via C++/WinRT.
  
      // abi 级别(直接调用 WinRT ), as 相当于 QueryInterface
      winrt::com_ptr<ABI::Windows::Foundation::IUriRuntimeClass> abiUri{
          contosoUri.as<ABI::Windows::Foundation::IUriRuntimeClass>() };
      HRESULT hr = abiUri->get_Port(&port); // Access the get_Port ABI function.
  }
  ```

3. 延迟初始化: 投影类型都有一个使用 `std::nullptr_t` 参数的构造函数, 用于延迟构造对应的运行时类型

4. `winrt::make`: 通过实现类型返回其投影接口(运行时接口的普通 `C++` 实现)或实例(运行时类及其实现类在同一编译单元)
5. 统一构造: 需要激活工厂(生成激活工厂的好方式是向 `IDL` 添加构造函数)
6. 投影类型可以使用 `as` 查询接口

## 使用 `C++/WinRT` 创作 `API`

* 实现 `WinRT` 接口
* 创作一个 `WinRT` 类
* 创作在 `Xaml` 中引用的 `WinRT` 类型

### 实现 `WinRT` 接口

只需要普通 `C++` 类即可. 可以利用 `winrt::implements` 模板简化操作. 比如实现 `WinRT` 接口 `Windows.ApplicationModel.Core.IFrameworkViewSource`:

```cpp
// 概念上的接口签名:
struct IFrameworkViewSource : IInspectable
{
    IFrameworkView CreateView();
};

// 使用 C++/WinRT 来实现它
// App.cpp

// ...

struct App : implements<App, IFrameworkViewSource>
{
    IFrameworkView CreateView()
    {
        return ...
    }
}

// ...
```

### 创作 `WinRT` 类型

1. 声明运行时类型(使用 `IDL` 语言)
2. 实现该运行时类型(使用 `C++/WinRT`)

使用 `IDL` 语言声明一个 `Windows` 运行时类型. 运行时类是一个可通过现代 `COM` 接口进行激活和使用(通常跨可执行文件)的类型.

```idl
// MyRuntimeClass.idl
namespace MyProject
{
    runtimeclass MyRuntimeClass
    {
        // Declaring a constructor (or constructors) in the IDL causes the runtime class to be
        // activatable from outside the compilation unit.
        MyRuntimeClass();
        String Name;
    }
}
```

实现该运行时类型:

```cpp
// MyRuntimeClass.h

// ...

namespace winrt::MyProject::implementation
{
    struct MyRuntimeClass : MyRuntimeClassT<MyRuntimeClass>
    {
        MyRuntimeClass() = default;

        winrt::hstring Name();
        void Name(winrt::hstring const& value);
    };
}

// winrt::MyProject::factory_implementation::MyRuntimeClass is here, too.
```

其中, `MyRuntimeClassT` 结构如下:

```cpp
namespace winrt::MyProject::implementation
{
    template<typename D, typename... T>
    struct MyRuntimeClassT : MyRuntimeClass_base<D, I...>
    {
        // ...
    };
}
```

其中, `MyRuntimeClass_base` 结构如下:

```cpp
namespace winrt::MyProject::implementation
{
    template<typename D, typename... T>
    struct MyRuntimeClass_base
      : winrt::implements<D, winrt::MyPoject::MyRuntimeclass, I...>
      , winrt::impl::requrie<D, ...>
      , winrt::impl::base<D, ...>
      , ...
    {
        // ...
    };
}
```

### 创建在 `Xaml` 中使用的 `WinRT` 类型

1. 创作 `WinRT` 运行时类
2. 使用此运行时类

例如, 一个 `WinRT` 实现类有以下类型:

名称|命名空间|说明
-|-|-
`MyRuntimeClass`|`MyProject`|运行时类型, 位于 `MyRuntimeClass.idl` 文件中, 用 `IDL` 声明 `namespace MyProject{ runtimeclass MyRuntimeClass{} }`
`MyRuntimeClass`|`winrt::MyProject::implementation`|实现类型, 位于 `MyRuntimeClass[.h|.cpp]` 文件中, 用于实现 `WinRT` 类型 `MyRuntimeClass`
`MyRuntimeClass`|`winrt::MyProject`|投影类型, 位于 `.../winrt/impl/MyProject.2.h` 文件中, 是 `WinRT` 类型 `MyRuntimeClass` 的投影

## 自定义事件

假设我们需要创建一个图像特效, 这个创建是在其它线程完成的，我们希望它在创建完成后发出通知.
现在我们自定义此事件通知(`EffectCompletedEvent`).

```cpp
// 一个图像特效创建完成事件, 预期事件处理器带有一个 bool 类型的参数
winrt::event<Windows::Foundation::EventHandler<bool>> EffectCompletedEvent;
// 保存事件令牌(用于在适当的地方销毁事件)
winrt::event_token EffectCompletedEventToken;
```

接下来, 需要实现注册和销毁此事件的接口.

```cpp
// 注册: handler 是符合 EffectCompletedEvent 要求的事件处理器(自由函数, Lambda, 成员函数或其它可调用物)
winrt::event_token OnEffectCompleted(Windows::Foundation::EventHandler<bool> const& handler);
// 销毁: 断开此事件的处理器
void               OnEffectCompleted(winrt::event_token const& token) noexcept;
```

以上描述了我们想要的事件及事件处理器, 现在来实现它.

```cpp
// 注册和销毁很简单

winrt::event_token OnEffectCompleted(Windows::Foundation::EventHandler<bool> const& handler)
{
  return EffectCompletedEvent.add(handler);
}

void OnEffectCompleted(winrt::event_token const& token) noexcept
{
  EffectCompletedEvent.remove(token);
}

```

现在, 我们实现了一个 `EffectCompletedEvent` 事件, 是时候来使用它了.

1. 事件处理器: 在事件发出之前, 必须有事件处理存在, 事件才会执行处理. 通常在类的构造函数或页面加载完成时注册好事件的处理器, 在类的析构函数或页面卸载时销毁事件处理器

```cpp
// 按照事件处理器的签名构造一个事件处理器, 这里示例一个成员函数
// 查看 api 文档来确定事件处理器的签名, 标准签名通常是这样的
// void handler_name(sender, event_args);
void MainPage::EffectCompletedHandler(Windows::Foundation::IInspectable const& sender, bool v)
{
  ...
}

// 这里在构造函数中注册它, 在析构函数中销毁
MainPage::MainPage()
{
  auto completed_handler = Windows::Foundation::EventHandler<bool>(get_weak(), &MainPage::EffectCompletedHandler);
  EffectCompletedEventToken = OnEffectCompleted(completed_handler);
}

MainPage::~MainPage()
{
  OnEffectCompleted(EffectCompletedEventToken);
}
```

不想手动销毁, 也可以利用 `winrt::event_revoker` 自动撤销事件处理器. 查看文档了解如何操作.

2. 发出事件: 在需要发出此事件的地方引发此事件的执行

```cpp
// winrt::event 有一个调用操作符
// template<typename... Args>
// void operator()(Args const&... args);
// 用来引发事件的执行

// 发出事件
EffectCompletedEvent(*this, true);
```

最后, 列出 `winrt::event` 结构

```cpp
template<typename Delegate>
struct event
{
  // 缺省构造
  event() = default;

  // 注册委托(事件处理器)
  event_token add(Delegate const&);
  
  // 撤销
  void remove(event_token const);

  // 引发事件执行
  template<typename... Args>
  void operator()(Args const& ...);

  // 是否有事件处理器
  explicit operator bool() const noexcept;
};
```

## 问题集锦

### `error: MDM2009`

描述:

同一个解决方案:

* 一个 `app` 项目 `A.app`
* 一个 `runtime component` 项目 `B.dll`

`2` 个项目都使用: `Win2D.uwp.1.25.0`, 其中 `A.app` 引用 `B.dll`

编译时出错:

```cmd
重复的引用 *.winmd
MDMERGE : error MDM2009: duplicate type error when using a grandchild reference
```

解决方法:

* 方法一: (可编译通过, 是否可行未测试)

  1. 先编译 B.dll

  > **重要** 完成后在 `.../x64/Debug/<B>/` 目录中 删除 `Microsoft.Graphics.Canvas.winmd` 文件

  2. 仅编译 A.app


* 方法二:

打开 `.../packages/Win2D.uwp.1.25.0/build/native/Win2D.uwp.targets` 文件

添加：

```xml

<ItemDefinitionGroup>
    <Reference>
        <Private Condition="'$(ConfigurationType)' != 'Application'">false</Private>
    </Reference>
</ItemDefinitionGroup>
```

原来的内容:

```xml
<?xml version="1.0" encoding="utf-8"?>
<Project xmlns="http://schemas.microsoft.com/developer/msbuild/2003">

  <PropertyGroup>
    <win2d-Platform Condition="'$(Platform)' == 'Win32'">x86</win2d-Platform>
    <win2d-Platform Condition="'$(Platform)' != 'Win32'">$(Platform)</win2d-Platform>
  </PropertyGroup>

  <ItemGroup>
    <Reference Include="$(MSBuildThisFileDirectory)..\..\lib\uap10.0\Microsoft.Graphics.Canvas.winmd">
      <Implementation>Microsoft.Graphics.Canvas.dll</Implementation>
    </Reference>
    <ReferenceCopyLocalPaths Include="$(MSBuildThisFileDirectory)..\..\runtimes\win10-$(win2d-Platform)\native\Microsoft.Graphics.Canvas.dll" />
  </ItemGroup>

  <ItemDefinitionGroup>
    <ClCompile>
      <AdditionalIncludeDirectories>$(MSBuildThisFileDirectory)..\..\Include;%(AdditionalIncludeDirectories)</AdditionalIncludeDirectories>
    </ClCompile>
  </ItemDefinitionGroup>

  <Import Project="$(MSBuildThisFileDirectory)..\Win2D.common.targets" />

</Project>
```

修改为:

```xml

<?xml version="1.0" encoding="utf-8"?>
<Project xmlns="http://schemas.microsoft.com/developer/msbuild/2003">

  <PropertyGroup>
    <win2d-Platform Condition="'$(Platform)' == 'Win32'">x86</win2d-Platform>
    <win2d-Platform Condition="'$(Platform)' != 'Win32'">$(Platform)</win2d-Platform>
  </PropertyGroup>

  <ItemGroup>
    <Reference Include="$(MSBuildThisFileDirectory)..\..\lib\uap10.0\Microsoft.Graphics.Canvas.winmd">
      <Implementation>Microsoft.Graphics.Canvas.dll</Implementation>
    </Reference>
    <ReferenceCopyLocalPaths Include="$(MSBuildThisFileDirectory)..\..\runtimes\win10-$(win2d-Platform)\native\Microsoft.Graphics.Canvas.dll" />
  </ItemGroup>

  <ItemDefinitionGroup>
    <ClCompile>
      <AdditionalIncludeDirectories>$(MSBuildThisFileDirectory)..\..\Include;%(AdditionalIncludeDirectories)</AdditionalIncludeDirectories>
    </ClCompile>

    // 这里是新加的

    <Reference>
        <Private Condition="'$(ConfigurationType)' != 'Application'">false</Private>
    </Reference>

  </ItemDefinitionGroup>

  <Import Project="$(MSBuildThisFileDirectory)..\Win2D.common.targets" />

</Project>
```
### WinRT originate error - 0x80004005 : 'Cannot find a Resource with the Name/Key TabViewButtonBackground'

可能出现在使用 `WinUI 3` 的项目中

解决方法:

1. 当 App[.xaml|.h|.cpp|.idl] 文件不在项目根目录时会出现此错误, 把它们移到项目根目录.
2. 如果仍然有此错误, 尝试修改项目文件, 添加以下内容:

```xml
<PropertyGroup>
   <DisableEmbeddedXbf>true</DisableEmbeddedXbf>
</PropertyGroup>
```
