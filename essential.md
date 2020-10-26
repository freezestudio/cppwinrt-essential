# `C++/WinRT` 要点

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