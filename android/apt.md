# APT

> APT(Annotation Processing Tool) 是一种处理注释的工具, 它对源代码文件进行检测找出其中的 Annotation，使用 Annotation 进行额外的处理。
>  Annotation 处理器在处理 Annotation 时可以根据源文件中的 Annotation 生成额外的源文件和其它的文件 (文件具体内容由 Annotation 处理器的编写者决定),APT 还会编译生成的源文件和原来的源文件，将它们一起生成 class 文件。

## APT 是什么？

`编译期`的<u>注解处理工具</u> (Annotation Processing Tool)。 这本是 Java 的一个工具，但 Android 也可以使用。

## 作用

他可以用来处理编译过程时的某些操作，比如 Java 文件的生成，注解的获取等。

## **谁在用？**

一些主流的三方库，如 `ButterKnife`、`EventBus`、`dagger2` 、阿里的`ARouter`路由框架等都用到了 APT技术来生成代码。

## APT 原理

APT 就是借助 Javax 的注解库,在编译阶段扫描代码,将自定义注解的元素传入到注解处理器的 process() 方法中,然后生成我们想要的代码.比如生成 Java 文件.

生成代码原理很简单,就是用一个 StringBuilder 拼接字符串,然后通过工具类生成 Java 文件
。

## 构建过程分析

1. 构建工程,划份代码生成模块/Apt 调用模块
2. 继承 AbstarctProcessor ,重写 process 方法
3. Build 代码生成模块打 jar 包
4. Apt 调用模块导入该 jar 包
5. Apt 调用模块将生成的代码注入源代码

## **如何在Android Studio中构建一个APT项目?**

**创建 Annotation Module**

首先，我们需要新建一个名称为 apt-annotation 的` Java Library`，主要放置一些项目中需要使用到的 Annotation 和关联代码。这里简单自定义了一个注解：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.CLASS) 
public @interface Test {   
}
```

配置 build.gradle，主要是规定 JDK 版本

```groovy
plugins {
    id 'java-library'
}

// 控制台中文设置UTF-8
tasks.withType(JavaCompile){
    options.encoding = "UTF-8"
}

java {
    sourceCompatibility = JavaVersion.VERSION_1_8
    targetCompatibility = JavaVersion.VERSION_1_8
}
```

**创建 Processor Module**

创建一个名为 apt-processor 的` Java Library`，这个类将会写代码生成的相关代码。核心就是在这里，配置 build.gradle:

```groovy
plugins {
    id 'java-library'
}

dependencies {
    implementation fileTree(dir: 'libs', includes: ['*.jar'])

    // 编译时期进行注解处理
    annotationProcessor 'com.google.auto.service:auto-service:1.0-rc4'
    compileOnly 'com.google.auto.service:auto-service:1.0-rc4'

    // 帮助我们通过类调用的方式来生成Java代码[JavaPoet]
    implementation 'com.squareup:javapoet:1.10.0'

    // 依赖于注解
    implementation project(':apt-annotation')
}

// 控制台中文设置UTF-8
tasks.withType(JavaCompile){
    options.encoding = "UTF-8"
}

java {
    sourceCompatibility = JavaVersion.VERSION_1_8
    targetCompatibility = JavaVersion.VERSION_1_8
}
```

> 1、定义编译的 jdk 版本为 1.8，这个很重要，不写会报错。
>
> 2、AutoService 主要的作用是注解 processor 类，并对其生成 META-INF 的配置信息。
>
> 3、JavaPoet 这个库的主要作用就是帮助我们通过类调用的形式来生成代码。
>
> 4、依赖上面创建的 apt-annotation Module。

**定义 Processor 类**

```java
@AutoService(Processor.class)
public class TestProcessor extends AbstractProcessor {
    @Override
    public Set<String> getSupportedAnnotationTypes() {
        return Collections.singleton(Test.class.getCanonicalName());
    }
    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
        return false;
    }
}
```

生成第一个类，我们接下来要生成下面这个 HelloWorld 的代码：

```java
package com.example.helloworld;
public final class HelloWorld {
    public static void main(String[] args) {
        System.out.println("Hello, JavaPoet!");
    }
}
```

修改上述 TestProcessor 的 process 方法

```java
@Override    
public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
	MethodSpec main = MethodSpec.methodBuilder("main")
		.addModifiers(Modifier.PUBLIC, Modifier.STATIC)
	    .returns(void.class)
	    .addParameter(String[].class, "args")
	    .addStatement("$T.out.println($S)", System.class, "Hello, JavaPoet!")
	    .build();
	TypeSpec helloWorld = TypeSpec.classBuilder("HelloWorld")
	    .addModifiers(Modifier.PUBLIC, Modifier.FINAL)
	    .addMethod(main)
	    .build();
	JavaFile javaFile = JavaFile.builder("com.example.helloworld", helloWorld)
	    .build();
	try {
	    javaFile.writeTo(processingEnv.getFiler());
	} catch (IOException e) {
	    e.printStackTrace();
	}
	return false;
}
```

**app 中使用**

```groovy
dependencies {

    ......

    implementation project(':apt-annotation')
    annotationProcessor project(':apt-processor')
}
```

在随意一个类添加 @Test 注解，比如在 MainActivity 中：

```java
@Test
public class MainActivity extends AppCompatActivity {
	@Override
	protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
	}
}
```

点击 Android Studio 的 ReBuild Project，可以在 app Module中的`build/generated/ap_generated_sources/debug/out`目录下，即可看到生成的代码。

![image-20210705145700781](https://i.loli.net/2021/07/05/cjA5DFYZrNgnzWM.png)

完整代码：

```java
package com.wcx.apt_processor;

import com.google.auto.service.AutoService;
import com.squareup.javapoet.JavaFile;
import com.squareup.javapoet.MethodSpec;
import com.squareup.javapoet.TypeSpec;
import com.wcx.apt_lib.Test;
import java.util.Collections;
import java.util.Set;

import javax.annotation.processing.AbstractProcessor;
import javax.annotation.processing.Filer;
import javax.annotation.processing.Messager;
import javax.annotation.processing.ProcessingEnvironment;
import javax.annotation.processing.Processor;
import javax.annotation.processing.RoundEnvironment;
import javax.annotation.processing.SupportedAnnotationTypes;
import javax.annotation.processing.SupportedOptions;
import javax.annotation.processing.SupportedSourceVersion;
import javax.lang.model.SourceVersion;
import javax.lang.model.element.Modifier;
import javax.lang.model.element.TypeElement;
import javax.tools.Diagnostic;

@AutoService(Processor.class) // 编译期绑定
//@SupportedAnnotationTypes({"com.wcx.apt_lib.Test"}) // 表示我要处理那个注解
//@SupportedSourceVersion(SourceVersion.RELEASE_8)
public class TestProcessor extends AbstractProcessor {

    // 编译期打印日志
    private Messager messager;

    // Java文件生成器
    private Filer filer;

    /**
     * 这个方法做一些初始化工作，方法中有一个ProcessingEnvironment类型的参数，ProcessingEnvironment是一个注解处理工具的集合。它包含了众多工具类。例如：
     * Filer        可以用来编写新文件
     * Messager     可以用来打印错误信息
     * Elements     是一个可以处理Element的工具类
     * @param processingEnv
     */
    @Override
    public synchronized void init(ProcessingEnvironment processingEnv) {
        super.init(processingEnv);
        messager = processingEnv.getMessager();
        filer = processingEnv.getFiler();
    }

    /**
     * 这个方法非常简单，只有一个返回值，用来指定当前正在使用的Java版本，通常return SourceVersion.latestSupported()即可。
     * 和TestProcessor上@SupportedSourceVersion注解一样效果
     * @return
     */
    @Override
    public SourceVersion getSupportedSourceVersion() {
        //return SourceVersion.latestSupported();
        return SourceVersion.RELEASE_8;
    }

    /**
     * 这个方法的返回值是一个Set集合，集合中指要处理的注解类型的名称(这里必须是完整的包名+类名，例如com.wcx.apt_lib.Test)。由于在本例中只需要处理@Test，因此Set集合中只需要添加@Test。
     * 和TestProcessor上@SupportedAnnotationTypes注解一样效果
     * @return
     */
    @Override
    public Set<String> getSupportedAnnotationTypes() {
        //HashSet<String> supportTypes = new LinkedHashSet<>();
        //supportTypes.add(BindView.class.getCanonicalName());
        //return supportTypes;
        return Collections.singleton(Test.class.getCanonicalName());
    }

    /**
     * 在这个方法的方法体中，我们可以校验被注解的对象是否合法、可以编写处理注解的代码，以及自动生成需要的java文件等。因此说这个方法是AbstractProcessor 中的最重要的一个方法。我们要处理的大部分逻辑都是在这个方法中完成。
     * 这个方法的返回值，是一个boolean类型，返回值表示注解是否由当前Processor处理
     * true     :则这些注解由此注解来处理，后续其它的 Processor 无需再处理它们；
     * false    :则这些注解未在此Processor中处理并，那么后续 Processor 可以继续处理它们。
     * @param annotations
     * @param roundEnv
     * @return
     */
    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
        if(annotations.isEmpty()){
            messager.printMessage(Diagnostic.Kind.NOTE, "没有发现被ARouter注解的类");
            return false; // 标注注解处理器没有工作
        }
        this.processingEnv.getMessager().printMessage(Diagnostic.Kind.NOTE, "我就是测试一下编码啦~~~~");
        MethodSpec main = MethodSpec.methodBuilder("main")
                .addModifiers(Modifier.PUBLIC, Modifier.STATIC)
                .returns(void.class)
                .addParameter(String[].class, "args")
                .addStatement("$T.out.println($S)", System.class, "Hello, JavaPoet!")
                .build();
        TypeSpec helloWorld = TypeSpec.classBuilder("HelloWorld")
                .addModifiers(Modifier.PUBLIC, Modifier.FINAL)
                .addMethod(main)
                .build();
        JavaFile javaFile = JavaFile.builder("com.example.helloworld", helloWorld)
                .build();
        try {
            javaFile.writeTo(filer);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return true;
    }
}
```

可以看到，在这个类上添加了`@AutoService`注解，它的作用是用来生成META-INF/services/javax.annotation.processing.Processor文件的，也就是我们在使用注解处理器的时候需要手动添加META-INF/services/javax.annotation.processing.Processor，而有了@AutoService后它会自动帮我们生成。[AutoService](https://github.com/google/auto/tree/master/service)是Google开发的一个库，使用时需要在factory-compiler中添加依赖，如下：

```groovy
implementation 'com.google.auto.service:auto-service:1.0-rc4'
```

![image-20210705154426228](https://i.loli.net/2021/07/05/sPIqjhAYREQnT8x.png)

## 关于 Apt 的几个问题

1. 为什么 ButterKnife 注解方法的修饰符不能是 private?

   之前我也遇到过这一问题,在接触了 Apt 后迎刃而解,这是因为 Apt 技术会在同一个包中自动生成代码,并且在使用时注入到源代码中,在同一个包, private 修饰的方法对 apt 来说是不可见的.

2. Apt 生成的代码在什么时候介入?

   BufferKnife 以及我们今天要手撸的 FastClick,它们都是跟随 Activity/View 初始化,在 setContentView 方法被执行后,我们会通过 FastClick.init() 方法,在该方法将Apt生成的方法注入源代码.

3. 为什么需要单独创建一个 Java 库模块?

   这是因为 Apt 相关的代码被放在一个叫做 Javax 的库中,而我们的 Android 库无法使用到 Javax 包,除非去网上下载一个 jar 文件作为我们的库文件.所以我们需要单独创建,刚好将模块功能隔离开,该模块只负责使用 Apt 生成代码.

   !> 为什么要强调上述两个模块一定要是Java Library？如果创建Android Library模块你会发现不能找到AbstractProcessor这个类，这是因为Android平台是基于OpenJDK的，而OpenJDK中不包含APT的相关代码。因此，在使用APT时，必须在Java Library中进行。

   
   

4. 使用 Apt 和使用反射有什么区别?

   Apt 是基于编译时注解, 之前我们使用反射加注解是基于运行时注解,但是这种方式会在运行阶段做耗时的遍历操作,因此性能问题也一直被人们诟病; Apt 也需要用到反射来创建对象,但是不包含耗时操作.所以在性能上有它的优势.

