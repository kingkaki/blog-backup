---
title: 【Struts2-命令-代码执行漏洞分析系列】S2-012
date: 2018-09-02 14:05:50
tags:
	- java
	- struts2
	- 漏洞分析
---



# 前言

漏洞环境改自vulhub的，环境地址 https://github.com/kingkaki/Struts2-Vulenv/tree/master/S2-012

# 漏洞信息

https://cwiki.apache.org/confluence/display/WW/S2-012

![](Struts2-命令-代码执行漏洞分析系列-S2-012\1.png)

> The second evaluation happens when redirect result reads it from the stack and uses the previously injected code as redirect parameter. This lets malicious users put arbitrary OGNL statements into any unsanitized String variable exposed by an action and have it evaluated as an OGNL expression to enable method execution and execute arbitrary methods, bypassing Struts and OGNL library protections.

当发生重定向时，OGNL表达式会进行二次评估，导致之前在`S2-003`、`S2-005`、`S2-009`进行的参数过滤未对重定向值进行过滤，导致了OGNL表达式的执行。



# 漏洞利用

环境中需要配置一个重定向设置，而且对重定向的参数不能进行限制（默认是没有限制

struts.xml

```xml
<!DOCTYPE struts PUBLIC
        "-//Apache Software Foundation//DTD Struts Configuration 2.0//EN"
        "http://struts.apache.org/dtds/struts-2.0.dtd">
<struts>

    <!-- <constant name="struts.enable.DynamicMethodInvocation" value="true" /> -->
    <constant name="struts.devMode" value="false" />

    <!-- Add packages here -->
    <package name="S2-012" extends="struts-default">
        <action name="user" class="com.demo.action.UserAction">
            <result name="redirect" type="redirect">/index.jsp?name=${name}</result>
            <result name="input">/index.jsp</result>
            <result name="success">/index.jsp</result>
        </action>
    </package>
</struts>
```

在重定向的参数中传入一个OGNL表达式`%{1+1}`

![](Struts2-命令-代码执行漏洞分析系列-S2-012\2.png)

明显可以看到被解析成了2，这也就造成了代码的执行



这样的话，只要传入`S2-001`中的poc就可以任意代码执行

![](Struts2-命令-代码执行漏洞分析系列-S2-012\3.png)



# 漏洞分析

在UserAction.java中的`execute()`下断点

![](Struts2-命令-代码执行漏洞分析系列-S2-012\4.png)

可以看到此时的参数信息，step over后会来到`xwork-core-2.2.3.jar!/com/opensymphony/xwork2/DefaultActionInvocation.class`中

然后不断step over下去可以回到这个堆栈中

![](Struts2-命令-代码执行漏洞分析系列-S2-012\5.png)

然后来到这这块代码区域

```java
} else {
    this.resultCode = this.invokeActionOnly();
}

if (!this.executed) {
    if (this.preResultListeners != null) {
        Iterator i$ = this.preResultListeners.iterator();

        while(i$.hasNext()) {
            Object preResultListener = (PreResultListener)i$.next();
            PreResultListener listener = (PreResultListener)preResultListener;
            String _profileKey = "preResultListener: ";

            try {
                UtilTimerStack.push(_profileKey);
                listener.beforeResult(this, this.resultCode);
            } finally {
                UtilTimerStack.pop(_profileKey);
            }
        }
    }

    if (this.proxy.getExecuteResult()) {
        this.executeResult();
    }
```



由于不满足`this.preResultListeners != null`就直接跳到了如下代码中

```java
if (this.proxy.getExecuteResult()) {
    this.executeResult();
}
```

![](Struts2-命令-代码执行漏洞分析系列-S2-012\6.png)

也就是在这里执行了OGNL表达式，造成了的代码执行



# 漏洞修复

> The OGNLUtil class was changed to deny eval expressions by default.

开启默认拒绝了恶意的OGNL表达式

对比了struts2.3.14和struts2.3.14.1

在XWorkConstans.class中

![](Struts2-命令-代码执行漏洞分析系列-S2-012\7.png)

在DefaultConfiguration.class中添加了如下语句

![](Struts2-命令-代码执行漏洞分析系列-S2-012\8.png)



# Reference Links

https://cwiki.apache.org/confluence/display/WW/S2-012

https://github.com/vulhub/vulhub/tree/master/struts2/s2-012







