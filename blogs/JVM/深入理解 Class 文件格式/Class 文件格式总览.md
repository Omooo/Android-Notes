---
Class 文件格式总览
---

#### 文件格式总览

```xml
ClassFile{
    u4                  magic;
    u2                  minor_version;
    u2                  major_version;
    u2                  constant_pool_count;
    cp_info             constant_pool[constant_pool_count-1];
    u2                  access_flags;
    u2                  this_class;
    u2                  super_class;
    u2                  interfaces_count;
    u2                  interfaces[interfaces_count];
    u2                  fields_count;
    field_info          fields[fields_count];
    u2                  methods_count;
    method_info         methods[methods_count];
    u2                  attributes_count;
    attributes_info     attributes[attributes_count];
}
```

#### 简要说明

首先说明的是，以上的 u2 代表这个域长度为 2 个字节，u4 即代表 4 个字节。

1. 根据[规范](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-4.html#jvms-4.1)，Class 文件前八个字节依次是 magic 魔数（取值必须是 0xCAFEBABE）、minor_version（次版本信息）和 major_version（主版本信息）。
2. constant_pool_count 表示常量池数组中元素的个数，而 constant_pool 是一个存储 cp_info 的数组。每一个 Class 文件都包含一个常量池。注意，cp 数组的索引从 1 开始。
3. access_flags，标明该类的访问权限，比如 public、private 之类的信息。
4. this_class 和 super_class，存储的是指向常量池数组元素的索引。通过这两个索引可以知道当前类的类名以及父类名，只是类名，但不包含包名。
5. interfaces_count 和 interfaces，这两个成员表示该类实现了多少个接口以及接口类的类名。和 this_class 一样，这两个成员也只是常量池数组里的索引号。真正的信息需要通过解析常量池的内容才能得到。
6. fields_count 和 fields 包含了成员变量的数量以及它们的信息，成员变量信息由 field_info 结构体表示。
7. methods_count 和 methods 包含了成员函数的数量以及它们的信息，成员函数信息由 method_info 结构体表示。
8. attributes_count 和 attributes 包含了属性信息。属性信息由 attributes_info 结构体表示。属性包含哪些信息呢？比如调试信息就记录了某句代码对应源文件哪一行、函数对应的 Java 字节码也属于属性信息的一种。另外，源文件中的注解也属于属性。

