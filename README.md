# Clang Plugins Overview
This document is intended as a brief introduction to clang plugins.
This work, unless otherwise specified, is licensed under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/).

## Building
This part of the guide is likely going to be what will go stale fastest, so checking an up-to-date book on clang for its build procedures is probably the best place to start troubleshooting.
Personally, I found [this book](https://learning.oreilly.com/library/view/llvm-techniques-tips/9781838824952/) to be very helpful, and would recommend it as a starting point for further reading.

Since clang plugins are dynamically loaded, the version of clang they're built against and the version of clang they're run with should match.
Because the plugin API isn't stable, this can make updating/porting plugins a pretty significant maintenance overhead.
This also means that unless the version of clang installed on your system includes debug symbols, you're going to want to build against a version of clang you've built manually to include debug symbols.

I like [git submodules](https://git-scm.com/book/en/v2/Git-Tools-Submodules) for this kind of task, but any system should work so long as the paths are adjusted properly.
To set up LLVM as a submodule, first:
```
$ mkdir vendor
$ git submodule add git@github.com:llvm/llvm-project.git vendor/llvm
```
This is a big repo, so cloning it can take several minutes (you can do `--depth` to limit the amount of history pulled, but make sure you leave enough history to check out a release version).
Then in `vendor/llvm`:
```
$ git checkout 'release/15.x' # Or choose another release version; or just work off the latest commit on main if you're daring
```
Now, we have to build clang and LLVM; this took me several hours on my laptop with numerous false starts because the linker kept getting killed when my laptop ran out of memory.
As such, this configuration command makes some choices (specificially using Ninja and `lld`, while only allowing one link job at a time) that might be unnecessary for someone on a beefy workstation; feel free to tweak if you want different performance.
Regardless, here's what worked for me (run in `vendor/llvm`, of course):
```
$ cmake -S llvm -B build -DCMAKE_BUILD_TYPE=Debug -G Ninja -DLLVM_USE_LINKER=lld -DLLVM_PARALLEL_LINK_JOBS=1 -DLLVM_ENABLE_PROJECTS=clang
```
then, to actually perform the build
```
$ cmake --build build
```

Now that LLVM is actually built, we need to cajole CMake (and all of LLVM's associated CMake scripts) to use the newly built version.
Unfortunately, to my knowledge all of CMake's supporting scripts expect that files are located as they would be in an installation root, so it seems the best way to do this is to use LLVM's install script, but pointed at somewhere different.
I "installed" to `vendor/llvm-install` using this command, run from `vendor/llvm/build`:
```
$ cmake -DCMAKE_INSTALL_PREFIX=../../llvm-install -P cmake_install.cmake
```
And I would make sure that `llvm-install` is in your `.gitignore`.

Then, point your top level `CMakeLists.txt` at the freshly built LLVM/clang:
```cmake
set(LLVM_DIR vendor/llvm-install/lib/cmake/llvm)
find_package(LLVM REQUIRED CONFIG NO_DEFAULT_PATH)
separate_arguments(LLVM_DEFINITIONS_LIST NATIVE_COMMAND ${LLVM_DEFINITIONS})
add_definitions(${LLVM_DEFINITIONS_LIST})
include_directories(SYSTEM ${LLVM_INCLUDE_DIRS})
list(APPEND CMAKE_MODULE_PATH "${LLVM_CMAKE_DIR}")
include(AddLLVM)

set(Clang_DIR vendor/llvm-install/lib/cmake/clang)
find_package(Clang REQUIRED CONFIG NO_DEFAULT_PATH)
include_directories(SYSTEM ${CLANG_INCLUDE_DIRS})
```

## Your First Plugin
Fundamentally, a clang plugin is a class that inherits from `PluginASTAction`, which must eventually get registered with clang so that it's run. This looks like:
```cpp
class ExampleAction : public PluginASTAction {
public:
  std::unique_ptr<ASTConsumer> CreateASTConsumer(CompilerInstance& inst, llvm::StringRef) override {
    // ...
    return std::unique_ptr<ASTConsumer>();
  }

  bool ParseArgs(const CompilerInstance& inst, const std::vector<std::string>& args) override {
    // Command line args to the plugin get parsed here. "true" means success, "false" aborts compilation.
    // For our purposes we ignore command line arguments and return true no matter what.
    return true;
  }
}
static FrontendPluginRegistry::Add<ExampleAction> X("example", "Performs your cool analysis");
```
leaving out all the necessary `#include`s and `using namespace ...`es.
At this point, it would be prudent to make sure that all libraries and versioning issues are sorted out such that this very simple example will compile properly (though it won't run without a proper return from `CreateASTConsumer`).

Once the plugin is built with cmake (if you're unfamiliar, do `cmake -S . -B build/` followed by `cmake --build build`), you'll have an object file (e.g., `example.o`) that can be used as a clang plugin. Congratulations!

## Clang AST
Taking a detour from code, there's some helpful tooling that makes writing/debugging plugins that make use of the AST much easier.
First, the command `clang -cc1 -ast-dump your_file.c` will pretty-print a text dump of the AST to the console, which can be helpful for getting an overview of the typical shape of the tree.
Next, the command `clang-query` is useful for experimenting with AST Matchers; a detailed writeup of its use for this purpose can be found [here](https://devblogs.microsoft.com/cppblog/exploring-clang-tooling-part-2-examining-the-clang-ast-with-clang-query/).
AST Matchers are a pseudo DSL provided by clang that allows one to concisely specify a pattern of AST nodes; a full reference is available [here](https://clang.llvm.org/docs/LibASTMatchersReference.html).

## Your First (Useful) Plugin
Consider this scenario: you're a TA for a computer science course teaching C that has very strict rules for grading code quality.
Specifically, each call to `malloc` must look like `type_t *val = (type_t *)malloc(n * sizeof(*val))`.
Grading this by hand is tedious and error prone, so you'd like clang to do it for you using its AST, emitting a diagnostic whenever a call to `malloc` that doesn't match this pattern is detected.
This means we must construct an `ASTConsumer` that traverses the AST looking for calls to `malloc` that break our pattern.
