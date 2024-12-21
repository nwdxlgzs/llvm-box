# LLVM-BOX

> clang: clang clang++ clang-cl clang-cpp\
> lld: ld.lld ld64.lld lld lld-link wasm-ld\
> llvm-objcopy: llvm-bitcode-strip llvm-install-name-tool llvm-objcopy llvm-strip\
> llvm-ar: llvm-ar llvm-dlltool llvm-lib llvm-ranlib

## 第零步：准备LLVM源码和环境

- 这个环节不做介绍，自己搭建吧，用到啥自己下，我这里是LLVM18

## 第一步：LLVM源码打patch

- 在`clang/lib/Driver/Job.cpp`找到代码
```cpp
if (!InProcess)
   return Command::Execute(Redirects, ErrMsg, ExecutionFailed);
```
注释掉这段代码
```cpp
//if (!InProcess)
//   return Command::Execute(Redirects, ErrMsg, ExecutionFailed);
```

- 在`clang/lib/Driver/ToolChains/Gnu.cpp`找到代码
> 这里只针对c/cpp编译elf/pe的情况，如果要处理别的情况请在`clang/lib/Driver/ToolChains/`找到对应工具链文件修改
```cpp
  if (!D.SysRoot.empty())
    CmdArgs.push_back(Args.MakeArgString("--sysroot=" + D.SysRoot));
```
在它的前方加
```cpp
  std::string _LinkerPath=ToolChain.GetLinkerPath();
  //ld.lld ld64.lld lld lld-link wasm-ld
  if(_LinkerPath=="ld.lld" || _LinkerPath=="ld64.ld" ||
    _LinkerPath=="lld" || _LinkerPath=="lld-link" ||
    _LinkerPath=="wasm-ld"){
    CmdArgs.push_back("lld");
    CmdArgs.push_back(Args.MakeArgString(_LinkerPath));
    _LinkerPath = D.ClangExecutable;
  } else if(_LinkerPath==D.ClangExecutable){
    CmdArgs.push_back("lld");
    CmdArgs.push_back("ld.lld");
  }
```
找到代码
```cpp
  const char *Exec = Args.MakeArgString(ToolChain.GetLinkerPath());
```
改成
```cpp
  const char *Exec = Args.MakeArgString(_LinkerPath);
```

- 删除`llvm\tools\bugpoint-passes\CMakeLists.txt`和`\llvm\lib\Transforms\Hello\CMakeLists.txt`中的if限定条件
```cmake
if(WIN32 OR CYGWIN OR ZOS)
  set(LLVM_LINK_COMPONENTS Core Support)
endif()
```
改成
```cmake
# if(WIN32 OR CYGWIN OR ZOS)
  set(LLVM_LINK_COMPONENTS Core Support)
# endif()
```

## 第二步：第一遍clang和lld的编译

> 自己更换NDK路径，我这里是`D:\OLLVM\android-ndk-r27\`

> 如果你要支持`WebAssembly`就加，记得加之前改linker设置（lld那个），我这里没改，为了告诉你加哪里就后端支持上加了
```bat
mkdir build
cd build
cmake -G Ninja -DCMAKE_TOOLCHAIN_FILE=D:\OLLVM\android-ndk-r27\build\cmake\android.toolchain.cmake -DANDROID_ABI=arm64-v8a -DANDROID_PLATFORM=android-24 -DLLVM_ENABLE_PROJECTS="clang;lld" -DCMAKE_BUILD_TYPE=MinSizeRel -DLLVM_INCLUDE_TESTS=OFF -DLLVM_INCLUDE_BENCHMARKS=OFF -DLLVM_INCLUDE_EXAMPLES=OFF -DLLVM_HOST_TRIPLE=arm64-linux-android -DLLVM_TARGETS_TO_BUILD="ARM;X86;AArch64;RISCV;WebAssembly" ../llvm
ninja -j16
```

执行完毕重复执行`ninja`会看到
```txt
ninja: no work to do.
```

## 第三步：llvm-box创建
- 新建`build\llvm-box.cpp`内容为
```cpp
#include <string.h>
#include <stdio.h>
#include <string.h>
#include "llvm/Support/LLVMDriver.h"
#include "llvm/ADT/ArrayRef.h"
#include "llvm/Support/InitLLVM.h"
#ifdef _WIN32
#include <direct.h>
#define getcwd _getcwd
#else
#include <unistd.h>
#endif
//如果没设置，某些环境（安卓）无法重新调用自己
static inline void set_library_path(const char* new_path) {
#ifdef _WIN32
    const char* env_var_name = "PATH";
#else
    const char* env_var_name = "LD_LIBRARY_PATH";
#endif
    const char* current_path = getenv(env_var_name);
    if (current_path == NULL || strlen(current_path) == 0) {
#ifdef _WIN32
        SetEnvironmentVariable(env_var_name, new_path);
#else
        setenv(env_var_name, new_path, 1);
#endif
    } else {
        if (strstr(current_path, new_path) != NULL)return;
        char new_env_value[8196];
#ifdef _WIN32
        snprintf(new_env_value, sizeof(new_env_value), "%s;%s", current_path, new_path);
        SetEnvironmentVariable(env_var_name, new_env_value);
#else
        snprintf(new_env_value, sizeof(new_env_value), "%s:%s", current_path, new_path);
        setenv(env_var_name, new_env_value, 1);
#endif
    }
}
//clang: clang clang++ clang-cl clang-cpp
extern int clang_main(int argc, char**, const llvm::ToolContext&);
//lld: ld.lld ld64.lld lld lld-link wasm-ld
extern int lld_main(int argc, char**, const llvm::ToolContext&);
//llvm-objcopy: llvm-bitcode-strip llvm-install-name-tool llvm-objcopy llvm-strip
extern int llvm_objcopy_main(int argc, char**, const llvm::ToolContext&);
//llvm-ar: llvm-ar llvm-dlltool llvm-lib llvm-ranlib
extern int llvm_ar_main(int argc, char**, const llvm::ToolContext&);



int main(int argc, const char** argv, char* const* envp) {
    if (argc < 2) {
        //通过重新指定可执行程序名字，重新模拟main函数
        printf("Usage: %s <executable> <executable-name> [args...]", argv[0]);
        return 1;
    }
    //如果没设置，某些环境（安卓）无法重新调用自己
    char cwd[1024];
    if (getcwd(cwd, sizeof(cwd)) != NULL) set_library_path(cwd);
    //拿到真实的执行文件
    const char* executable = argv[1];
    //退掉自己（llvm-box）
    --argc;
    ++argv;
    //退掉executable（executable用于切换对应的main）
    --argc;
    ++argv;
    if (strcmp(executable, "clang") == 0) {
        llvm::InitLLVM X(argc, argv);
        return clang_main(argc, const_cast<char**>(argv), { argv[0], nullptr, false });
    }
    if (strcmp(executable, "lld") == 0) {
        llvm::InitLLVM X(argc, argv);
        return lld_main(argc, const_cast<char**>(argv), { argv[0], nullptr, false });
    }
    if (strcmp(executable, "llvm-objcopy") == 0) {
        llvm::InitLLVM X(argc, argv);
        return llvm_objcopy_main(argc, const_cast<char**>(argv), { argv[0], nullptr, false });
    }
    if (strcmp(executable, "llvm-ar") == 0) {
        llvm::InitLLVM X(argc, argv);
        return llvm_ar_main(argc, const_cast<char**>(argv), { argv[0], nullptr, false });
    }
    printf("Unknown program: %s -> %s\n", executable, argv[0]);
    printf("llvm-box support tools: \n"
           "<executable>: <executable-name> ...\n"
           "clang: clang clang++ clang-cl clang-cpp\n"
           "lld: ld.lld ld64.lld lld lld-link wasm-ld\n"
           "llvm-objcopy: llvm-bitcode-strip llvm-install-name-tool llvm-objcopy llvm-strip\n"
           "llvm-ar: llvm-ar llvm-dlltool llvm-lib llvm-ranlib\n"
    );
    return 1;
}
```
> 不要贪图内嵌更多组件，选项会冲突（重复定义选项，或者某些选择同时激活两个组件，其中一个报错也会终止）

编译命令根据`build.ninja`中的对应文件生成命令改，这里偷懒静态库部分直接指定`lib/*.a`了，毕竟要的就是`clang`和`lld`以及配套资源。

> 自己更换NDK路径，我这里是`D:\OLLVM\android-ndk-r27\`，项目目录`D:\OLLVM\llvm\oneClang\`

```bat
D:\OLLVM\android-ndk-r27\toolchains\llvm\prebuilt\windows-x86_64\bin\clang++.exe --target=aarch64-none-linux-android24 --sysroot=D:/OLLVM/android-ndk-r27/toolchains/llvm/prebuilt/windows-x86_64/sysroot -DGTEST_HAS_RTTI=0 -D__STDC_CONSTANT_MACROS -D__STDC_FORMAT_MACROS -D__STDC_LIMIT_MACROS -ID:/OLLVM/llvm/oneClang/build/tools/clang/tools/driver -ID:/OLLVM/llvm/oneClang/clang/tools/driver -ID:/OLLVM/llvm/oneClang/clang/include -ID:/OLLVM/llvm/oneClang/build/tools/clang/include -ID:/OLLVM/llvm/oneClang/build/include -ID:/OLLVM/llvm/oneClang/build/tools/lld/tools/lld -ID:/OLLVM/llvm/oneClang/lld/tools/lld -ID:/OLLVM/llvm/oneClang/lld/include -ID:/OLLVM/llvm/oneClang/build/tools/lld/include -ID:/OLLVM/llvm/oneClang/llvm/include -g -DANDROID -fdata-sections -ffunction-sections -funwind-tables -fstack-protector-strong -no-canonical-prefixes -D_FORTIFY_SOURCE=2 -Wformat -Werror=format-security -fPIC -fno-semantic-interposition -fvisibility-inlines-hidden -Werror=date-time -Werror=unguarded-availability-new -Wall -Wextra -Wno-unused-parameter -Wwrite-strings -Wcast-qual -Wmissing-field-initializers -pedantic -Wno-long-long -Wc++98-compat-extra-semi -Wimplicit-fallthrough -Wcovered-switch-default -Wno-noexcept-type -Wnon-virtual-dtor -Wdelete-non-virtual-dtor -Wsuggest-override -Wstring-conversion -Wmisleading-indentation -Wctad-maybe-unsupported -fdiagnostics-color -ffunction-sections -fdata-sections -fno-common -Woverloaded-virtual -Wno-nested-anon-types -Os -DNDEBUG -std=c++17 -fPIE  -fno-exceptions -funwind-tables -fno-rtti -o llvm-box.cpp.o -c D:\OLLVM\llvm\oneClang\build\llvm-box.cpp
```

得到编译产物`llvm-box.cpp.o`，然后准备链接指令构建。

```ninja
# Link build statements for EXECUTABLE target 某某某
```
或者
```ninja
build bin/某某某:
```

可以快速找到`bin/某某某`文件链接时的配置然后除了带`main`的`.o`，都收集，然后就拼凑出了下面的编译命令。

```bat
D:\OLLVM\android-ndk-r27\toolchains\llvm\prebuilt\windows-x86_64\bin\clang++.exe --target=aarch64-none-linux-android24 --sysroot=D:/OLLVM/android-ndk-r27/toolchains/llvm/prebuilt/windows-x86_64/sysroot -Os -DNDEBUG -g -DANDROID -fdata-sections -ffunction-sections -funwind-tables -fstack-protector-strong -no-canonical-prefixes -D_FORTIFY_SOURCE=2 -Wformat -Werror=format-security   -fPIC -fno-semantic-interposition -fvisibility-inlines-hidden -Werror=date-time -Werror=unguarded-availability-new -Wall -Wextra -Wno-unused-parameter -Wwrite-strings -Wcast-qual -Wmissing-field-initializers -pedantic -Wno-long-long -Wc++98-compat-extra-semi -Wimplicit-fallthrough -Wcovered-switch-default -Wno-noexcept-type -Wnon-virtual-dtor -Wdelete-non-virtual-dtor -Wsuggest-override -Wstring-conversion -Wmisleading-indentation -Wctad-maybe-unsupported -fdiagnostics-color -fdata-sections -fno-common -Woverloaded-virtual -Wno-nested-anon-types -static-libstdc++ -Wl,--build-id=sha1 -Wl,--no-rosegment -Wl,--no-undefined-version -Wl,--fatal-warnings -Wl,--no-undefined -Qunused-arguments -Wl,--color-diagnostics -Wl,--gc-sections -Wl,--export-dynamic llvm-box.cpp.o tools/clang/tools/driver/CMakeFiles/clang.dir/driver.cpp.o tools/clang/tools/driver/CMakeFiles/clang.dir/cc1_main.cpp.o tools/clang/tools/driver/CMakeFiles/clang.dir/cc1as_main.cpp.o tools/clang/tools/driver/CMakeFiles/clang.dir/cc1gen_reproducer_main.cpp.o tools/llvm-objcopy/CMakeFiles/llvm-objcopy.dir/ObjcopyOptions.cpp.o tools/llvm-objcopy/CMakeFiles/llvm-objcopy.dir/llvm-objcopy.cpp.o tools/lld/tools/lld/CMakeFiles/lld.dir/lld.cpp.o tools\llvm-ar\CMakeFiles\llvm-ar.dir\llvm-ar.cpp.o lib/*.a -lm -lz -latomic -o llvm-box
```

别着急走，`llvm-box`现在应该非常大，我们先进行第一次文件瘦身（或者上面编译时别用`-g`等参数），具体保不保留内部符号看个人了，只想去除调试也行，只去除调试它更安全，去除全部可去除的信息时需要确定程序可以运行，好在实际是可以运行的。

```bat
D:\OLLVM\android-ndk-r27\toolchains\llvm\prebuilt\windows-x86_64\bin\llvm-strip.exe --strip-debug llvm-box
D:\OLLVM\android-ndk-r27\toolchains\llvm\prebuilt\windows-x86_64\bin\llvm-strip.exe --strip-all llvm-box
```

如果你有点追求，可以尝试再套`upx`，进一步压缩体积，但是请注意不要追求大小，文件过小程序启动时解压负担会很大，会更慢。

```bat
upx llvm-box -o llvm-box_upx
```

如果你现在就着急push到安卓

```bat
adb push llvm-box /sdcard/libllvm-box.so
adb push llvm-box_upx /sdcard/libllvm-box.so
```

如果你手头有Androlua+源码，我这里提供一个编译demo，可以参考测试。（`-Wl,-s`等同于`--strip-all`）

```sh
cd AndroLua+项目目录（luajava源码拷贝到lua目录直接全编译）
llvm-box clang clang lctype.c linit.c lopcodes.c ltablib.c lapi.c ldblib.c liolib.c loslib.c ltm.c lauxlib.c ldebug.c llex.c lparser.c luajava.c lbaselib.c ldo.c lmathlib.c lstate.c lundump.c lbitlib.c ldump.c lmem.c lstring.c lutf8lib.c lcode.c lfunc.c loadlib.c lstrlib.c lvm.c lcorolib.c lgc.c lobject.c ltable.c lzio.c 
-save-temps=obj --target=aarch64-linux-android --sysroot=NDK中的sysroot目录 -resource-dir NDK中的sysroot同级目录的lib/clang/18 -I头文件目录 -shared -fPIC -Os -Wno-parentheses-equality -Wno-string-plus-int -Wl,-s -o libluajava.so
```

关于不同平台架构编译的说明

| CPU    | ABI                             |
| ------ | ------------------------------- |
| ARMv5  | armeabi                         |
| ARMv7  | armeabi, armeabi-v7a            |
| ARMv8  | armeabi, armeabi-v7a, arm64-v8a |
| ARMv9  | arm64-v8a, arm64-v9a            |
| MIPS   | mips                            |
| MIPS64 | mips, mips64                    |
| x86    | x86                             |
| x86_64 | x86, x86_64,                    |

- `armeabi-v7a`: `--target=armv7a-linux-androideabi`
- `arm64-v8a`: `--target=aarch64-linux-android`;`--target=aarch64-linux-android -march=armv9.5-a`
- `x86`: `--target=i686-linux-android -mstackrealign`
- `x86_64`: `--target=x86_64-linux-android`
- `riscv64`: `--target=riscv64-linux-android`

> 编译的补充说明：直接`execvp`调用是不支持`*.c`通配符的，`sh`传递过去时支持通配符。\
> `NDK`编译的`x86`默认是`i686`（`i[3-9]86`）。\
> `clang18`最高支持`armv9.5-a`，但是由于`ARMv9`支持`arm64-v8a`，有可能见不到`arm64-v9a`文件夹。\
> 古早一点的`thumb`属于`ARM32`（最早出现在`ARMv4T`），通过`thumb-linux-android`设置。
