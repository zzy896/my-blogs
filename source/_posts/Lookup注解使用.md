---
title: spring方法注入@Lookup注解使用
date: 2019-09-25 15:24:58
tags:
---

##情景分析
在Spring的诸多应用场景中bean都是单例形式，当一个单利bean需要和一个非单利bean组合使用或者一个非单利bean和另一个非单利bean组合使用时，我们通常都是将依赖以属性的方式放到bean中来引用，然后以@Autowired来标记需要注入的属性。但是这种方式在bean的生命周期不同时将会出现很明显的问题，假设单利bean A需要一个非单利bean B（原型），我们在A中注入bean B，每次调用bean A中的方法时都会用到bean B，我们知道Spring Ioc容器只在容器初始化时执行一次，也就是bean A中的依赖bean B只有一次注入的机会，但是实际上bean B我们需要的是每次调用方法时都获取一个新的对象（原型）所以问题明显就是：我们需要bean B是一个原型bean，而事实上bean B的依赖只注入了一次变成了事实上的单利bean。

###代码说明
```bash
@Component
@Scope("prototype")
public class PrototypeBean {
    private static final Logger logger= LoggerFactory.getLogger(PrototypeBean.class);
    
    public void say() {
        logger.info("say something...");
    }
}
@Component
public class SingletonBean {
    private static final Logger logger = LoggerFactory.getLogger(SingletonBean.class);
    
    @Autowired
    private PrototypeBean bean;
    
    public void print() {
        logger.info("Bean SingletonBean's HashCode : {}",bean.hashCode());
        bean.say();
    }
}
@SpringBootApplication
public class SampleApplication {
    private static final Logger logger = LoggerFactory.getLogger(SampleApplication.class);
    public static void main(String[] args) {
        SpringApplication.run(SampleApplication.class, args);
    }

    @Bean public CommandLineRunner test(final SingletonBean bean) {
        return (args)-> {
            logger.info("测试单例bean和原型bean的调用");
            int i =0;
            while(i<3) {
                i++;
                bean.print();
            }
        };
    }
}
```
##结果
```bash
2019-09-25 15:04:29,721 INFO :-- [main .. ] o.s.SampleApplication 测试单例bean和原型bean的调用 
2019-09-25 15:04:29,723 INFO :-- [main .. ] o.s.a.SingletonBean Bean SingletonBean's HashCode : 1713129148 
2019-09-25 15:04:29,723 INFO :-- [main .. ] o.s.a.PrototypeBean say something... 
2019-09-25 15:04:29,723 INFO :-- [main .. ] o.s.a.SingletonBean Bean SingletonBean's HashCode : 1713129148 
2019-09-25 15:04:29,724 INFO :-- [main .. ] o.s.a.PrototypeBean say something... 
2019-09-25 15:04:29,724 INFO :-- [main .. ] o.s.a.SingletonBean Bean SingletonBean's HashCode : 1713129148 
2019-09-25 15:04:29,724 INFO :-- [main .. ] o.s.a.PrototypeBean say something... 
```
我们看到每次输出PrototypeBean的HashCode都是一样的，证明我们实际上并没有达到使用原型bean的目的。

###解决方案
1、在bean A中引入ApplicationContext每次调用方法时用上下文的getBean(name,class)方法去重新获取bean B的实例。
2、使用@Lookup注解。
这两种解决方案都能解决我们遇到的问题，但是第二种相对而言更简单。以下给出两种解决方案的代码示例。

###通过应用上下文ApplicationContext获取获取
```bash
@Component
public class SingletonBean {
    private static final Logger logger = LoggerFactory.getLogger(SingletonBean.class);
    
    @Autowired
    private ApplicationContext context;
    
    public void print() {
        PrototypeBean bean = getFromApplicationContext();
        logger.info("Bean SingletonBean's HashCode : {}",bean.hashCode());
        bean.say();
    }
    
    /**
     * 每次都从ApplicatonContext中获取新的bean引用
     * @return PrototypeBean instance
     */
    PrototypeBean getFromApplicationContext() {
        return this.context.getBean("prototypeBean",PrototypeBean.class);
    }
}
```
###结果
```bash
2019-09-25 15:10:01,485 INFO :-- [main .. ] o.s.SampleApplication 测试单例bean和原型bean的调用 
2019-09-25 15:10:01,487 INFO :-- [main .. ] o.s.a.SingletonBean Bean SingletonBean's HashCode : 376601041 
2019-09-25 15:10:01,487 INFO :-- [main .. ] o.s.a.PrototypeBean say something... 
2019-09-25 15:10:01,487 INFO :-- [main .. ] o.s.a.SingletonBean Bean SingletonBean's HashCode : 2056499811 
2019-09-25 15:10:01,487 INFO :-- [main .. ] o.s.a.PrototypeBean say something... 
2019-09-25 15:10:01,488 INFO :-- [main .. ] o.s.a.SingletonBean Bean SingletonBean's HashCode : 890733699 
2019-09-25 15:10:01,488 INFO :-- [main .. ] o.s.a.PrototypeBean say something... 
```
我们看到每次我们调用print()方法时都会重新从应用上下文获取新的引用，达到了使用原型的目的。

###通过@Lookup注解实现方法注入
使用方法注入的方法需要满足以下语法要求

```bash
<public|protected> [abstract] <return-type> theMethodName(no-arguments);
```

```bash
@Component
public abstract class SingletonBean {
    private static final Logger logger = LoggerFactory.getLogger(SingletonBean.class);
    
    public void print() {
        PrototypeBean bean = methodInject();
        logger.info("Bean SingletonBean's HashCode : {}",bean.hashCode());
        bean.say();
    }
    // 也可以写成 @Lookup("prototypeBean") 来指定需要注入的bean
    @Lookup
    protected abstract PrototypeBean methodInject();
}
```

结果

```bash
2019-09-25 15:18:50,105 INFO :-- [main .. ] o.s.SampleApplication 测试单例bean和原型bean的调用 
2019-09-25 15:18:50,108 INFO :-- [main .. ] o.s.a.SingletonBean Bean SingletonBean's HashCode : 1349373781 
2019-09-25 15:18:50,108 INFO :-- [main .. ] o.s.a.PrototypeBean say something... 
2019-09-25 15:18:50,108 INFO :-- [main .. ] o.s.a.SingletonBean Bean SingletonBean's HashCode : 1046820071 
2019-09-25 15:18:50,109 INFO :-- [main .. ] o.s.a.PrototypeBean say something... 
2019-09-25 15:18:50,109 INFO :-- [main .. ] o.s.a.SingletonBean Bean SingletonBean's HashCode : 1722645488 
2019-09-25 15:18:50,110 INFO :-- [main .. ] o.s.a.PrototypeBean say something... 
```