---
title: 在Python中调用C++代码
date: 2023-11-15 20:57:49
copyright: true
tags: [Python, C++, pybind11]
categories:
- 技术笔记
- 编程语言

---


&emsp;&emsp;Python与C++的互操作性为开发者提供了多种灵活的选择，在本篇文章中将会总结一些在Python中调用C++代码的方法，包括使用ctypes库直接调用C++动态链接库函数，借助SWIG或Boost.Python自动生成或手动编写接口，以及利用Cython和Pybind11等现代工具创建Python与C++之间的绑定。这些方法简化了在Python中调用C++代码的过程，使得开发者能够轻松利用C++的性能优势。本文将着重介绍 **Pybind11** 这一工具。

<!--more-->


---


# 方案筛选

1. **Ctypes**: Ctypes 是 Python 内置的一个标准库，可以用来调用动态链接库（DLL）中的 C/C++ 函数。通过一套类型映射的方式将Python与二进制动态链接库相连接。
2. **SWIG**（Simplified Wrapper and Interface Generator）：SWIG 是一个能够自动生成 C/C++ 程序和其他高级语言（如 Python）之间的包装器的工具。它可以将 C/C++ 代码包装成可以被 Python 直接调用的模块。但由于支持的语言众多，因此在 Python 端性能表现不是太好。
3. **Boost.Python**: Boost.Python 是 C++ Boost 库中的一个子模块，它提供了一组 C++ 类和函数，用于将 C++ 代码包装成 Python 可以直接调用的模块。但最大的缺点是需要依赖庞大的 Boost 库，编译和依赖关系包袱重。
4. **Cython**: Cython 是一个用于将 Python 代码转换为 C/C++ 代码的编译器，可以通过将 C/C++ 代码嵌入到 Python 中。
5. **Pybind11**：Pybind11 是一个轻量级的开源库，可以将 C++ 代码封装成可以被 Python 直接调用的模块。它提供了简洁而直观的语法，使得将 C++ 代码封装成 Python 接口变得更加容易。
## 对比

1. 底层实现：Ctypes 是使用 Python 自带的标准库，通过**调用动态链接库**（DLL）中的 C/C++ 函数来实现。SWIG、Boost.Python、Cython 和 Pybind11 则是通过**生成封装代码**来实现，将 C/C++ 代码封装成可以被 Python 直接调用的模块。
2. 使用难度：Ctypes 的使用相对较简单，只需要导入函数原型并调用即可。SWIG 在配置和使用上较为复杂，需要编写接口文件和配置文件。Boost.Python 和 Pybind11 的使用相对较简单。
## 开源库的选择参考

- HiGHS，选择了 Pybind11；
- Tensorflow：已于 2019 年将 SIWG 切换为 pybind11；
- 目前市面上大部分 AI 计算框架，如 TensorFlow、Pytorch、阿里 X-Deep Learning、百度 PaddlePaddle 等，均使用 pybind11 来提供 C++到 Python 端接口封装。


<br/><br/>

# pybind11 使用总结
参考：[Pybind11 文档](https://daobook.github.io/pybind11/basics.html)

## 模块引入
&emsp;&emsp;pybind11 是一个 header-only 的库，只需要 C++ 项目里直接 include pybind11 的头文件就能使用。可以 `git submodule` 添加子模块：
```bash
git submodule add https://github.com/pybind/pybind11.git pybind11
cd pybind11/
git checkout tags/v2.10.0

mkdir build
cd build
cmake ..
cmake --build . --config Release --target check
make check -j 4
```
在 CMakeLists.txt 里 `add_subdirectory` pybind11 的路径，再用其提供的 `pybind11_add_module` 就能创建 pybind11 的模块了。
```bash
cmake_minimum_required(VERSION 3.25)
project(pybind_test)

set(MY_PYBIND ${MY_CURR}/third_party/pybind11-2.5.0)

add_subdirectory(${MY_PYBIND})
pybind11_add_module(example_pb example_pb.cpp)
```
如果想在已有 C++ 动态库上扩展 pybind11 绑定，那么`target_link_libraries`链接该动态库就可以了。
（示例代码：[https://github.com/ikuokuo/start-pybind11](https://github.com/ikuokuo/start-pybind11)）



## 使用 pybind11 封装C++

### C++ 文件

```cpp
#include "vdot.h"
double dot(std::vector<double> &a, std::vector<double> &b) {
  double res = 0;
  for (int i = 0; i < (int)a.size(); ++i) {
    res += a[i] * b[i];
  }
  return res;
}
```

### pybind11 文件

```cpp
#include <pybind11/pybind11.h>
#include <pybind11/stl.h>
#include "cpp/vdot_cpp/vdot.h"

namespace py = pybind11;
PYBIND11_MODULE(np, m) { // (Python包名为np, 实例对象)
  m.doc() = "";
  m.def("vdot", &dot); // m.def("Python函数名", &C函数名);
}
```
### 编写CMake
#### 编译 C++ 的库
使用 start-pybind11 提供的宏进行编译C++动态库（静态库也可以）。
```cmake
# add_my_library(LIB_NAME [SRCS srcs] [LIBS libs] [SHARED] [THREAD])
add_my_library(vdotlib  # C++编译后的库名
        SRCS vdot.cpp   # C++源文件
        SHARED  # 动态库
        THREAD)
```
在父层文件夹添加该子目录
```cmake
add_subdirectory(${MY_CURR}/vdot_cpp)
```
#### 编译 pybind11 的.so库
使用 start-pybind11 提供的宏进行编译C++动态库（静态库也可以）。
```cmake
add_pb_library(np # 库的名字（ Python 的包名 ）
        SRCS vdot_py.cpp  # binding文件
        LIBS vdotlib			# C++编译后的库名，静态动态均可
        SHARED
        THREAD)
```
在父层文件夹添加该子目录
```cmake
add_subdirectory(${MY_CURR}/vdot)
```

### 让 Python 的.so库可以找到C++库
把C++编译后的库文件导入动态连接库的搜索路径
```bash
p_c="/Users/sxj/CLionProjects/start-pybind11-new/_output/lib/vdot_cpp"
export DYLD_LIBRARY_PATH=$p_c${DYLD_LIBRARY_PATH:+:${DYLD_LIBRARY_PATH}}
echo $DYLD_LIBRARY_PATH 
```
或者直接移动 .dylib 库（或 .a 库）到 .so 库相同目录。

### 把 .so库 加入 Python 的搜索路径
```bash
p_so="/Users/sxj/CLionProjects/MDecomper0922/_output/lib/pybind"
export PYTHONPATH=$p_so${PYTHONPATH:+:${PYTHONPATH}}
echo $PYTHONPATH 
```

然后就可以使用 `import` 加 .so 的名字来使用了。


## 支持的数据类型
参考：[https://daobook.github.io/pybind11/advanced/cast/index.html](https://daobook.github.io/pybind11/advanced/cast/index.html)
&emsp;&emsp;`float`, `double`，`bool`，`char`，`const char *`，`std::string`，`std::pair<T1, T2>`，`std::tuple<...>`，`std::complex<T>`，`std::array<T, Size>`，`std::vector<T>`，`std::set<T>`，`std::function<...>`，`Eigen::Matrix<...>`，`Eigen::SparseMatrix<...>`  ……

**STL 容器**
pybind11 支持 STL 容器自动转换，当需要处理 STL 容器时，只要额外包括头文件`<pybind11/stl.h>`即可。

**bytes、string 类型传递**
由于在 Python3 中 string 类型默认为 UTF-8 编码，如果从 C++端传输 string 类型的 protobuf 数据到 Python，则会出现 “UnicodeDecodeError” 的报错，所以需要使用 `py::bytes`。
```cpp
m.def("return_bytes",
    []() {
        std::string s("\xba\xd0\xba\xd0");  // Not valid UTF-8
        return py::bytes(s);  // Return the data without transcoding
    }
);
```

**智能指针**
[智能指针 - pybind11中文文档](https://charlottelive.github.io/pybind11-Chinese-docs/10.%E6%99%BA%E8%83%BD%E6%8C%87%E9%92%88.html)


## 函数
### 声明函数参数名称和默认值
```cpp
m.def("add", &add, "A function which adds two numbers",
      py::arg("i") = 1, py::arg("j") = 2);
```
### 返回指针
```cpp
Data *get_data() { return _data; }
m.def("get_data", &get_data, py::return_value_policy::reference);
```

### 运算符重载
```cpp
class Vector2 {
public:
    Vector2(float x, float y) : x(x), y(y) { }

    Vector2 operator+(const Vector2 &v) const { return Vector2(x + v.x, y + v.y); }
    Vector2 operator*(float value) const { return Vector2(x * value, y * value); }
    Vector2& operator+=(const Vector2 &v) { x += v.x; y += v.y; return *this; }
    Vector2& operator*=(float v) { x *= v; y *= v; return *this; }

    friend Vector2 operator*(float f, const Vector2 &v) {
        return Vector2(f * v.x, f * v.y);
    }

    std::string toString() const {
        return "[" + std::to_string(x) + ", " + std::to_string(y) + "]";
    }
private:
    float x, y;
};
```
```cpp
#include <pybind11/operators.h>

PYBIND11_MODULE(example, m) {
    py::class_<Vector2>(m, "Vector2")
        .def(py::init<float, float>())
        .def(py::self + py::self)
        .def(py::self += py::self)
        .def(py::self *= float())
        .def(float() * py::self)
        .def(py::self * float())
        .def(-py::self)
        .def("__repr__", &Vector2::toString);
}
```


## 面向对象
### 公有变量
```cpp
.def_readwrite("name", &Pet::name)
```
### 私有变量
```cpp
.def_property("name", &Pet::getName, &Pet::setName)
```
### 继承
```cpp
struct Pet {
    Pet(const std::string &name) : name(name) { }
    std::string name;
};

struct Dog : Pet {
    Dog(const std::string &name) : Pet(name) { }
    std::string bark() const { return "woof!"; }
};
```
```cpp
py::class_<Pet>(m, "Pet")
   .def(py::init<const std::string &>())
   .def_readwrite("name", &Pet::name);
py::class_<Dog, Pet>(m, "Dog")
    .def(py::init<const std::string &>())
    .def("bark", &Dog::bark);
```
### 重载
```cpp
struct Pet {
    void set(int age_) { age = age_; }
    void set(const std::string &name_) { name = name_; }
};
py::class_<Pet>(m, "Pet")
    .def("set", py::overload_cast<int>(&Pet::set), "Set the pet's age")
    .def("set", py::overload_cast<const std::string &>(&Pet::set), "Set the pet's name");
```
```cpp
struct Widget {
    int foo(int x, float y);
    int foo(int x, float y) const;
};

py::class_<Widget>(m, "Widget")
   .def("foo", py::overload_cast<int, float>(&Widget::foo))
   .def("foo",   py::overload_cast<int, float>(&Widget::foo, py::const_));
```
### 内部类和内部枚举
```cpp
struct Pet {
    struct Attributes {
        float age = 0;
    };
    enum Kind {
        Dog = 0,
        Cat
    };
};
py::class_<Pet> pet(m, "Pet");
py::class_<Pet::Attributes> attributes(pet, "Attributes")
    .def(py::init<>())
    .def_readwrite("age", &Pet::Attributes::age);
py::enum_<Pet::Kind>(pet, "Kind")
    .value("Dog", Pet::Kind::Dog)
    .value("Cat", Pet::Kind::Cat)
    .export_values();
```

## 手动编译
```bash
c++ -O3 -Wall -shared -std=c++11 -fPIC $(python3-config --includes) -Iextern/pybind11/include example.cpp -o example$(python3-config --extension-suffix)
```
## py::cast
用于在C++代码中进行Python对象类型的转换 
```cpp
#include <pybind11/pybind11.h>

namespace py = pybind11;

int main() {
    py::initialize_interpreter(); // 初始化Python解释器

    // 将Python整数对象转换为C++整数
    py::object py_int = py::int_(42);
    int cpp_int = py::cast<int>(py_int);
    std::cout << "C++ int: " << cpp_int << std::endl;

    // 将C++整数转换为Python整数对象
    int cpp_int2 = 123;
    py::object py_int2 = py::cast<py::object>(cpp_int2);
    std::cout << "Python int: " << py::str(py_int2) << std::endl;

    py::finalize_interpreter(); // 清理Python解释器

    return 0;
}
```


<br/><br/>

# 开源示例
## 示例 start-pybind11  运行命令记录
[GitHub - ikuokuo/start-pybind11: Start pybind11](https://github.com/ikuokuo/start-pybind11)
**【注意】** 切换 Python 的环境为 3.9，首先需要在CLion中设置`Python Interpreter`为指定版本的 conda 环境（本地测试成功的为`py39`）。完全退出CLion，在命令行`conda activate py39` 切换环境后再次打开`open CLion.app`后即可更改运行的Python环境。

**编译**
```bash
-DBUILD_PYTHON_BINDINGS=True
```
```bash
cd start-pybind11/
make install
```

**运行**

加入 Python 的环境变量
```bash
source setup.bash first_steps
```
```bash
import first_steps_pb as pb
pb.add(1, 2)
```

## HiGHS 运行命令记录
**编译**

```bash
-DBUILD_PYTHON=True -DBUILD_DEPS=ON
```
```bash
mkdir build
cd build
cmake ..
cmake --build .
```

**安装**

```bash
sudo cmake --install .
```
Python包安装（cd 到项目根目录，借助 `setup.py` 进行安装）
```bash
pip install -e ./
```
同时需要安装依赖
```bash
pip install pybind11
pip install pyomo
```
测试Python接口
```bash
pytest -v ./highspy/tests/
```

<br/><br/>

# 参考网站

* [pybind11的最佳实践](https://zhuanlan.zhihu.com/p/192974017)
* [pybind11文档](https://charlottelive.github.io/pybind11-Chinese-docs/04.%E9%A6%96%E6%AC%A1%E5%B0%9D%E8%AF%95.html)



<br/><br/><br/><br/>
