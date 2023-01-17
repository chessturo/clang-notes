add_library(example SHARED plugin.cpp)
target_compile_options(example PUBLIC -fno-exceptions -fno-rtti)
