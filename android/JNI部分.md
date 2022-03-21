### CPU 架构适配需要注意哪些问题

#### 不同的 CUP 架构之前兼容性如何

cpu 的架构有 ：x86，x86_64，arm64_v8a，armeabi-v7a，armeabi。

<img src="https://cdn.jsdelivr.net/gh/LvKang-insist/PicGo/202203091753037.png" alt="image-20220309175321982" style="zoom:50%;" />

兼容性如上所示，armeabi 可以兼容其他的，x86_64 兼容 x86，arm64-v8a 兼容 v7a。

#### Native 库加载

如果有一个在 arm64-v8a 的手机上运行的 app，在加载 so 的时候，它会优先去选择自己对应的 so 库，如下图所示：

<img src="https://cdn.jsdelivr.net/gh/LvKang-insist/PicGo/202203091758563.png" alt="image-20220309175828517" style="zoom: 50%;" />

如果找不到 v8a,就会继续寻找自己兼容的架构，如 v7a。

- 兼容模式运行的一些问题

  - 兼容模式运行的 native 库无法获得最好的性能

    所以 x86 电脑上运行 arm 的虚拟机会很慢

  - 兼容模式容易出现一些难以排查的问题

  - 系统优先加载对应架构目录下的 so 库

- SDK 开发者应当提供哪些 so 库

  应该结合目标的群体提供合适的架构。

  有些时候在 lib 目录下只提供一个 v7a 的目录，然后将 v8a 的 so 库也放进去，因为有些 so 可能适合在 v7a 上运行，而有些适合在 v8a 上运行。这种情况下又不想去提供两套架构的so库，就可以这么干。微信就是这样做的。

  还有就是通过线上监控的反馈，来分析该为哪些 so 提供 v8a 的版本等。

  动态从云端加载 so 库

- 优化 so 体积

  - 默认隐藏所有符号，只公开必要的

    - -fvisibility = hidden

  - 禁用 C++ Exception & RTTI

    - -fno -exceptions -fno -rtti

  - 不要使用 iostream，应优先使用 Android Log

  - 使用 gc -sections 去除无用代码

    - LOCAL_CFLAGS += ffuncation-sections -fdata-sections
    - LOCAL_LDFLAGS += -WL, --gc-sections

  - 构建时分包

    可以根据 cpu 的架构打出 n 个包，然后通过应用商场按照 CUP 架构分发安装包即可。

#### SDK 开发者需要注意什么

- 尽量不在 Native 层开发，降低问题跟踪维护成功
- 尽量优化 Native 库的体积，降低开发者使用的成本
- 必须提供完整的 CPU 架构依赖



### Java Native 方法和 Native 函数是怎么绑定的

- 静态绑定

  通过命名规则映射，栗子：

  ```java
  package com.example.cmakendkdemo;
  public class JNITools {
  
      public static native int addNumber(int a, int b);
  
      public static native int subNumber(int a, int b);
  
      public static native int mulNumber(int a, int b);
  
      public static native int divNumber(int a, int b);
  }
  
  ```

  ```c++
  extern "C"
  JNIEXPORT jint JNICALL
  Java_com_example_cmakendkdemo_JNITools_addNumber(JNIEnv *env, jclass clazz, jint a, jint b) {
      return a + b;
  }
  extern "C"
  JNIEXPORT jint JNICALL
  Java_com_example_cmakendkdemo_JNITools_subNumber(JNIEnv *env, jclass clazz, jint a, jint b) {
      return a - b;
  }
  extern "C"
  JNIEXPORT jint JNICALL
  Java_com_example_cmakendkdemo_JNITools_mulNumber(JNIEnv *env, jclass clazz, jint a, jint b) {
      return a * b;
  }
  extern "C"
  JNIEXPORT jint JNICALL
  Java_com_example_cmakendkdemo_JNITools_divNumber(JNIEnv *env, jclass clazz, jint a, jint b) {
      return a / b;
  }
  ```

  1. 名称规则

     可以看到，java 方法对应的 c++ 代码的名称都是通过 `Java+包名+类名+方法名称` 来定义的，其中你的  `.` 都被替换为了 `_` 。

  2. 参数

     1. JNIEnv
     2. java 方法是静态的，这里就是 jclass，如果是非静态，就是 jobject，表示方法引用的对象，剩下的就是从 java 传过来的参数了。

  3. 其他

     - extern "C"

       编译这个 native 函数的时候必须要按照 C 的规则保留名字，不能去混淆这个名字，不然经过混淆就找不到名字了。

     - JINEXPORT 
     - jint  ：返回值
     - JNICALL :

- 动态绑定

  通过 JNI 函数注册

  ```c++
  static int jniRegisterMethods(JNIEnv *env, const char *className, const JNINativeMethod *methods,
                                int numMethods) {
      //1,获取 Class
      jclass clazz;
      LogI("JNI", "Registering %s natives \n", className);
      clazz = env->FindClass(className);
  	
      //注册方法
      int result = 0;
      if ((env->RegisterNatives(clazz, gJni_Methods_table, numMethods) < 0) {
          return -1;
      }
  
      (env)->DeleteLocalRef(clazz)
      return result;
  }
  ```

  - 动态绑定可以在任何时刻触发
  - 动态绑定之前根据静态规则查找 Native 函数
  - 动态绑定可以在病毒后的任意时刻取消

- 对比

  |                     | 动态绑定         | 静态绑定                                  |
  | ------------------- | ---------------- | ----------------------------------------- |
  | Native 函数名       | 无要求           | 按照固有的规则编写且采用 C 的名称修饰规则 |
  | Native 函数可见性   | 无要求           | 可见                                      |
  | 动态更换            | 可以             | 不可见                                    |
  | 调用性能            | 无需查找         | 有额外查找开销                            |
  | 开发体验            | 几乎无副作用     | 重构代码较为繁琐                          |
  | Android Studio 支持 | 不能自动关联跳转 | 自动关联 JNI 函数可跳转                   |

  

 	