---
APT、JavaPoet 实现 ButterKnife
---

#### 目录

1. 思维导图
2. 概述
3. APT
4. AutoService
5. JavaPoet
6. ButterKnife 的实现
7. 参考

#### 思维导图

![](https://i.loli.net/2019/01/26/5c4c00a69d7a5.png)

#### 概述

用 APT、JavaPoet、AutoService 实现简单的 ButterKnife，APT 负责处理编译时注解，JavaPoet 用于生成 Java 代码，AutoService 负责注册注解处理器。

#### APT 注解处理器 

APT（Annotation Processing Tool）即注解处理器，是一种注解处理工具，用来在编译器扫描和处理注解，通过注解来生成 Java 文件。即以注解作为桥梁，通过预先规定好的代码生成规则来自动生成 Java 文件。此类注解框架的代表有 ButterKnife、Dagger2、EventBus 等。

Java API 已经提供了扫描源码并解析注解的框架，开发者可以通过继承 AbstractProcessor 类来实现自己的注解处理逻辑。APT 的原理是在注解了某些代码元素（如字段、函数、类等）后，在编译时编译器会检查 AbstractProcessor 的子类，并且自动调用其 process() 方法，然后将添加了指定注解的所有代码元素作为参数传递给该方法，开发者在根据注解元素在编译期输出对应的 Java 代码。

##### AbstractProcessor

实现一个注解处理器，需要继承 AbstractProcessor ，如下：

```java
public class BindViewProcessor extends AbstractProcessor {

    private Elements mElementsUtils;
    private Types mTypesUtils;
    private Filter mFilter;
    private Messager mMessager;

    /**
     * 初始化方法
     * 可以初始化一些给注解处理器使用的工具类
     */
    @Override
    public synchronized void init(ProcessingEnvironment processingEnvironment) {
        super.init(processingEnvironment);
        mElementsUtils = processingEnvironment.getElementUtils();
    }

    /**
     * 指定目标注解对象
     */
    @Override
    public Set<String> getSupportedAnnotationTypes() {
        Set<String> hashSet = new HashSet<>();
        hashSet.add(BindView.class.getCanonicalName());
        return hashSet;
    }

    /**
     * 指定使用的 Java 版本
     */
    @Override
    public SourceVersion getSupportedSourceVersion() {
        return SourceVersion.latestSupported();
    }

    /**
     * 处理注解
     */
    @Override
    public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment) {
		//...
        return true;
    }
}
```

对于 APT，其实主要是有很多 API 不熟悉。

Elements：用于处理程序元素的工具类；

Types：用于处理类型数据的工具类；

Filter：用于给注解处理器创建文件；

Messager：用于给注解处理器报告错误、警告、提示等信息。

##### Element 元素相关

注解处理器工具扫描 Java 源文件，源文件中的每一部分都是程序中的 Element 元素，如包、类、方法、字段等。例如源代码中的类声明信息代表 TypeElement 类型元素，方法声明信息代表 ExecutableElement 类型元素，有了这些结构，就能完整的表示整个源代码信息了。

Element 元素分为以下类型：

1. ExcecutableElement

   可执行元素，包括类或接口的方法、构造方法或初始化程序。

2. PackageElement

   包元素，提供对有关包及其成员的信息的访问。

3. TypeElement

   类或接口元素，提供对有关类型及其成员的信息的访问。

4. TypeParameterElement

   表示一般类、接口、方法或构造方法元素的形式类型参数，类型参数声明一个 TypeVariable

5. VariableElement

   表示一个字段、enum 常量、方法或构造方法参数、局部变量或异常参数。

#### AutoServcie 注册注解处理器

以前要注册注解处理器要在 module 的 META_INFO 目录新建 services 目录，并创建一个名为 Java.annotation.processing.Processor 的文件，然后在文件中写入要注册的注解处理器的全民。

后来 Google 推出了 AutoService 注解库来实现注册注解处理器的注册，通过在注解处理器上加上 @AutoService(Processor.class) 注解，即可在编译时生成 META_INFO 信息。

```java
@AutoService(Processor.class)
public class BindViewProcessor extends AbstractProcessor {
}
```

#### JavaPoet 生成 Java 代码

JavaPoet 中有几个常用的类：

MethodSpec：代表一个构造方法或方法声明；

TypeSpec：代表一个类、接口、或者枚举声明；

FieldSpec：代表一个成员变量、字段声明；

JavaFile：包含一个顶级类的 Java 文件；

关于它的使用，直接看官方文档即可：

[https://github.com/square/javapoet](https://github.com/square/javapoet)

#### ButterKnife 的实现

分为四步：

1. 定义注解
2. 注解处理器处理注解
3. 生成 Java 文件
4. 引入

##### 定义注解

```java
@Retention(RetentionPolicy.CLASS)
@Target(ElementType.FIELD)
public @interface BindView {
    int value();
}
```

##### 注解处理器处理注解、生成 Java 文件

```java
@AutoService(Processor.class)
public class BindViewProcessor extends AbstractProcessor {

    private Elements mElementsUtils;
    private Types mTypesUtils;
    private Filter mFilter;
    private Messager mMessager;

    /**
     * 初始化方法
     * 可以初始化一些给注解处理器使用的工具类
     */
    @Override
    public synchronized void init(ProcessingEnvironment processingEnvironment) {
        super.init(processingEnvironment);
        mElementsUtils = processingEnvironment.getElementUtils();
    }

    /**
     * 指定目标注解对象
     */
    @Override
    public Set<String> getSupportedAnnotationTypes() {
        Set<String> hashSet = new HashSet<>();
        hashSet.add(BindView.class.getCanonicalName());
        return hashSet;
    }

    /**
     * 指定使用的 Java 版本
     */
    @Override
    public SourceVersion getSupportedSourceVersion() {
        return SourceVersion.latestSupported();
    }

    /**
     * 处理注解
     */
    @Override
    public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment) {
        //获取所有包含 BindView 注解的元素
        Set<? extends Element> elementSet = roundEnvironment.getElementsAnnotatedWith(BindView.class);
        Map<TypeElement, Map<Integer, VariableElement>> typeElementMapHashMap = new HashMap<>();
        for (Element element : elementSet) {
            //因为 BindView 的作用对象是 FIELD，因此 element 可以直接转化为 VariableElement
            VariableElement variableElement = (VariableElement) element;
            //getEnclosingElement 方法返回封装此 Element 的最里层元素
            //如果 Element 直接封装在另一个元素的声明中，则返回该封装元素
            //此处表示的即是 Activity 类对象
            TypeElement typeElement = (TypeElement) variableElement.getEnclosingElement();
            Map<Integer, VariableElement> variableElementMap = typeElementMapHashMap.get(typeElement);
            if (variableElementMap == null) {
                variableElementMap = new HashMap<>();
                typeElementMapHashMap.put(typeElement, variableElementMap);
            }
            //获取注解的值，即 ViewId
            BindView bindAnnotation = variableElement.getAnnotation(BindView.class);
            int viewId = bindAnnotation.value();
            variableElementMap.put(viewId, variableElement);
        }
        for (TypeElement key : typeElementMapHashMap.keySet()) {
            Map<Integer, VariableElement> elementMap = typeElementMapHashMap.get(key);
            String packageName = ElementUtil.getPackageName(mElementsUtils, key);
            JavaFile javaFile = JavaFile.builder(packageName, generateCodeByPoet(key, elementMap)).build();
            try {
                javaFile.writeTo(processingEnv.getFiler());
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        return true;
    }

    /**
     * 生成 Java 类
     *
     * @param typeElement        注解对象的上层元素对象，即 Activity 对象
     * @param variableElementMap Activity 包含的注解对象以及注解的目标对象
     * @return
     */
    private TypeSpec generateCodeByPoet(TypeElement typeElement, Map<Integer, VariableElement> variableElementMap) {
        //自动生成的文件以 Activity 名 + ViewBinding 进行命名
        return TypeSpec.classBuilder(ElementUtil.getEnclosingClassName(typeElement) + "ViewBinding")
                .addModifiers(Modifier.PUBLIC)
                .addMethod(generateMethodByPoet(typeElement, variableElementMap))
                .build();
    }

    /**
     * 生成方法
     *
     * @param typeElement        注解对象上层元素对象，即 Activity 对象
     * @param variableElementMap Activity 包含的注解对象以及注解的目标对象
     * @return
     */
    private MethodSpec generateMethodByPoet(TypeElement typeElement, Map<Integer, VariableElement> variableElementMap) {
        ClassName className = ClassName.bestGuess(typeElement.getQualifiedName().toString());
        //方法参数名
        String parameter = "_" + StringUtil.toLowerCaseFirstChar(className.simpleName());
        MethodSpec.Builder methodBuilder = MethodSpec.methodBuilder("bind")
                .addModifiers(Modifier.PUBLIC, Modifier.STATIC)
                .returns(void.class)
                .addParameter(className, parameter);
        for (int viewId : variableElementMap.keySet()) {
            VariableElement element = variableElementMap.get(viewId);
            //被注解的字段名
            String name = element.getSimpleName().toString();
            //被注解的字段的对象类型的全名称
            String type = element.asType().toString();
            String text = "{0}.{1}=({2})({3}.findViewById({4}));\n";
            methodBuilder.addCode(MessageFormat.format(text, parameter, name, type, parameter, String.valueOf(viewId)));
        }
        return methodBuilder.build();
    }
}
```

##### 引入

```java
public class ButterKnife {
    public static void bind(Activity activity) {
        Class clazz = activity.getClass();
        try {
            Class bindViewClass = Class.forName(clazz.getName() + "ViewBinding");
            Method method = bindViewClass.getMethod("bind", clazz);
            method.invoke(null, activity);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```



#### 参考

[教你实现一个轻量级的注解处理器 APT](https://mp.weixin.qq.com/s/3zrAzOUGpovRRbuYnce3uw)

[拆 JakeWharton 系列之 ButterKnife](https://juejin.im/post/58f388d1da2f60005d369a09)

[ButterKnife原理分析(二)注解的处理](https://www.jianshu.com/p/bcddc376c0ef)



