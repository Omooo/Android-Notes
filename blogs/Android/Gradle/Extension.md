---
Extension
---

#### 目录

1. 简单 Extension
2. 嵌套 Extension
3. 嵌套容器 Extension

#### 简单 Extension

```
student {
    name 'Omooo'
    age 18
    isMale false
}
```

```groovy
class Student {
    String name
    int age
    boolean isMale
}
```

```groovy
project.extensions.create('student', Student.class)
Student student = project.extensions.getByType(Student.class)
println(student.name)
```

#### 嵌套 Extension

```
student {
    name 'Omooo'
    age 18
    isMale false

    info {
        qq 2333
        email '2333@qq.com'
        isDog true
    }
}
```

```groovy
class Student {
    String name
    int age
    boolean isMale
    Info info

    @SuppressWarnings("UnstableApiUsage")
    Student(ObjectFactory factory) {
        info = factory.newInstance(Info.class)
    }

    def info(Action<Info> action) {
        action.execute(info)
    }


    static class Info {
        String email
        int qq
        boolean isDog
    }
}
```

```groovy
project.extensions.create('student', Student.class, project.objects)
Student student = project.extensions.getByType(Student.class)
println(student.name)
println(student.info.qq)
```

#### 嵌套容器 Extension

```
animal {
    count 2333

    dog {
        form 'Animal'
        isMale false
    }

    catConfig {
        shanghaiCat {
            from 'Shanghai'
            weight 20000.0f
        }

        beijingCat {
            from 'Beijing'
            weight 300f
        }
    }
}
```

```groovy
class Dog {
    String form
    boolean isMale

    @Override
    String toString() {
        return "Dog{" +
                "form='" + form + '\'' +
                ", isMale=" + isMale +
                '}'
    }
}
```

```groovy
class Cat {
    String name

    String from
    float weight

    Cat(String name) {
        this.name = name
    }

    @Override
    String toString() {
        return "Cat{" +
                "name='" + name + '\'' +
                ", from='" + from + '\'' +
                ", weight=" + weight +
                '}'
    }
}
```

```groovy
class CatExtFactory implements NamedDomainObjectFactory<Cat> {

    private Instantiator instantiator

    CatExtFactory(Instantiator instantiator1) {
        this.instantiator = instantiator1
    }

    @Override
    Cat create(String s) {
        return instantiator.newInstance(Cat.class, s)
    }
}
```

```groovy
class Animal {
    int count
    Dog dog
    private NamedDomainObjectContainer<Cat> catContainer

    Animal(Instantiator instantiator,
           NamedDomainObjectContainer<Cat> catContainer) {
        this.dog = instantiator.newInstance(Dog.class)
        this.catContainer = catContainer
    }

    void dog(Action<Dog> action) {
        action.execute(dog)
    }

    void catConfig(Action<? extends NamedDomainObjectContainer<Cat>> action) {
        action.execute(catContainer)
    }

    @Override
    String toString() {
        return "dog info:" + dog.toString() + "\ncat info:" + catContainer
    }
}
```

```groovy
        Instantiator instantiator = ((DefaultGradle) project.getGradle())
                .getServices().get(Instantiator.class)
        NamedDomainObjectContainer<Cat> catContainer =
                project.container(Cat.class, new CatExtFactory(instantiator))
        project.extensions.create('animal', Animal.class, instantiator, catContainer)

        project.task('showAnimalInfo'){
            doLast{
                Animal animal = project.extensions.getByName('animal')
                println(animal.toString())
            }
        }
```

