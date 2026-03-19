---
tags:
  - language-server
  - cxx
draft: true
---
`clangd` 是本人偏好的 C++ language server。原因之一是在 macOS 上默认的 C++ 编译器为 `clang`，`clangd` 对于 LLVM 侧的编译工具理应有更好的支持。并且，即使是用 `gcc` 编译的项目，`clangd` 也有着令人满意的表现。原因之二是我觉得 vscode 默认的 Microsoft's C/C++ extension 中的 `cpptool` 很难用。

---
以下记录使用 `clangd` 的一些心得：

`clangd` 通常配合 `compile_commands.json` 或编译数据库使用。对于使用 CMake 构建的项目，这意味着通常需要在 `CMakeLists.txt` 里写 `set(CMAKE_EXPORT_COMPILE_COMMANDS ON)`，或在 cmake configure 阶段命令行参数传入 `-DCMAKE_EXPORT_COMPILE_COMMANDS=ON` 。

可以在项目的目录下创建 `.clangd` 文件（参考[配置文档](https://clangd.llvm.org/config)），如：

```yaml
CompileFlags:
	Compiler: /Library/Developer/CommandLineTools/usr/bin/clang++
	Remove:
		- -std=*
	Add:
		- -std=c++20
		- -stdlib=libc++
		- -pthread
		- -Wall
		- -Wextra
		- -pedantic
		- -Iinclude
```

在 VSCode 中，可以编辑 `.vscode/settings.json` 控制 `clangd` language server 的行为，如：

```json
{
	"clangd.path": "/path/to/your/clangd",
	"clangd.arguments": [
		// Specifying the directory for `compile_commands.json`
		"--compile-commands-dir=${workspaceFolder}",
		"--query-driver=/Library/Developer/CommandLineTools/usr/bin/clang++"
	]
}
```

为了更精确的解析，建议启用 `--compile-commands-dir=${workspaceFolder}`。项目目录下的 `.clangd` 文件仅适用于项目树下的文件，但 system header library 位于工作区之外。对于指定了 C++ 标准的项目，`clangd` server 在 system header library 中不一定找得到项目编译数据库，因此没有精确的编译参数信息，解析时会 fallback 成系统默认值。

可以配合 `.clang-tidy` 和 `.clang-format` 使用。

