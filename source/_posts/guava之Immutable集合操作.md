---
title: guava之Immutable集合操作
date: 2020-01-01 09:58:29
tags:
---
***guava是google的一个库，弥补了java语言的很多方面的不足，很多在java8中已有实现，暂时不展开。Collections是jdk提供的一个工具类。***
**1、JDK中实现immutable集合**
在JDK中提供了Collections.unmodifiableXXX系列方法来实现不可变集合, 但是存在一些问题，下面我们先看一个具体实例：
``` bash
public class ImmutableTest {

    @Test
    public void testJDKImmutable(){
        List<String> list=new ArrayList<String>();
        list.add("a");
        list.add("b");
        list.add("c");

        //通过list创建一个不可变的unmodifiableList集合
        List<String> unmodifiableList=Collections.unmodifiableList(list);
        System.out.println(unmodifiableList);

        //通过list添加元素
        list.add("ddd");
        System.out.println("往list添加一个元素:"+list);
        System.out.println("通过list添加元素之后的unmodifiableList:"+unmodifiableList);

        //通过unmodifiableList添加元素
        unmodifiableList.add("eee");
        System.out.println("往unmodifiableList添加一个元素:"+unmodifiableList);

    }
}
```
运行结果：
``` bash
[a,b,c]
往list添加一个元素:[a,b,c,ddd]
通过list添加元素之后的unmodifiableList:[a,b,c,ddd]

java.lang.UnsupportedOperationException 
      at java.util.Collections$UnmodifiableCollection.add(Collections.java:1055)
```
通过运行结果我们可以看出：虽然unmodifiableList不可以直接添加元素，但是我的list是可以添加元素的，而list的改变也会使unmodifiableList改变。
所以说Collections.unmodifiableList实现的不是真正的不可变集合。

**2、Guava的immutable集合**
Guava提供了对JDK里标准集合类里的immutable版本的简单方便的实现，以及Guava自己的一些专门集合类的immutable实现。当你不希望修改一个集合类，
或者想做一个常量集合类的时候，使用immutable集合类就是一个最佳的编程实践。

注意：每个Guava immutable集合类的实现都拒绝null值。我们做过对Google内部代码的全面的调查，并且发现只有5%的情况下集合类允许null值，而95%的情况下
都拒绝null值。万一你真的需要能接受null值的集合类，你可以考虑用Collections.unmodifiableXXX。
immutable集合可以有以下几种方式来创建：
　　1、用copyOf方法, 譬如, ImmutableSet.copyOf(set)
　　2、使用of方法，譬如，ImmutableSet.of(“a”, “b”, “c”)或者ImmutableMap.of(“a”, 1, “b”, 2)
　　3、使用Builder类
举例：

``` bash
@Test
   public void testGuavaImmutable(){

       List<String> list=new ArrayList<String>();
       list.add("a");
       list.add("b");
       list.add("c");

       ImmutableList<String> imlist=ImmutableList.copyOf(list);
       System.out.println("imlist："+imlist);

       ImmutableList<String> imOflist=ImmutableList.of("peida","jerry","harry");
       System.out.println("imOflist："+imOflist);

       ImmutableSortedSet<String> imSortList=ImmutableSortedSet.of("a", "b", "c", "a", "d", "b");
       System.out.println("imSortList："+imSortList);

       list.add("baby");
       //关键看这里是否imlist也添加新元素了
       System.out.println("list添加新元素之后看imlist:"+imlist);

       ImmutableSet<Color> imColorSet =
               ImmutableSet.<Color>builder()
                       .add(new Color(0, 255, 255))
                       .add(new Color(0, 191, 255))
                       .build();

       System.out.println("imColorSet:"+imColorSet);
   }
```
运行结果：发现imlist并未改变。
``` bash
mlist:[a,b,c]
imOflist:[peida,jerry,harry]
imSortList:[a,b,c,d]
list添加新元素之后看imlist:[a,b,c]
imColorSet:[java.awt.Color[r=0,g=255,b=255],java.awt.Color[r=0,g=191,b=255]]

```
对于排序的集合来说有例外，因为元素的顺序在构建集合的时候就被固定下来了。譬如，ImmutableSet.of(“a”, “b”, “c”, “a”, “d”, “b”)，对于这个集合的遍历顺序来说就是”a”, “b”, “c”, “d”。
更智能的copyOf
copyOf方法比你想象的要智能，ImmutableXXX.copyOf会在合适的情况下避免拷贝元素的操作－先忽略具体的细节，但是它的实现一般都是很“智能”的。譬如：

```bash
@Test
        public void testCotyOf(){
            ImmutableSet<String> imSet=ImmutableSet.of("peida","jerry","harry","lisa");
            System.out.println("imSet："+imSet);

            //set直接转list
            ImmutableList<String> imlist=ImmutableList.copyOf(imSet);
            System.out.println("imlist："+imlist);

            //list直接转SortedSet
            ImmutableSortedSet<String> imSortSet=ImmutableSortedSet.copyOf(imSet);
            System.out.println("imSortSet："+imSortSet);

            List<String> list=new ArrayList<String>();
            for(int i=0;i<=10;i++){
                list.add(i+"x");
            }
            System.out.println("list："+list);

            //截取集合部分元素
            ImmutableList<String> imInfolist=ImmutableList.copyOf(list.subList(2, 8));
            System.out.println("imInfolist："+imInfolist);
```
运行结果

```bash
imSet:[peida,jerry,harry,lisa]
imlist:[peida,jerry,harry,lisa]
imSortSet:[harry,jerry,lisa,peida]
list:[0x,1x,2x,3x,4x,5x,6x,7x,8x,9x,10x]
imInfoList:[2x,3x,4x,5x,6x,7x]
```
Guava集合和不可变对应关系
```bash
可变集合类型               可变集合源：JDK or Guava?         Guava不可变集合
Collection                       JDK                         ImmutableCollection
List                             JDK                         ImmutableList
Set                              JDK                         ImmutableSet
SortedSet/NavigableSet           JDK                         ImmutableSortedSet
Map                              JDK                         ImmutableMap
SortedMap                        JDK                         ImmutableSortedMap
Multiset                         Guava                       ImmutableMultiset
SortedMultiset                   Guava                       ImmutableSortedMultiset
Multimap                         Guava                       ImmutableMultimap
ListMultimap                     Guava                       ImmutableListMultimap
SetMultimap                      Guava                       ImmutableSetMultimap
BiMap                            Guava                       ImmutableBiMap
ClassToInstanceMap               Guava                       ImmutableClassToInstanceMap
Table                            Guava                       ImmutableTable
```
