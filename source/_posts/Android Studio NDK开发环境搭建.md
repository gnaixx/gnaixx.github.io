title: "android studio NDK开发环境搭建"
date: 2016-03-07 22:54:43
categories: android
tags: [android, ndk]
toc: true
description: Android studio在很早版本已经开始支持NDK开发了，但是好像一直没有出正式版的gradle插件，现在最新的版本插件版本是0.6.0-alpha5。之前使用好像有点问题，具体忘记了。所以一直使用0.4.0版本开发。

---

**0.4.0**是`Experimental Plugin`版本所以在使用的时候和正式版本的配置还是有挺大的差别的。[http://tools.android.com/tech-docs/new-build-system/gradle-experimental/0-4-0]()是官方文档，上面给出了`Experimental Plugin`版本使用说明，但是实际使用中还是发现了不少问题，比如签名配置。

###Change Config

主要修改包括三个文件

```shell
./app/build.gradle
./build.gradle
./gradle/wrapper/gradle-wrapper.properties
```
####./gradle/wrapper/gradle-wrapper.properties
每个插件版本都对应着一个gradle版本       

| Plugin Version | Gragle Version | 
| -------------- | -------------- |    
| 0.1.0          | 2.5            |
| 0.2.0          | 2.5            |
| 0.3.0-alpha3   | 2.6            |
| 0.4.0          | 2.8            |
| 0.6.0-alpha1   | 2.8            |
| 0.6.0-alpha5   | 2.10           |

   

因为我使用的是0.4.0版本所以需要将`./gradle/wrapper/gradle-wrapper.properties`中配置改为   

```properties
#Wed Oct 21 11:34:03 PDT 2015
distributionBase=GRADLE_USER_HOME
distributionPath=wrapper/dists
zipStoreBase=GRADLE_USER_HOME
zipStorePath=wrapper/dists
distributionUrl=https\://services.gradle.org/distributions/gradle-2.8-all.zip
```

####./build.gradle
修改classpath为当前的0.4.0版本插件   

```gradle
buildscript {
    repositories {
        jcenter()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle-experimental:0.4.0'

        // NOTE: Do not place your application dependencies here; they belong
        // in the individual module build.gradle files
    }
}

allprojects {
    repositories {
        jcenter()
    }
}

task clean(type: Delete) {
    delete rootProject.buildDir
}
```

####./app/build.gradle
`./app/build.gradle`文件需要改动的东西比较多，很多配置与正式版本不一样。

- 用`'com.android.model.application'`替换` 'com.android.application'`如果是library模块改为`com.android.model.library`
- 在`android外层`添加根节点`model { }`
- 需要添加`=`给属性赋值
- 添加元素需要用add函数

下面这个是我自己项目的配置：

```gradle
apply plugin: 'com.android.model.application'

model {
    def signConf

    android {
        compileSdkVersion = 23
        buildToolsVersion = "23.0.2"

        defaultConfig.with {
            applicationId = "com.example.gnaix.ndk"
            minSdkVersion.apiLevel = 15
            targetSdkVersion.apiLevel = 22
            versionCode = 1
            versionName = "1.0"

            buildConfigFields.with {
                create() {
                    type = "int"
                    name = "VALUE"
                    value = "1"
                }
            }
        }
    }
    
    //编译设置
    android.buildTypes {
        release {
            minifyEnabled = true
            proguardFiles.add(file('proguard-rules.pro'))
            signingConfig = signConf
            ndk.with {
                debuggable = false
            }
        }
        debug{
            minifyEnabled = false
            proguardFiles.add(file('proguard-rules.pro'))
            ndk.with {
                debuggable = true
            }
        }
    }

	//签名设置
    android.signingConfigs {
        create("myConfig") {
            keyAlias = 'gnaix'
            keyPassword = '990828785'
            storeFile = new File('/Users/xiangqing/Documents/workspace/android/gnaix.jks')
            storePassword = '990828785'
            storeType = "jks"
            signConf = it
        }
    }
    
	//ndk定义
    android.ndk {
        moduleName = "native"  //设置库(so)文件名称
		 //CFlags.add("-DCUSTOM_DEFINE")
        ldLibs.addAll(["log", "android", "EGL", "GLESv1_CM"])
        //ldFlags.add("-L/custom/lib/path")
        stl = "stlport_static"
    }

	//ndk编译参数
    android.productFlavors {
        create("arm") {
            ndk.abiFilters.add("armeabi")
        }
        create("arm7") {
            ndk.abiFilters.add("armeabi-v7a")
        }
        create("arm8") {
            ndk.abiFilters.add("arm64-v8a")
        }
        create("x86") {
            ndk.abiFilters.add("x86")
        }
        create("x86-64") {
            ndk.abiFilters.add("x86_64")
        }
        create("mips") {
            ndk.abiFilters.add("mips")
        }
        create("mips-64") {
            ndk.abiFilters.add("mips64")
        }
        // To include all cpu architectures, leaves abiFilters empty
        create("all")
    }
}

dependencies {
    compile fileTree(dir: 'libs', include: ['*.jar'])
    testCompile 'junit:junit:4.12'
    compile 'com.android.support:appcompat-v7:23.2.0'
}

```

###Signing Config
下面这段代码是官方文档给出的签名配置，但是在实际编译中报错了。改成上面的配置则可以了。   
<font color="red">PS:</font>之前我都是新建`gradle.properties`文件配置签名信息，方便多项目应用，可是在这里好像不行。  

```gradle
apply plugin: "com.android.model.application"

model {
    android {
        compileSdkVersion = 23
        buildToolsVersion = "23.0.2"
    }

    android.buildTypes {
        release {
            signingConfig = $("android.signingConfigs.myConfig")
        }
    }
    android.signingConfigs {
        create("myConfig") {
            storeFile = new File("/path/to/debug.keystore")
            storePassword = "android"
            keyAlias = "androiddebugkey"
            keyPassword = "android"
            storeType = "jks"
        }
    }
}
```

###Source Set
默认情况下C/C++文件是存放在`src/main/jni`目录下。可以配置android.sources节点改变路径。不过我还是喜欢默认路径。   

```gradle
model {
    android {
        compileSdkVersion = 22
        buildToolsVersion = "22.0.1"
    }
    android.ndk {
        moduleName = "native"
    }
    android.sources {
        main {
            jni {
                source {
                    srcDir "src"
                }
            }
        }
    }
}
```
如果有已经编译好的SO包要添加引用： 

```gradle
android.sources {
        main {
            jniLibs {
                dependencies {
                    library file("./libs/armeabi/libnative.so") abi "armeabi"
                    library file("./libs/armeabi-v7a/libnative.so") abi "armeabi-v7a"
                    library file("./libs/x86/libnative.so") abi "x86"
                    library file("./libs/mips/libnative.so") abi "mips"
                }
            }
        }
    }
```
注意这里需要写SO包命全称，加上lib。

###Other Config
文档中还介绍了将SO作为一个独立的lib编译的方式，在实际操作中我也失败了，当时项目时间紧也就没有研究了=_=#。    
***文档示例/lib/build.gradle：***
   
```gradle
apply plugin: "com.android.model.native"

model {
    android {
        compileSdkVersion = 21
    }
    android.ndk {
        moduleName = "hello"
    }
    android.sources {
        main {
            jni {
                exportedHeaders {
                    srcDir "src/main/headers"
                }
            }
        }
    }
}
```

文档中还提到其他配置的使用方法，因为项目中没有用到，我也没试过。   
示例项目地址[https://github.com/gnaix92/as-ndk](https://github.com/gnaix92/as-ndk)


**下一篇将会介绍SO开发的JNI调用**





