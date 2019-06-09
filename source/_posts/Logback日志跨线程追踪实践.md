---
title: Logback日志跨线程追踪实践
date: 2019-06-09 20:23:44
tags:
---
###1. 自定义日志模板参数：Logback的Pattern模板
当一个请求过来,我们想要知道当前请求具体跑了那些流程该怎么做呢？ 噔噔噔噔..我们的男主Logback自定义Pattern
模板即将登场.

在我们打印日志的时候,通常我们都会把一些重要的参数信息写到日志里面,方便我们后期从日志里面定位问题,其他的
内部系统调用我们的程序的时候，我们可以要求他们那边在header头里面增加trace-id这样的唯一标识，如果没有该参
数的话，我们自己手动生成一个唯一的标识，方便我们将整条请求链路构建起来.
``` bash
<property name="CONSOLE_LOG_PATTERN"
 value="[%yellow(%date{yyyy-MM-dd HH:mm:ss})] [%highlight(%-5level)] [%cyan(%traceId)] 
 [%magenta(%thread)] [%blue(%file:%line)] [%green(%logger)] : %.4000m%n"/>
```
上面这里是Logback的定义变量,重点关注[%cyan(%traceId)]这个参数(ps:其他的日志系统如log4j2都有类似的实现)  
要实现让Logback识别我们自定义的标识符,我们需要继承两个方法ClassicConverter跟PatternLayout,具体实现如下:
``` bash
public class PatternLogbackLayout extends PatternLayout {
  static {
    defaultConverterMap.put("traceId", TraceIdPatternConverter.class.getName());
  }
}

public class TraceIdPatternConverter extends ClassicConverter {
	@Override
	public String convert(ILoggingEvent iLoggingEvent) {
		String traceId = LogHandlerInterceptor.getTrack();
		return StringUtils.isEmpty(traceId) ? "traceId" : traceId;
	}
}
```
通过spring的拦截器我们将请求的header头里面的参数赋值给traceId,然后配置logback.xml，让logback去识别我们定义的traceId
``` bash
<appender name="stdout" class="ch.qos.logback.core.ConsoleAppender">
   <encoder class="ch.qos.logback.core.encoder.LayoutWrappingEncoder">
      <layout class="com.blacksearch.logback.PatternLogbackLayout">
      	<pattern>${CONSOLE_LOG_PATTERN}</pattern>
      </layout>
   </encoder>
</appender>
```
通过以上配置,我们已经可以在日志里面看到traceId了
``` bash
[2019-05-10 16:47:38] [INFO ] [538e2c0d-3800-4c86-b320-260bdd945c0c] [http-nio-8080-exec-6] [TestLogTrackController.java:15] [com.blacksearch.controller.TestLogTrackController] : 测试
[2019-05-10 16:47:38] [DEBUG] [538e2c0d-3800-4c86-b320-260bdd945c0c] [http-nio-8080-exec-6] [AbstractMessageConverterMethodProcessor.java:298] [org.springframework.web.servlet.mvc.method.annotation.RequestResponseBodyMethodProcessor] : Nothing to write: null body
```
这样子在服务器上面我们可以通过
``` bash
grep '538e2c0d-3800-4c86-b320-260bdd945c0c'
```
查看到当前请求的所有日志~或者上ELK之后 直接搜索538e2c0d-3800-4c86-b320-260bdd945c0c 方便开心~

###2. 当我们使用多线程的时候,我们发现,原先定义的标识居然消失了！！！
解决方案：自定义抽象类实现Runnable 首先我们先来复现问题. eg:

``` bash
@RequestMapping("/asynLogTrack")
public String asynLogTrack(){
	logger.info("ces--------");
	new Thread(new Runnable() {
		@Override
		public void run() {
			logger.info("ces");
		}
	}).start();
	return null;
}
[2019-05-10 16:55:18] [INFO ] [5b446567-97be-42f7-b6b0-205a6b431e87] [http-nio-8080-exec-2] [TestLogTrackController.java:21] [com.blacksearch.controller.TestLogTrackController] : ces--------
[2019-05-10 16:55:18] [INFO ] [traceId] [Thread-10] [TestLogTrackController.java:25] [com.blacksearch.controller.TestLogTrackController] : ces
[2019-05-10 16:55:18] [DEBUG] [5b446567-97be-42f7-b6b0-205a6b431e87] [http-nio-8080-exec-2] [AbstractMessageConverterMethodProcessor.java:298] [org.springframework.web.servlet.mvc.method.annotation.RequestResponseBodyMethodProcessor] : Nothing to write: null body
```
我们看到打印 ces 的打印记录,我们发现，traceId居然丢失了!!! 原因在于,栈是线程私有,当我们新建线程的时候,新建的
线程跟原来的线程互相独立,也就是说logback无法在新建的线程上获取到值。既然如何,那么我们该如何让Logback在
新线程上获取到值呢？正如标题而言。

``` bash
public abstract class TrackRunnable implements Runnable {

	public abstract void trackRun();

	private String track = LogHandlerInterceptor.getTrack();

	@Override
	public void run() {
		try {
			LogHandlerInterceptor.setTrace(track);
			trackRun();
		}finally {
			LogHandlerInterceptor.remove();
		}
	}
}
```
我们采用抽象类实现Runnable,然后在run()方法里面通过ThreadLocal去重新赋值。eg:
``` bash
@RequestMapping("/asynLogTrackHasTrace")
public String asynLogTrackHasTrace(){
	logger.info("ces1------");
	new Thread(new TrackRunnable() {
		@Override
		public void trackRun() {
			try {
				Thread.sleep(4000);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
			logger.info("ces2-----------");
		}
	}).start();
	return null;
}
[2019-05-10 17:07:20] [INFO ] [0a7f30d3-8e1d-49b4-9bca-4b72fd1c0ccd] [http-nio-8080-exec-5] [TestLogTrackController.java:33] [com.blacksearch.controller.TestLogTrackController] : ces1------
[2019-05-10 17:07:20] [DEBUG] [0a7f30d3-8e1d-49b4-9bca-4b72fd1c0ccd] [http-nio-8080-exec-5] [AbstractMessageConverterMethodProcessor.java:298] [org.springframework.web.servlet.mvc.method.annotation.RequestResponseBodyMethodProcessor] : Nothing to write: null body
[2019-05-10 17:07:20] [DEBUG] [traceId] [http-nio-8080-exec-5] [FrameworkServlet.java:1130] [org.springframework.web.servlet.DispatcherServlet] : Completed 200 OK
[2019-05-10 17:07:24] [INFO ] [0a7f30d3-8e1d-49b4-9bca-4b72fd1c0ccd] [Thread-11] [TestLogTrackController.java:42] [com.blacksearch.controller.TestLogTrackController] : ces2-----------

```



