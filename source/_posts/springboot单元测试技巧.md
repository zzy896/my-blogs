---
title: springboot单元测试技巧
date: 2020-02-21 20:08:33
tags:
---
**直奔主题：**

**引入依赖**
```bash
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```
标准的Spring Boot测试单元代码结构：
```bash
import org.junit.runner.RunWith;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

@RunWith(SpringRunner.class)
@SpringBootTest
public class ApplicationTest {
	
}
```
**JUnit4注解**
@BeforeClass、@AfterClass：在每个类加载的开始和结束时运行，必须为静态方法，执行一次。
@Before、@After：在每个测试方法开始之前和结束之后运行，例如Test1()和Test2()执行前后各执行一次。
```bash
@RunWith(SpringRunner.class)
@SpringBootTest
public class TestApplicationTests {

    @BeforeClass
    public static void beforeClassTest() {
        System.out.println("before class test");
    }
    
    @Before
    public void beforeTest() {
        System.out.println("before test");
    }
    
    @Test
    public void Test1() {
        System.out.println("test 1+1=2");
        Assert.assertEquals(2, 1 + 1);
    }
    
    @Test
    public void Test2() {
        System.out.println("test 2+2=4");
        Assert.assertEquals(4, 2 + 2);
    }
    
    @After
    public void afterTest() {
        System.out.println("after test");
    }
    
    @AfterClass
    public static void afterClassTest() {
        System.out.println("after class test");
    }
}
```
运行结果：
```bash
before class test
before test
test 1+1=2
after test
before test
test 2+2=4
after test
after class test
```

**org.junit.Assert常用方法**
```bash
assertEquals("message",A,B)，判断A对象和B对象是否相等，这个判断在比较两个对象时调用了equals()方法。
assertSame("message",A,B)，判断A对象与B对象是否相同，使用的是==操作符。
assertTrue("message",A)，判断A条件是否为真。
assertFalse("message",A)，判断A条件是否不为真。
assertNotNull("message",A)，判断A对象是否不为null。
assertArrayEquals("message",A,B)，判断A数组与B数组是否相等。
```

**MockMvc**
在单元测试中，使用MockMvc前需要进行初始化，如下所示：
```bash
private MockMvc mockMvc;

@Autowired
private WebApplicationContext wac;

@Before
public void setupMockMvc(){
    mockMvc = MockMvcBuilders.webAppContextSetup(wac).build();
}
```
**MockMvc模拟MVC请求**
模拟一个get请求：
```bash
mockMvc.perform(MockMvcRequestBuilders.get("/hello?name={name}","hello"));
```

模拟一个post请求：
```bash
mockMvc.perform(MockMvcRequestBuilders.post("/user/{id}", 1));
```

模拟文件上传：
```bash
mockMvc.perform(MockMvcRequestBuilders.fileUpload("/fileupload").file("file", "文件内容".getBytes("utf-8")));
```

模拟请求参数：
// 模拟发送一个message参数，值为hello
```bash
mockMvc.perform(MockMvcRequestBuilders.get("/hello").param("message", "hello"));
```

使用MultiValueMap构建参数：
```bash
MultiValueMap<String, String> params = new LinkedMultiValueMap<String, String>();
params.add("name", "Tom");
params.add("moblie", "123");
mockMvc.perform(MockMvcRequestBuilders.get("/hobby/save").params(params));
```

模拟发送JSON参数：
```bash
String jsonStr = "{\"username\":\"Dopa\",\"passwd\":\"ac3af72d9f95161a502fd326865c2f15\",\"status\":\"1\"}";
mockMvc.perform(MockMvcRequestBuilders.post("/user/save").content(jsonStr.getBytes()));
```

实际测试中，要手动编写这么长的JSON格式字符串很繁琐也很容易出错，可以借助Spring Boot自带的Jackson技术来序列化一个Java对象,如下所示：
```bash
User user = new User();
user.setUsername("Dopa");
user.setPasswd("ac3af72d9f95161a502fd326865c2f15");
user.setStatus("1");

String userJson = mapper.writeValueAsString(user);
mockMvc.perform(MockMvcRequestBuilders.post("/user/save").content(userJson.getBytes()));
```
其中，mapper为com.fasterxml.jackson.databind.ObjectMapper对象。

模拟Session和Cookie：
```bash
mockMvc.perform(MockMvcRequestBuilders.get("/index").sessionAttr(name, value));
mockMvc.perform(MockMvcRequestBuilders.get("/index").cookie(new Cookie(name, value)));
```

设置请求的Content-Type：
```bash
mockMvc.perform(MockMvcRequestBuilders.get("/index").contentType(MediaType.APPLICATION_JSON_UTF8));
```
设置返回格式为JSON：
```bash
mockMvc.perform(MockMvcRequestBuilders.get("/user/{id}", 1).accept(MediaType.APPLICATION_JSON));
```
模拟HTTP请求头：
```bash
mockMvc.perform(MockMvcRequestBuilders.get("/user/{id}", 1).header(name, values));
```
MockMvc处理返回结果
期望成功调用，即HTTP Status为200：
```bash
mockMvc.perform(MockMvcRequestBuilders.get("/user/{id}", 1))
    .andExpect(MockMvcResultMatchers.status().isOk());
```    
期望返回内容是application/json：
```bash
mockMvc.perform(MockMvcRequestBuilders.get("/user/{id}", 1))
    .andExpect(MockMvcResultMatchers.content().contentType(MediaType.APPLICATION_JSON));
```    
检查返回JSON数据中某个值的内容：
```bash
mockMvc.perform(MockMvcRequestBuilders.get("/user/{id}", 1))
    .andExpect(MockMvcResultMatchers.jsonPath("$.username").value("Tom"));
```    
这里使用到了jsonPath，$代表了JSON的根节点。更多关于jsonPath的介绍可参考 https://github.com/json-path/JsonPath。
判断Controller方法是否返回某视图：
```bash            
mockMvc.perform(MockMvcRequestBuilders.post("/index"))
    .andExpect(MockMvcResultMatchers.view().name("index.html"));
```
比较Model：
```bash
mockMvc.perform(MockMvcRequestBuilders.get("/user/{id}", 1))
    .andExpect(MockMvcResultMatchers.model().size(1))
    .andExpect(MockMvcResultMatchers.model().attributeExists("password"))
    .andExpect(MockMvcResultMatchers.model().attribute("username", "Tom"));
```    
比较forward或者redirect：
```bash
mockMvc.perform(MockMvcRequestBuilders.get("/index"))
    .andExpect(MockMvcResultMatchers.forwardedUrl("index.html"));
```    
// 或者
```bash
mockMvc.perform(MockMvcRequestBuilders.get("/index"))
    .andExpect(MockMvcResultMatchers.redirectedUrl("index.html"));
```

比较返回内容，使用content()：
// 返回内容为hello
```bash
mockMvc.perform(MockMvcRequestBuilders.get("/index"))
    .andExpect(MockMvcResultMatchers.content().string("hello"));
```
// 返回内容是XML，并且与xmlCotent一样
```bash
mockMvc.perform(MockMvcRequestBuilders.get("/index"))
    .andExpect(MockMvcResultMatchers.content().xml(xmlContent));
```
// 返回内容是JSON ，并且与jsonContent一样
```bash
mockMvc.perform(MockMvcRequestBuilders.get("/index"))
    .andExpect(MockMvcResultMatchers.content().json(jsonContent));
```    
输出响应结果：
```bash
mockMvc.perform(MockMvcRequestBuilders.get("/index"))
    .andDo(MockMvcResultHandlers.print());
```    
####测试Service
Service代码：
```bash
@Repository("userService")
public class UserServiceImpl extends BaseService<User> implements UserService {
   ...
   ...
}
```
在测试方法上加上@Transactional注，数据能自动回滚
```bash
@RunWith(SpringRunner.class)
@SpringBootTest
public class UserServiceTest {

    @Autowired
    UserService userService;

    @Test
    @Transactional
    public void test() {
        User user = new User();
        user.setUsername("Junit");
        user.setPasswd("123456");
        user.setStatus("1");
        user.setCreateTime(new Date());
        this.userService.save(user);
    }
}
```
测试通过，查看数据库发现数据并没有被插入，这样很好的避免了不必要的数据污染。   

####测试Controller
```bash
@RestController
public class UserController {
    @Autowired
    UserService userService;

    @GetMapping("user/{userName}")
    public User getUserByName(@PathVariable(value = "userName") String userName) {
        return this.userService.findByName(userName);
    }

    @PostMapping("user/save")
    public void saveUser(@RequestBody User user) {
        this.userService.saveUser(user);
    }
}
```
getUserByName(@PathVariable(value = "userName") String userName)方法的测试类
```bash
@RunWith(SpringRunner.class)
@SpringBootTest
public class UserControllerTest {

    private MockMvc mockMvc;
    
    @Autowired
    private WebApplicationContext wac;
    
    @Before
    public void setupMockMvc(){
        mockMvc = MockMvcBuilders.webAppContextSetup(wac).build();
    }
    
    @Test
    public void test() throws Exception {
        mockMvc.perform(
            MockMvcRequestBuilders.get("/user/{userName}", "scott")
            .contentType(MediaType.APPLICATION_JSON_UTF8))
        .andExpect(MockMvcResultMatchers.status().isOk())
        .andExpect(MockMvcResultMatchers.jsonPath("$.username").value("scott"))
        .andDo(MockMvcResultHandlers.print());
    }
}
```
saveUser(@RequestBody User user)方法的测试类：
```bash
@RunWith(SpringRunner.class)
@SpringBootTest
public class UserControllerTest {

    private MockMvc mockMvc;
    
    @Autowired
    private WebApplicationContext wac;
    
    @Autowired
    ObjectMapper mapper;
    
    
    @Before
    public void setupMockMvc(){
        mockMvc = MockMvcBuilders.webAppContextSetup(wac).build();
    }
	
    @Test
    @Transactional
    public void test() throws Exception {
        User user = new User();
        user.setUsername("Dopa");
        user.setPasswd("ac3af72d9f95161a502fd326865c2f15");
        user.setStatus("1");
        
        String userJson = mapper.writeValueAsString(user);
        mockMvc.perform(
            MockMvcRequestBuilders.post("/user/save")
            .contentType(MediaType.APPLICATION_JSON_UTF8)
            .content(userJson.getBytes()))
        .andExpect(MockMvcResultMatchers.status().isOk())
        .andDo(MockMvcResultHandlers.print());
    }
}
```

单元测试时模拟一个登录用户信息Session，MockMvc也提供了解决方案，可在初始化的时候模拟一个HttpSession：
```bash
private MockMvc mockMvc;
private MockHttpSession session;

@Autowired
private WebApplicationContext wac;

@Before
public void setupMockMvc(){
    mockMvc = MockMvcBuilders.webAppContextSetup(wac).build();
    session = new MockHttpSession();
    User user =new User();
    user.setUsername("Dopa");
    user.setPasswd("ac3af72d9f95161a502fd326865c2f15");
    session.setAttribute("user", user); 
}
```       