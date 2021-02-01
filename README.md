# Django reCaptcha v2 [![Build Status](https://travis-ci.org/kbytesys/django-recaptcha2.svg?branch=master)](https://travis-ci.org/kbytesys/django-recaptcha2)
----

这个集成应用在 Django 上为可视化渲染以及支持多种 recaptcha 形式的<a href="https://developers.google.com/recaptcha/intro"> Google reCaptcha v2 </a>实现了 recaptcha 字段。目前已支持reCAPTCHA 隐藏形式在 Django 上的自动渲染，请阅读以下相关文档。

如果你在寻找 Django 上的 Google reCaptcha v3 项目，可以去看看另一个专门的repository https://github.com/longhuan1999/django-recaptcha3

----

## 如何安装

通过 pip 安装所需的软件包 （或下载源码自行安装）：

```bash
pip install django-recaptcha2
```

然后将 django-recaptcha2 添加到您 Django 项目配置文件的 installed apps 中：

```python
INSTALLED_APPS = (
    ...
    'snowpenguin.django.recaptcha2',
    ...
)
```

并将您的 reCaptcha 私钥和公钥添加到您 Django 项目的 settings.py 中：

```python
RECAPTCHA_PRIVATE_KEY = 'your private key'
RECAPTCHA_PUBLIC_KEY = 'your public key'
# 如果您需要从其它地方加载 reCaptcha，而不是 https://google.com
# （比如：绕过防火墙限制）， 你可以通过以下设置指定要使用的代理，但此设置仅会修改前端加载 reCaptcha 的途径。
# RECAPTCHA_PROXY_HOST = 'https://recaptcha.net'
```

**如果你的服务器在国内，则还需要修改 django-recaptcha3 的源码：**

      1. 请先 pip uninstall django-recaptcha3 进行卸载， 然后下载 release 中的源码压缩包。
  
      2. 修改 django-recaptcha3-master\snowpenguin\django\recaptcha3\fields.py#38 :
  
```python
        try:
            r = requests.post(
                'https://www.google.com/recaptcha/api/siteverify', // 此处域名改为 www.recaptcha.net
                {
                    'secret': self._private_key,
                    'response': response_token
                },
                timeout=5
            )
```

      3. 在 django-recaptcha3-master 目录下执行 pip install . 。

如果需要为您的 django 项目所用到的的域名创建 apikey ，则可以访问此<a href="https://www.google.com/recaptcha/admin">网站</a>。

## "I'm not a robot" 的用法 
### 表单和控件
您可以使用此应用程序提供的字段便捷地创建一个包含 reCaptcha 的表单：

```python
from snowpenguin.django.recaptcha2.fields import ReCaptchaField
from snowpenguin.django.recaptcha2.widgets import ReCaptchaWidget

class ExampleForm(forms.Form):
    [...]
    captcha = ReCaptchaField(widget=ReCaptchaWidget())
    [...]
```

如果您不想用 settings.py 中设置的私钥，可以在字段中添加 "private_key" 参数来指定私钥，并且您可以将一些参数传递到部件构造器中：

```python
class ReCaptchaWidget(Widget):
    def __init__(self, explicit=False, container_id=None, theme=None, type=None, size=None, tabindex=None,
                 callback=None, expired_callback=None, attrs={}, *args, **kwargs):
```

如果将 explicit 布尔值设置为 true ，则将在可视化渲染支持下渲染此字段。如果要在同一个页面中使用多个带有 reCaptcha 的表单，这将很有用。查看模板和示例部分可以获取更多信息。

您可以自定义 reCaptcha 的 theme, type, size, tabindex, callback and expired_callback 参数。如果您想修改这些参数可以参考 reCaptcha
<a href="https://developers.google.com/recaptcha/docs/display#config">文档</a>.
警告：此应用不会验证传入的参数。

### Recaptcha "container id"
目前，Recaptcha 的默认容器ID（container id）是：

* recaptcha-{$fieldname} 用于自动渲染
* recaptcha-{$fieldname}-{%fiverandomdigits} 用于可视化渲染

当您在拥有相同字段名且在同一页面的不同表单中使用多个 Recaptcha 实例时，这样可以避免名称冲突。

**注意：** 您始终可以在部件构造类中使用 "container_id" 参数覆盖 "container id"，但请注意：您需要自己手动检查您提供的ID是否已被使用。

### 模板
您可以使用一些模板标签来简化 reCaptcha 的使用：
 
* recaptcha_init: 为 reCaptcha api 添加 script 标签。您需要将此标签放在 "head" 元素中的某个位置
* recaptcha_explicit_init: 为 reCaptcha api 添加支持可视化渲染的 script 标签。您需要将此标签放在 "body" 结束标签之前的某个位置。如果您使用了此标签，则不必再使用 "recaptcha_init"。
* recaptcha_explicit_support: 此标签添加 reCaptcha 用于可视化渲染的回调函数。此标签还添加了 ReCaptchaWidget 在使用 explicit = True 初始化时需要用到的一些函数和 javascript 变量。您需要将此标签放在 "head" 元素中的某个位置。
* recaptcha_key: 如果要在您的 HTML 模板中手动使用 reCaptcha 而不通过 forms ，则需要 sitekey（又名 public api key）。此标签将返回带有配置文件中公钥的字符串。
  
您可以照常使用该表单。

### 强制设置构件的文本语音
您可以在recaptha2 init 标签中禁用自动检测语言即指定文字语言：

```django
{% load recaptcha2 %}
<html>
  <head>
      {% recaptcha_init 'es' %}
  </head>
```

or

```django
    </form>
    {% recaptcha_explicit_init 'es'%}
  </body>
</html>
```

可在<a href="https://developers.google.com/recaptcha/docs/language">此页面</a>查询语言代码。

### 示例
#### 简单渲染的示例

只需创建一个带有 reCaptcha 字段的表单，然后遵循以下模板示例：

```django
{% load recaptcha2 %}
<html>
  <head>
      {% recaptcha_init %}
  </head>
  <body>
    <form action="?" method="POST">
      {% csrf_token %}
      {{ form }}
      <input type="submit" value="Submit">
    </form>
  </body>
</html>
```

#### 可视化渲染的示例

创建一个带有 explicit = True 属性的表单，并像这样编写模板：

```django
{% load recaptcha2 %}
<html>
  <head>
    {% recaptcha_explicit_support %}
  </head>
  <body>
    <form action="?" method="POST">
      {% csrf_token %}
      {{ form }}
      <input type="submit" value="Submit">
    </form>
    {% recaptcha_explicit_init %}
  </body>
</html>
```

#### 渲染多个的示例

您可以使用仅带有 explicit = True 属性的表单渲染多个 reCaptcha ：

```django
{% load recaptcha2 %}
<html>
  <head>
      {% recaptcha_explicit_support %}
  </head>
  <body>
    <form action="{% url 'form1_post' %}" method="POST">
      {% csrf_token %}
      {{ form1 }}
      <input type="submit" value="Submit">
    </form>
    <form action="{% url 'form2_post' %}" method="POST">
      {% csrf_token %}
      {{ form2 }}
      <input type="submit" value="Submit">
    </form>
    {% recaptcha_explicit_init %}
  </body>
</html>
```

#### 应用支持的混合手动渲染

您同样可以使用此应用的可视化渲染支持在你模板的其中一个表单中实现 reCaptcha ：

```django
{% load recaptcha2 %}
<html>
    <head>
        {% recaptcha_explicit_support %}
    </head>
    <body>
        [...]
        <div id='recaptcha'></div>
        <script>
            django_recaptcha_callbacks.push(function() {
                grecaptcha.render('recaptcha', {
                    'theme': 'dark',
                    'sitekey': '{% recaptcha_key %}'
                })
            });
        </script>
        [...]
        {% recaptcha_explicit_init %}
    </body>
</html>
```

## "Invisible"（隐藏形式）的用法
这种绑定的实现和用法更加简单，您无需使用可视化渲染即可添加 reCAPTCHA 的多个实例。

### 表单和控件
您可以使用此应用提供的字段便捷地创建一个启用 reCaptcha 的表单：

```python
from snowpenguin.django.recaptcha2.fields import ReCaptchaField
from snowpenguin.django.recaptcha2.widgets import ReCaptchaHiddenInput

class ExampleForm(forms.Form):
    [...]
    captcha = ReCaptchaField(widget=ReCaptchaHiddenInput())
    [...]
```

如果您不想用 settings.py 中设置的私钥，可以在字段中添加 "private_key" 参数来指定私钥。

### 模板
您只需要在页面 head 中添加 "recaptcha_init" 标签，并将隐藏式的 reCAPTCHA 提交按钮放置在表单内：

```django
<form id='myform1' action="?" method="POST">
      {% csrf_token %}
      {{ form }}
      {% recaptcha_invisible_button submit_label='Submit' %}
</form>
```

您可以使用其定义中包含的参数来自定义按钮：

```python
def recaptcha_invisible_button(public_key=None, submit_label=None, extra_css_classes=None,
                               form_id=None, custom_callback=None):
``` 

您可以覆盖 reCAPTCHA 公钥，更改按钮的 label ，应用额外的CSS类，强制按钮提交由ID标识的表单或提供自定义函数的名称。请查看示例以了解其工作原理。
You can override the reCAPTCHA public key, change the label of the button, apply extra css classes, force
the button to submit a form identified by id or provide the name of a custom callback. Please check the samples
to understand how it works.

### 示例
#### 简单的示例

```django
{% load recaptcha2 %}
<html>
  <head>
      {% recaptcha_init %}
  </head>
  <body>
    <form action="?" method="POST">
      {% csrf_token %}
      {{ form }}
      {% recaptcha_invisible_button submit_label='Submit' %}
    </form>
  </body>
</html>
```

**注意：** 该按钮将使用 "Element.closest" 函数查找第一个 "form" 元素。IE不支持它，因此请使用 polyfill（例如：https://polyfill.io ）。如果您不想添加额外的 JavaScript 库，请使用表单ID或自定义函数。

#### 表单 id

```django
{% load recaptcha2 %}
<html>
  <head>
      {% recaptcha_init %}
  </head>
  <body>
    <form id='myform' action="?" method="POST">
      {% csrf_token %}
      {{ form }}
      {% recaptcha_invisible_button submit_label='Submit' form_id='myform' %}
    </form>
  </body>
</html>
```

#### 自定义函数

```django
{% load recaptcha2 %}
<html>
  <head>
      {% recaptcha_init %}
  </head>
  <body>
    <form id='myform' action="?" method="POST">
      {% csrf_token %}
      {{ form }}
      {% recaptcha_invisible_button submit_label='Submit' custom_callback='mycallback' %}
      <script>
          function mycallback(token) {
              someFunction();
              document.getElementById("myform").submit();
          }
      </script>
    </form>
  </body>
</html>
```

### 待办事项

- 仅支持自动绑定，但是您可以在表单内添加虚拟小部件，并在模板中添加所需的 javascript 代码，以便以编程方式使用绑定和调用。

- 您只能在配置中配置一个 reCAPTCHA 密钥。这其实不是一个问题，因为如果您要使用隐藏式的 reCAPTCHA ，则不再需要使用“旧版本”。如果需要同时使用这两种实现，则仍可以在字段、标签和部件构造函数中设置公钥和私钥。

- ReCaptchaHiddenInput 可能是创建一些 "I'm not a robot" reCAPTCHA 模板标签以代替 ReCaptchaWidget 的起点（可能在将来的版本中）

## 测试
### Test unit support
您无法在测试中模拟 api 调用，但是可以禁用 recaptcha 字段来进行测试工作。

只需在测试中设置 RECAPTCHA_DISABLE env 变量即可：

```python
os.environ['RECAPTCHA_DISABLE'] = 'True'
```

警告：您可以使用任何单词代替 "True" ，clean 函数将仅检查变量是否存在。

### Test unit with recaptcha2 disabled
```python
import os
import unittest

from yourpackage.forms import MyForm

class TestCase(unittest.TestCase):
    def setUp(self):
        os.environ['RECAPTCHA_DISABLE'] = 'True'

    def test_myform(self):
        form = MyForm({
            'field1': 'field1_value'
        })
        self.assertTrue(form.is_valid())

    def tearDown(self):
        del os.environ['RECAPTCHA_DISABLE']
```
