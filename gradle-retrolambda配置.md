1. 下载jdk1.8, 并在系统环境变量中配置。
2. 右键项目，---》 open module settings ---》 sdk Location，设置 jdk location为java 8 安装地址。
3. android studio ---》 settings----》 path variables 增加新的路径变量 JAVA8_HOME,JAVA7_HOME，分别指向对应的安装路径。
4. 项目的build gradle配置如下
```java

buildscript {
    repositories {
        jcenter()
        mavenCentral()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:1.5.0'

        // NOTE: Do not place your application dependencies here; they belong
        // in the individual module build.gradle files\

        classpath 'me.tatarka:gradle-retrolambda:3.2.0'
    }
}

allprojects {
    repositories {
        jcenter()
        mavenCentral()
    }
}

task clean(type: Delete) {
    delete rootProject.buildDir
}
```

5. 其他module的buidl gradle配置如下
```java
apply plugin: 'com.android.application'
apply plugin: 'me.tatarka.retrolambda'

android {
    compileSdkVersion 23
    buildToolsVersion "23.0.2"

    defaultConfig {
        applicationId "com.wsy.realm"
        minSdkVersion 9
        targetSdkVersion 23
        versionCode 1
        versionName "1.0"
    }
    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }
    compileOptions {
        sourceCompatibility JavaVersion.VERSION_1_8
        targetCompatibility JavaVersion.VERSION_1_8
    }
    //保证Processor使用遇到第一个包中定义
    packagingOptions {
        pickFirst 'META-INF/services/javax.annotation.processing.Processor'
    }
}

retrolambda {
    javaVersion JavaVersion.VERSION_1_7
}

//设置编码
tasks.withType(JavaCompile) {
    options.encoding = "UTF-8"
}

dependencies {
    compile fileTree(dir: 'libs', include: ['*.jar'])
    testCompile 'junit:junit:4.12'
    compile 'com.android.support:appcompat-v7:23.1.1'
    compile 'io.realm:realm-android:0.87.0'
    compile 'com.jakewharton:butterknife:7.0.1'
}

```