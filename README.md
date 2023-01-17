# Clang Plugins Overview
This document is intended as a brief introduction to clang plugins.
This work, unless otherwise specified, is licensed under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/).

## Building
This part of the guide is likely going to be what will go stale fastest, so checking an up-to-date book on clang for its build proceedures is probably the best place to start troubleshooting.
Personally, I found [this book](https://learning.oreilly.com/library/view/llvm-techniques-tips/9781838824952/) to be very helpful, and would recommend it as a starting point for further reading.
This said, clang is built with cmake, so using cmake to build your plugin can make the process a lot easier.
Since the clang repo itself is quite large, and I don't have any plans to contribute upsteam, I prefer to use an out-of-tree build.
To do this, once all the proper libraries are installed on your system, you can use a `CMakeLists.txt` with the following to expose the proper header files to your compiler:
```cmake
find_package(LLVM REQUIRED CONFIG)
add_definitions(${LLVM_DEFINITIONS})
include_directories(${LLVM_INCLUDE_DIRS})
link_directories(${LLVM_LIBRARY_DIR})
list(APPEND CMAKE_MODULE_PATH "${LLVM_CMAKE_DIR}")
include(AddLLVM)

find_package(Clang REQUIRED CONFIG)
include_directories(SYSTEM ${CLANG_INCLUDE_DIRS})
```
you'll also want to make sure RTTI and exceptions are disabled to make sure that there aren't any ABI issues, e.g., via
```cmake
add_compile_options(-fno-rtti -fno-exceptions)
```

Since clang plugins are dynamically loaded, the version of clang they're built against and the version of clang they're run with should match.
Because the plugin API isn't stable, this can make updating/porting plugins a pretty significant maintainance overhead.

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

