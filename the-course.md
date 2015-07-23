# 教程

想要用 Python 和 Flask 开发应用吗？让我们来边看例子边学习。本教程中我们将会创建 一个微博应用。这个应用只支持单一用户，只能创建文本条目，也没有不能订阅和评论， 但是已经具备一个初学者需要掌握的功能。这个应用只需要使用 Flask 和 SQLite ， SQLite 是 Python 自带的。

如果你想要事先下载完整的源代码或者用于比较，请查看[示例源代码](http://github.com/mitsuhiko/flask/tree/master/examples/flaskr/) 。

## Flaskr 介绍

我们把教程中的博客应用称为 flaskr ，当然你也可以随便取一个没有 web-2.0 气息的名字 ;) 它的基本功能如下：

1. 让用户可以根据配置文件中的信息登录和注销。只支持一个用户。
2. 用户登录以后可以添加新的博客条目。条目由文本标题和支持 HTML 代码的内容组成。 因为我们信任用户，所以不对内容中的 HTML 进行净化处理。
3. 页面以倒序（新的在上面）显示所有条目。并且用户登录以后可以在这个页面添加新的条目。


我们直接在应用中使用 SQLite3 ，因为在这种规模的应用中 SQlite3 已经够用了。如果 是大型应用，那么就有必要使用能够好的处理数据库连接的 [SQLAlchemy](http://www.sqlalchemy.org/) 了，它可以 同时对应多种数据库，并做其他更多的事情。如果你的数据更适合使用 NoSQL 数据库， 那么也可以考虑使用某种流行的 NoSQL 数据库。

这是教程应用完工后的截图：

![](images/5-1/png)

## 步骤 0 ：创建文件夹¶
在开始之前需要为应用创建下列文件夹:

```
/flaskr
    /static
    /templates
```

flaskr 文件夹不是一个 Python 包，只是一个用来存放我们文件的地方。我们将把以后要用到的数据库模式和主模块放在这个文件夹中。 static 文件夹中的文件是用于供应用用户通过 HTTP 访问的文件，主要是 CSS 和 javascript 文件。 Flask 将会在 templates 文件夹中搜索 Jinja2 模板，所有在教程中的模板都放在 templates 文件夹中。

## 步骤 1 ：数据库模式

首先我们要创建数据库模式。本应用只需要使用一张表，并且由于我们使用 SQLite ， 所以这一步非常简单。把以下内容保存为 schema.sql 文件并放在我们上一步创建的 flaskr 文件夹中就行了：

```
drop table if exists entries;
create table entries (
  id integer primary key autoincrement,
  title text not null,
  text text not null
);
```

这个模式只有一张名为 entries 的表，表中的字段为 id 、 title 和 text 。 id 是主键，是自增整数型字段，另外两个字段是非空的字符串型字段。

## 步骤 2 ：应用构建代码

现在我们已经准备好了数据库模式了，下面来创建应用模块。我们把模块命名为 flaskr.py ，并放在 flaskr 文件夹中。为了方便初学者学习，我们把库的导入与相关配置放在了一起。对于小型应用来说，可以把配置直接放在模块中。但是更加清晰的 方案是把配置放在一个独立的 .ini 或 .py 文件中，并在模块中导入配置的值。

在 flaskr.py 文件中:

```
# all the imports
import sqlite3
from flask import Flask, request, session, g, redirect, url_for, \
     abort, render_template, flash

# configuration
DATABASE = '/tmp/flaskr.db'
DEBUG = True
SECRET_KEY = 'development key'
USERNAME = 'admin'
PASSWORD = 'default'
```

接着创建真正的应用，并用同一文件中的配置来初始化，在 flaskr.py 文件中:

```
# create our little application :)
app = Flask(__name__)
app.config.from_object(__name__)
```

[from_object()](http://dormousehole.readthedocs.org/en/latest/api.html#flask.Config.from_object) 会查看给定的对象（如果该对象是一个字符串就会直接导入它），搜索对象中所有变量名均为大字字母的变量。在我们的应用中，已经将配 置写在前面了。你可以把这些配置放到一个独立的文件中。

通常，从一个配置文件中导入配置是比较好的做法，我们使用 [from_envvar()](http://dormousehole.readthedocs.org/en/latest/api.html#flask.Config.from_envvar) 来完成这个工作，把上面的 from_object() 一行替换为:

```
app.config.from_envvar('FLASKR_SETTINGS', silent=True)
```

这样做就可以设置一个 FLASKR_SETTINGS 的环境变量来指定一个配置文件，并根据该文件来重载缺省的配置。 silent 开关的作用是告诉 Flask 如果没有这个环境变量 不要报错。

secret_key （密钥）用于保持客户端会话安全，请谨慎地选择密钥，并尽可能的使 复杂而且不容易被猜到。 DEBUG 标志用于开关交互调试器。因为调试模式允许用户执行服务器上的代码，所以永远不要在生产环境中打开调试模式 ！

我们还添加了一个方便连接指定数据库的方法。这个方法可以用于在请求时打开连接，也可以用于 Python 交互终端或代码中。以后会派上用场。

```
def connect_db():
    return sqlite3.connect(app.config['DATABASE'])
```

最后，在文件末尾添加以单机方式启动服务器的代码:

```
if __name__ == '__main__':
    app.run()
```

到此为止，我们可以顺利运行应用了。输入以下命令开始运行:

```
python flaskr.py
```

你会看到服务器已经运行的信息，其中包含应用访问地址。

因为我们还没创建视图，所以当你在浏览器中访问这个地址时，会得到一个 404 页面未 找到错误。很快我们就会谈到视图，但我们先要弄好数据库。

### 外部可见的服务器

想让你的服务器被公开访问？详见[外部可见的服务器](http://dormousehole.readthedocs.org/en/latest/quickstart.html#public-server) 。


## 步骤 3 ：创建数据库

如前所述 Flaskr 是一个数据库驱动的应用，更准确地说是一个关系型数据库驱动的 应用。关系型数据库需要一个数据库模式来定义如何储存信息，因此必须在第一次运行 服务器前创建数据库模式。

使用 sqlite3 命令通过管道导入 schema.sql 创建模式:

```
sqlite3 /tmp/flaskr.db < schema.sql
```

上述方法的不足之处是需要额外的 sqlite3 命令，但这个命令不是每个系统都有的。而且还必须提供数据库的路径，容易出错。因此更好的方法是在应用中添加一个数据库初始化函数。

添加的方法是：首先从 contextlib 库中导入 [contextlib.closing()](http://docs.python.org/dev/library/contextlib.html#contextlib.closing) 函数，即在 flaskr.py 文件的导入部分添加如下内容:

```
from contextlib import closing
```

接下来，可以创建一个用来初始化数据库的 init_db 函数，其中我们使用了先前创建的 connect_db 函数。把这个初始化函数放在 flaskr.py 文件中的`connect_db` 函数 下面:

```
def init_db():
    with closing(connect_db()) as db:
        with app.open_resource('schema.sql', mode='r') as f:
            db.cursor().executescript(f.read())
        db.commit()
```

[closing()](http://docs.python.org/dev/library/contextlib.html#contextlib.closing) 帮助函数允许我们在 with 代码块保持数据库连接打开。应用对象的 [open_resource()](http://dormousehole.readthedocs.org/en/latest/api.html#flask.Flask.open_resource) 方法支持也支持这个功能， 可以在 with 代码块中直接使用。这个函数打开一个位于来源位置（你的 flaskr 文件夹）的文件并允许你读取文件的内容。这里我们用于在数据库连接上执行代码。

当我们连接到数据库时，我们得到一个提供指针的连接对象（本例中的 db ）。这个 指针有一个方法可以执行完整的代码。最后我们提供要做的修改。 SQLite 3 和其他事务型数据库只有在显式提交时才会真正提交。

现在可以创建数据库了。打开 Python shell ，导入，调用函数:

```
>>> from flaskr import init_db
>>> init_db()
```

### 故障处理

如果出现表无法找到的问题，请检查是否写错了函数名称（应该是 init_db ）， 是否写错了表名（例如单数复数错误）。

## 步骤 4 ：请求数据库连接

现在我们已经学会如何打开并在代码中使用数据库连接，但是如何优雅地在请求时使用它 呢？我们会在每一个函数中用到数据库连接，因此有必要在请求之前初始化连接，并在请求之后关闭连接。

Flask 中可以使用 [before_request()](http://dormousehole.readthedocs.org/en/latest/api.html#flask.Flask.before_request) 、 [after_request()](http://dormousehole.readthedocs.org/en/latest/api.html#flask.Flask.after_request) 和 [teardown_request()](http://dormousehole.readthedocs.org/en/latest/api.html#flask.Flask.teardown_request) 装饰器达到这个目的:

```
@app.before_request
def before_request():
    g.db = connect_db()

@app.teardown_request
def teardown_request(exception):
    db = getattr(g, 'db', None)
    if db is not None:
        db.close()
    g.db.close()
```

使用 before_request() 装饰的函数会在请求之前调用，且不传递 参数。使用 after_request() 装饰的函数会在请求之后调用，且 传递发送给客户端响应对象。它们必须传递响应对象，所以在出错的情况下就不会执行。 因此我们就要用到 teardown_request() 装饰器了。这个装饰器下 的函数在响应对象构建后被调用。它们不允许修改请求，并且它们的返回值被忽略。如果 请求过程中出错，那么这个错误会传递给每个函数；否则传递 None 。

我们把数据库连接保存在 Flask 提供的特殊的 g 对象中。这个对象与 每一个请求是一一对应的，并且只在函数内部有效。不要在其它对象中储存类似信息， 因为在多线程环境下无效。这个特殊的 g 对象会在后台神奇的工作，保证系统正常运行。

若想更好地处理这种资源，请参阅在 [Flask 中使用 SQLite 3](http://dormousehole.readthedocs.org/en/latest/patterns/sqlite3.html#sqlite3) 。


### Hint

我该把这些代码放在哪里？

如果你按教程一步一步读下来，那么可能会疑惑应该把这个步骤和以后的代码放在哪 里？比较有条理的做法是把这些模块级别的函数放在一起，并把新的 before_request 和 teardown_request 函数放在前文的 init_db 函数 下面（即按照教程的顺序放置）。

如果你已经晕头转向了，那么你可以参考一下 示例源代码 。在 Flask 中，你可以把应用的所有代码都放在同一个 Python 模块中。但是你没有必要这样做，尤其是当你的应用变大了的时候，更不应当这样。


## 步骤 5 ：视图函数

现在数据库连接弄好了，接着开始写视图函数。我们共需要四个视图函数：

### 显示条目
这个视图显示所有数据库中的条目。它绑定应用的根地址，并从数据库中读取 title 和 text 字段。 id 最大的记录（最新的条目）在最上面。从指针返回的记录集是一个包含 select 语句查询结果的元组。对于教程应用这样的小应用，做到这样就已经够好了。但是你可能想要把结果转换为字典，具体做法参见简化查询 中的例子。

这个视图会把条目作为字典传递给 show_entries.html 模板，并返回渲染结果:

```
@app.route('/')
def show_entries():
    cur = g.db.execute('select title, text from entries order by id desc')
    entries = [dict(title=row[0], text=row[1]) for row in cur.fetchall()]
    return render_template('show_entries.html', entries=entries)
```

### 添加一个新条目

这个视图可以让一个登录后的用户添加一个新条目。本视图只响应 POST 请求，真正的表单显示在 show_entries 页面中。如果一切顺利，我们会 flash() 一个消息给下一个请求并重定向回到 show_entries 页面:

```
@app.route('/add', methods=['POST'])
def add_entry():
    if not session.get('logged_in'):
        abort(401)
    g.db.execute('insert into entries (title, text) values (?, ?)',
                 [request.form['title'], request.form['text']])
    g.db.commit()
    flash('New entry was successfully posted')
    return redirect(url_for('show_entries'))
```

注意，我们在本视图中检查了用户是否已经登录（即检查会话中是否有 logged_in 键，且对应的值是否为 True ）。

### 安全性建议

请像示例代码一样确保在构建 SQL 语句时使用问号。否则当你使用字符串构建 SQL 时容易遭到 SQL 注入攻击。更多内容参见 在 Flask 中使用 SQLite 3 。

## 登录和注销

这些函数用于用户登录和注销。登录视图根据配置中的用户名和密码验证用户并在会话中设置 logged_in 键值。如果用户通过验证，键值设为 True ，那么用户会被重定向到 show_entries 页面。另外闪现一个信息，告诉用户已登录成功。如果出现错误，模板会 提示错误信息，并让用户重新登录:

```
@app.route('/login', methods=['GET', 'POST'])
def login():
    error = None
    if request.method == 'POST':
        if request.form['username'] != app.config['USERNAME']:
            error = 'Invalid username'
        elif request.form['password'] != app.config['PASSWORD']:
            error = 'Invalid password'
        else:
            session['logged_in'] = True
            flash('You were logged in')
            return redirect(url_for('show_entries'))
    return render_template('login.html', error=error)
```

登出视图则正好相反，把键值从会话中删除。在这里我们使用了一个小技巧：如果你使用字典的 [pop()](http://docs.python.org/dev/library/stdtypes.html#dict.pop) 方法并且传递了第二个参数（键的缺省值），那么当字典中有 这个键时就会删除这个键，否则什么也不做。这样做的好处是我们不用检查用户是否已经登录了。

```
@app.route('/logout')
def logout():
    session.pop('logged_in', None)
    flash('You were logged out')
    return redirect(url_for('show_entries'))
```

## 步骤 6 ：模板

现在开始写模板。如果我们现在访问 URL ，那么会得到一个 Flask 无法找到模板文件的 异常。 Flask 使用 Jinja2 模板语法并默认开启自动转义。也就是说除非用 Markup 标记一个值或在模板中使用 |safe 过滤器，否则 Jinja2 会把如 < 或 > 之类的特殊字符转义为与其 XML 等价字符。

我们还使用了模板继承以保存所有页面的布局统一。

请把以下模板放在 templates 文件夹中：

## layout.html

这个模板包含 HTML 骨架、头部和一个登录链接（如果用户已登录则变为一个注销链接 ）。如果有闪现信息，那么还会显示闪现信息。 {% block body %} 块会被子模板中同名（ body ）的块替换。

session 字典在模板中也可以使用。你可以使用它来检验用户是否已经 登录。注意，在 Jinja 中可以访问对象或字典的不存在的属性和成员。如例子中的 'logged_in' 键不存在时代码仍然能正常运行：

```
<!doctype html>
<title>Flaskr</title>
<link rel=stylesheet type=text/css href="{{ url_for('static', filename='style.css') }}">
<div class=page>
  <h1>Flaskr</h1>
  <div class=metanav>
  {% if not session.logged_in %}
    <a href="{{ url_for('login') }}">log in</a>
  {% else %}
    <a href="{{ url_for('logout') }}">log out</a>
  {% endif %}
  </div>
  {% for message in get_flashed_messages() %}
    <div class=flash>{{ message }}</div>
  {% endfor %}
  {% block body %}{% endblock %}
</div>
```

## show_entries.html

这个模板扩展了上述的 layout.html 模板，用于显示信息。注意， for 遍历了我们通过 [render_template()](http://dormousehole.readthedocs.org/en/latest/api.html#flask.render_template) 函数传递的所有信息。模板还告诉表单使用 POST 作为 HTTP 方法向 add_entry 函数提交数据：

```
{% extends "layout.html" %}
{% block body %}
  {% if session.logged_in %}
    <form action="{{ url_for('add_entry') }}" method=post class=add-entry>
      <dl>
        <dt>Title:
        <dd><input type=text size=30 name=title>
        <dt>Text:
        <dd><textarea name=text rows=5 cols=40></textarea>
        <dd><input type=submit value=Share>
      </dl>
    </form>
  {% endif %}
  <ul class=entries>
  {% for entry in entries %}
    <li><h2>{{ entry.title }}</h2>{{ entry.text|safe }}
  {% else %}
    <li><em>Unbelievable.  No entries here so far</em>
  {% endfor %}
  </ul>
{% endblock %}
```

## login.html

最后是简单显示用户登录表单的登录模板：

```
{% extends "layout.html" %}
{% block body %}
  <h2>Login</h2>
  {% if error %}<p class=error><strong>Error:</strong> {{ error }}{% endif %}
  <form action="{{ url_for('login') }}" method=post>
    <dl>
      <dt>Username:
      <dd><input type=text name=username>
      <dt>Password:
      <dd><input type=password name=password>
      <dd><input type=submit value=Login>
    </dl>
  </form>
{% endblock %}
```

## 步骤 7 ：添加样式

现在万事俱备，只剩给应用添加一些样式了。只要把以下内容保存为 static 文件夹中的 style.css 文件就行了：

```
body            { font-family: sans-serif; background: #eee; }
a, h1, h2       { color: #377ba8; }
h1, h2          { font-family: 'Georgia', serif; margin: 0; }
h1              { border-bottom: 2px solid #eee; }
h2              { font-size: 1.2em; }

.page           { margin: 2em auto; width: 35em; border: 5px solid #ccc;
                  padding: 0.8em; background: white; }
.entries        { list-style: none; margin: 0; padding: 0; }
.entries li     { margin: 0.8em 1.2em; }
.entries li h2  { margin-left: -1em; }
.add-entry      { font-size: 0.9em; border-bottom: 1px solid #ccc; }
.add-entry dl   { font-weight: bold; }
.metanav        { text-align: right; font-size: 0.8em; padding: 0.3em;
                  margin-bottom: 1em; background: #fafafa; }
.flash          { background: #cee5F5; padding: 0.5em;
                  border: 1px solid #aacbe2; }
.error          { background: #f0d6d6; padding: 0.5em; }
```

## 额外赠送：测试应用

现在你已经完成了整个应用，一切都运行正常。为了方便以后进行完善修改，添加自动测试不失为一个好主意。本教程中的应用已成为[测试 Flask 应用](http://dormousehole.readthedocs.org/en/latest/testing.html#testing)文档中演示如何进行 单元测试的例子，可以去看看测试 Flask 应用是多么容易

*© Copyright 2013, Armin Ronacher. Created using [Sphinx](http://sphinx.pocoo.org/).*
