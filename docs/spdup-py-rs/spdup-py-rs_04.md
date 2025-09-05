# 第四章：*第五章*：为我们的 pip 模块创建 Rust 接口

在*第四章*，“在 Python 中构建 pip 模块”，我们构建了一个 Python 的`pip`模块。现在，我们将构建相同的`pip`模块在 Rust 中，并管理接口。有些人可能更喜欢 Python 来完成某些任务；其他人可能会说 Rust 更好。在本章中，我们将简单地根据需要使用两者。为了实现这一点，我们将构建一个 Rust 的`pip`模块，它可以被安装并直接导入到我们的 Python 代码中。我们还将构建 Python 入口点，它们直接与我们的编译后的 Rust 代码通信，以及 Python 适配器/接口，以使我们的模块的用户体验变得简单、安全，并通过具有所有我们希望用户使用的功能的**用户界面**（**UIs**）来锁定。

本章将涵盖以下主题：

+   使用`pip`打包 Rust

+   使用`pyO3`crate 构建 Rust 接口

+   为我们的 Rust 包构建测试

+   与 Python、Rust 和 Numba 比较速度

覆盖这些主题将使我们能够构建 Rust 模块并在我们的 Python 系统中使用它们。这对于 Python 开发者来说是一个主要优势；你可以在 Python 程序中无缝地使用更快、更安全、资源消耗更少的代码。

# 技术要求

我们需要安装**Python 3**。为了充分利用本章内容，我们还需要拥有一个 GitHub 账户，因为我们将会使用 GitHub 来打包我们的代码，可以通过此链接访问：[`github.com/maxwellflitton/flitton-fib-rs`](https://github.com/maxwellflitton/flitton-fib-rs)。

本章的代码可以在[`github.com/PacktPublishing/Speed-up-your-Python-with-Rust/tree/main/chapter_five`](https://github.com/PacktPublishing/Speed-up-your-Python-with-Rust/tree/main/chapter_five)找到。

# 使用 pip 打包 Rust

在本节中，我们将设置我们的`pip`包，使其能够利用 Rust 代码。这将使我们能够使用 Python 设置工具导入我们的 Rust `pip`包，为我们的系统编译它，并在 Python 代码中使用它。对于本章，我们实际上是在构建与*第四章*，“在 Python 中构建 pip 模块”中构建的相同的斐波那契模块。建议为我们的 Rust 模块创建另一个 GitHub 仓库；然而，没有任何阻止你重构现有的 Python `pip`模块。为了构建我们的 Rust `pip`模块，我们必须执行以下步骤：

1.  定义我们的包的`gitignore`和`Cargo`。

1.  为我们的包配置 Python 设置过程。

1.  为我们的包创建一个 Rust 库。

## 定义我们的包的 gitignore 和 Cargo

要开始，我们必须确保我们的 Git 不会跟踪我们不希望上传的文件，并且我们的`Cargo`具有正确的依赖项，*步骤 1*。

1.  首先，我们可以从`gitignore`开始。如果你选择使用与我们在上一章中定义的相同的 GitHub 仓库，那么 Python 的所有文件已经定义在 GitHub 仓库根目录下的`.gitignore`文件中。如果不是这样，那么在你创建新的 GitHub 仓库时，我们必须在`Add .gitignore`部分选择 Python 模板。无论如何，一旦我们在`.gitignore`文件中有 Python 的`gitignore`模板，我们必须添加我们包 Rust 部分的`gitignore`要求。为此，我们在`.gitignore`文件中添加以下代码：

    ```rs
     /target/
    ```

    是的，这就是我们 Rust 代码的全部内容。这比我们需要忽略的 Python 文件要少得多。

1.  现在我们已经定义了`gitignore`，我们可以继续定义我们包根目录下的`Cargo.toml`文件，最初用以下代码定义我们包的元数据：

    ```rs
    [package]
    name = "flitton_fib_rs"
    version = "0.1.0"
    authors = ["Maxwell Flitton 
      <maxwellflitton@gmail.com>"]
    edition = "2018"
    ```

    这没有什么新东西；我们在这里所做的只是定义我们包的名称和通用信息。

1.  然后我们继续用以下代码定义依赖项：

    ```rs
    [dependencies]
    [dependencies.pyo3]
    version = "0.13.2"
    features = ["extension-module"]
    ```

    我们可以看到，我们在`dependencies`部分没有定义任何依赖项。我们将依赖于`pyo3`crate 来使我们的 Rust 代码能够与我们的 Python 代码交互。我们在撰写本书时声明了 crate 的最新版本，以及我们想要启用`extension-module`功能，因为我们将会使用`pyo3`来创建我们的 Rust 模块。

1.  然后我们用以下代码定义我们的库数据：

    ```rs
    [lib]
    name = "flitton_fib_rs"
    crate-type = ["cdylib"]
    ```

    必须注意，我们已经定义了一个`crate-type`变量。Crate 类型为编译器提供如何链接 Rust crates 的信息。这可以是静态的或动态的。例如，如果我们将`crate-type`变量定义为`bin`，这将把我们的 Rust 代码编译成一个可运行的执行文件。主文件必须存在于我们的模块中，因为这将作为入口点。我们也可以将`crate-type`变量定义为`lib`，这样它就会被编译成一个可以被其他 Rust 程序使用的库。我们可以进一步定义，要么是静态库要么是动态库。将`crate-type`变量定义为`cdylib`告诉编译器我们想要一个由其他语言加载的动态系统库。如果我们不添加这个，我们将在通过`pip`安装我们的库时无法编译我们的代码。我们的库应该能够为 Linux 和 Windows 编译。然而，我们需要一些链接参数来确保我们的库也能在 macOS 上工作。

1.  为了做到这一点，我们需要在`.cargo/config`文件中定义配置：

    ```rs
    [target.x86_64-apple-darwin]
    rustflags = [
        "-C", "link-arg=-undefined",
        "-C", "link-arg=dynamic_lookup",
    ]
    [target.aarch64-apple-darwin]
    rustflags = [
        "-C", "link-arg=-undefined",
        "-C", "link-arg=dynamic_lookup",
    ]
    ```

这样，我们已经为我们的 Rust 库定义了所有需要的内容。现在，我们继续下一步，配置我们模块的 Python 部分。

## 配置我们的包的 Python 设置过程

当涉及到设置 Python 部分时，我们将在模块根目录的`setup.py`文件中定义这个。最初，我们将用以下代码导入我们需要的所有需求：

```rs
#!/usr/bin/env python
```

```rs
from setuptools import dist
```

```rs
dist.Distribution().fetch_build_eggs(['setuptools_rust'])
```

```rs
from setuptools import setup
```

```rs
from setuptools_rust import Binding, RustExtension
```

我们将使用`setuptools_rust`模块来管理我们的 Rust 代码。然而，我们无法确定用户是否已经安装了`setuptools_rust`，我们需要它来运行我们的设置代码。由于这个原因，我们不能依赖于需求列表，因为安装需求发生在我们导入`setuptools_rust`之后。为了解决这个问题，我们使用`dist`模块来获取这个脚本所需的`setuptools_rust`模块。用户不会永久安装`setuptools_rust`，而是为了脚本使用它。现在这已经完成，我们可以用以下代码定义我们的设置：

```rs
setup(
```

```rs
    name="flitton-fib-rs",
```

```rs
    version="0.1",
```

```rs
    rust_extensions=[RustExtension(
```

```rs
        ".flitton_fib_rs.flitton_fib_rs",
```

```rs
        path="Cargo.toml", binding=Binding.PyO3)],
```

```rs
    packages=["flitton_fib_rs"],
```

```rs
    classifiers=[
```

```rs
            "License :: OSI Approved :: MIT License",
```

```rs
            "Development Status :: 3 - Alpha",
```

```rs
            "Intended Audience :: Developers",
```

```rs
            "Programming Language :: Python",
```

```rs
            "Programming Language :: Rust",
```

```rs
            "Operating System :: POSIX",
```

```rs
            "Operating System :: MacOS :: MacOS X",
```

```rs
        ],
```

```rs
    zip_safe=False,
```

```rs
)
```

在这里，我们可以看到我们定义了模块的元数据，就像我们在上一章中所做的那样。我们还可以看到我们定义了一个`rust_extensions`参数，指向我们将在 Rust 文件中定义的实际 Rust 模块，正如我们在以下图中可以看到的那样：

![图 5.1 – 我们设置模块的流程

![图 5.1 – 我们设置模块的流程

图 5.1 – 我们设置模块的流程

我们还指向我们的`Cargo.toml`文件，因为当我们安装我们的 Rust 模块时，我们必须编译我们依赖项中的其他 Rustcrate。我们还必须声明我们的模块不是安全压缩的。这同样是 C 模块的标准做法。现在我们已经完成了所有设置配置，我们可以继续到下一步，构建我们的基本 Rust 模块，这将使我们能够使用`pip install`安装 Rust 代码，并在我们的 Python 代码中使用它：

1.  对于我们的 Rust 代码，我们最初需要在`src/lib.rs`文件中导入所有的`pyo3`需求，以下代码为示例：

    ```rs
    use pyo3::prelude::*;
    use pyo3::wrap_pyfunction;
    ```

    这所做的是使我们的 Rust 代码能够利用`pyo3`crate 中所有的宏。我们还将把 Rust 函数封装到模块中。

1.  然后我们用以下代码定义一个基本的*hello world*函数：

    ```rs
    #[pyfunction]
    fn say_hello() {
        println!("saying hello from Rust!");
    }
    ```

    我们可以看到，我们已经将`pyo3`的 Python 函数宏应用到`say_hello`函数上。

1.  现在我们有了函数，我们可以用以下代码在同一个文件中定义我们的模块：

    ```rs
    #[pymodule]
    fn flitton_fib_rs(_py: Python, m: &PyModule) -> \
      PyResult<()> {
        m.add_wrapped(wrap_pyfunction!(say_hello));
        Ok(())
    }
    ```

    在这里，我们可以看到我们定义了模块为`flitton_fib_rs`。在使用时，这将必须以`flitton_fib_rs`的形式导入。然后我们使用`pymodule`宏。这个函数正在加载模块。我们必须在最后定义一个结果。鉴于我们没有任何复杂的逻辑，我们将定义结果为`Ok`。我们不需要对 Python 做任何事情；然而，我们将我们的封装`say_hello`函数添加到我们的模块中。`wrap_pyfunction`宏本质上接受一个 Python 实例并返回一个 Python 函数。现在我们已经定义了 Rust 代码，我们必须构建我们的 Python 入口点。

1.  这相当简单；我们只需要用以下代码在`flitton_fib_rs/__init__.py`文件中导入我们的函数：

    ```rs
    from .flitton_fib_rs import *
    ```

    我们将在本章后面详细介绍这是如何工作的，因为我们将会安装这个包并运行它。

## 安装我们的包的 Rust 库

目前，我们拥有部署我们的包并通过 `pip` 安装所需的一切。考虑到这一点，我们将我们的包上传到我们的 GitHub 仓库，这在 *第四章* 的 *配置 Python pip 模块的设置工具* 部分有所介绍，即 *在 Python 中构建 pip 模块*。

一旦我们完成这些，我们就可以使用以下命令安装我们的 `pip` 包，所有操作都在一行中完成：

```rs
pip install git+https://github.com/maxwellflitton/flitton-
```

```rs
  fib-rs@main
```

您的 GitHub 仓库的 URL 可能不同。在安装过程中，这个过程可能会挂起一段时间。结果应该给出以下输出：

```rs
Collecting git+https://github.com/maxwellflitton
```

```rs
/flitton-fib-rs@main
```

```rs
Cloning https://github.com/maxwellflitton/
```

```rs
flitton-fib-rs (to revision main) to /private
```

```rs
/var/folders/8n/
```

```rs
7295fgp11dncqv9n0sk6j_cw0000gn/T/pip-req-build-kcmv4ldt
```

```rs
Running command git clone -q https:
```

```rs
//github.com/maxwellflitton/flitton-fib-rs 
```

```rs
/private/var/folders/8n
```

```rs
/7295fgp11dncqv9n0sk6j_cw0000gn/T/pip-req-build-kcmv4ldt
```

```rs
Installing collected packages: flitton-fib-rs
```

```rs
  Running setup.py install for flitton-fib-rs ... done
```

```rs
Successfully installed flitton-fib-rs-0.1
```

这是因为我们是在基于我们的系统编译这个包。在这里，我们可以看到我们从仓库的 `main` 分支收集代码并运行 `setup.py` 文件。我们实际上所做的是将 Rust 代码编译成二进制文件，并将其放置在我们 `__init__.py` 入口点文件旁边，以下是我们模块的文件布局：

```rs
├── flitton_fib_rs
```

```rs
│   ├── __init__.py
```

```rs
│   └── flitton_fib_rs.cpython-38-darwin.so
```

这就是为什么我们的 `from .flitton_fib_rs import *` 代码在入口点中可以工作。

现在所有这些都已经安装到我们的 Python 包中，我们可以运行我们的 Python 控制台并输入以下命令：

```rs
>>> from flitton_fib_rs import say_hello
```

```rs
>>> say_hello()
```

```rs
saying hello from Rust!
```

这里就是它了！我们已经让 Rust 与 Python 一起工作，并且我们已经成功地将我们的 Rust 代码打包成 `pip` 模块。这是一个彻底的变革。我们现在可以不重写我们的 Python 系统，就可以利用 Rust 代码。然而，我们只有一个 Rust 代码文件。如果我们想充分利用将 Rust 与 Python 融合的能力，我们需要学习如何构建更大的 Rust 系统。 

# 使用 pyO3 crate 构建 Rust 接口

构建接口不仅仅意味着向我们的 Rust 模块中添加更多函数并将它们包装起来。在某种程度上，我们确实需要做这些；然而，探索如何从其他 Rust 文件中导入它们是很重要的。我们还必须探索和了解我们在构建模块时 Rust 和 Python 之间可以建立的关系。为了实现这一点，我们将执行以下步骤：

1.  在我们的 Rust 包中构建我们的 Fibonacci 模块。

1.  为我们的包创建命令行工具。

1.  为我们的包创建适配器。

在 *第一步* 中，我们只需使用 Rust 代码构建我们的模块。*第二步* 和 *第三步* 更侧重于 Python，用 Python 代码包装我们的 Rust 代码，以便简化 Rust 模块与外部 Python 代码的交互。在 *第六章* 的 *在 Rust 中与 Python 对象协同工作* 中，我们将直接在我们的 Rust 代码中与 Python 对象交互。考虑到所有这些，让我们通过首先在 Rust 中使用 *第一步* 构建 Fibonacci 代码来构建我们的 Python 接口。

## 构建 Fibonacci 的 Rust 代码

在这一步，我们将构建我们的 Fibonacci 模块，跨越多个 Rust 文件。为了实现这一点，我们的模块的文件结构如下所示：

```rs
├── Cargo.toml
```

```rs
├── README.md
```

```rs
├── flitton_fib_rs
```

```rs
│   ├── __init__.py
```

```rs
├── setup.py
```

```rs
├── src
```

```rs
│   ├── fib_calcs
```

```rs
│   │   ├── fib_number.rs
```

```rs
│   │   ├── fib_numbers.rs
```

```rs
│   │   └── mod.rs
```

```rs
│   ├── lib.rs
```

在这里，我们可以看到我们已经将我们的斐波那契代码添加到了`src/fib_calcs`目录下，因为我们记得`fib_numbers.rs`依赖于`fib_number.rs`。

现在，让我们按照以下步骤进行：

1.  我们可以在`fib_number.rs`文件中用以下代码最初定义我们的斐波那契数函数：

    ```rs
    use pyo3::prelude::pyfunction;
    #[pyfunction]
    pub fn fibonacci_number(n: i32) -> u64 {
        if n < 0 {
             panic!("{} is negative!", n);
        }
         match n {
              0     => panic!("zero is not a right \
                       argument to fibonacci_number!"),
              1 | 2 => 1,
              _     => fibonacci_number(n - 1) + 
                       fibonacci_number(n - 2)
        }
    }
    ```

    在这里，我们可以看到我们导入了`pyfunction`宏来应用于我们的函数。到目前为止，我们已经熟悉了计算斐波那契数；然而，与之前的示例不同，我们必须注意我们移除了`如果输入的斐波那契数要计算的值为 3`的匹配语句。这是因为该匹配语句显著加快了代码的执行速度，而我们希望在本章的最后部分进行公平的速度比较。

1.  现在我们已经定义了我们的斐波那契数函数，我们可以在`fib_numbers.rs`文件中定义我们的`fibonacci_numbers`函数，代码如下：

    ```rs
    use std::vec::Vec;
    use pyo3::prelude::pyfunction;
    use super::fib_number::fibonacci_number;
    #[pyfunction]
    pub fn fibonacci_numbers(numbers: Vec<i32>) -> \
      Vec<u64> {
        let mut vec: Vec<u64> = Vec::new();
        for n in numbers.iter() {
            vec.push(fibonacci_number(*n));
        }
        return vec
    }
    ```

    在这里，我们可以看到我们接受了一个整数向量，遍历它们，并将它们追加到一个空向量中，返回包含所有计算出的斐波那契数的向量。在这里，我们导入了`fibonacci_number`函数。

1.  然而，我们记得如果我们不在`src/mod.rs`文件中用以下代码定义它们，我们就无法导入它们，并且这两个函数将不会在直接目录之外可用。

    ```rs
    pub mod fib_number;
    pub mod fib_numbers;
    ```

1.  现在我们已经定义了两个函数并在我们的`src/mod.rs`文件中声明了它们，我们现在能够将它们导入到我们的`lib.rs`文件中。我们通过首先声明`fib_calcs`模块，然后使用以下代码导入函数来完成这项工作：

    ```rs
    mod fib_calcs;
    use fib_calcs::fib_number::__pyo3_get_function \
      _fibonacci_number;
    use fib_calcs::fib_numbers::__pyo3_get_function \
      _fibonacci_numbers;
    pub mod fib_numbers;
    ```

    这里需要注意的是，我们的函数前缀为`__pyo3_get_function_`。这使我们能够保留应用于函数的宏。如果我们直接导入函数，我们就无法将它们添加到模块中，这将导致在安装我们的包时出现编译错误。

1.  现在我们已经导入了函数并准备好了，我们可以导入包装它们并将它们添加到模块中，代码如下：

    ```rs
    #[pymodule]
    fn flitton_fib_rs(_py: Python, m: &PyModule) -> \
      PyResult<()> {
        m.add_wrapped(wrap_pyfunction!(say_hello));
        m.add_wrapped(wrap_pyfunction!(fibonacci_number));
        m.add_wrapped(wrap_pyfunction!(fibonacci_numbers);
        Ok(())
    }
    ```

1.  现在我们已经构建了我们的模块，我们可以测试它们。我们通过将我们的更改上传到 GitHub 仓库，使用`pip uninstall`来卸载我们的`pip`模块，并使用`pip install`来安装我们的新包来完成这项工作。一旦我们的新包安装完成，我们就可以在 Python 终端中导入并使用我们的新函数，如下所示：

    ```rs
    >>> from flitton_fib_rs import fibonacci_number, 
    fibonacci_numbers
    >>> fibonacci_number(20)
    6765
    >>> fibonacci_numbers([20, 21, 22])
    [6765, 10946, 17711]
    >>>
    ```

在这里，我们可以看到我们可以导入并使用我们在 Rust 中编写的跨越多个文件的斐波那契数！我们现在已经到了一个阶段，没有任何事情阻止我们构建自己的 Rust Python `pip`包。如果你在 Rust 中有特定的要解决的问题，比如你的 Python 程序难以计算的计算密集型任务，现在没有任何事情阻止你解决这个问题。

现在我们已经费尽周折地将用 Rust 编写的 Python 包打包，我们可以进一步利用我们的包，通过命令行功能。使用 `pip` 安装的包是方便、强大的命令行工具。在下一节中，我们将直接从命令行访问我们包中的 Rust 代码。

## 为我们的包创建命令行工具

你可能已经注意到，为了使用我们的斐波那契函数，我们必须启动一个 Python 控制台，导入函数，然后使用它们。如果我们只想在控制台中计算一个斐波那契数，这并不很高效。我们可以通过定义入口点来删除计算斐波那契数在终端中所需的这些不必要的程序。

考虑到我们在 `setup.py` 文件中定义了我们的命令行入口点，在作为我们的 Rust 函数包装器的 Python 文件中定义我们的入口点是有意义的（因为我们仍然想要 Rust 的速度优势），如下面的图所示：

![图 5.2 – 模块入口点流程](img/Figure_5.2_B17720.jpg)

![img/Figure_5.2_B17720.jpg](img/Figure_5.2_B17720.jpg)

图 5.2 – 模块入口点流程

这个包装可以通过导入 `argparse` 和我们在 Rust 模块中制作的 `fibonacci_number` 函数来创建一个简单的 Python 函数，该函数获取用户输入，然后将它传递给 Rust 函数，并打印出结果。我们可以通过以下步骤实现这一点：

1.  我们可以通过将以下代码添加到我们创建的 `flitton_fib_rs/fib_number_command.py` 文件中，来构建一个收集参数并调用 Rust 代码的 Python 函数：

    ```rs
    import argparse
    from .flitton_fib_rs import fibonacci_number
    def fib_number_command() -> None:
        parser = argparse.ArgumentParser(
            description='Calculate Fibonacci numbers')
        parser.add_argument('--number', action='store', \
          type=int, required=True,help="Fibonacci \
            number to becalculated")
        args = parser.parse_args()
        print(f"Your Fibonacci number is: "
              f"{fibonacci_number(n=args.number)}")
    }
    ```

    我们必须记住，当我们的 Rust 二进制文件编译时，它将在 `flitton_fib_rs` 目录中，就在我们刚刚创建的文件旁边。

1.  接下来，我们在 `setup.py` 文件中定义入口点。现在我们有了我们的函数，我们可以在 `setup.py` 文件中通过声明此文件和函数的路径来指向它，并使用以下代码为 `entry_points` 参数指定路径：

    ```rs
    entry_points={
        'console_scripts': [
            'fib-number = flitton_fib_rs.'
            'fib_number_command:'
            'fib_number_command',
        ],
    },
    ```

1.  一旦完成，我们就已经完全配置了我们包中的 Python 入口点。最后，我们可以通过传递参数到入口点来测试我们的命令行。现在，如果我们更新我们的 GitHub 仓库并在 Python 环境中重新安装我们的包，我们可以通过输入以下命令来测试我们的命令行：

    ```rs
    fib-number --number 20
    ```

    这将给出以下输出：

    ```rs
    Your Fibonacci number is: 6765
    ```

我们可以看到我们的命令行工具正在工作。现在，我们已经复制了与我们在 *第四章* 中之前相同的函数，即 *在 Python 中构建 pip 模块*。然而，我们现在必须更进一步。我们在包中融合了两种不同的语言。为了完全掌握我们的 `pip` 包，我们需要探索如何命令和细化 Rust 与 Python 之间的交互。

在我们的下一步中，我们将构建适配器，使我们能够做到这一点。

## 为我们的包创建适配器

在我们尝试构建适配器接口之前，我们需要了解什么是**适配器**。适配器是一种设计模式，它管理两个不同模块、应用程序、语言等之间的接口。设计模式的标题描述了我们正在做的事情。例如，如果你购买了一款新的 MacBook Pro，你会意识到你只有 USB-C 端口。而不是打开你的 MacBook 并重新布线，以便它可以接受你的标准 USB 闪存驱动器，你购买了一个适配器。适配器有多个优点。当涉及到模块化软件工程时，这给我们带来了优势。

例如，假设模块 A 依赖于模块 B。我们可以在模块 A 中创建适配器来管理这两个模块之间的接口，而不是在模块 A 中导入模块 B 的各个方面。这样，我们就能获得很多灵活性。例如，模块 C 可以构建为模块 B 的改进版。我们不需要通过模块 A 查找并尝试根除模块 B 的使用，我们知道它们都在适配器中得到了利用。我们甚至可以缓慢地创建一个转向模块 C 的第二个适配器。如果我们想删除一个模块或将其移除，再次，我们只需删除适配器就可以立即切断与其他模块的连接。适配器简单且给我们带来了极大的灵活性。

考虑到我们关于适配器的讨论，我们创建 Rust 代码和 Python 之间的适配器是有意义的。鉴于 Python 系统本质上是在使用我们的 Rust 代码，因此在我们 Python 中构建适配器是有意义的。

为了演示如何做到这一点，我们将创建一个接受列表或整数的适配器。然后它选择正确的 Rust 函数并实现它。然而，对于这个适配器的用途，我们可以设想一个有很多错误数据被输入到模块中的场景。我们不希望每次传递错误数据时都出错，但我们确实想分类计算是否失败，并统计我们完成的正确计算的数量。这似乎很具体，我们必须记住，就像 MacBook 一样，我们可以有多个适配器。如果我们未来需要修改或删除，没有什么可以阻止我们这样做。

然而，在我们开始编写代码之前，我们需要理解适配器涉及的层级，如下所述：

![Figure 5.3 – Rust 模块的 Python 适配器的层级

![img/Figure_5.3_B17720.jpg]

图 5.3 – Rust 模块的 Python 适配器的层级

在前面的图中，我们可以看到 Python 对象来自类型。然而，我们可以通过**元类**来介入这些对象是如何从类型中调用的。当涉及到元类时，我们必须构建一个元类来定义我们的计数器是如何被调用的。我们的计数器将是通用的。我们不知道用户将如何使用我们的接口。他们可能会遍历一系列数据点，为每个数据点调用我们的适配器。我们需要确保无论调用多少适配器，它们都指向同一个计数器。这可能会有些令人困惑。这将在我们构建时变得更加清晰。

### 使用单例设计模式构建适配器接口

首先，我们必须定义我们的`Singleton`元类：

1.  这可以在我们创建的`flitton_fib_rs/singleton.py`文件中使用以下代码完成：

    ```rs
    class Singleton(type):
        _instances = {}
        def __call__(cls, *args, **kwargs):
            if cls not in cls._instances:
                cls._instances[cls] = super(Singleton, \
                  cls).__call__(*args, **kwargs)
            return cls._instances[cls]
    ```

1.  在这里，我们可以看到我们的`Singleton`类直接继承自`type`。在这里，发生的事情是我们有一个名为`_instances`的字典，这个字典的键是类类型。当一个具有`Singleton`作为元类的类被调用时，该类的类型会在字典中进行检查。如果该类型不在字典中，则它将被构造并放入字典中。然后，字典中的实例被返回。这本质上意味着我们无法有两个类的实例。这个过程在以下图中展示：![图 5.4 – Singleton 元类的逻辑流程]

    ![img/Figure_5.4_B17720.jpg]

    图 5.4 – Singleton 元类的逻辑流程

1.  现在，我们将使用我们的`Singleton`类来构建我们的计数器。这可以在我们创建的`flitton_fib_rs/counter.py`文件中使用以下代码完成：

    ```rs
    from .singleton import Singleton
    class Counter(metaclass=Singleton):
        def __init__(self, initial_value: int = 0) -> \
          None:
            self._value: int = initial_value
        def increase_count(self) -> None:
            self._value += 1
        @property
        def value(self) -> int:
            return self._value
    ```

    现在，我们的`Counter`类不能在同一个程序中构造两次。因此，我们可以确保无论我们调用多少次，都只有一个`Counter`类。

1.  我们现在可以在我们的主要适配器上使用它。我们将把我们的主要适配器放在我们创建的`flitton_fib_rs/fib_number_adapter.py`文件中。首先，我们使用以下代码导入所有需要的函数和对象：

    ```rs
    from typing import Union, List, Optional
    from .flitton_fib_rs import fibonacci_number, \
        fibonacci_numbers
    from .counter import Counter
    ```

    在这里，我们可以看到我们导入了所需的`typing`。我们还导入了我们将使用的 Rust 斐波那契数和我们的计数器。现在我们已经导入了所需的内容，我们可以构建我们的接口构造器。

1.  对于我们的适配器，我们需要一个数字输入，一个表示过程是否成功的状态，以及实际的结果，这将是我们计算出的斐波那契数，或者如果失败，将是一个错误信息。我们还将有一个计数器，并且我们将在对象的构建过程中处理输入。这可以用以下代码表示：

    ```rs
    class FlittonFibNumberAdapter:
        def __init__(self,
            number_input: Union[int, List[int]]) -> None:
            self.input: Union[int, List[int]] = \
              number_input
            self.success: bool = False
            self.result: Optional[Union[int, List[int]]] \
              = None
            self.error_message: Optional[str] = None
            self._counter: Counter = Counter()
            self._process_input()
    ```

    记住，尽管我们调用了计数器，但它是一个单例模式；因此，计数器将是所有适配器实例中的相同实例。现在我们已经定义了所有正确的属性，我们必须定义什么是实际的成功。

1.  这是我们声明成功状态为`True`并增加计数器的地方。这可以通过以下`FlittonFibNumberAdapter`实例函数表示，如下所示：

    ```rs
        def _define_success(self) -> None:
            self.success = True
            self._counter.increase_count()
    ```

    这很顺畅；因为我们已经为计数器定义了一个干净的接口，所以不需要太多解释。现在我们已经定义了成功，我们需要处理输入，因为有两个不同的函数，一个接受列表，另一个接受整数。

1.  我们可以通过`FlittonFibNumberAdapter`实例函数将正确的输入传递给正确的函数，如下所示：

    ```rs
        def _process_input(self) -> None:
            if isinstance(self.input, int):
                self.result = fibonacci_number( \
                    n=self.input)
                self._define_success()
            elif isinstance(self.input, list):
                self.result = fibonacci_numbers( \
                    numbers=self.input)
                self._define_success()
            else:
                self.error_message = "input needs to be \
                  a list of ints or an int"
    ```

    在这里，我们可以看到如果没有传递整数列表，我们定义一个错误信息。如果我们传递了正确的输入，我们定义结果为函数的结果并调用`_define_success`函数。

1.  剩下的唯一事情是将计数器暴露给外部用户。这可以通过以下`FlittonFibNumberAdapter`属性来完成：

    ```rs
        @property
        def count(self) -> int:
            return self._counter.value
    ```

1.  再次强调，计数器界面简洁，因此无需解释。我们的适配器接口现已完成。我们所需做的就是通过以下代码将其导入到`src/__init__.py`文件中，以便向用户暴露：

    ```rs
    from .fib_number_adapter import \
      FlittonFibNumberAdapter
    ```

一切都已完成。我们现在可以更新我们的 GitHub 仓库，并在 Python 环境中重新安装我们的包。

### 在 Python 控制台中测试适配器接口

我们现在可以使用以下 Python 控制台命令测试我们的适配器，如下所示：

```rs
>>> from flitton_fib_rs import FlittonFibNumberAdapter
```

```rs
>>> test = FlittonFibNumberAdapter(10)
```

```rs
>>> test_two = FlittonFibNumberAdapter(15)
```

```rs
>>> test_two.count
```

```rs
2
```

```rs
>>> test.count
```

```rs
2
```

```rs
>>> test_two.success
```

```rs
True
```

```rs
>>> test_two.result
```

```rs
610
```

在这里，我们可以看到我们可以从模块中导入我们的适配器。然后我们可以定义两个不同的适配器。我们可以看到两个适配器之间的计数器是一致的，这意味着我们的单例模式工作得很好！两个适配器都指向同一个`Counter`实例！所有我们的适配器都将指向那个相同的`Counter`实例。我们还可以看到成功状态为`True`，并且我们可以访问计算结果：

1.  现在，在同一个 Python 控制台，我们可以通过以下 Python 控制台命令测试一个不正确的输入是否会导致失败并且不会增加计数：

    ```rs
    >>> test_three = FlittonFibNumberAdapter(
                                   "should fail"
                                 )
    >>> test_three.count
    2
    >>> test_three.result
    >>> test_three.success
    False
    >>> test_three.error_message
    'input needs to be a list of ints or an int'
    >>>
    ```

1.  在这里，我们可以看到计数器没有增加，成功状态为`False`，并且出现了一个错误信息。最终的输入测试可以通过输入一个整数列表，使用下面的 Python 控制台命令来完成：

    ```rs
    3, as they are all pointing to the same Counter instance and there was one failure out of the four. 
    ```

1.  在同一个 Python 控制台命令中调用它们，可以揭示这是否为真，如下所示：

    ```rs
    >>> test.count
    3
    >>> test_two.count
    3
    >>> test_three.count
    3
    >>> test_four.count
    3
    ```

如此，我们已经完全配置了我们的模块的 Python 接口。

在本节中，我们为我们的 Rust `pip`包添加了 Python 接口。你可能会想向`flitton_fib_rs`目录中添加额外的目录并填充整个 Python 模块。然而，当包被安装时，`flitton_fib_rs`目录中的额外目录不会被复制。这也是可以的。我们本质上是在构建 Rust `pip`包。Rust 既快又安全，我们应该尽可能多地依赖它。`flitton_fib_rs`目录中的 Python 适配器和命令应该在那里平滑接口。例如，如果我们想以特定的方式管理我们接口的内存，那么在 Python 接口作为包装器中这样做是有意义的，因为 Python 将是导入和使用`pip`包的系统。如果你发现自己将适配器和命令行函数以外的任何东西放入`flitton_fib_rs`模块，那是一个警告信号，你应该尝试将其放入 Rust 模块本身。我们已经手动测试了我们的包；然而，我们需要确保我们的 Rust 斐波那契计算函数按预期工作。

在下一节中，我们将为我们的 Rust 代码创建单元测试。

# 为我们的 Rust 包构建测试

在上一节中，我们在*第四章*，*在 Python 中构建 pip 模块*，为我们 Python 代码构建了单元测试。在本节中，我们将为我们的斐波那契函数构建单元测试。这些测试不需要任何额外的包或依赖项。我们可以使用 Cargo 来管理我们的测试。这可以通过在`src/fib_calcs/fib_number.rs`文件中添加我们的测试代码来完成。步骤如下：

1.  我们通过在`src/fib_calcs/fib_number.rs`文件中创建一个模块来实现这一点，以下代码如下：

    ```rs
    #[cfg(test)]
    mod fibonacci_number_tests {
          use super::fibonacci_number;
    }
    ```

    在这里，我们可以看到我们在同一文件中定义了一个模块，并用`#[cfg(test)]`宏装饰了这个模块。

1.  我们还可以看到我们必须导入这个函数，因为它在模块之上。在这个模块内部，我们可以运行标准测试，检查我们传入的整数是否使用以下代码计算出我们期望的斐波那契数：

    ```rs
         #[test]
         fn test_one() {
              assert_eq!(fibonacci_number(1), 1);
         }
         #[test]
         fn test_two() {
              assert_eq!(fibonacci_number(2), 1);
         }
         #[test]
         fn test_three() {
               assert_eq!(fibonacci_number(3), 2);
         }
         #[test]
         fn test_twenty() {
               assert_eq!(fibonacci_number(20), 6765);
         }
    ```

    在这里，我们可以看到我们用`#[test]`宏装饰了我们的测试函数。如果它们没有产生我们期望的结果，那么`assert_eq!`和测试将失败。我们还必须注意，如果我们传入零或负值，我们的函数将引发 panic。

1.  这些可以通过测试函数进行测试，如下所示：

    ```rs
          #[test]
          #[should_panic]
          fn test_0() {
              fibonacci_number(0);
          }
          #[test]
          #[should_panic]
          fn test_negative() {
              fibonacci_number(-20);
          }
    ```

    在这里，我们传入失败的输入。如果它们没有 panic，那么测试将失败，因为我们用`#[should_panic]`宏装饰了它。

1.  现在我们已经为`fibonacci_number`函数创建了测试，我们可以在`src/fib_calcs/fib_numbers.rs`文件中构建我们的`fibonacci_numbers`函数测试，以下代码如下：

    ```rs
    #[cfg(test)]
    mod fibonacci_numbers_tests {
          use super::fibonacci_numbers;
          #[test]
          fn test_run() {
          let outcome = fibonacci_numbers([1, 2, 3, \
            4].to_vec());
            assert_eq!(outcome, [1, 1, 2, 3]);
          }
    }
    ```

1.  在这里，我们可以看到这与我们的其他测试有相同的布局。如果我们想运行我们的测试，我们可以使用以下命令来运行它们：

    ```rs
    cargo test
    ```

    这给我们以下输出：

    ```rs
    running 7 tests
    test fib_calcs::fib_number::fibonacci_number_tests::test_th
    ree ... ok
    test fib_calcs::fib_numbers::fibonacci_numbers_tests::test_
    run ... ok
    test fib_calcs::fib_number::fibonacci_number_tests::
    test_two ... ok
    test fib_calcs::fib_number::fibonacci_number_tests::test_on
    e ... ok
    test fib_calcs::fib_number::fibonacci_number_tests::
    test_twenty ... ok
    test fib_calcs::fib_number::fibonacci_number_tests::
    test_negative ... ok
    test fib_calcs::fib_number::fibonacci_number_tests::
    test_0 ... ok
    test result: ok. 7 passed; 0 failed; 0 ignored; 0 
    measured; 0 
    filtered out; finished in 0.00s
         Running target/debug/deps/flitton_fib_rs-
    07e3ba4b0bc8cc1e
    running 0 tests
    test result: ok. 0 passed; 0 failed; 0 ignored; 0 
    measured; 0 
    filtered out; finished in 0.00s
       Doc-tests flitton_fib_rs
    running 0 tests
    test result: ok. 0 passed; 0 failed; 0 ignored; 0 
    measured; 0 
    filtered out; finished in 0.00s
    ```

在这里，我们可以看到所有测试都已运行并通过。如果我们回想一下*第四章*，*在 Python 中构建 pip 模块*，我们会记得我们使用了模拟。

Rust 仍在开发`mockall`，支持模拟，可以在以下 URL 找到：[`docs.rs/mockall/0.10.0/mockall/`](https://docs.rs/mockall/0.10.0/mockall/)。另一个可以用于模拟的更干净的 crate 可以在以下 URL 找到：[`docs.rs/mocktopus/0.7.11/mocktopus/`](https://docs.rs/mocktopus/0.7.11/mocktopus/)。

我们现在已经介绍了如何构建我们的模块并对其进行测试。我们现在已经完成了带有测试和 Python 界面的 Rust `pip`模块的构建。现在我们可以测试我们 Rust 模块的速度，看看会发生什么，以及 Rust 模块作为工具有多强大。

# 比较 Python、Rust 和 Numba 的速度

我们现在已经使用命令行工具、Python 接口和单元测试在 Rust 中构建了一个`pip`模块。这是我们拥有的一个闪亮的新工具。让我们对其进行测试。我们知道 Rust 本身比 Python 快。然而，我们知道`pyo3`绑定会减慢我们的速度吗？还有另一种加快 Python 代码速度的方法，那就是使用 Numba，这是一个将 Python 代码编译以加快速度的 Python 包。如果我们可以用 Numba 达到同样的速度，我们还需要经历创建 Rust 包的所有匆忙吗？在本节中，我们将多次运行我们的 Fibonacci 函数，在 Python、Numba 和我们的 Rust 模块中。需要注意的是，Numba 的安装可能会很头疼。例如，我无法在我的 MacBook Pro M1 上安装它。我不得不在一个 Linux 笔记本电脑上安装 Numba 来运行这一节。你不需要运行本节中的代码；这更多的是为了演示目的。如果你想尝试运行测试脚本，所有步骤都提供如下：

1.  首先，我们必须安装我们构建的 Rust `pip`模块。然后我们使用以下命令安装 Numba：

    ```rs
    pip install numba
    ```

1.  一旦完成这些，我们就有了所需的一切。在任何 Python 脚本中，我们使用以下代码导入所需的包：

    ```rs
    from time import time
    from flitton_fib_rs.flitton_fib_rs import \
      fibonacci_number
    from numba import jit
    ```

    我们正在使用`time`模块来计时每次运行所需的时间。我们还使用了来自我们的 Rust `pip`模块的 Fibonacci 函数，并且还需要 Numba 的`jit`装饰器。**jit**代表**即时**。这是因为 Numba 在加载函数时对其进行编译。

1.  我们现在使用以下代码定义我们的标准 Python 函数：

    ```rs
    def python_fib_number(number: int) -> int:
        if number < 0:
            raise ValueError(
                "Fibonacci has to be equal or above zero"
            )
        elif number in [1, 2]:
            return 1
        else:
            return numba_fib_number(number - 1) + \
                   numba_fib_number(number - 2)
    ```

1.  我们可以看到，这是与 Rust 代码构建相同的逻辑。我们想要确保我们的测试是可靠的比较。然后我们使用以下代码定义用`jit`编译的 Python 函数：

    ```rs
    @jit(nopython=True)
    def numba_fib_number(number: int) -> int:
        if number < 0:
            raise ValueError("Fibonacci has to be equal \
              or above zero")
        elif number in [1, 2]:
            return 1
        else:
            return numba_fib_number(number - 1) + \
                   numba_fib_number(number - 2)
    ```

1.  我们可以看到，这是相同的。唯一的区别是我们用`jit`进行了装饰，并将`nopython`设置为`True`以获得最佳性能。然后我们使用以下代码运行所有这些：

    ```rs
    t0 = time()
    for i in range(0, 30):
        numba_fib_number(35)
    t1 = time()
    print(f"the time taken for numba is: {t1-t0}")
    t0 = time()
    for i in range(0, 30):
        numba_fib_number(35)
    t1 = time()
    print(f"the time taken for numba is: {t1 - t0}")
    ```

    在这里，我们可以看到我们循环从 `0` 到 `30` 的范围，并使用数字 `35` 调用我们的函数 `30` 次。然后我们打印出完成这一过程所需的时间。我们注意到我们做了两次。这是因为第一次运行将涉及编译函数。

1.  当我们运行此代码时，我们得到以下控制台输出：

    ```rs
    the time taken for numba is: 2.6187334060668945
    the time taken for numba is: 2.4959869384765625
    ```

1.  在这里，我们可以看到第二次运行时节省了一些时间，因为它没有进行编译。多次运行此测试表明这种减少是标准的。现在，我们使用以下代码设置我们的标准 Python 测试：

    ```rs
    t0 = time()
    for i in range(0, 30):
        python_fib_number(35)
    t1 = time()
    print(f"the time taken for python is: {t1 - t0}")
    ```

1.  运行此测试将得到以下控制台输出：

    ```rs
    the time taken for python is: 2.889884853363037
    ```

1.  我们可以看到，与我们的 Numba 函数相比，运行纯 Python 代码时速度会显著降低。现在，我们可以继续进行最后的测试，即我们的 Rust 测试，该测试由以下代码定义：

    ```rs
    t0 = time()
    for i in range(0, 30):
        fibonacci_number(35)
    t1 = time()
    print(f"the time taken for rust is: {t1 - t0}")
    ```

1.  运行此测试将给我们以下控制台输出：

    ```rs
    the time taken for rust is: 0.9373788833618164
    ```

在这里，我们可以看到 Rust 函数要快得多。这并不意味着 `Numba` 是浪费时间。当涉及到 Python 优化时，Numba 在某些情况下可以表现得很好。在其他情况下，Python 优化根本不会影响它们。考虑到它们应用起来有多容易，始终值得检查是否可以加快速度。然而，我们现在也知道 Rust 总是比纯 Python 代码快。

# 摘要

在本章中，我们构建了一个完整的 Python `pip` 模块，其中包含命令行工具、接口和 Rust 代码。我们管理了 Rust 和 Python 开发的 `gitignore`。然后我们定义了我们的设置工具，用于打包我们的 Python 代码和模块，以及具有 Python 绑定的 Rust 代码的编译。一旦这些定义完成，我们就学习了如何构建跨多个 Rust 文件的 Rust 函数，这些函数可以用 `pyo3` 绑定包装。

我们的开发并没有仅仅停留在 Rust 上。我们还探索了 Python 的单例和适配器设计模式，为我们的用户提供更高级的 Python 接口。然后我们使用单元测试和速度检查测试我们的代码。必须指出的是，我们本章没有涵盖 GitHub actions。GitHub actions 与前一章中的定义方式相同。我们不是使用 Python 单元测试来运行测试，而是使用 Cargo 等工具来运行我们的测试。然而，上传到 PyPI 要复杂一些。为了涵盖这一点，在 *进一步阅读* 部分提供了如何预编译和上传 Rust `pip` 模块的示例。

我们现在拥有一种强大的技能，那就是构建利用 Rust 的 Python `pip` 模块。然而，我们依赖 Python 来构建我们的接口。在下一章中，我们将在 Rust 代码中处理 Python 对象。因此，我们将能够将更高级的 Python 数据对象传递到我们的 Rust 代码中。我们还将使我们的 Rust 代码能够返回完整的 Python 对象。

# 问题

1.  你如何为 `pyo3` Rust Python `pip` 模块定义 `setup.py` 文件？

1.  在安装后，我们的 `pip` 模块在 Python 环境中的布局是什么？为什么我们不能构建跨越多个目录的 Python 模块？

1.  什么是单例设计模式？

1.  适配器设计模式是什么？使用设计模式的优势是什么？

1.  元类是什么？我们如何使用它？

# 答案

1.  在我们进行 `setup.py` 文件中的任何其他操作之前，我们必须使用 `dist` 包来安装 `setuptools_rust`。我们定义设置参数并使用来自 `setuptools_rust` 的 `RustExtension` 对象，指向编译好的 Rust 模块安装后的位置。

1.  当 `pip` 模块安装时，二进制的 Rust 文件位于定义模块 Python 文件的同一目录中。然而，该目录中的目录不会被复制，因此它们将在安装过程中丢失。

1.  单例设计模式确保所有对特定类的引用都指向该类的一个实例。

1.  适配器模式是一种管理两个模块之间交互的接口。其优势在于模块之间的灵活性。我们知道所有交互的位置，如果我们想切断模块，我们只需要删除适配器。这使我们能够根据需要切换模块。

1.  元类位于类型和对象之间的一种类。正因为如此，我们可以用它来查看我们如何管理调用我们的对象。

# 进一步阅读

+   *Mre* – 在 PyPI 上部署 Rust 包的 GitHub Actions 示例（2021）: [`github.com/mre/hyperjson/blob/master/.github/workflows/ci.yml`](https://github.com/mre/hyperjson/blob/master/.github/workflows/ci.yml%0D)

+   *《精通面向对象 Python》*，*Steven F. Lott*，*Packt Publishing*（2019）

+   *《PyO3 用户指南》*: [`pyo3.rs/v0.13.2/`](https://pyo3.rs/v0.13.2/)
