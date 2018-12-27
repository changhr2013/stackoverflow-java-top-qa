## How do servlets work? Instantiation, shared variables and multithreading

### 问题：

假设，我有一个 web 服务器可以支持无数的 servlets，对于通过这些 servlets 的信息，我正在获取这些 servlets 的上下文环境，并设置 session 变量。
现在，如果有两个或者更多的 user 用户发送请求到这个服务器，session 变量会发生什么变化？session 对于所有的 user 是公共的还是不同的 user 拥有不同的 session。如果用户彼此之间的 session 是不同的，那么服务器怎么区分辨别不同的用户呢？
另外一些相似的问题，如果有 N 个用户访问一个具体的 servlets，那么这个 servlets 是只在第一个用户第一次访问的时候实例化，还是为每一个用户各自实例化呢？

### 答案：

#### ServletContext

当 servletcontainer（像 tomcat）启动的时候，它会部署和加载所有的 webapplications，当一个 webapplication 加载完成后，servletcontainer 就会创建一个 ServletContext，并且保存在服务器的内存中。这个 webapp 的 web.xml 会被解析，web.xml 中的每个 `<servlet>, <filter> and <listener>` 或者通过注解 `@WebServlet, @WebFilter and @WebListener`，都会被创建一次并且也保存在服务器的内存中。对于所有filter，`init()` 方法会被直接触发，当 servletcontainer 关闭的时候，它会 unload 所有的 webapplications,触发所有实例化的 servlets 和 filters 的 `destroy()` 方法,最后，servletcontext 和所有的 servlets，filter 和 listener 实例都会被销毁。

#### HttpServletRequest and HttpServletResponse

servletcontainer 是附属于 webserver 的，而这个 webserver 会持续监听一个目标端口的 `HTTP request` 请求，这个端口在开发中经常会被设置成 8080，而在生产环境会被设置成 80。当一个客户端（比如用户的浏览器）发送一个 HTTP request，servletcontainer 就会创建新的 HttpServletRequest 对象和 HttpServletResponse 对象。

在有filter的情况下，`doFilter()` 方法会被触发。当代码调用 `chain.doFilter(request, response)` 时候，请求会经过下一个过滤器 filter，如果没有了过滤器，会到达 servlet。在 servlets 的情况下，`service()` 触发，然后根据 `request.getMethod()` 确定执行 doGet() 还是 `doPost()`，如果当前 servlet 找不到请求的方法，返回 405 error。

request 对象提供了 HTTP 请求所有的信息，比如 request headers 和 request body，response 对象提供了控制和发送 HTTP 响应的的能力，并且以你想要的方式，比如设置 headers 和 body。当HTTP响应结束，请求和响应对象会被销毁（实际上，大多数 container 将会清洗到这些对象的状态然后回收这些事例以重新利用）。

#### httpSession

当客户端第一次访问 webapp 或者通过```request.getSession()```方法第一次获取httpSession
，servletcontainer 将会创建一个新的 HttpSession 对象，产生一个长的唯一的 ID 标记 session（可以通过session.getId()）,并且将这个 session 存储在server内存中。servletcontainer 同时会在HTTP response 的 Header中设置 `Set-Cookie` cookie 值，其中 cookie name 为 JSESSIONID，cookie value 为唯一的长 ID 值。

在接下来的连续请求中，客户端浏览器都要 cookie 通过 header 带回，然后 servletcontainer 会根据 cookie 中的 JSESSIONID 值，获得 server 内存中的对应的 httpSession。

只要没超过`<session-timeout>`设定的值，httpSession 对象会一直存在，`<session-timeout>`大小可以在 web.xml 中设定，默认是30分钟。所以如果连续30分钟之内客户端不再访问 webapp，servletcontainer 就会销毁对应的 session。接下来的 request 请求即使 cookies 依旧存在，但是却不再有对应的 session 了。servletcontainer 会创建新的 session。

另外一方面，session cookie在浏览器端有默认的生命时长，就是只要浏览器一直在运行，所以当浏览器关闭，浏览器端的cookie会被销毁。
#### 最后

- 只要webapp存在，ServletContext 一定会存在。并且ServletContext 是被所有session和request共享的。

- 只要客户端用同一个浏览器和webapp交互并且该session没有在服务端超时，HttpSession 就会一直存在。并且在同一个会话中所有请求都是共享的。

- 只有当完整的response响应到达，HttpServletRequest 和 HttpServletResponse才不再存活，并且不被共享。

- 只要webapp存在，servlet、filter和listener就会存在。他们被所有请求和会话共享。

- 只要问题中的对象存在，任何设置在ServletContext, HttpServletRequest 和 HttpSession中的属性就会存在。

#### 线程安全

就是说，你主要关注的是线程安全性。你应该了解到，servlets和filter 是被所有请求共享的。这正是Java的美妙之处，它的多线程和不同的线程可以充分利用同样的实例instance，否则对于每一个 request 请求都要重复创建和调用 init() 和 destroy() 开销太大。

但是你也应该注意到，你不应该把任何请求或会话作用域的数据作为一个servlet或过滤器的实例变量。这样会被其他会话的请求共享，并且那是线程不安全的！下面的例子阐明的这点：

```java
public class ExampleServlet extends HttpServlet {

    private Object thisIsNOTThreadSafe;

    protected void doGet(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        Object thisIsThreadSafe;

        thisIsNOTThreadSafe = request.getParameter("foo"); // BAD!! Shared among all requests!
        thisIsThreadSafe = request.getParameter("foo"); // OK, this is thread safe.
    }
}
```

> 原文链接：[http://stackoverflow.com/questions/3106452/how-do-servlets-work-instantiation-shared-variables-and-multithreading](http://stackoverflow.com/questions/3106452/how-do-servlets-work-instantiation-shared-variables-and-multithreading)