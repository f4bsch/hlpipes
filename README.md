# hlpipes

Integrate Halide pipelines with CMake projects

# Use in your existing CMake Porject
Point the CMake variable `HALIDE_DIR` to Halide path.
Set `HLPIPES_ROOT` to the directory path where your Halide generator `*_generator.cpp` sources are.

# Tested Toolchains
* Android Studio 3.0
* Visual Studio 2017
* CLion 


# Tested Targets
* arm-64-android (cross)
* x86_64-android (cross)
* x86_64-windows
* x86_64-linux


# Supported Features
* Profile
* OpenGL(ES)
* SSE4.1
