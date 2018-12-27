## 如何避免在JSP文件中使用Java代码

### 问题

如何避免在 JSP 文件中使用 Java 代码？

我对Java EE不是很熟悉，我知道类似如下的三行代码

```jsp
<%= x+1 %>
<%= request.getParameter("name") %>
<%! counter++; %>
```

这三行代码是学校教的老式代码。在 JSP 2，存在一些方法可以避免在 JSP 文件中使用 Java 代码。有人可以告诉我在 JSP 2 中如何避免使用 Java 代码吗，这些方法该如何使用？

### 回答

在大约十年前，taglibs（比如 JSTL）和 EL（EL表达式，`${}`）诞生的时候，在 JSP 中使用 scriptlets（类似`<% %>`）这种做法，就确实已经是不被鼓励使用的做法了。

scriptlets 主要的缺点有：

1. **重用性** ：你不可以重用 scriptlets

2. **可替换性** ：你不可以让 scriptlets 抽象化

3. **面向对象能力** ：你不可以使用继承或组合

4. **调试性** ：如果 scriptlets 中途抛出了异常，你只能获得一个空白页

5. **可测试性** ：scriptlets 不能进行单元测试

6. **可维护性** ：（这句有些词语不确定）需要更多的时间去维护 混合的/杂乱的/冲突的 代码逻辑

Oracle 自己也在 [JSP coding conventions](http://www.oracle.com/technetwork/articles/javase/code-convention-138726.html) 一文中推荐在功能可以被标签库所替代的时候避免使用 scriptlets 语法。以下引用它提出的几个观点：

> 在JSP 1.2规范中，强烈推荐使用 JSTL 来减少 JSP scriptlets 语法的使用。一个使用 JSTL 的页面，总得来说会更加地容易阅读和维护。

> ...

> 在任何可能的地方，当标签库能够提供相同的功能时，尽量避免使用JSP scriptlets语法。这会让页面更加容易阅读和维护，帮助将 业务逻辑 从 表现层逻辑 中分离，也会让页面往更符合JSP 2.0风格的方向发展（JSP 2.0规范中，支持但是极大弱化了JSP scriptlets语法）

> ...

> 本着适应 模型-显示层-控制器（MVC） 设计模式中关于减少业务逻辑层与显示层之间的耦合的精神，**JSP scriptlets语法不应该**被用来编写业务逻辑。相应的，JSP scriptlets语法在传送一些服务端返回的处理客户端请求的数据（也称为value objects）的时候会被使用。尽管如此，使用一个 controller servlet 来处理或者用自定义标签来处理会更好。

-----------

**如何替换 scriptlets 语句，取决于代码/逻辑的目的。更常见的是，被替换的语句会被放在另外的一些更值得放的Java类里**（这里翻译得不一定清楚）

* 如果你想在每个请求、每个页面请求都运行**相同的** Java 代码，，比如说 检查一个用户是否在登录状态，就要实现一个 过滤器，在 doFilter() 方法中编写正确的代码，例如：

```java
public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws ServletException, IOException {
    if (((HttpServletRequest) request).getSession().getAttribute("user") == null) {
        ((HttpServletResponse) response).sendRedirect("login"); // Not logged in, redirect to login page.
    } else {
        chain.doFilter(request, response); // Logged in, just continue request.
    }
}
```

当你在 `<url-pattern>` 中做好恰当的地址映射，覆盖所有应该被覆盖的JSP文件，也就不需要再 JSP 文件中添加这些相同的 Java 代码

-----------

* 如果你想执行一些 Java 代码来**预处理**一个请求，例如，预加载某些从数据库加载的数据来显示在一些表格里，可能还会有一些查询参数，那么，实现一个 Servlet，在 doGet() 方法里编写正确的代码，例如

```java
protected void doGet(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
    try {
        List<Product> products = productService.list(); // Obtain all products.
        request.setAttribute("products", products); // Store products in request scope.
        request.getRequestDispatcher("/WEB-INF/products.jsp").forward(request, response); // Forward to JSP page to display them in a HTML table.
    } catch (SQLException e) {
        throw new ServletException("Retrieving products failed!", e);
    }
}
```

这个方法能够更方便地处理异常。这样会在渲染、展示 JSP 页面时访问数据库。在数据库抛出异常的时候，你可以根据情况返回不同的响应或页面。在上面的例子，出错时默认会展示500页面，你也可以改变 `web.xml` 的 `<error-page>` 来自定义异常处理错误页。

-----------

* 如果你想执行一些Java代码来**后置处理（postprocess）**一个请求，例如处理表单提交，那么，实现一个 Servlet，在 doPost() 里写上正确的代码：

```java
protected void doPost(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
    String username = request.getParameter("username");
    String password = request.getParameter("password");
    User user = userService.find(username, password);

    if (user != null) {
        request.getSession().setAttribute("user", user); // Login user.
        response.sendRedirect("home"); // Redirect to home page.
    } else {
        request.setAttribute("message", "Unknown username/password. Please retry."); // Store error message in request scope.
        request.getRequestDispatcher("/WEB-INF/login.jsp").forward(request, response); // Forward to JSP page to redisplay login form with error.
    }
}
```

这个处理不同目标结果页的方法会比原来更加简单： 可以显示一个带有表单验证错误提示的表单（在这个特别的例子中，你可以用EL表达式`${message}`来显示错误提示），或者仅仅跳转到成功的页面

-----------

* 如果你想执行一些 Java 代码来**控制**执行计划（control the execution plan） 和/或 request 和 response 的跳转目标，用 [MVC模式](http://stackoverflow.com/questions/3541077/design-patterns-web-based-applications/3542297#3542297) 实现一个 Servlet，例如：

```java
protected void service(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
    try {
        Action action = ActionFactory.getAction(request);
        String view = action.execute(request, response);

        if (view.equals(request.getPathInfo().substring(1)) {
            request.getRequestDispatcher("/WEB-INF/" + view + ".jsp").forward(request, response);
        } else {
            response.sendRedirect(view);
        }
    } catch (Exception e) {
        throw new ServletException("Executing action failed.", e);
    }
}
```

或者使用一些 MVC 框架例如[JSF](http://stackoverflow.com/tags/jsf/info), [Spring MVC](http://stackoverflow.com/tags/spring-mvc/info), [Wicket](http://stackoverflow.com/tags/wicket/info) 这样你就不用自定义 servlet，只要写一些页面和 javabean class 就可以了。

---------

* 如果你想执行一些 Java 代码来**控制JSP页面的数据渲染流程（control the flow inside a JSP page）**，那么你需要使用一些（已经存在的）流程控制标签库，比如[JSTL core](http://docs.oracle.com/javaee/5/jstl/1.1/docs/tlddocs/c/tld-summary.html)，例如，在一个表格显示`List<Product>`

```java
<%@ taglib uri="http://java.sun.com/jsp/jstl/core" prefix="c" %>
...
<table>
    <c:forEach items="${products}" var="product">
        <tr>
            <td>${product.name}</td>
            <td>${product.description}</td>
            <td>${product.price}</td>
        </tr>
    </c:forEach>
</table>
```

这些 XML 风格的标签可以很好地适应 HTML 代码，代码变得更好阅读（也因此更好地维护），相比于杂乱无章的 scriptlets 的分支大括号（Where the heck does this closing brace belong to?" (到底这个结束大括号是属于哪个代码段的？)）。一个简单的设置可以配置你的 Web 程序让在使用 scriptlets 的时候自动抛出异常

```xml
<jsp-config>
    <jsp-property-group>
        <url-pattern>*.jsp</url-pattern>
        <scripting-invalid>true</scripting-invalid>
    </jsp-property-group>
</jsp-config>
```

在JSP的继承者[Facelets](http://stackoverflow.com/tags/facelets/info)里（Java EE提供的MVC框架[JSF](http://stackoverflow.com/tags/jsf/info)），已经**不**可能使用scriptlets语法了。这是一个让你强制使用“正确的方法”的方法

-----------

* 如果你想执行一些 Java 代码来在 JSP 中 **访问和显示** 一些“后端”数据，你需要使用 EL（表达式），`${}`，例如，显示已经提交了的数值：

```jsp
<input type="text" name="foo" value="${param.foo}" />
```

`${param.foo}`会显示`request.getParameter("foo")`这句话的输出结果。

-----------

* 如果你想在 JSP 直接执行一些工具类 Java 代码（典型的，一些 public static 方法），你需要定义它，并使用 EL 表达式函数。这是 JSTL 里的标准函数标签库，但是你也可以[轻松地创建自己需要的功能](http://docs.oracle.com/javaee/5/tutorial/doc/bnahq.html#bnaiq)，下面是一个使用有用的`fn:escapeXml` 来避免 XSS 攻击的例子。

```java
<%@ taglib uri="http://java.sun.com/jsp/jstl/functions" prefix="fn" %>
...
<input type="text" name="foo" value="${fn:escapeXml(param.foo)}" />
```

注意，XSS 并不是 Java/JSP/JSTL/EL/任何技术相关 的东西，这个问题是**任何** Web 应用程序都需要关心的问题，scriptlets 并没有为这个问题提供良好的解决方案，至少没有标准的 Java API 的解决方案。JSP 的继承者 Facelets 内含了 HTML 转义功能，所以在 Facelets 里你不用担心 XSS 攻击的问题。

See Also:

* [JSP, Servlet, JSF的不同点在哪里？](http://stackoverflow.com/questions/2095397/what-is-the-difference-between-jsf-servlet-and-jsp/2097732#2097732)

* [Servlet, ServletContext, HttpSession 和 HttpServletRequest/Response 是如何工作的?](http://stackoverflow.com/questions/3106452/java-servlet-instantiation-and-session-variables/3106909#3106909)

* [JSP, Servlet and JDBC的基本MVC例子](http://stackoverflow.com/questions/5003142/jsp-using-mvc-and-jdbc)

* [Java Web应用程序中的设计模式](http://stackoverflow.com/questions/3541077/design-patterns-web-based-applications/)

* [JSP/Servlet中的隐藏功能](http://balusc.blogspot.com/2010/01/hidden-features-of-jspservlet.html)

> 原文地址：[http://stackoverflow.com/questions/3177733/how-to-avoid-java-code-in-jsp-files](http://stackoverflow.com/questions/3177733/how-to-avoid-java-code-in-jsp-files)