---
title: Java知识总结之——Type
date: 2021-06-02 14:13:56
tags:
---



Type是Java 编程语言中所有类型的公共高级接口（官方解释），也就是Java中所有类型的“爹”,它并不是我们平常工作中经常使用的 int、String、List、Map等数据类型，而是从Java语言角度来说，对基本类型、引用类型向上的抽象；

***Type体系中类型的包括：***
```bash
1.原始类型(Type):不仅仅包含我们平常所指的类，还包括枚举、数组、注解等
2.参数化类型(ParameterizedType):就是我们平常所用到的泛型List<String>、Map<K,V>,Set<T>,Class<?>
3.数组类型(GenericArrayType):并不是我们工作中所使用的数组String[] 、byte[]，而是带有泛型的数组，即T[]
4.类型变量(TypeVariable):比如 T a
5.基本类型(Class):原始类型，每个类(貌似接口也有)都会有个Class对象
```

***Type及相关类所在包位置：java.lang.reflect***
Type类几乎在各种框架中都能看到，尤其是涉及代理，反射的业务。理解好Type类也会对今后框架封装，源码解读有很大好处。

**1.Type**
``` bash
public interface Type {
    /**
     * Returns a string describing this type, including information
     * about any type parameters.
     *
     * @implSpec The default implementation calls {@code toString}.
     *
     * @return a string describing this type
     * @since 1.8
     */
    default String getTypeName() {
        return toString();
    }
}
```
从java1.8开始，type默认有个getTypeName()方法

**2.ParameterizedType**
```bash
public interface ParameterizedType extends Type {
    // 获取<>中实际的类型参数，以Type数组形式返回
    Type[] getActualTypeArguments();
    // 获取<>前面的类型
    Type getRawType();
    // 如果这个类型是某个类型所属，则获取这个所有者的类型，否则返回null,比如Map.Entry<Sting,String>，会返回Map
    Type getOwnerType();
}
```

**3.TypeVariable**
类型变量，即泛型中的变量；例如：T、K、V等变量，可以表示任何类；在这需要强调的是，TypeVariable代表着泛型中的变量，而ParameterizedType则代表整个泛型；
```bash
public interface TypeVariable<D extends GenericDeclaration> extends Type, AnnotatedElement {
    // 获取泛型的上限，无显示定义(extends),默认为Object
    Type[] getBounds();
    // 获取声明改类型变量实体(即获取类，方法或构造器名)
    D getGenericDeclaration();
    // 获取名称，即K、V、E之类名称
    String getName();
    // 
     AnnotatedType[] getAnnotatedBounds();
}
```
**4. GenericArrayType**
泛型数组类型，用来描述ParameterizedType、TypeVariable类型的数组；即List<T>[] 、T[]等
```bash
public interface GenericArrayType extends Type {
    // 获得这个数组元素类型，比如T[]  则获得T的type
    Type getGenericComponentType();
}
```

**5.Class**
与上三种不同，Class是Type的一个实现类，属于原始类型，是Java反射的基础，对java类的的抽象。在程序运行期间，每一个类都对应一个Class对象，这个对象包含了类的修饰符、方法、属性、构造等信息，所以我们可以对这个Class对象进行相应的操作，这就是Java的反射。

**6.WildcardType**
泛型表达式（或者通配符表达式），即? extends Number这样的表达式;WildcardType虽然是Type的子接口，但却不是Java类型中的一种：
```bash
public interface WildcardType extends Type {
    // 获取泛型表达式上界（上限extends）
    Type[] getUpperBounds();
    // 获取泛型表达式下界（下限super）
    Type[] getLowerBounds();
}
```
**7.各种Type测试**
****7.1 ParameterizedType****
```bash
public class Main {
    public static void main(String[] args) throws NoSuchFieldException {
        // 获取ParameterizedTypeTest类的list属性
        Field fieldList = ParameterizedTypeTest.class.getDeclaredField("list");
        // 获取该属性的泛型类型
        Type typeList = fieldList.getGenericType();
        System.out.println("list域类型名："+typeList.getTypeName());
        System.out.println("list域实际的Type类型："+typeList.getClass().getName());

        System.out.println(".........................................................\n");

        // 获取ParameterizedTypeTest类的list属性
        Field fieldSet = ParameterizedTypeTest.class.getDeclaredField("set");
        // 获取该属性的泛型类型
        Type typeSet = fieldSet.getGenericType();
        System.out.println("set域类型名："+typeSet.getTypeName());
        System.out.println("set域实际的Type类型："+typeSet.getClass().getName());

        System.out.println(".........................................................\n");
        // 获取ParameterizedTypeTest类的list属性
        Field fieldMap = ParameterizedTypeTest.class.getDeclaredField("map");
        // 获取该属性的泛型类型
        Type typeMap = fieldMap.getGenericType();
        System.out.println("map域类型名："+typeMap.getTypeName());
        System.out.println("map域实际的Type类型："+typeMap.getClass().getName());
        if (typeMap instanceof ParameterizedType){
            ParameterizedType mapParameterizedType = (ParameterizedType) typeMap;
            // 获取泛型中的实际参数
            Type[] types = mapParameterizedType.getActualTypeArguments();
            System.out.println("map域泛型参数类型[0]:"+types[0]);
            System.out.println("map域泛型参数类型[1]:"+types[1]);
            System.out.println("map域声明泛型参数的类类型："+mapParameterizedType.getRawType());
            System.out.println("泛型的拥有者类型："+mapParameterizedType.getOwnerType());
        }


        System.out.println(".........................................................\n");
        // 获取ParameterizedTypeTest类的list属性
        Field fieldMap2 = ParameterizedTypeTest.class.getDeclaredField("map2");
        // 获取该属性的泛型类型
        Type typeMap2 = fieldMap2.getGenericType();
        System.out.println("map2域类型名："+typeMap2.getTypeName());
        System.out.println("map2域实际的Type类型："+typeMap2.getClass().getName());
        if (typeMap2 instanceof ParameterizedType){
            ParameterizedType mapParameterizedType2 = (ParameterizedType) typeMap2;
            // 获取泛型中的实际参数
            Type[] types = mapParameterizedType2.getActualTypeArguments();
            System.out.println("map2域泛型参数类型[0]:"+types[0]);
            System.out.println("map2域泛型参数类型[1]:"+types[1]);
            System.out.println("map2域声明泛型参数的类类型："+mapParameterizedType2.getRawType());
            System.out.println("泛型的拥有者类型："+mapParameterizedType2.getOwnerType());
        }
    }



    public static class ParameterizedTypeTest<T>{
        private List<T> list = null;
        private Set<T>  set = null;
        private Map<String ,T> map = null;
        private Map.Entry<String,Integer>  map2;
    }
}
```
***测试结果：***
```bash
list域类型名：java.util.List<T>
list域实际的Type类型：sun.reflect.generics.reflectiveObjects.ParameterizedTypeImpl
.........................................................

set域类型名：java.util.Set<T>
set域实际的Type类型：sun.reflect.generics.reflectiveObjects.ParameterizedTypeImpl
.........................................................

map域类型名：java.util.Map<java.lang.String, T>
map域实际的Type类型：sun.reflect.generics.reflectiveObjects.ParameterizedTypeImpl
map域泛型参数类型[0]:class java.lang.String
map域泛型参数类型[1]:T
map域声明泛型参数的类类型：interface java.util.Map
泛型的拥有者类型：null
.........................................................

map2域类型名：java.util.Map$Entry<java.lang.String, java.lang.Integer>
map2域实际的Type类型：sun.reflect.generics.reflectiveObjects.ParameterizedTypeImpl
map2域泛型参数类型[0]:class java.lang.String
map2域泛型参数类型[1]:class java.lang.Integer
map2域声明泛型参数的类类型：interface java.util.Map$Entry
泛型的拥有者类型：interface java.util.Map
```

****7.2 GenericArrayType****
```bash
public class Main {

    public static void main(String[] args) throws NoSuchFieldException {
        Field fieldListArray = GenericArrayTypeTest.class.getDeclaredField("lists");
        Type typeListArray = fieldListArray.getGenericType();
        System.out.println("lists域Type类型："+typeListArray.getClass().getName());
        GenericArrayType genericArrayType1 = (GenericArrayType) typeListArray;
        System.out.println("lists域元素类型:"+genericArrayType1.getGenericComponentType());
        System.out.println("lists域元素Type类型:"+genericArrayType1.getGenericComponentType().getClass().getName());
        System.out.println("......................................................\n");

        Field fieldT = GenericArrayTypeTest.class.getDeclaredField("t");
        Type typeT = fieldT.getGenericType();
        System.out.println("t域Type类型："+typeT.getClass().getName());
        GenericArrayType genericArrayType2 = (GenericArrayType) typeT;
        System.out.println("t域元素类型:"+genericArrayType2.getGenericComponentType());
        System.out.println("t域元素Type类型:"+genericArrayType2.getGenericComponentType().getClass().getName());

    }



    public static class GenericArrayTypeTest<T>{
        private T[] t;
        private List<String>[] lists;
    }
}
```
***执行结果：***
```bash
lists域Type类型：sun.reflect.generics.reflectiveObjects.GenericArrayTypeImpl
lists域元素类型:java.util.List<java.lang.String>
lists域元素Type类型:sun.reflect.generics.reflectiveObjects.ParameterizedTypeImpl
......................................................

t域Type类型：sun.reflect.generics.reflectiveObjects.GenericArrayTypeImpl
t域元素类型:T
t域元素Type类型:sun.reflect.generics.reflectiveObjects.TypeVariableImpl
```

****7.3 TypeVariable****
```bash
public class Main {

    public static void main(String[] args) throws NoSuchFieldException {
        Field aField = TypeVariableTest.class.getDeclaredField("a");
        Type aFieldType = aField.getGenericType();
        Class aFieldClass = aField.getType();
        System.out.println("TypeVariableTest<T> a 域 type名："+aFieldType.getTypeName());
        System.out.println("TypeVariableTest<T> a 域 type的class名："+aFieldType.getClass().getCanonicalName());
        System.out.println("TypeVariableTest<T> a 域 Class名: "+aFieldClass.getCanonicalName());

        if (aFieldType instanceof TypeVariable){
            TypeVariable typeVariable = (TypeVariable) aFieldType;
            String name = typeVariable.getName();
            Type[] bounds = typeVariable.getBounds();
            System.out.println("a 域类型是 TypeVariable类型！");
            System.out.println("a 域 type'name: "+name);
            for (int i=0;i<bounds.length;i++){
                System.out.println("a 域 type'bounds["+i+"] Type'name: "+bounds[i].getTypeName());
            }
            GenericDeclaration genericDeclaration = typeVariable.getGenericDeclaration();
            System.out.println("声明a 域变量的实体："+genericDeclaration);
        }

        System.out.println("-----------------------------------------------------------\n");
        Field bField  = TypeVariableTest.class.getDeclaredField("b");
        Type bFieldType = bField.getGenericType();
        Class bFieldClass = bField.getType();
        System.out.println("TypeVariableTest<T> b 域 type名："+bFieldType.getTypeName());
        System.out.println("TypeVariableTest<T> b 域 type的Class名："+bFieldType.getClass().getCanonicalName());
        System.out.println("TypeVariableTest<T> b 域 Class名："+bFieldClass.getCanonicalName());

    }

    public static class TypeVariableTest<T> {
        T a;
        List<String> b;
    }

}
```

***输出结果:***
```bash
TypeVariableTest<T> a 域 type名：T
TypeVariableTest<T> a 域 type的class名：sun.reflect.generics.reflectiveObjects.TypeVariableImpl
TypeVariableTest<T> a 域 Class名: java.lang.Object
a 域类型是 TypeVariable类型！
a 域 type'name: T
a 域 type'bounds[0] Type'name: java.lang.Object
声明a 域变量的实体：class Main$TypeVariableTest
-----------------------------------------------------------

TypeVariableTest<T> b 域 type名：java.util.List<java.lang.String>
TypeVariableTest<T> b 域 type的Class名：sun.reflect.generics.reflectiveObjects.ParameterizedTypeImpl
TypeVariableTest<T> b 域 Class名：interface java.util.List
```

****7.4 Class****
Type接口的实现类，在Java中，每个.class文件在程序运行期间，都对应着一个Class对象，这个对象保存着这个类的全部信息。

就算是各种Type也是有它的Class对象。

****7.5 WildcardType****
通配符类型，例如List<? extends Number>
```bash
public class Main {

    public static void main(String[] args) throws NoSuchFieldException {
        Field aField = TypeVariableTest.class.getDeclaredField("a");
        Type aFieldType = aField.getGenericType();
        Class aFieldClass = aField.getType();
        System.out.println("TypeVariableTest a 域 type名："+aFieldType.getTypeName());
        System.out.println("TypeVariableTest a 域 type的class名："+aFieldType.getClass().getCanonicalName());
        System.out.println("TypeVariableTest a 域 Class名: "+aFieldClass.getCanonicalName());

        if (aFieldType instanceof ParameterizedType){
            ParameterizedType parameterizedType = (ParameterizedType) aFieldType;
            System.out.println("a 域Type 是 ParameterizedType ！");
            Type[] typesString = parameterizedType.getActualTypeArguments();
            System.out.println("a 域泛型参数[0]的type名："+typesString[0].getTypeName());
            System.out.println("a 域泛型参数[0]的type类型名："+typesString[0].getClass().getCanonicalName());
            
            WildcardType type0 = (WildcardType) typesString[0];
            Type[] type0UpperBounds = type0.getUpperBounds();
            Type[] type0LowerBounds = type0.getLowerBounds();

            System.out.println("泛型参数[0]的上限type:"+type0UpperBounds[0]);
        }
    }

    public static class TypeVariableTest {
        List<? extends String> a;
    }

}
```
输出结果：
```bash
TypeVariableTest a 域 type名：java.util.List<? extends java.lang.String>
TypeVariableTest a 域 type的class名：sun.reflect.generics.reflectiveObjects.ParameterizedTypeImpl
TypeVariableTest a 域 Class名: java.util.List
a 域Type 是 ParameterizedType ！
a 域泛型参数[0]的type名：? extends java.lang.String
a 域泛型参数[0]的type类型名：sun.reflect.generics.reflectiveObjects.WildcardTypeImpl
泛型参数[0]的上限type:class java.lang.String
```

**8.应用**
更快的读懂一些开源项目源码：几乎每个受欢迎的Java开源项目都会使用注解，反射，代理模式，适配器模式，而这些应用都需要判断Type,如果不搞懂几个Type，很容易一头雾水。
更好的利用Type和Java特性去设计代码


