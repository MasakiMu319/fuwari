---
title: 0x01 Python venv
published: 2024-12-09 16:00:00
description: '浅析 Python venv.'
image: ''
tags: ['Python', 'pyvenv']
category: 'Python'
draft: false
---

>   写本文的动机来自对于 “shebeng” 机制的好奇，遂挖掘 python 的虚拟环境到底干了什么事儿。 太长可以直接跳到文末的 Conclusion 部分 🙂。

`venv` 模块支持创建轻量的虚拟环境，每一个都有他们自己独立的 Python packages 集合安装在 [site](https://docs.python.org/3/library/site.html#module-site) 目录中。一个虚拟环境创建在一个现有的 Python 安装上，被称为虚拟环境的基础 Python，并且可选地与基础环境中的 packages 隔离，因此只有明确被安装在虚拟环境中的才可用。

当在虚拟环境使用时，命令行安装工具例如 pip 会安装 Python packages 到当前的虚拟环境中而无需明确地声明。

一个虚拟环境时：

-   用于包含一个特定的 Python 解释器，用于支持项目（库或者应用）的软件库和二进制文件。这些默认与其他安装在操作系统上的虚拟环境和 Python 解释器、库文件是隔离的。
-   包含一个文件夹，通常被在项目中被命名为 `.venv` 或者 `venv` ，或者在一个包含多虚拟环境的容器目录中，例如 `~/.virtualenvs` .
-   无需纳入版本控制系统检查例如 Git。
-   被视为一次性的 —— 它应该是方便删除和再创建的，你不应将任何项目代码放在该环境中；
-   不被视为可移动或者可复制的 —— 你只需要再目标未知重新创建新的环境。

## Creating virtual enviroments

```bash
python -m venv /path/to/new/virtual/enviroment
```

这会创建一个目标目录（包括必须的父级目录）并新建一个 `pyvenv.cfg` 文件，其中的 `home` key 指向运行命令所使用的已安装的 Python。还会创建一个 `bin` 子目录（在 Windows 上是 `Scripts`）包含一个复制或者可执行 Python 的符号链接（根据创建环境时的平台或者参数进行调整）。同时创建一个 `lib/pythonX.Y/site-packages` 子目录（在 Windows 上是 `Libsite-packages` )。如果被指定一个已存在的目录，则会复用。

`pyvenv.cfg` 文件也包括 `include-system-site-packages` key，如果在调用 `venv` 时使用 `—system-site-packages` 选项时会被设为 true。

除非使用 `—without-pip` 选项，否则 `ensurepip` 会被调用去安装 `pip` 到虚拟环境中。

可以在使用 `venv` 时提供多个路径，这样每个提供的路径下都会创建一个相同的虚拟环境。

## How venvs work

当一个来自虚拟环境的 Python 解释器运行的时候，`sys.prefix` 和 `sys.exec_prefix` 会指向虚拟环境的目录，而 `sys.base_prefix` 和 `sys.base_exec_prefix` 会指向创建环境的基础 Python。因此只需检查 `sys.prefix != sys.base_prefix` 即可确定当前解释器是否来自虚拟环境。

```bash
Python 3.12.4 (v3.12.4:8e8a4baf65, Jun  6 2024, 17:33:18) [Clang 13.0.0 (clang-1300.0.29.30)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
>>> import sys
>>> sys.prefix
'/Users/Side-Projects/ModelTools/.venv'
>>> sys.base_prefix
'/Library/Frameworks/Python.framework/Versions/3.12'
>>> sys.exec_prefix
'/Users/Side-Projects/ModelTools/.venv'
>>> sys.base_exec_prefix
'/Library/Frameworks/Python.framework/Versions/3.12'
```

你不需要显示激活一个虚拟环境，因为你可以在调用 Python 的时候指定虚拟环境 Python 解释器的完整路径。进一步来说，所有安装在虚拟环境中的脚本都应该直接可运行，而无需激活。

为了实现这一目标，安装在虚拟环境的脚本有一个名为 “shebang” 行，指向了环境的 Python 解释器，`#!/<path-to-venv>/bin/python` 。这意味着脚本总会使用给定的解释器运行，无论 PATH 值是什么。在 Windows 上，shebang 行同样生效，前提是 [Python Launcher for Windows](https://docs.python.org/3/using/windows.html#launcher) 已经被正确安装。

当一个虚拟环境被激活后，`VIRTUAL_ENV` 环境变量会被设置到环境的 PATH。因为明确激活一个虚拟环境不是必须的，VIRTUAL_ENV 不能用作确定是否使用虚拟环境的依据。

## Conclusion

我们总结虚拟环境创建和运行的机制：

1.  首先通过 `venv` module 创建的虚拟环境会自动创建一个对应的目录，其中包含了激活虚拟环境的脚本与指向创建虚拟环境所使用的 Python 解释器符号链接或者 Copy，以及后续安装 packages 的目录 `lib/pythonX.Y/site-packages`；

2.  使用虚拟环境目录下提供的激活脚本，我们就进入了包含特定 Python 解释器的虚拟环境。简单来说就是将 `VIRTUAL_ENV` 写入当前的环境变量 `PATH` 最前面，而 `VIRTUAL_ENV` 对应的值就是我们在步骤 1 中创建的虚拟环境目录路径； 这一点我们可以通过查看激活脚本得到验证：

    ```bash
    VIRTUAL_ENV='/Users/Side-Projects/ModelTools/.venv'
    if ([ "$OSTYPE" = "cygwin" ] || [ "$OSTYPE" = "msys" ]) && $(command -v cygpath &> /dev/null) ; then
        VIRTUAL_ENV=$(cygpath -u "$VIRTUAL_ENV")
    fi
    export VIRTUAL_ENV
    
    _OLD_VIRTUAL_PATH="$PATH"
    PATH="$VIRTUAL_ENV/bin:$PATH"
    export PATH
    ```

3.  当我们在 `shebeng` 中声明 `#!/usr/bin/env python3` 时，`/usr/bin/env` 保证了在激活虚拟环境后一定使用的是当前环境的环境变量，而由于此时 python3 能在最前面的虚拟环境目录下找到，所以具有绝对优先使用权，因此就能在不显示激活虚拟环境的情况下成功调用相应 Python 解释器；

4.  当然，如果 shebeng 中直接声明了 Python 解释器的路径 `#!.venv/bin/python`，那么就能实现不激活虚拟环境而使用相应 Python 解释器运行的效果。