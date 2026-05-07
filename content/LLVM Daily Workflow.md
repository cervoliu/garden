---
tags:
  - LLVM
---
After pulling upstream, use this workflow:

```sh
cd /home/cervol/llvm-project/llvm
git pull upstream main
cmake --build --preset alive2-deps-clang18-lld
```

That keeps the build using your preset config, including Ninja, Clang 18, LLD, ccache, and the local cache dir.

For routine downstream use, avoid rebuilding default `all` unless you need it. Build only the tools Alive2 usually needs:

```sh
cd /home/cervol/llvm-project
CCACHE_DIR=/home/cervol/llvm-project/build-clang18-lld/.ccache \
  ninja -C build-clang18-lld opt llvm-config llvm-lit FileCheck
```

If you need Clang too:

```sh
CCACHE_DIR=/home/cervol/llvm-project/build-clang18-lld/.ccache \
  ninja -C build-clang18-lld clang opt llvm-config llvm-lit FileCheck
```

Check completion with:

```sh
ninja -C /home/cervol/llvm-project/build-clang18-lld -n
```

For a target-limited rebuild, `ninja -n` should say no work for those targets if you ask the same targets:

```sh
ninja -C /home/cervol/llvm-project/build-clang18-lld -n clang opt llvm-config llvm-lit FileCheck
```

Best practices:

- Use the preset build command when doing a broad rebuild:
  ```sh
  cd /home/cervol/llvm-project/llvm
  cmake --build --preset alive2-deps-clang18-lld
  ```
- Use explicit Ninja targets for day-to-day work.
- Do not clean or recreate `build-clang18-lld` unless the cache/config is broken.
- If running Ninja directly, always set:
  ```sh
  CCACHE_DIR=/home/cervol/llvm-project/build-clang18-lld/.ccache
  ```
- After a pull, expect some rebuild. If PCH or TableGen outputs changed, many downstream objects can become dirty.
- If you interrupt a rebuild, resume it later with the same command; otherwise the next day it may still show thousands of pending tasks.

For long rebuilds, I’d run:

```sh
cd /home/cervol/llvm-project/llvm
cmake --build --preset alive2-deps-clang18-lld
```

and let it finish once after each meaningful upstream update.