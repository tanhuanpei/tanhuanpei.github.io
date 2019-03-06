---
title: Android进阶之APT in Android
date: 2019-03-06 14:17:41
tags: APT 进阶
---

#### 概述

**APT**(Annotation Processing Tool)是一种处理注释的工具，它对源代码文件进行检测找出其中的Annotation，使用APT进行额外的处理。APT在处理Annotation时可以根据源文件中的Annotation生成额外的源文件和其他的文件（文件具体内容由Annotation处理器的编写者决定），APT还会编译生成的源文件和原来的源文件，将它们一起生成class文件。

#### 实现自定义注解

利用APT实现自定义注解，主要由以下几大步骤完成。

##### 第一步：自定义Annotation

通过Java提供的元注解，可以实现自定义Annotation。下面先介绍Java的元注解。

元注解，负责注解其他注解，Java 5.0定义了4个标准的meta-annotation类型，他们被用来提供对其他annotation类型作说明。

1. @Target

   规定Annotation所修饰的对象范围。

   - ElementType.CONTRUCTOR：构造器声明
   - ElementType.FIELD：成员变量、对象、属性（包括enum实例）
   - ElementType.LOCAL_VARIABLE：局部变量声明
   - ElementType.METHOD：方法声明
   - ElementType.PACKAGE：包声明
   - ElementType.PARAMETER：参数声明
   - ElementType.TYPE：类、接口（包括注解类型）或enum实例

   使用实例

   ```java
   @Target(ElementType.TYPE)
   public @interface Table {
       /**
        * 数据表名称注解，默认值为类名称
        * @return
        */
       public String tableName() default "className";
   }
   ```

2. @Retention

   标示对Annotation的“生命周期”限制。

   - RetentionPolicy.SOURCE：只在源代码中保留，一般用来增加代码的理解性或者帮助代码检查之类的，比如Override
   - RetentionPolicy.CLASS：默认的选择，能把注解保留到编译后的字节码文件中，仅仅到字节码文件中，运行时是无法得到的
   - RetentionPolicy.RUNTIME：注解不仅能保留到class字节码文件中，还能在运行通过反射获取到，这也是我们常用到

   使用实例

   ```java
   @Documented
   @Target(PARAMETER)
   @Retention(RUNTIME)
   public @interface Body {
   
   }
   ```

3. @Document

   是一个标注注解，没有成员，用于标示可以被工具文档化。

   ```java
   @Documented
   @Retention(RetentionPolicy.RUNTIME)
   @Target(ElementType.ANNOTATION_TYPE)
   public @interface Documented {
   }
   ```

4. @Inherited

   是一个标注注解，没有成员，用于标示被标注的类型是可以被继承的，如果此注解用于一个class，则此class的注解也将应用于子类。

注解元素其参数成员只能是基本类型**byte**,**short**,**char**,**int**,**long**,**float**,**double**,**boolean**八种基本数据类型和 **String**,**Enum**,**Class**,**annotations**等数据类型,以及这一些类型的数组。这些参数必须有确定的值，要么在定义注解是指定默认值，要么在使用注解时指定，非基本类型的注解元素不可为null，因为，使用空字符串或者0作为默认值是一种常用的做法。

```java
@Documented
@Target(PARAMETER)
@Retention(RUNTIME)
public @interface QueryMap {
  /** Specifies whether parameter names and values are already URL encoded. */
  boolean encoded() default false;
}
```

```java
@Documented
@Target(METHOD)
@Retention(RUNTIME)
public @interface HEAD {
  /**
   * A relative or absolute path, or full URL of the endpoint. This value is optional if the first
   * parameter of the method is annotated with {@link Url @Url}.
   * <p>
   * See {@linkplain retrofit2.Retrofit.Builder#baseUrl(HttpUrl) base URL} for details of how
   * this is resolved against a base URL to create the full endpoint URL.
   */
  String value() default "";
}
```



###### 创建Annotation Module

知道了理论知识之后，我们新建一个名称为annotation的Java Library，主要放置自定义的Annotation，这里自定义一个适用于方法的注解

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface SysTrace {
}
```



配置build.gradle，主要是规定jdk版本，不同工程不同jdk版本可能会有编译的问题。

```java
apply plugin: 'java'

dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])
}

sourceCompatibility = "1.7"
targetCompatibility = "1.7"
```



##### 第二步：识别和处理Annotation

Java API已经提供了扫描源码并解析注解的框架，开发者可以通过继承`AbstractProcessor`类来实现自己的注解解析逻辑。APT的原理就是在注解了某些代码元素（如字段、函数、类等）后，在编译时编译器会检查`AbstractProcessor`的子类，并且自动调用其`process()`方法，然后将添加了指定注解的所有代码元素作为参数传递给此方法，开发者再根据注解元素在编译期输出对应的Java代码。



###### 创建Process Module

创建一个名为process的Java Library，解析注解的代码将放在这里。配置build.gradle文件

```groovy
apply plugin: 'java'

dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])
    implementation 'com.google.auto.service:auto-service:1.0-rc3'
    implementation 'com.squareup:javapoet:1.7.0'
    implementation project(':annotation')
}
sourceCompatibility = "1.7"
targetCompatibility = "1.7"
```



###### 新建Processor处理类

`AbstractProcessor`是javac扫描和处理注解的关键类。我们需要继承自这个类来完成代码生成的功能。

```java
@AutoService(Process.class)
@SupportedAnnotationTypes("com.example.annotation.DebugTrace")
public class TraceProcessor extends AbstractProcessor {

    @Override
    public synchronized void init(ProcessingEnvironment processingEnv) {
        super.init(processingEnv);
    }

    @Override
    public Set<String> getSupportedAnnotationTypes() {
        Set<String> annotationType = new HashSet<>();
        annotationType.add("com.example.annotation.DebugTrace");
        return annotationType;
    }

    @Override
    public SourceVersion getSupportedSourceVersion() {
        return SourceVersion.latestSupported();
    }

    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {

        return true;
    }

}
```

下面我们对重写的各个方法的作用进行介绍：

- `init(ProcessingEnvironment processingEnv)`：javac会在Processor创建时调用并执行初始化操作，该方法会传入一个参数ProcessingEnvironment，通过env可以方法Elements、Types、Filer等工具类。
- `getSupportedAnnotationTypes()`：指定哪些注解需要注册，也可以使用注解`@SupportedAnnotationTypes`
- `getSupportedSourceVersion() `：指定支持的 java 版本，通常返回`SourceVersion.latestSupported()`，如果只想支持到 Java 7 可以返回 `SourceVersion.RELEASE_7`
- `process(Set<? extends TypeElement> annotations, RoundEnvironment env)`: 它是每个 processor 的主方法，可以在这个方法中扫描和处理注解，并生成新的 java 源代码，通过参数 RoundEnvironment env 可以找到我们想要的某一个被注解的元素。return true表示注解已经被此processor消费，后续的其他processor不需要再处理。return flase则表示后续其他processor可能也处理需要处理这些注解。

`process()`是最重要的方法，通过传递的参数我们可以拿到注解信息以及其注解的元素，再通过[javapoet](https://github.com/square/javapoet )拼接代码。

注解和代码生成我们都已经处理，但`TraceProcessor`类什么时候被执行呢？javac在构建项目时，会自动检测并读取 javax.annotation.processing.Processor 来注册列出来的 Processor，这样我们的代码生成逻辑就能够得到执行。

![1548927537819](../../.config/Typora/typora-user-images/1548927537819.png)

```java
com.example.processor.TraceProcessor;
```

如果不想这么麻烦，我们可以使用Google提供的auto-service库，在processor类添加`@AutoService(Process.class)`，这样就自动完成注册功能。



##### 第三步：引用annotationProcessor，使用Annotation

在前面我们已经自定义注解并处理了注解解析流程，现在只需要在用到注解的工程，通过annotationProcessor指令添加我们的注解处理器。

```groovy
dependencies {
 	......
    implementation project(':annotation')
    annotationProcessor project(':processor')
}
```

`annotationProcessor`是APT工具中的一种，随着Android Gradle 2.2插件发布，已经内置到最新Gradle插件，不需要引用，可以直接在build.gradle文件中使用。~~`android-apt`~~,另外一个APT工具，已由Android官方`annotationProcessor`替代。

APT在编译时自动查找所有继承自`AbstractProcessor`的类，然后调用他们的`process`方法去处理，这样就拥有了在编译过程中执行代码的能力。

至此，已经把自定义注解和编译时注解的流程讲述完毕。