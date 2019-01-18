---
APT
---

#### 目录

1. 思维导图
2. 概述
3. ButterKnife 的实现
4. 参考

#### 思维导图

#### 概述

APT（Annotation Processing Tool）即注解处理器，是一种注解处理工具，用来在编译器扫描和处理注解，通过注解来生成 Java 文件。即以注解作为桥梁，通过预先规定好的代码生成规则来自动生成 Java 文件。此类注解框架的代表有 ButterKnife、Dagger2、EventBus 等。

Java API 已经提供了扫描源码并解析注解的框架，开发者可以通过继承 AbstractProcessor 类来实现自己的注解处理逻辑。APT 的原理是在注解了某些代码元素（如字段、函数、类等）后，在编译时编译器会检查 AbstractProcessor 的子类，并且自动调用其 process() 方法，然后将添加了指定注解的所有代码元素作为参数传递给该方法，开发者在根据注解元素在编译期输出对应的 Java 代码。