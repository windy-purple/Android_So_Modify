#Anroid逆向学习从编写so到静动态调试分析arm的一次总结
##一、前言
  **最近跟着教我兄弟学逆向这篇教程学习Android逆向，在第七课后作业反复折腾了好几天，正好在折腾的时候对前面的学习总结一波，动态分析一下arm汇编（静态看arm感觉跟看天书没什么区别。。。），涉及到的东西都很简单基础，大神就不要浪费时间了！！！**

---
##二、所使用到的工具
- **Android studio v3.3**
- **IDA v7.0**
- **AndroidKiller**
- **ApkToolBox v1.6.4**

---

##三、编写所需要用到的so和apk文件
  **关于怎么编写Android应用和so文件，网上一大堆超详细的教程，这里就不再细说了，只简单说一下so文件的编写过程。**  
  **1、新建一个java类，使用`System.loadLibrary("so_name");`来加载so文件，在创建native层函数，我这里创建的一个名为add，形参为两个整数，返回值为一个整数的native函数**
![创建java类](https://raw.githubusercontent.com/windy-purple/Android_So_Modify/master/5.PNG)
  **2、在Android Studio的终端使用`javac java_name.java`命令编译刚才添加的类**
![编译java文件](https://raw.githubusercontent.com/windy-purple/Android_So_Modify/master/6.jpg)
  **3、跳转到java目录，生成.h文件，生成命令格式为`javah -jni Android项目包名.类名`**
![生成.h文件](https://raw.githubusercontent.com/windy-purple/Android_So_Modify/master/7.jpg)
  **4、在main文件夹下面新建jni文件夹，然后将上一步在java文件夹下面生成好的.h文件复制到刚新建好的jni文件夹下面，并在相应函数下面编写逻辑代码(我这里比较简单，只实现了两个整数相加并返回结果)，然后新建一个util.c的空文件（不加上这个文件会报错。。。）**
![新建jni文件夹](https://raw.githubusercontent.com/windy-purple/Android_So_Modify/master/8.png)
![编写函数逻辑代码](https://raw.githubusercontent.com/windy-purple/Android_So_Modify/master/9.PNG)
  **5、在build.gradle文件中添加相应配置，并且在src目录下建立CMakeLists.txt文件**
![添加ndk配置](https://raw.githubusercontent.com/windy-purple/Android_So_Modify/master/10.PNG)
  **[代码]**
```
 ndk{
            moduleName "myjni"
        }
        externalNativeBuild{
            cmake {
                cppFlags ""
                abiFilters "arm64-v8a","armeabi-v7a","x86","x86_64"
            }
        }
    }
    externalNativeBuild {
        cmake {
            path "CMakeLists.txt"
        }
    }
```
**[CMakeLists文件内容]**  

```
\# Sets the minimum version of CMake required to build the native
\# library. You should either keep the default value or only pass a
\# value of 3.4.0 or lower.

cmake_minimum_required(VERSION 3.4.1)

\# Creates and names a library, sets it as either STATIC
\# or SHARED, and provides the relative paths to its source code.
\# You can define multiple libraries, and CMake builds it for you.
\# Gradle automatically packages shared libraries with your APK.

add_library( # Sets the name of the library.AndroidStudio开始支持Cmake了，ndk感觉挺费劲的，这个是不是好玩点，，这里是要生成的库的文件名 libtest.so
             \#这里是liuxin
             myjni  \#so文件名字
             \# Sets the library as a shared library.
             SHARED

             \# Provides a relative path to your source file(s).
             \# Associated headers in the same location as their source
             \# file are automatically included.对应的C文件的目录位置
             src/main/jni/main.c)

\# Searches for a specified prebuilt library and stores the path as a
\# variable. Because system libraries are included in the search path by
\# default, you only need to specify the name of the public NDK library
\# you want to add. CMake verifies that the library exists before
\# completing its build.

find_library( \# Sets the name of the path variable.
              log-lib

              \# Specifies the name of the NDK library that
              \# you want CMake to locate.
              log )

\# Specifies libraries CMake should link to your target library. You
\# can link multiple libraries, such as libraries you define in the
\# build script, prebuilt third-party libraries, or system libraries.

target_link_libraries( # Specifies the target library.指定依赖库
                      \#这里是liuxin
                       myjni  \#so文件名字

                       \# Links the target library to the log library
                       \# included in the NDK.关联日志记录库文件，在ndk目录中
                       ${log-lib} )
```

  **6、在`Build->Rebuild Project`编译好so文件，so文件位置存放在`build->intermediates->cmake->debug->obj`目录下，选取相应的so文件在main的JniLibs目录下(该目录需要自己建立)，然后编译好apk即可**
![编译so](https://raw.githubusercontent.com/windy-purple/Android_So_Modify/master/11.png)

---

##四、破解该apk，将结果变为调用该so中该函数时无论参数输入多少，返回结果恒等于0
  **1、将apk拖进夜神中，观察一波(这里结果为52,参入参数为22和30)**
![夜神结果](https://raw.githubusercontent.com/windy-purple/Android_So_Modify/master/4.PNG)
![传入参数](https://raw.githubusercontent.com/windy-purple/Android_So_Modify/master/12.PNG)

  **2、将该apk拖进AndroidKiller中反编译，在jd中查看java代码(这里就不再分析smali代码了，直接看java)，可以看到在关键函数中调用myTest类的add函数，在jd中双击该类跟进，发现加载了so文件，并且定义了native函数int add(int,int)，所以经过上面分析要修改返回值需要修改so文件（也可以在smali层直接修改，但这篇文章主要讲so，如果有兴趣的可以去smali层修改）**
![1](https://raw.githubusercontent.com/windy-purple/Android_So_Modify/master/13.jpg)
![2](https://raw.githubusercontent.com/windy-purple/Android_So_Modify/master/14.jpg)
![3](https://raw.githubusercontent.com/windy-purple/Android_So_Modify/master/15.PNG)

 **3、使用ida静态分析myjni这个so文件。在AndroidKiller中找到该so文件，右键打开文件路径，然后拖进ida中，在export窗口(提供给外界调用的函数名集合的一个窗口）中找到add函数，双击进入该函数，可以看到汇编指令就这两条`ADDS            R0, R3, R2`  
`BX              LR` (因为我的函数功能过于简单所以就2条汇编指令，作为学习只有就不要纠结那么多了)，第一条意思很简单就是将r3和r2寄存器的值相加复制给r0寄存器，第二条指令意思是跳转到lr寄存器中所指地址中去执行下面的指令(lr是链路寄存器，用于保存函数返回地址，就是相当于存储了函数返回后下一条指令的地址)**
![4](https://raw.githubusercontent.com/windy-purple/Android_So_Modify/master/16.jpg)
![5](https://raw.githubusercontent.com/windy-purple/Android_So_Modify/master/17.jpg)
![6](https://raw.githubusercontent.com/windy-purple/Android_So_Modify/master/18.jpg)
![7](https://raw.githubusercontent.com/windy-purple/Android_So_Modify/master/19.jpg)
![arm](https://raw.githubusercontent.com/windy-purple/Android_So_Modify/master/arm%E5%AF%84%E5%AD%98%E5%99%A8.PNG)
  **4、动态调试。静态其实看着还是挺懵逼的，作为一个arm汇编的初学者，真的是搞不清楚调用函数过程中参数传到那个寄存器中去了，返回值跑哪里去了（暂时只关注这两点），所以那就动态调试so吧（记住一定要用真机调试，反正我用夜神模拟器调试就木有成功过，网上有大佬分析说的是模拟器底层还是x86的汇编，不是arm，所以有各种各样的奇葩错误无法解决）（而且要root）。**  
    **（1）、将手机连接好，并进入调试模式，将ida的dbgsrv->android_server拷贝到手机的/data/local/tmp目录下面(打开cmd，输入`adb push ida路径/dbgsrv/android_serevr /data/local/tmp`拷贝文件至手机)，然后输入adb shell进入调试模式下，执行`su`获取root权限，`cd /data/local/tmp`进入android_server所在目录下面，`chmod 777 android_server`赋予android_server文件777(可读可写可执行)权限，`./android_server`执行android_server文件，最后另外打开一个cmd窗口，执行`adb forward tcp:23946 tcp:23946`进行端口转发(23946是ida的默认端口，因为木有反调试所以懒得改了)。**
![20](https://raw.githubusercontent.com/windy-purple/Android_So_Modify/master/20.PNG)
![21](https://raw.githubusercontent.com/windy-purple/Android_So_Modify/master/21.PNG)
    **(2)、在手机上点击要调试的app启动，然后打开ida，在弹出的初始界面中，选择go这个选项，直接进入ida，然后选择Debugger->Attach->Remote ARMLinux/Android Debugger选项，在弹出的窗口中点击Debug Options选项，勾选下图所示三个选项（这三个选项名字太长了，麻烦看一下图吧），然后点击ok，在点击ok,弹出选择进程的界面，找到要调试的进程（可以使用serarch搜索进程），点击，然后点击ok，然后ida会附加到要调试的进程，在ida右侧的module哪里显示了所有加载的so文件，可以左键点击然后Ctrl+F搜索so文件（我这里so文件名为libmyJni.so，所以我搜索my就行了），找到对应的so文件后，双击即可弹出so文件对应的函数框(我这里是add函数)，然后双击对应的函数，ida会跳转到这个函数中去（我这儿就是跳转到了add函数中）。**

![22](https://raw.githubusercontent.com/windy-purple/Android_So_Modify/master/22.jpg)
![23](https://raw.githubusercontent.com/windy-purple/Android_So_Modify/master/23.png)
![24](https://raw.githubusercontent.com/windy-purple/Android_So_Modify/master/25.jpg)
![25](https://raw.githubusercontent.com/windy-purple/Android_So_Modify/master/24.jpg)
![26](https://raw.githubusercontent.com/windy-purple/Android_So_Modify/master/26.PNG)
![27](https://raw.githubusercontent.com/windy-purple/Android_So_Modify/master/27.PNG)
![29](https://raw.githubusercontent.com/windy-purple/Android_So_Modify/master/28.PNG)
![28](https://raw.githubusercontent.com/windy-purple/Android_So_Modify/master/29.PNG)
![30](https://raw.githubusercontent.com/windy-purple/Android_So_Modify/master/30.PNG)
    **3、经过上一步的配置，我们以及成功进入到要调试的函数中了，现在差开始调试了，在`ADDS            R0, R3, R2`处下一个断点（鼠标左键点击这行汇编代码，然后按F2即可下断点），然后按F9运行，在手机上点击按钮，即可看到程序停在了这行代码处，然后按F8单步调试，在右边寄存器处可以看到相关寄存器的16进制值，这里我们可以看到r0寄存器的16进制值为34（10进制为52），可见函数返回结果所用的寄存器为r0，r2寄存器16进制值为16（10进制值为22），对应了我们传进去的第一个参数22，r3寄存器的16进制值为1E（10进制为30），对应了我们传进去的第二个参数30。**

![31](https://raw.githubusercontent.com/windy-purple/Android_So_Modify/master/31.PNG)
![32](https://raw.githubusercontent.com/windy-purple/Android_So_Modify/master/32.PNG)
    **4、经过上面的动态分析，我们已经很清楚的知道该函数汇编运行过程--将一个参数值传入寄存器R2，第二个参数值传入寄存器R3，相加结果返回值送入寄存器R0。现在我们需要将结果很等于0，那么我们只需将R0复值为0返回即可。具体思路是将`ADDS R0,R2,R3`这行汇编代码改为`MOV R0,#0`即可。现在我们打开ida导入so文件，找到`ADDS R0,R2,R3`，我们可以点击ida的菜单栏的Options->General，在弹出的窗口中，将bytes改为4，即可显示处汇编指令对应的机器码，现在我们只需要将对应的机器码修改为`mov R0,#0`对应的机器码即可（可以用ApkToolBox这个工具的arm转机器码这个功能查看汇编对应的机器码）。我们可以使用patch来修改对应的机器码，首先，鼠标左键点击要修改的那行汇编代码，然后我们点击菜单栏的Edit->Patch program->Change bytes，在弹出的窗口修改对应的机器码(因为是Thumb模式，所以修改两个字节即可,这里对应的修改为的机器码为00 20)，然后点击菜单栏的Edit->Patch program->Apple patches to input file...即可保存修改。**

![37](https://raw.githubusercontent.com/windy-purple/Android_So_Modify/master/37.jpg)
![38](https://raw.githubusercontent.com/windy-purple/Android_So_Modify/master/38.PNG)
![3](https://raw.githubusercontent.com/windy-purple/Android_So_Modify/master/3.PNG)
![33](https://raw.githubusercontent.com/windy-purple/Android_So_Modify/master/33.png)
![34](https://raw.githubusercontent.com/windy-purple/Android_So_Modify/master/34.PNG)
![35](https://raw.githubusercontent.com/windy-purple/Android_So_Modify/master/35.png)
    **5、再将得到修改的so文件复制，在AndroidKiller中替换lib目录下所有so文件，然后重新编译即可，在将得到的apk文件安装好即可看到点击按钮显示结果为0。**

![36](https://raw.githubusercontent.com/windy-purple/Android_So_Modify/master/36.jpg)
![1](https://raw.githubusercontent.com/windy-purple/Android_So_Modify/master/1.PNG)

---

##五、结束语
  **相关附件链接：链接：https://pan.baidu.com/s/12a_l4JcuJj4i6nJty0xXXQ 提取码：licr 。如果觉得我写得还可以的话，可以给个免费评分哟，截图太难了！！！（手动狗头一波）**