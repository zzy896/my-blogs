---
title: 简述Spring的ApplicationEvent使用
date: 2020-02-14 13:46:33
tags:
---
spring 事件为bean 与 bean之间传递消息。一个bean处理完了希望其余一个接着处理.这时我们就需要其余的一个bean监听当前bean所发送的事件.
spring事件使用步骤如下:

1.先自定义事件：你的事件需要继承 ApplicationEvent

2.定义事件监听器: 需要实现 ApplicationListener

3.使用容器对事件进行发布

首先定义一个事件
```bash
public class TestEvent extends ApplicationEvent {

    private String name;

    private String msg;

    public TestEvent(Object source){
        super(source);
    }

    public TestEvent(Object source, String name, String msg) {
        super(source);
        this.name = name;
        this.msg = msg;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getMsg() {
        return msg;
    }

    public void setMsg(String msg) {
        this.msg = msg;
    }
}
```

其次定义事件监听
```bash
@Component
public class TestEventListener implements ApplicationListener<TestEvent> {

    @Async
    @Override
    public void onApplicationEvent(TestEvent testEvent) {
        System.out.println("姓名："+testEvent.getName()+"得到消息："+testEvent.getMsg());
    }
}
```

使用容器发布事件
```bash
@Component
public class TestEventPublisher {

    @Autowired
    private ApplicationContext applicationContext;

    public void pushlish(String name, String msg){
        applicationContext.publishEvent(new TestEvent(this, name,msg));
    }
}
```

测试事件是否能够生效
```bash
@Configuration
@ComponentScan("cn.*.event")
public class EventConfig {
}


public class TestMain {

    private static AnnotationConfigApplicationContext context;

    public static void main(String[] args) {
        start();
    }

    private static void start() {
        if (context == null) {
             context=new AnnotationConfigApplicationContext(EventConfig.class);
        }
        context.getBean(TestEventPublisher.class).pushlish("hangyu","申请退款！");
    }
}
```