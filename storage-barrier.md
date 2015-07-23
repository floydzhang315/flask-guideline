# Flask 方案

有一些东西是大多数网络应用都会用到的。比如许多应用都会使用关系型数据库和用户 验证，在请求之前连接数据库并得到当前登录用户的信息，在请求之后关闭数据库连接。

更多用户贡献的代码片断和方案参见 [Flask 代码片断归档](http://flask.pocoo.org/snippets/) 。

## 大型应用

对于大型应用来说使用包代替模块是一个好主意。使用包非常简单。假设有一个小应用如 下:

```
/yourapplication
    /yourapplication.py
    /static
        /style.css
    /templates
        layout.html
        index.html
        login.html
        ...
```

### 简单的包

要把上例中的小应用装换为大型应用只要在现有应用中创建一个名为 yourapplication 的新文件夹，并把所有东西都移动到这个文件夹内。然后把 yourapplication.py 更名 为 __init__.py 。（请首先删除所有 .pyc 文件，否则基本上会出问题）

修改完后应该如下例:

```
/yourapplication
    /yourapplication
        /__init__.py
        /static
            /style.css
        /templates
            layout.html
            index.html
            login.html
            ...
```

但是现在如何运行应用呢？原本的 python yourapplication/__init__.py 无法运行 了。因为 Python 不希望包内的模块成为启动文件。但是这不是一个大问题，只要在 yourapplication 文件夹旁添加一个 runserver.py 文件就可以了，其内容如下:

```
from yourapplication import app
app.run(debug=True)
```

我们从中学到了什么？现在我们来重构一下应用以适应多模块。只要记住以下几点：

1. Flask 应用对象必须位于 __init__.py 文件中。这样每个模块就可以安全地导入 了，且 __name__ 变量会解析到正确的包。
2. 所有视图函数（在顶端有 route() 的）必须在 __init__.py 文件中被导入。不是导入对象本身，而是导入视图模块。请 在应用对象创建之后 导入视图对象。

__init__.py 示例:

```
from flask import Flask
app = Flask(__name__)

import yourapplication.views
```

views.py 内容如下:

```
from yourapplication import app

@app.route('/')
def index():
    return 'Hello World!'
```

最终全部内容如下:

```
/yourapplication
    /runserver.py
    /yourapplication
        /__init__.py
        /views.py
        /static
            /style.css
        /templates
            layout.html
            index.html
            login.html
            ...
```

#### 回环导入

回环导入是指两个模块互相导入，本例中我们添加的 views.py 就与 __init__.py 相互依赖。每个 Python 程序员都讨厌回环导入。一般情况下回环导入是个坏主意，但 在这里一点问题都没有。原因是我们没有真正使用 __init__.py 中的视图，只是 保证模块被导入，并且我们在文件底部才这样做。

但是这种方式还是有些问题，因为没有办法使用装饰器。要找到解决问题的灵感请参阅 大型应用 一节。

### 使用蓝图

对于大型应用推荐把应用分隔为小块，每个小块使用蓝图辅助执行。关于这个主题的介绍 请参阅[使用蓝图的模块化应用](blueprint-model.md)一节 。


## 应用工厂

如果你已经在应用中使用了包和蓝图（ 使用蓝图的模块化应用 ），那么还有许多方法可以更 进一步地改进你的应用。常用的方案是导入蓝图后创建应用对象，但是如果在一个函数中 创建对象，那么就可以创建多个实例。

那么这样做有什么用呢？

1. 用于测试。可以针对不同的情况使用不同的配置来测试应用。
2. 用于多实例，如果你需要运行同一个应用的不同版本的话。当然你可以在服务器上 使用不同配置运行多个相同应用，但是如果使用应用工厂，那么你可以只使用一个 应用进程而得到多个应用实例，这样更容易操控。

那么如何做呢？

### 基础工厂

方法是在一个函数中设置应用，具体如下:

```
def create_app(config_filename):
    app = Flask(__name__)
    app.config.from_pyfile(config_filename)

    from yourapplication.model import db
    db.init_app(app)

    from yourapplication.views.admin import admin
    from yourapplication.views.frontend import frontend
    app.register_blueprint(admin)
    app.register_blueprint(frontend)

    return app
```

这个方法的缺点是在导入时无法在蓝图中使用应用对象。但是你可以在一个请求中使用它。 如何通过配置来访问应用？使用 [current_app](http://dormousehole.readthedocs.org/en/latest/api.html#flask.current_app):

```
from flask import current_app, Blueprint, render_template
admin = Blueprint('admin', __name__, url_prefix='/admin')

@admin.route('/')
def index():
    return render_template(current_app.config['INDEX_TEMPLATE'])
```

这里我们在配置中查找模板的名称。

扩展对象初始化时不会绑定到一个应用，应用可以使用 db.init_app 来设置扩展。 扩展对象中不会储存特定应用的状态，因此一个扩展可以被多个应用使用。关于扩展设计 的更多信息请参阅 [Flask 扩展开发](app-extend.md) 。

当使用 Flask-SQLAlchemy 时，你的 model.py 可能是这样的:

```
from flask.ext.sqlalchemy import SQLAlchemy
# no app object passed! Instead we use use db.init_app in the factory.
db = SQLAlchemy()

# create some models
```

### 使用应用

因此，要使用这样的应用就必须先创建它。下面是一个运行应用的示例 run.py 文件:

```
from yourapplication import create_app
app = create_app('/path/to/config.cfg')
app.run()
```

### 改进工厂

上面的工厂函数还不是足够好，可以改进的地方主要有以下几点：

1. 为了单元测试，要想办法传入配置，这样就不必在文件系统中创建配置文件。
2. 当设置应用时从蓝图调用一个函数，这样就可以有机会修改属性（如挂接请求前/后 处理器等）。
4. 如果有必要的话，当创建一个应用时增加一个 WSGI 中间件。


## 应用调度

应用调度是在 WSGI 层面组合多个 WSGI 应用的过程。可以组合多个 Flask 应用，也可以 组合 Flask 应用和其他 WSGI 应用。通过这种组合，如果有必要的话，甚至可以在同一个 解释器中一边运行 Django ，一边运行 Flask 。这种组合的好处取决于应用内部是如何 工作的。

应用调度与模块化的最大不同在于应用调度中的每个 应用是完全独立的，它们以各自的配置运行，并在 WSGI 层面被调度。

### 说明

下面所有的技术说明和举例都归结于一个可以运行于任何 WSGI 服务器的 application 对象。对于生产环境，参见 部署方式 。对于开发环境， Werkzeug 提供了一个内建开发服务器，它使用 werkzeug.serving.run_simple() 来运行:

```
from werkzeug.serving import run_simple
run_simple('localhost', 5000, application, use_reloader=True)
```

注意 [run_simple](http://werkzeug.pocoo.org/docs/serving/#werkzeug.serving.run_simple) 不能用于生产环境，生产 环境服务器参见成熟的 WSGI 服务器 。

为了使用交互调试器，应用和简单服务器都应当处于调试模式。下面是一个简单的 “ hello world ”示例，使用了调试模式和 run_simple:

```
from flask import Flask
from werkzeug.serving import run_simple

app = Flask(__name__)
app.debug = True

@app.route('/')
def hello_world():
    return 'Hello World!'

if __name__ == '__main__':
    run_simple('localhost', 5000, app,
               use_reloader=True, use_debugger=True, use_evalex=True)
```

### 组合应用

如果你想在同一个 Python 解释器中运行多个独立的应用，那么你可以使用 [werkzeug.wsgi.DispatcherMiddleware](http://werkzeug.pocoo.org/docs/middlewares/#werkzeug.wsgi.DispatcherMiddleware) 。其原理是：每个独立的 Flask 应用都 是一个合法的 WSGI 应用，它们通过调度中间件组合为一个基于前缀调度的大应用。

假设你的主应用运行于 / ，后台接口位于 /backend:

```
from werkzeug.wsgi import DispatcherMiddleware
from frontend_app import application as frontend
from backend_app import application as backend

application = DispatcherMiddleware(frontend, {
    '/backend':     backend
})
```

### 根据子域调度

有时候你可能需要使用不同的配置来运行同一个应用的多个实例。可以把应用创建过程 放在一个函数中，这样调用这个函数就可以创建一个应用的实例，具体实现参见 应用工厂 方案。

最常见的做法是每个子域创建一个应用，配置服务器来调度所有子域的应用请求，使用 子域来创建用户自定义的实例。一旦你的服务器可以监听所有子域，那么就可以使用一个 很简单的 WSGI 应用来动态创建应用了。

WSGI 层是完美的抽象层，因此可以写一个你自己的 WSGI 应用来监视请求，并把请求分配 给你的 Flask 应用。如果被分配的应用还没有创建，那么就会动态创建应用并被登记 下来:

```
from threading import Lock

class SubdomainDispatcher(object):

    def __init__(self, domain, create_app):
        self.domain = domain
        self.create_app = create_app
        self.lock = Lock()
        self.instances = {}

    def get_application(self, host):
        host = host.split(':')[0]
        assert host.endswith(self.domain), 'Configuration error'
        subdomain = host[:-len(self.domain)].rstrip('.')
        with self.lock:
            app = self.instances.get(subdomain)
            if app is None:
                app = self.create_app(subdomain)
                self.instances[subdomain] = app
            return app

    def __call__(self, environ, start_response):
        app = self.get_application(environ['HTTP_HOST'])
        return app(environ, start_response)
```

调度器示例:

```
from myapplication import create_app, get_user_for_subdomain
from werkzeug.exceptions import NotFound

def make_app(subdomain):
    user = get_user_for_subdomain(subdomain)
    if user is None:
        # 如果子域没有对应的用户，那么还是得返回一个 WSGI 应用
        # 用于处理请求。这里我们把 NotFound() 异常作为应用返回，
        # 它会被渲染为一个缺省的 404 页面。然后，可能还需要把
        # 用户重定向到主页。
        return NotFound()

    # 否则为特定用户创建应用
    return create_app(user)

application = SubdomainDispatcher('example.com', make_app)
```

### 根据路径调度


根据 URL 的路径调度非常简单。上面，我们通过查找 Host 头来判断子域，现在 只要查找请求路径的第一个斜杠之前的路径就可以了:

```
from threading import Lock
from werkzeug.wsgi import pop_path_info, peek_path_info

class PathDispatcher(object):

    def __init__(self, default_app, create_app):
        self.default_app = default_app
        self.create_app = create_app
        self.lock = Lock()
        self.instances = {}

    def get_application(self, prefix):
        with self.lock:
            app = self.instances.get(prefix)
            if app is None:
                app = self.create_app(prefix)
                if app is not None:
                    self.instances[prefix] = app
            return app

    def __call__(self, environ, start_response):
        app = self.get_application(peek_path_info(environ))
        if app is not None:
            pop_path_info(environ)
        else:
            app = self.default_app
        return app(environ, start_response)
```

与根据子域调度相比最大的不同是：根据路径调度时，如果创建函数返回 None ，那么 就会回落到另一个应用:

```
from myapplication import create_app, default_app, get_user_for_prefix

def make_app(prefix):
    user = get_user_for_prefix(prefix)
    if user is not None:
        return create_app(user)

application = PathDispatcher(default_app, make_app)
```

## 实现 API 异常处理

在 Flask 上经常会执行 RESTful API 。开发者首先会遇到的问题之一是用于 API 的内建 异常处理不给力，回馈的内容不是很有用。

对于非法使用 API ，比使用 abort 更好的解决方式是实现你自己的异常处理类型， 并安装相应句柄，输出符合用户格式要求的出错信息。

### 简单的异常类

基本的思路是引入一个新的异常，回馈一个合适的可读性高的信息、一个状态码和一些 可选的负载，给错误提供更多的环境内容。

以下是一个简单的示例:

```
from flask import jsonify

class InvalidUsage(Exception):
    status_code = 400

    def __init__(self, message, status_code=None, payload=None):
        Exception.__init__(self)
        self.message = message
        if status_code is not None:
            self.status_code = status_code
        self.payload = payload

    def to_dict(self):
        rv = dict(self.payload or ())
        rv['message'] = self.message
        return rv
```

这样一个视图就可以抛出带有出错信息的异常了。另外，还可以通过 payload 参数以 字典的形式提供一些额外的负载。

### 注册一个错误处理句柄

现在，视图可以抛出异常，但是会立即引发一个内部服务错误。这是因为没有为这个错误 处理类注册句柄。句柄增加很容易，例如:

```
@app.errorhandler(InvalidAPIUsage)
def handle_invalid_usage(error):
    response = jsonify(error.to_dict())
    response.status_code = error.status_code
    return response
```

### 在视图中的用法

以下是如何在视图中使用该功能:

```
@app.route('/foo')
def get_foo():
    raise InvalidUsage('This view is gone', status_code=410)
```

## URL 处理器

New in version 0.7.

Flask 0.7 引入了 URL 处理器，其作用是为你处理大量包含相同部分的 URL 。假设你有 许多 URL 都包含语言代码，但是又不想在每个函数中都重复处理这个语言代码，那么就可 可以使用 URL 处理器。

在与蓝图配合使用时， URL 处理器格外有用。下面我们分别演示在应用中和蓝图中使用 URL 处理器。

### 国际化应用的 URL

假设有应用如下:

```
from flask import Flask, g

app = Flask(__name__)

@app.route('/<lang_code>/')
def index(lang_code):
    g.lang_code = lang_code
    ...

@app.route('/<lang_code>/about')
def about(lang_code):
    g.lang_code = lang_code
    ...
```

上例中出现了大量的重复：必须在每一个函数中把语言代码赋值给 g 对象。当然，如果使用一个装饰器可以简化这个工作。但是，当你需要生成由一个函数 指向另一个函数的 URL 时，还是得显式地提供语言代码，相当麻烦。

我们使用 url_defaults() 函数来简化这个问题。这个函数可以自动 把值注入到 url_for() 。以下代码检查在 URL 字典中是否存在语言代码， 端点是否需要一个名为 'lang_code' 的值:

```
@app.url_defaults
def add_language_code(endpoint, values):
    if 'lang_code' in values or not g.lang_code:
        return
    if app.url_map.is_endpoint_expecting(endpoint, 'lang_code'):
        values['lang_code'] = g.lang_code
```

URL 映射的 [is_endpoint_expecting()](http://werkzeug.pocoo.org/docs/routing/#werkzeug.routing.Map.is_endpoint_expecting) 方法可用于检查 端点是否需要提供一个语言代码。

上例的逆向函数是 [url_value_preprocessor()](http://dormousehole.readthedocs.org/en/latest/api.html#flask.Flask.url_value_preprocessor) 。这些函数在请求 匹配后立即根据 URL 的值执行代码。它们可以从 URL 字典中取出值，并把取出的值放在 其他地方:

```
@app.url_value_preprocessor
def pull_lang_code(endpoint, values):
    g.lang_code = values.pop('lang_code', None)
```

这样就不必在每个函数中把 lang_code 赋值给 g 了。你还可以作 进一步改进：写一个装饰器把语言代码作为 URL 的前缀。但是更好的解决方式是使用 蓝图。一旦 'lang_code' 从值的字典中弹出，它就不再传送给视图函数了。精简后的代码如下:

```
from flask import Flask, g

app = Flask(__name__)

@app.url_defaults
def add_language_code(endpoint, values):
    if 'lang_code' in values or not g.lang_code:
        return
    if app.url_map.is_endpoint_expecting(endpoint, 'lang_code'):
        values['lang_code'] = g.lang_code

@app.url_value_preprocessor
def pull_lang_code(endpoint, values):
    g.lang_code = values.pop('lang_code', None)

@app.route('/<lang_code>/')
def index():
    ...

@app.route('/<lang_code>/about')
def about():
    ...
```

### 国际化的蓝图 URL

因为蓝图可以自动给所有 URL 加上一个统一的前缀，所以应用到每个函数就非常方便了。 更进一步，因为蓝图 URL 预处理器不需要检查 URL 是否真的需要要一个 'lang_code' 参数，所以可以去除 [url_defaults()](http://dormousehole.readthedocs.org/en/latest/api.html#flask.Flask.url_defaults))函数中的 逻辑判断:

```
from flask import Blueprint, g

bp = Blueprint('frontend', __name__, url_prefix='/<lang_code>')

@bp.url_defaults
def add_language_code(endpoint, values):
    values.setdefault('lang_code', g.lang_code)

@bp.url_value_preprocessor
def pull_lang_code(endpoint, values):
    g.lang_code = values.pop('lang_code')

@bp.route('/')
def index():
    ...

@bp.route('/about')
def about():
    ...
```

## 使用 Distribute 部署

[distribute](http://pypi.python.org/pypi/distribute) 的前身是 setuptools ，它是一个扩展库，通常用于分发 Python 库和 扩展。它的英文名称的就是“分发”的意思。它扩展了 Python 自带的一个基础模块安装 系统 distutils ，支持多种更复杂的结构，方便了大型应用的分发部署。它的主要特色：

* 支持依赖 ：一个库或者应用可以声明其所依赖的其他库的列表。依赖库将被自动 安装。
* 包注册 ：可以在安装过程中注册包，这样就可以通过一个包查询其他包的信息。 这套系统最有名的功能是“切入点”，即一个包可以定义一个入口，以便于其他包挂接， 用以扩展包。
* 安装管理 ： distribute 中的 easy_install 可以为你安装其他库。你也可以 使用早晚会替代 easy_install 的 [pip](http://pypi.python.org/pypi/pip) ，它除了安装包还可以做更多的事。

Flask 本身，以及其他所有在 cheeseshop 中可以找到的库要么是用 distribute 分发的， 要么是用老的 setuptools 或 distutils 分发的。

在这里我们假设你的应用名称是 yourapplication.py ，且没有使用模块，而是一个[包](http://dormousehole.readthedocs.org/en/latest/patterns/packages.html#larger-applications) 。 distribute 不支持分发标准模块，因此我们不 讨论模块的问题。关于如何把模块转换为包的信息参见[大型应用方案](huge-app.md)。

使用 distribute 将使发布更复杂，也更加自动化。如果你想要完全自动化处理，请同时 阅读 使用 Fabric 部署 一节。

### 基础设置脚本

因为你已经安装了 Flask ，所以你应当已经安装了 setuptools 或 distribute 。如果 没有安装，不用怕，有一个 [distribute_setup.py](http://python-distribute.org/distribute_setup.py) 脚本可以帮助你安装。只要下载这个 脚本，并在你的 Python 解释器中运行就可以了。

标准声明: [最好使用 virtualenv](http://dormousehole.readthedocs.org/en/latest/installation.html#virtualenv) 。

你的设置代码应用放在 setup.py 文件中，这个文件应当位于应用旁边。这个文件名只是 一个约定，但是最好不要改变，因为大家都会去找这个文件。

是的，即使你使用 distribute ，你导入的包也是 setuptools 。 distribute 完全 向后兼容于 setuptools ，因此它使用相同的导入名称。

Flask 应用的基础 setup.py 文件示例如下:

```
from setuptools import setup

setup(
    name='Your Application',
    version='1.0',
    long_description=__doc__,
    packages=['yourapplication'],
    include_package_data=True,
    zip_safe=False,
    install_requires=['Flask']
)
```

请记住，你必须显式的列出子包。如果你要 distribute 自动为你搜索包，你可以使用 find_packages 函数:

```
from setuptools import setup, find_packages

setup(
    ...
    packages=find_packages()
)
```

大多数 setup 的参数可以望文生义，但是 include_package_data 和 zip_safe 可以不容易理解。 include_package_data 告诉 distribute 要搜索一个 MANIFEST.in 文件，把文件内容所匹配的所有条目作为包数据安装。可以通过使用这个 参数分发 Python 模块的静态文件和模板（参见 分发资源 ）。 zip_safe 标志可用于强制或防止创建 zip 压缩包。通常你不会想要把包安装为 zip 压缩文件，因为一些工具不支持压缩文件，而且压缩文件比较难以调试。

### 分发资源

如果你尝试安装上文创建的包，你会发现诸如 static 或 templates 之类的文件夹 没有被安装。原因是 distribute 不知道要为你添加哪些文件。你要做的是：在你的 setup.py 文件旁边创建一个 MANIFEST.in 文件。这个文件列出了所有应当添加到 tar 压缩包的文件:

```
recursive-include yourapplication/templates *
recursive-include yourapplication/static *
```

不要忘了把 setup 函数的 include_package_data 参数设置为 True ！否则即使把 内容在 MANIFEST.in 文件中全部列出来也没有用。

### 声明依赖

依赖是在 install_requires 参数中声明的，这个参数是一个列表。列表中的每一项都是 一个需要在安装时从 PyPI 获得的包。缺省情况下，总是会获得最新版本的包，但你可以 指定最高版本和最低版本。示例:

```
install_requires=[
    'Flask>=0.2',
    'SQLAlchemy>=0.6',
    'BrokenPackage>=0.7,<=1.0'
]
```

我前面提到，依赖包都从 PyPI 获得的。但是如果要从别的地方获得包怎么办呢？你只要 还是按照上述方法写，然后提供一个可选地址列表就行了:

```
dependency_links=['http://example.com/yourfiles']
```

请确保页面上有一个目录列表，且页面上的链接指向正确的 tar 压缩包。这样 distribute 就会找到文件了。如果你的包在公司内部网络上，请提供指向服务器的 URL 。

### 安装 / 开发

要安装你的应用（理想情况下是安装到一个 virtualenv ），只要运行带 install 参数 的 setup.py 脚本就可以了。它会将你的应用安装到 virtualenv 的 site-packages 文件夹下，同时下载并安装依赖:

```
$ python setup.py install
```

如果你正开发这个包，同时也希望相关依赖被安装，那么可以使用 develop 来代替:

```
$ python setup.py develop
```

这样做的好处是只安装一个指向 site-packages 的连接，而不是把数据复制到那里。这样 在开发过程中就不必每次修改以后再运行 install 了。


## 使用 Fabric 部署

Fabric 是一个 Python 工具，与 Makefiles 类似，但是能够在远程服务器上执行 命令。如果与适当的 Python 包（ 大型应用 ）与优良的配置（ 配置管理 ）相结合那么 Fabric 将是在外部服务器上部署 Flask 的利器。

在下文开始之前，有几点需要明确：

* Fabric 1.0 需要要被安装到本地。本教程假设使用的是最新版本的 Fabric 。
* 应用已经是一个包，且有一个可用的 setup.py 文件（ 使用 Distribute 部署 ）。
* 在下面的例子中，我们假设远程服务器使用 mod_wsgi 。当然，你可以使用你自己 喜欢的服务器，但是在示例中我们选择 Apache + mod_wsgi ，因为它们设置方便， 且在没有 root 权限情况下可以方便的重载应用。

### 创建第一个 Fabfile

fabfile 是控制 Fabric 的东西，其文件名为 fabfile.py ，由 fab 命令执行。在 这个文件中定义的所有函数都会被视作 fab 子命令。这些命令将会在一个或多个主机上 运行。这些主机可以在 fabfile 中定义，也可以在命令行中定义。本例将在 fabfile 中 定义主机。

下面是第一个例子，比较级。它可以把当前的源代码上传至服务器，并安装到一个预先存在 的 virtual 环境:

```
from fabric.api import *

# 使用远程命令的用户名
env.user = 'appuser'
# 执行命令的服务器
env.hosts = ['server1.example.com', 'server2.example.com']

def pack():
    # 创建一个新的分发源，格式为 tar 压缩包
    local('python setup.py sdist --formats=gztar', capture=False)

def deploy():
    # 定义分发版本的名称和版本号
    dist = local('python setup.py --fullname', capture=True).strip()
    # 把 tar 压缩包格式的源代码上传到服务器的临时文件夹
    put('dist/%s.tar.gz' % dist, '/tmp/yourapplication.tar.gz')
    # 创建一个用于解压缩的文件夹，并进入该文件夹
    run('mkdir /tmp/yourapplication')
    with cd('/tmp/yourapplication'):
        run('tar xzf /tmp/yourapplication.tar.gz')
        # 现在使用 virtual 环境的 Python 解释器来安装包
        run('/var/www/yourapplication/env/bin/python setup.py install')
    # 安装完成，删除文件夹
    run('rm -rf /tmp/yourapplication /tmp/yourapplication.tar.gz')
    # 最后 touch .wsgi 文件，让 mod_wsgi 触发应用重载
    run('touch /var/www/yourapplication.wsgi')
```

上例中的注释详细，应当是容易理解的。以下是 fabric 提供的最常用命令的简要说明：

* run - 在远程服务器上执行一个命令
* local - 在本地机器上执行一个命令
* put - 上传文件到远程服务器上
* cd - 在服务器端改变目录。必须与 with 语句联合使用。

### 运行 Fabfile

那么如何运行 fabfile 呢？答案是使用 fab 命令。要在远程服务器上部署当前版本的 代码可以使用这个命令:

```
$ fab pack deploy
```

但是这个命令需要远程服务器上已经创建了 /var/www/yourapplication 文件夹，且 /var/www/yourapplication/env 是一个 virtual 环境。更进一步，服务器上还没有 创建配置文件和 .wsgi 文件。那么，我们如何在一个新的服务器上创建一个基础环境 呢？

这个问题取决于你要设置多少台服务器。如果只有一台应用服务器（多数情况下），那么 在 fabfile 中创建命令有一点多余。当然，你可以这么做。这个命令可以称之为 setup 或 bootstrap 。在使用命令时显式传递服务器名称:

```
$ fab -H newserver.example.com bootstrap
```

设置一个新服务器大致有以下几个步骤：

1. 在 /var/www 创建目录结构:

```
$ mkdir /var/www/yourapplication
$ cd /var/www/yourapplication
$ virtualenv --distribute env
```

2. 上传一个新的 application.wsgi 文件和应用配置文件（如 application.cfg ） 到服务器上。

3. 创建一个新的用于 yourapplication 的 Apache 配置并激活它。要确保激活 .wsgi 文件变动监视，这样在 touch 的时候可以自动重载应用。（ 更多信息参见 [mod_wsgi (Apache)](http://dormousehole.readthedocs.org/en/latest/deploying/mod_wsgi.html#mod-wsgi-deployment) ）

现在的问题是： application.wsgi 和 application.cfg 文件从哪里来？

### WSGI 文件

WSGI 文件必须导入应用，并且还必须设置一个环境变量用于告诉应用到哪里去搜索配置。 示例:

```
import os
os.environ['YOURAPPLICATION_CONFIG'] = '/var/www/yourapplication/application.cfg'
from yourapplication import app
```

应用本身必须像下面这样初始化自己才会根据环境变量搜索配置:

```
app = Flask(__name__)
app.config.from_object('yourapplication.default_config')
app.config.from_envvar('YOURAPPLICATION_CONFIG')
```

这个方法在[配置管理](config-manange.md)一节已作了详细的介绍。

### 配置文件

上文已谈到，应用会根据 YOURAPPLICATION_CONFIG 环境变量找到正确的配置文件。 因此我们应当把配置文件放在应用可以找到的地方。在不同的电脑上配置文件是不同的， 所以一般我们不对配置文件作版本处理。

一个流行的方法是在一个独立的版本控制仓库为不同的服务器保存不同的配置文件，然后 在所有服务器进行检出。然后在需要的地方使用配置文件的符号链接（例如： /var/www/yourapplication ）。

不管如何，我们这里只有一到两台服务器，因此我们可以预先手动上传配置文件。

### 第一次部署

现在我们可以进行第一次部署了。我已经设置好了服务器，因此服务器上应当已经有了 virtual 环境和已激活的 apache 配置。现在我们可以打包应用并部署它了:

```
$ fab pack deploy
```

Fabric 现在会连接所有服务器并运行 fabfile 中的所有命令。首先它会打包应用得到一个 tar 压缩包。然后会执行分发，把源代码上传到所有服务器并安装。感谢 setup.py 文件，所需要的依赖库会自动安装到 virtual 环境。

### 下一步

在前文的基础上，还有更多的方法可以全部署工作更加轻松：

* 创建一个初始化新服务器的 bootstrap 命令。它可以初始化一个新的 virtual 环境、正确设置 apache 等等。
* 把配置文件放入一个独立的版本库中，把活动配置的符号链接放在适当的地方。
* 还可以把应用代码放在一个版本库中，在服务器上检出最新版本后安装。这样你可以 方便的回滚到老版本。
* 挂接测试功能，方便部署到外部服务器进行测试。
使用 Fabric 是一件有趣的事情。你会发现在电脑上打出 fab deploy 是非常神奇的。 你可以看到你的应用被部署到一个又一个服务器上。


## 在 Flask 中使用 SQLite 3

在 Flask 中，你可以方便的按需打开数据库连接，并且在环境解散时关闭这个连接（ 通常是请求结束的时候）。

以下是一个在 Flask 中使用 SQLite 3 的例子:

```
import sqlite3
from flask import g

DATABASE = '/path/to/database.db'

def get_db():
    db = getattr(g, '_database', None)
    if db is None:
        db = g._database = connect_to_database()
    return db

@app.teardown_appcontext
def close_connection(exception):
    db = getattr(g, '_database', None)
    if db is not None:
        db.close()
```

为了使用数据库，所有应用都必须准备好一个处于激活状态的环境。使用 get_db 函数可以得到数据库连接。当环境解散时，数据库连接会被切断。

注意：如果你使用的是 Flask 0.9 或者以前的版本，那么你必须使用 flask._app_ctx_stack.top ，而不是 g 。因为 [flask.g](http://dormousehole.readthedocs.org/en/latest/api.html#flask.g) 对象是绑定到 请求的，而不是应用环境。

示例:

```
@app.route('/')
def index():
    cur = get_db().cursor()
    ...
```

> Note 请记住，解散请求和应用环境的函数是一定会被执行的。即使请求前处理器执行失败或根本没有执行，解散函数也会被执行。因此，我们必须保证在关闭数据库连接之前 数据库连接是存在的。


### 按需连接

上述方式（在第一次使用时连接数据库）的优点是只有在真正需要时才打开数据库连接。 如果你想要在一个请求环境之外使用数据库连接，那么你可以手动在 Python 解释器打开 应用环境:

```
with app.app_context():
    # now you can use get_db()
```

### 简化查询

现在，在每个请求处理函数中可以通过访问 g.db 来得到当前打开的数据库连接。为了 简化 SQLite 的使用，这里有一个有用的行工厂函数。该函数会转换每次从数据库返回的 结果。例如，为了得到字典类型而不是元组类型的返回结果，可以这样:

```
def make_dicts(cursor, row):
    return dict((cur.description[idx][0], value)
                for idx, value in enumerate(row))

db.row_factory = make_dicts
```

或者更简单的:

```
db.row_factory = sqlite3.Row
```

此外，把得到游标，执行查询和获得结果组合成一个查询函数不失为一个好办法:

```
def query_db(query, args=(), one=False):
    cur = get_db().execute(query, args)
    rv = cur.fetchall()
    cur.close()
    return (rv[0] if rv else None) if one else rv
```

上述的方便的小函数与行工厂联合使用与使用原始的数据库游标和连接相比要方便多了。

可以这样使用上述函数:

```
for user in query_db('select * from users'):
    print user['username'], 'has the id', user['user_id']
```

只需要得到单一结果的用法:

```
user = query_db('select * from users where username = ?',
                [the_username], one=True)
if user is None:
    print 'No such user'
else:
    print the_username, 'has the id', user['user_id']
```

如果要给 SQL 语句传递参数，请在语句中使用问号来代替参数，并把参数放在一个列表中 一起传递。不要用字符串格式化的方式直接把参数加入 SQL 语句中，这样会给应用带来 [SQL 注入](http://en.wikipedia.org/wiki/SQL_injection)的风险。

### 初始化模式

关系数据库是需要模式的，因此一个应用常常需要一个 schema.sql 文件来创建 数据库。因此我们需要使用一个函数来其于模式创建数据库。下面这个函数可以完成这个 任务:

```
def init_db():
    with app.app_context():
        db = get_db()
        with app.open_resource('schema.sql', mode='r') as f:
            db.cursor().executescript(f.read())
        db.commit()
```

可以使用上述函数在 Python 解释器中创建数据库：

```
>>> from yourapplication import init_db
>>> init_db()
```

## 在 Flask 中使用 SQLAlchemy

许多人喜欢使用 [SQLAlchemy](http://www.sqlalchemy.org/) 来访问数据库。建议在你的 Flask 应用中使用包来代替 模块，并把模型放入一个独立的模块中（参见[大型应用](huge-app.md) ）。虽然这 不是必须的，但是很有用。

有四种 SQLAlchemy 的常用方法，下面一一道来：

### Flask-SQLAlchemy 扩展

因为 SQLAlchemy 是一个常用的数据库抽象层，并且需要一定的配置才能使用，因此我们 为你做了一个处理 SQLAlchemy 的扩展。如果你需要快速的开始使用 SQLAlchemy ，那么 推荐你使用这个扩展。

你可以从 [PyPI](http://pypi.python.org/pypi/Flask-SQLAlchemy) 下载 [Flask-SQLAlchemy](http://packages.python.org/Flask-SQLAlchemy/) 。

### 声明

SQLAlchemy 中的声明扩展是使用 SQLAlchemy 的最新方法，它允许你像 Django 一样， 在一个地方定义表和模型然后到处使用。除了以下内容，我建议你阅读 声明 的官方 文档。

以下是示例 database.py 模块:

```
from sqlalchemy import create_engine
from sqlalchemy.orm import scoped_session, sessionmaker
from sqlalchemy.ext.declarative import declarative_base

engine = create_engine('sqlite:////tmp/test.db', convert_unicode=True)
db_session = scoped_session(sessionmaker(autocommit=False,
                                         autoflush=False,
                                         bind=engine))
Base = declarative_base()
Base.query = db_session.query_property()

def init_db():
    # 在这里导入定义模型所需要的所有模块，这样它们就会正确的注册在
    # 元数据上。否则你就必须在调用 init_db() 之前导入它们。
    import yourapplication.models
    Base.metadata.create_all(bind=engine)
```

要定义模型的话，只要继承上面创建的 Base 类就可以了。你可能会奇怪这里为什么 不用理会线程（就像我们在 SQLite3 的例子中一样使用 g 对象）。 原因是 SQLAlchemy 已经用 scoped_session 为我们做好了此 类工作。

如果要在应用中以声明方式使用 SQLAlchemy ，那么只要把下列代码加入应用模块就可以 了。 Flask 会自动在请求结束时或者应用关闭时删除数据库会话:

```
from yourapplication.database import db_session

@app.teardown_appcontext
def shutdown_session(exception=None):
    db_session.remove()
```

以下是一个示例模型（放入 models.py 中）:

```
from sqlalchemy import Column, Integer, String
from yourapplication.database import Base

class User(Base):
    __tablename__ = 'users'
    id = Column(Integer, primary_key=True)
    name = Column(String(50), unique=True)
    email = Column(String(120), unique=True)

    def __init__(self, name=None, email=None):
        self.name = name
        self.email = email

    def __repr__(self):
        return '<User %r>' % (self.name)
```

可以使用 init_db 函数来创建数据库：

```
>>> from yourapplication.database import init_db
>>> init_db()
```

在数据库中插入条目示例：

```
>>> from yourapplication.database import db_session
>>> from yourapplication.models import User
>>> u = User('admin', 'admin@localhost')
>>> db_session.add(u)
>>> db_session.commit()
```

查询很简单：

```
>>> User.query.all()
[<User u'admin'>]
>>> User.query.filter(User.name == 'admin').first()
<User u'admin'>
```

### 人工对象关系映射

人工对象关系映射相较于上面的声明方式有优点也有缺点。主要区别是人工对象关系映射 分别定义表和类并映射它们。这种方式更灵活，但是要多些代码。通常，这种方式与声明 方式一样运行，因此请确保把你的应用在包中分为多个模块。

示例 database.py 模块:

```
from sqlalchemy import create_engine, MetaData
from sqlalchemy.orm import scoped_session, sessionmaker

engine = create_engine('sqlite:////tmp/test.db', convert_unicode=True)
metadata = MetaData()
db_session = scoped_session(sessionmaker(autocommit=False,
                                         autoflush=False,
                                         bind=engine))
def init_db():
    metadata.create_all(bind=engine)
```

就像声明方法一样，你需要在请求后或者应用环境解散后关闭会话。把以下代码放入你的 应用模块:

```
from yourapplication.database import db_session

@app.teardown_appcontext
def shutdown_session(exception=None):
    db_session.remove()
```

以下是一个示例表和模型（放入 models.py 中）:

```
from sqlalchemy import Table, Column, Integer, String
from sqlalchemy.orm import mapper
from yourapplication.database import metadata, db_session

class User(object):
    query = db_session.query_property()

    def __init__(self, name=None, email=None):
        self.name = name
        self.email = email

    def __repr__(self):
        return '<User %r>' % (self.name)

users = Table('users', metadata,
    Column('id', Integer, primary_key=True),
    Column('name', String(50), unique=True),
    Column('email', String(120), unique=True)
)
mapper(User, users)
```

查询和插入与声明方式的一样。

### SQL 抽象层

如果你只需要使用数据库系统（和 SQL ）抽象层，那么基本上只要使用引擎:

```
from sqlalchemy import create_engine, MetaData

engine = create_engine('sqlite:////tmp/test.db', convert_unicode=True)
metadata = MetaData(bind=engine)
```

然后你要么像前文中一样在代码中声明表，要么自动载入它们:

```
users = Table('users', metadata, autoload=True)
```

可以使用 insert 方法插入数据。为了使用事务，我们必须先得到一个连接：

```
>>> con = engine.connect()
>>> con.execute(users.insert(), name='admin', email='admin@localhost')
```

SQLAlchemy 会自动提交。

可以直接使用引擎或连接来查询数据库：

```
>>> users.select(users.c.id == 1).execute().first()
(1, u'admin', u'admin@localhost')
```

查询结果也是类字典元组：

```
>>> r = users.select(users.c.id == 1).execute().first()
>>> r['name']
u'admin'
```

你也可以把 SQL 语句作为字符串传递给 execute() 方法：

```
>>> engine.execute('select * from users where id = :1', [1]).first()
(1, u'admin', u'admin@localhost')
```

关于 SQLAlchemy 的更多信息请移步其[官方网站](http://sqlalchemy.org/) 。


## 上传文件

是的，这里要谈的是一个老问题：文件上传。文件上传的基本原理实际上很简单，基本上是：

1. 一个带有 enctype=multipart/form-data 的 <form> 标记，标记中含有 一个 <input type=file> 。
2. 应用通过请求对象的 files 字典来访问文件。
3. 使用文件的 [save()](http://werkzeug.pocoo.org/docs/datastructures/#werkzeug.datastructures.FileStorage.save) 方法把文件永久 地保存在文件系统中。

### 简介

让我们从一个基本的应用开始，这个应用上传文件到一个指定目录，并把文件显示给用户。 以下是应用的引导代码:

```
import os
from flask import Flask, request, redirect, url_for
from werkzeug import secure_filename

UPLOAD_FOLDER = '/path/to/the/uploads'
ALLOWED_EXTENSIONS = set(['txt', 'pdf', 'png', 'jpg', 'jpeg', 'gif'])

app = Flask(__name__)
app.config['UPLOAD_FOLDER'] = UPLOAD_FOLDER
```

首先我们导入了一堆东西，大多数是浅显易懂的。werkzeug.secure_filename() 会 在稍后解释。UPLOAD_FOLDER 是上传文件要储存的目录，ALLOWED_EXTENSIONS 是允许 上传的文件扩展名的集合。接着我们给应用手动添加了一个 URL 规则。一般现在不会做 这个，但是为什么这么做了呢？原因是我们需要服务器（或我们的开发服务器）为我们提供 服务。因此我们只生成这些文件的 URL 的规则。

为什么要限制文件件的扩展名呢？如果直接向客户端发送数据，那么你可能不会想让用户 上传任意文件。否则，你必须确保用户不能上传 HTML 文件，因为 HTML 可能引起 XSS 问题（参见跨站脚本攻击（XSS） ）。如果服务器可以执行 PHP 文件，那么还必须确保不允许上传 .php 文件。但是谁又会在服务器上安装 PHP 呢，对不？ :)

下一个函数检查扩展名是否合法，上传文件，把用户重定向到已上传文件的 URL:

```
def allowed_file(filename):
    return '.' in filename and \
           filename.rsplit('.', 1)[1] in ALLOWED_EXTENSIONS

@app.route('/', methods=['GET', 'POST'])
def upload_file():
    if request.method == 'POST':
        file = request.files['file']
        if file and allowed_file(file.filename):
            filename = secure_filename(file.filename)
            file.save(os.path.join(app.config['UPLOAD_FOLDER'], filename))
            return redirect(url_for('uploaded_file',
                                    filename=filename))
    return '''
    <!doctype html>
    <title>Upload new File</title>
    <h1>Upload new File</h1>
    <form action="" method=post enctype=multipart/form-data>
      <p><input type=file name=file>
         <input type=submit value=Upload>
    </form>
    '''
```

那么 [secure_filename()](http://werkzeug.pocoo.org/docs/utils/#werkzeug.utils.secure_filename) 函数到底是有什么用？有一条原则是“ 永远不要信任用户输入”。这条原则同样适用于已上传文件的文件名。所有提交的表单数据 可能是伪造的，文件名也可以是危险的。此时要谨记：在把文件保存到文件系统之前总是要 使用这个函数对文件名进行安检。

#### 进一步说明

你可以会好奇 secure_filename() 做了哪些工作，如果不使用 它会有什么后果。假设有人把下面的信息作为 filename 传递给你的应用:

```
filename = "../../../../home/username/.bashrc"
```

假设 ../ 的个数是正确的，你会把它和 UPLOAD_FOLDER 结合在一起，那么用户 就可能有能力修改一个服务器上的文件，这个文件本来是用户无权修改的。这需要了解 应用是如何运行的，但是请相信我，黑客都是病态的 :)

现在来看看函数是如何工作的：

```
>>> secure_filename('../../../../home/username/.bashrc')
'home_username_.bashrc'
```

现在还剩下一件事：为已上传的文件提供服务。 Flask 0.5 版本开始我们可以使用一个 函数来完成这个任务:

```
from flask import send_from_directory

@app.route('/uploads/<filename>')
def uploaded_file(filename):
    return send_from_directory(app.config['UPLOAD_FOLDER'],
                               filename)
```

另外，可以把 uploaded_file 注册为 build_only 规则，并使用 SharedDataMiddleware 。这种方式可以在 Flask 老版本中 使用:

```
from werkzeug import SharedDataMiddleware
app.add_url_rule('/uploads/<filename>', 'uploaded_file',
                 build_only=True)
app.wsgi_app = SharedDataMiddleware(app.wsgi_app, {
    '/uploads':  app.config['UPLOAD_FOLDER']
})
```

如果你现在运行应用，那么应该一切都应该按预期正常工作。

### 改进上传

New in version 0.6.

Flask 到底是如何处理文件上传的呢？如果上传的文件很小，那么会把它们储存在内存中。 否则就会把它们保存到一个临时的位置（通过 tempfile.gettempdir() 可以得到 这个位置）。但是，如何限制上传文件的尺寸呢？缺省情况下， Flask 是不限制上传文件 的尺寸的。可以通过设置配置的 MAX_CONTENT_LENGTH 来限制文件尺寸:

```
from flask import Flask, Request

app = Flask(__name__)
app.config['MAX_CONTENT_LENGTH'] = 16 * 1024 * 1024
```

上面的代码会把尺寸限制为 16 M 。如果上传了大于这个尺寸的文件， Flask 会抛出一个 RequestEntityTooLarge 异常。

Flask 0.6 版本中添加了这个功能。但是通过继承请求对象，在较老的版本中也可以实现 这个功能。更多信息请参阅 Werkzeug 关于文件处理的文档。

### 上传进度条

在不久以前，许多开发者是这样实现上传进度条的：分块读取上传的文件，在数据库中储存 上传的进度，然后在客户端通过 JavaScript 获取进度。简而言之，客户端每 5 秒钟向 服务器询问一次上传进度。觉得讽刺吗？客户端在明知故问。

现在有了更好的解决方案，更快且更可靠。网络发生了很大变化，你可以在客户端使用 HTML5 、 JAVA 、 Silverlight 或 Flash 获得更好的上传体验。请查看以下库，学习 一些优秀的上传的示例：

* [Plupload](http://www.plupload.com/) - HTML5, Java, Flash
* [SWFUpload](http://www.swfupload.org/) - Flash
*  [JumpLoader](http://jumploader.com/) - Java

###一个更简便的方案

因为所有应用中上传文件的方案基本相同，因此可以使用 Flask-Uploads 扩展来实现 文件上传。这个扩展实现了完整的上传机制，还具有白名单功能、黑名单功能以及其他 功能。


## 缓存

当你的应用变慢的时候，可以考虑加入缓存。至少这是最简单的加速方法。缓存有什么用？ 假设有一个函数耗时较长，但是这个函数在五分钟前返回的结果还是正确的。那么我们就 可以考虑把这个函数的结果在缓存中存放一段时间。

Flask 本身不提供缓存，但是它的基础库之一 Werkzeug 有一些非常基本的缓存支持。它 支持多种缓存后端，通常你需要使用的是 memcached 服务器。

### 设置一个缓存

与创建 [Flask](http://dormousehole.readthedocs.org/en/latest/api.html#flask.Flask) 对象类似，你需要创建一个缓存对象并保持它。如果你 正在使用开发服务器，那么你可以创建一个 [SimpleCache](http://werkzeug.pocoo.org/docs/contrib/cache/#werkzeug.contrib.cache.SimpleCache) 对象。这个对象提供简单的缓存，它把 缓存项目保存在 Python 解释器的内存中:

```
from werkzeug.contrib.cache import SimpleCache
cache = SimpleCache()
```

如果你要使用 memcached ，那么请确保有 memcache 模块支持（你可以从 PyPI 获得模块）和一个正在运行的 memcached 服务器。 连接 memcached 服务器示例:

```
from werkzeug.contrib.cache import MemcachedCache
cache = MemcachedCache(['127.0.0.1:11211'])
```

如果你正在使用 App Engine ，那么你可以方便地连接到 App Engine memcache 服务器:

```
from werkzeug.contrib.cache import GAEMemcachedCache
cache = GAEMemcachedCache()
```

### 使用缓存

现在的问题是如何使用缓存呢？有两个非常重要的操作： [get()](http://werkzeug.pocoo.org/docs/contrib/cache/#werkzeug.contrib.cache.BaseCache.get) 和 [set()](http://werkzeug.pocoo.org/docs/contrib/cache/#werkzeug.contrib.cache.BaseCache.set) 。下面展示如何使用它们：

get() 用于从缓存中获得项目，调用时使用 一个字符串作为键名。如果项目存在，那么就会返回这个项目，否则返回 None:

```
rv = cache.get('my-item')
```

set() 用于把项目添加到缓存中。第一个参数 是键名，第二个参数是键值。还可以提供一个超时参数，当超过时间后项目会自动删除。

下面是一个完整的例子:

```
def get_my_item():
    rv = cache.get('my-item')
    if rv is None:
        rv = calculate_value()
        cache.set('my-item', rv, timeout=5 * 60)
    return rv
```

## 视图装饰器

Python 有一个非常有趣的功能：函数装饰器。这个功能可以使网络应用干净整洁。 Flask 中的每个视图都是一个装饰器，它可以被注入额外的功能。你可以已经用过了 [route()](http://dormousehole.readthedocs.org/en/latest/api.html#flask.Flask.route) 装饰器。但是，你有可能需要使用你自己的装饰器。假设有 一个视图，只有已经登录的用户才能使用。如果用户访问时没有登录，则会被重定向到 登录页面。这种情况下就是使用装饰器的绝佳机会。

### 检查登录装饰器

让我们来实现这个装饰器。装饰器是一个返回函数的函数。听上去复杂，其实很简单。只要 记住一件事：装饰器用于更新函数的 __name__ 、 __module__ 和其他属性。这一点很 容易忘记，但是你不必人工更新函数属性，可以使用一个类似于装饰器的函数（ [functools.wraps()](http://docs.python.org/dev/library/functools.html#functools.wraps) ）。

下面是检查登录装饰器的例子。假设登录页面为 'login' ，当前用户被储存在 g.user 中，如果还没有登录，其值为 None:

```
from functools import wraps
from flask import g, request, redirect, url_for

def login_required(f):
    @wraps(f)
    def decorated_function(*args, **kwargs):
        if g.user is None:
            return redirect(url_for('login', next=request.url))
        return f(*args, **kwargs)
    return decorated_function
```

如何使用这个装饰器呢？把这个装饰器放在最靠近函数的地方就行了。当使用更进一步的 装饰器时，请记住要把 route() 装饰器放在最外面:

```
@app.route('/secret_page')
@login_required
def secret_page():
    pass
```

### 缓存装饰器

假设有一个视图函数需要消耗昂贵的计算成本，因此你需要在一定时间内缓存这个视图的 计算结果。这种情况下装饰器是一个好的选择。我们假设你像 缓存 方案中一样设置了缓存。

下面是一个示例缓存函数。它根据一个特定的前缀（实际上是一个格式字符串）和请求的 当前路径生成缓存键。注意，我们先使用了一个函数来创建装饰器，这个装饰器用于装饰 函数。听起来拗口吧，确实有一点复杂，但是下面的示例代码还是很容易读懂的。

被装饰代码按如下步骤工作

1. 基于基础路径获得当前请求的唯一缓存键。
2. 从缓存中获取键值。如果获取成功则返回获取到的值。
3. 否则调用原来的函数，并把返回值存放在缓存中，直至过期（缺省值为五分钟）。

代码:

```
from functools import wraps
from flask import request

def cached(timeout=5 * 60, key='view/%s'):
    def decorator(f):
        @wraps(f)
        def decorated_function(*args, **kwargs):
            cache_key = key % request.path
            rv = cache.get(cache_key)
            if rv is not None:
                return rv
            rv = f(*args, **kwargs)
            cache.set(cache_key, rv, timeout=timeout)
            return rv
        return decorated_function
    return decorator
```

注意，以上代码假设存在一个可用的实例化的 cache 对象，更多信息参见 缓存 方案。

### 模板装饰器

不久前， TurboGear 的人发明了模板装饰器这个通用模式。其工作原理是返回一个字典， 这个字典包含从视图传递给模板的值，模板自动被渲染。以下三个例子的功能是相同的:

```
@app.route('/')
def index():
    return render_template('index.html', value=42)

@app.route('/')
@templated('index.html')
def index():
    return dict(value=42)

@app.route('/')
@templated()
def index():
    return dict(value=42)
```

正如你所见，如果没有提供模板名称，那么就会使用 URL 映射的端点（把点转换为斜杠） 加上 '.html' 。如果提供了，那么就会使用所提供的模板名称。当装饰器函数返回 时，返回的字典就被传送到模板渲染函数。如果返回的是 None ，就会使用空字典。如果 返回的不是字典，那么就会直接传递原封不动的返回值。这样就可以仍然使用重定向函数或 返回简单的字符串。

以下是装饰器的代码:

```
from functools import wraps
from flask import request

def templated(template=None):
    def decorator(f):
        @wraps(f)
        def decorated_function(*args, **kwargs):
            template_name = template
            if template_name is None:
                template_name = request.endpoint \
                    .replace('.', '/') + '.html'
            ctx = f(*args, **kwargs)
            if ctx is None:
                ctx = {}
            elif not isinstance(ctx, dict):
                return ctx
            return render_template(template_name, **ctx)
        return decorated_function
    return decorator
```

### 端点装饰器

当你想要使用 werkzeug 路由系统，以便于获得更强的灵活性时，需要和 Rule 中定义的一样，把端点映射到视图函数。这样就需要 用的装饰器了。例如:

```
from flask import Flask
from werkzeug.routing import Rule

app = Flask(__name__)
app.url_map.add(Rule('/', endpoint='index'))

@app.endpoint('index')
def my_index():
    return "Hello world"
```

## 使用 WTForms 进行表单验证

当你必须处理浏览器提交的表单数据时，视图代码很快会变得难以阅读。有一些库可以 简化这个工作，其中之一便是 [WTForms](http://wtforms.simplecodes.com/) ，下面我们将介绍这个库。如果你必须处理 许多表单，那么应当尝试使用这个库。

如果要使用 WTForms ，那么首先要把表单定义为类。我推荐把应用分割为多个模块（ [大型应用](huge-app.md)），并为表单添加一个独立的模块。

####  使用一个扩展获得大部分 WTForms 的功能

Flask-WTF 扩展可以实现本方案的所有功能，并且还提供一些小的辅助工具。使用 这个扩展可以更好的使用表单和 Flask 。你可以从 PyPI 获得这个扩展。


### 表单

下面是一个典型的注册页面的示例:

```
from wtforms import Form, BooleanField, TextField, PasswordField, validators

class RegistrationForm(Form):
    username = TextField('Username', [validators.Length(min=4, max=25)])
    email = TextField('Email Address', [validators.Length(min=6, max=35)])
    password = PasswordField('New Password', [
        validators.Required(),
        validators.EqualTo('confirm', message='Passwords must match')
    ])
    confirm = PasswordField('Repeat Password')
    accept_tos = BooleanField('I accept the TOS', [validators.Required()])
```

### 视图

在视图函数中，表单用法示例如下:

```
@app.route('/register', methods=['GET', 'POST'])
def register():
    form = RegistrationForm(request.form)
    if request.method == 'POST' and form.validate():
        user = User(form.username.data, form.email.data,
                    form.password.data)
        db_session.add(user)
        flash('Thanks for registering')
        return redirect(url_for('login'))
    return render_template('register.html', form=form)
```

注意，这里我们默认视图使用了 SQLAlchemy （在 Flask 中使用 SQLAlchemy），当然这不是必须的。请根据你的实际情况修改代码。

请记住以下几点：

1. 如果数据是通过 HTTP POST 方法提交的，请根据 form 的 值创建表单。如果是通过 GET 方法提交的，则相应的是 args 。
2. 调用 validate() 函数来验证数据。如果验证通过，则 函数返回 True ，否则返回 False 。
3. 通过 form.<NAME>.data 可以访问表单中单个值。

### 模板中的表单

现在我们来看看模板。把表单传递给模板后就可以轻松渲染它们了。看一看下面的示例 模板就可以知道有多轻松了。 WTForms 替我们完成了一半表单生成工作。为了做得更好， 我们可以写一个宏，通过这个宏渲染带有一个标签的字段和错误列表（如果有的话）。

以下是一个使用宏的示例 _formhelpers.html 模板：

```
{% macro render_field(field) %}
  <dt>{{ field.label }}
  <dd>{{ field(**kwargs)|safe }}
  {% if field.errors %}
    <ul class=errors>
    {% for error in field.errors %}
      <li>{{ error }}</li>
    {% endfor %}
    </ul>
  {% endif %}
  </dd>
{% endmacro %}
```

上例中的宏接受一堆传递给 WTForm 字段函数的参数，为我们渲染字段。参数会作为 HTML 属性插入。例如你可以调用 render_field(form.username, class='username') 来 为输入元素添加一个类。注意： WTForms 返回标准的 Python unicode 字符串，因此我们 必须使用 |safe 过滤器告诉 Jinja2 这些数据已经经过 HTML 转义了。

以下是使用了上面的 _formhelpers.html 的 register.html 模板：

```
{% from "_formhelpers.html" import render_field %}
<form method=post action="/register">
  <dl>
    {{ render_field(form.username) }}
    {{ render_field(form.email) }}
    {{ render_field(form.password) }}
    {{ render_field(form.confirm) }}
    {{ render_field(form.accept_tos) }}
  </dl>
  <p><input type=submit value=Register>
</form>
```

更多关于 WTForms 的信息请移步 [WTForms 官方网站](http://wtforms.simplecodes.com/) 。


## 模板继承

Jinja 最有力的部分就是模板继承。模板继承允许你创建一个基础“骨架”模板。这个模板 中包含站点的常用元素，定义可以被子模板继承的 块 。

听起来很复杂其实做起来简单，看看下面的例子就容易理解了。

### 基础模板

这个模板的名称是 layout.html ，它定义了一个简单的 HTML 骨架，用于显示一个 简单的两栏页面。“子”模板的任务是用内容填充空的块：

```
<!doctype html>
<html>
  <head>
    {% block head %}
    <link rel="stylesheet" href="{{ url_for('static', filename='style.css') }}">
    <title>{% block title %}{% endblock %} - My Webpage</title>
    {% endblock %}
  </head>
  <body>
    <div id="content">{% block content %}{% endblock %}</div>
    <div id="footer">
      {% block footer %}
      &copy; Copyright 2010 by <a href="http://domain.invalid/">you</a>.
      {% endblock %}
    </div>
  </body>
</html>
```

在这个例子中， {% block %} 标记定义了四个可以被子模板填充的块。 block 标记告诉模板引擎这是一个可以被子模板重载的部分。

### 子模板

子模板示例：

```
{% extends "layout.html" %}
{% block title %}Index{% endblock %}
{% block head %}
  {{ super() }}
  <style type="text/css">
    .important { color: #336699; }
  </style>
{% endblock %}
{% block content %}
  <h1>Index</h1>
  <p class="important">
    Welcome on my awesome homepage.
{% endblock %}
```

这里 {% extends %} 标记是关键，它告诉模板引擎这个模板“扩展”了另一个模板， 当模板系统评估这个模板时会先找到父模板。这个扩展标记必须是模板中的第一个标记。 如果要使用父模板中的块内容，请使用 {{ super() }} 。


## 消息闪现

一个好的应用和用户界面都需要良好的反馈。如果用户得不到足够的反馈，那么应用最终 会被用户唾弃。 Flask 的闪现系统提供了一个良好的反馈方式。闪现系统的基本工作方式 是：在且只在下一个请求中访问上一个请求结束时记录的消息。一般我们结合布局模板来 使用闪现系统。

### 简单的例子

以下是一个完整的示例:

```
from flask import Flask, flash, redirect, render_template, \
     request, url_for

app = Flask(__name__)
app.secret_key = 'some_secret'

@app.route('/')
def index():
    return render_template('index.html')

@app.route('/login', methods=['GET', 'POST'])
def login():
    error = None
    if request.method == 'POST':
        if request.form['username'] != 'admin' or \
           request.form['password'] != 'secret':
            error = 'Invalid credentials'
        else:
            flash('You were successfully logged in')
            return redirect(url_for('index'))
    return render_template('login.html', error=error)

if __name__ == "__main__":
    app.run()
```

以下是实现闪现的 layout.html 模板：

```
<!doctype html>
<title>My Application</title>
{% with messages = get_flashed_messages() %}
  {% if messages %}
    <ul class=flashes>
    {% for message in messages %}
      <li>{{ message }}</li>
    {% endfor %}
    </ul>
  {% endif %}
{% endwith %}
{% block body %}{% endblock %}
```

以下是 index.html 模板：

```
{% extends "layout.html" %}
{% block body %}
  <h1>Overview</h1>
  <p>Do you want to <a href="{{ url_for('login') }}">log in?</a>
{% endblock %}
```

login 模板：

```
{% extends "layout.html" %}
{% block body %}
  <h1>Login</h1>
  {% if error %}
    <p class=error><strong>Error:</strong> {{ error }}
  {% endif %}
  <form action="" method=post>
    <dl>
      <dt>Username:
      <dd><input type=text name=username value="{{
          request.form.username }}">
      <dt>Password:
      <dd><input type=password name=password>
    </dl>
    <p><input type=submit value=Login>
  </form>
{% endblock %}
```

### 闪现消息的类别

New in version 0.3.

闪现消息还可以指定类别，如果没有指定，那么缺省的类别为 'message' 。不同的 类别可以给用户提供更好的反馈。例如错误消息可以使用红色背景。

使用 flash() 函数可以指定消息的类别:

```
flash(u'Invalid password provided', 'error')
```

模板中的 [get_flashed_messages()](http://dormousehole.readthedocs.org/en/latest/api.html#flask.get_flashed_messages) 函数也应当返回类别，显示消息的循环 也要略作改变：

```
{% with messages = get_flashed_messages(with_categories=true) %}
  {% if messages %}
    <ul class=flashes>
    {% for category, message in messages %}
      <li class="{{ category }}">{{ message }}</li>
    {% endfor %}
    </ul>
  {% endif %}
{% endwith %}
```

上例展示如何根据类别渲染消息，还可以给消息加上前缀，如 <strong>Error:</strong> 。

### 过滤闪现消息

New in version 0.9.

你可以视情况通过传递一个类别列表来过滤 [get_flashed_messages()](http://dormousehole.readthedocs.org/en/latest/api.html#flask.get_flashed_messages) 的 结果。这个功能有助于在不同位置显示不同类别的消息。

```
{% with errors = get_flashed_messages(category_filter=["error"]) %}
{% if errors %}
<div class="alert-message block-message error">
  <a class="close" href="#">×</a>
  <ul>
    {%- for msg in errors %}
    <li>{{ msg }}</li>
    {% endfor -%}
  </ul>
</div>
{% endif %}
{% endwith %}
```

## 通过 jQuery 使用 AJAX

[jQuery](http://jquery.com/) 是一个小型的 JavaScript 库，通常用于简化 DOM 和 JavaScript 的使用。它 是一个非常好的工具，可以通过在服务端和客户端交换 JSON 来使网络应用更具有动态性。

JSON 是一种非常轻巧的传输格式，非常类似于 Python 原语（数字、字符串、字典和列表 ）。 JSON 被广泛支持，易于解析。 JSON 在几年之前开始流行，在网络应用中迅速取代 了 XML 。

### 载入 jQuery

为了使用 jQuery ，你必须先把它下载下来，放在应用的静态目录中，并确保它被载入。 理想情况下你有一个用于所有页面的布局模板。在这个模板的 <body> 的底部添加一个 script 语句来载入 jQuery ：

```
<script type=text/javascript src="{{
  url_for('static', filename='jquery.js') }}"></script>
```

另一个方法是使用 Google 的 AJAX 库 API 来载入 jQuery：

```
<script src="//ajax.googleapis.com/ajax/libs/jquery/1.9.1/jquery.min.js"></script>
<script>window.jQuery || document.write('<script src="{{
  url_for('static', filename='jquery.js') }}">\x3C/script>')</script>
```

在这种方式中，应用会先尝试从 Google 下载 jQuery ，如果失败则会调用静态目录中的 备用 jQuery 。这样做的好处是如果用户已经去过使用与 Google 相同版本的 jQuery 的 网站后，访问你的网站时，页面可能会更快地载入，因为浏览器已经缓存了 jQuery 。

### 我的网站在哪里？

我的网站在哪里？如果你的应用还在开发中，那么答案很简单：它在本机的某个端口上， 且在服务器的根路径下。但是如果以后要把应用移到其他位置（例如< http://example.com/myapp> ）上呢？在服务端，这个问题不成为问题，可以使用 url_for() 函数来得到答案。但是如果我们使用 jQuery ，那么就不能硬码应用的路径，只能使用动态路径。怎么办？

一个简单的方法是在页面中添加一个 script 标记，设置一个全局变量来表示应用的根 路径。示例：

```
<script type=text/javascript>
  $SCRIPT_ROOT = {{ request.script_root|tojson|safe }};
</script>
```

在 Flask 0.10 版本以前版本中，使用 |safe 是有必要的，是为了使 Jinja 不要 转义 JSON 编码的字符串。通常这样做不是必须的，但是在 script 内部我们需要这么 做。

#### 进一步说明

在 HTML 中， script 标记是用于声明 CDATA 的，也就是说声明的内容不会被 解析。<script> 与 </script> 之间的内容都会被作为脚本处理。这也意味着 在 script 标记之间不会存在任何 </ 。在这里 |tojson 会正确处理问题， 并为你转义斜杠（ {{ "</script>"|tojson|safe }} 会被渲染为 "<\/script>" ）。

在 Flask 0.10 版本中更进了一步，把所有 HTML 标记都用 unicode 转义了，这样使 Flask 自动把 HTML 转换为安全标记。

## JSON 视图函数

现在让我们来创建一个服务端函数，这个函数接收两个 URL 参数（两个需要相加的数字 ），然后向应用返回一个 JSON 对象。下面这个例子是非常不实用的，因为一般会在 客户端完成类似工作，但这个例子可以简单明了地展示如何使用 jQuery 和 Flask:

```
from flask import Flask, jsonify, render_template, request
app = Flask(__name__)

@app.route('/_add_numbers')
def add_numbers():
    a = request.args.get('a', 0, type=int)
    b = request.args.get('b', 0, type=int)
    return jsonify(result=a + b)

@app.route('/')
def index():
    return render_template('index.html')
```

正如你所见，我还添加了一个 index 方法来渲染模板。这个模板会按前文所述载入 jQuery 。模板中有一个用于两个数字相加的表单和一个触发服务端函数的链接。

注意，这里我们使用了 [get()](http://werkzeug.pocoo.org/docs/datastructures/#werkzeug.datastructures.MultiDict.get) 方法。它 不会调用失败。如果字典的键不存在，就会返回一个缺省值（这里是 0 ）。更进一步 它还会把值转换为指定的格式（这里是 int ）。在脚本（ API 、 JavaScript 等） 触发的代码中使用它特别方便，因为在这种情况下不需要特殊的错误报告。

## HTML

你的 index.html 模板要么继承一个已经载入 jQuery 和设置好 $SCRIPT_ROOT 变量的 layout.html 模板，要么在模板开头就做好那两件事。下面就是应用的 HTML 示例 （ index.html ）。注意，我们把脚本直接放入了 HTML 中。通常更好的方式是放在 独立的脚本文件中：

```
<script type=text/javascript>
  $(function() {
    $('a#calculate').bind('click', function() {
      $.getJSON($SCRIPT_ROOT + '/_add_numbers', {
        a: $('input[name="a"]').val(),
        b: $('input[name="b"]').val()
      }, function(data) {
        $("#result").text(data.result);
      });
      return false;
    });
  });
</script>
<h1>jQuery Example</h1>
<p><input type=text size=5 name=a> +
   <input type=text size=5 name=b> =
   <span id=result>?</span>
<p><a href=# id=calculate>calculate server side</a>
```

这里不讲述 jQuery 运行详细情况，仅对上例作一个简单说明：

1. $(function() { ... }) 定义浏览器在页面的基本部分载入完成后立即执行的. 代码。
2. $('selector') 选择一个元素供你操作。
3. element.bind('event', func) 定义一个用户点击元素时运行的函数。如果函数 返回 false ，那么缺省行为就不会起作用（本例为转向 # URL ）。
4. \$.getJSON(url, data, func) 向 url 发送一个 GET 请求，并把 data 对象的内容作为查询参数。一旦有数据返回，它将调用指定的函数，并把返回值作为 函数的参数。注意，我们可以在这里使用先前定义的 $SCRIPT_ROOT 变量。

如果你没有一个完整的概念，请从 github 下载[示例源代码](http://github.com/mitsuhiko/flask/tree/master/examples/jqueryexample) 。

## 自定义出错页面

Flask 有一个方便的 [abort()](http://dormousehole.readthedocs.org/en/latest/api.html#flask.abort) 函数，它可以通过一个 HTTP 出错代码退出 一个请求。它还提供一个包含基本说明的出错页面，页面显示黑白的文本，很朴素。

用户可以根据错误代码或多或少知道发生了什么错误。

### 常见出错代码

以下出错代码是用户常见的，即使应用正常也会出现这些出错代码：

**404 Not Found**
这是一个古老的“朋友，你使用了一个错误的 URL ”信息。这个信息出现得如此 频繁，以至于连刚上网的新手都知道 404 代表：该死的，我要看的东西不见了。 一个好的做法是确保 404 页面上有一些真正有用的东西，至少要有一个返回首页 的链接。
**403 Forbidden**
如果你的网站上有某种权限控制，那么当用户访问未获授权内容时应当发送 403 代码。因此请确保当用户尝试访问未获授权内容时得到正确的反馈。
**410 Gone**
你知道 “404 Not Found” 有一个名叫 “410 Gone” 的兄弟吗？很少有人使用这个 代码。如果资源以前曾经存在过，但是现在已经被删除了，那么就应该使用 410 代码，而不是 404 。如果你不是在数据库中把文档永久地删除，而只是给文档打 了一个删除标记，那么请为用户考虑，应当使用 410 代码，并显示信息告知用户 要找的东西已经删除。
**500 Internal Server Error**
这个代码通常表示程序出错或服务器过载。强烈建议把这个页面弄得友好一点， 因为你的应用 迟早 会出现故障的（参见[掌握应用错误](http://dormousehole.readthedocs.org/en/latest/errorhandling.html#application-errors)）。

### 出错处理器

一个出错处理器是一个函数，就像一个视图函数一样。与视图函数不同之处在于出错处理器 在出现错误时被调用，且传递错误。错误大多数是一个 [HTTPException](http://werkzeug.pocoo.org/docs/exceptions/#werkzeug.exceptions.HTTPException) ，但是有一个例外：当出现内部服务器错误 时会把异常实例传递给出错处理器。

出错处理器使用 [errorhandler()](http://dormousehole.readthedocs.org/en/latest/api.html#flask.Flask.errorhandler) 装饰器注册，注册时应提供异常的 出代码。请记住， Flask 不会 为你设置出错代码，因此请确保在返回响应时，同时提供 HTTP 状态代码。

以下是一个处理 “404 Page Not Found” 异常的示例:

```
from flask import render_template

@app.errorhandler(404)
def page_not_found(e):
    return render_template('404.html'), 404
```

示例模板：

```
{% extends "layout.html" %}
{% block title %}Page Not Found{% endblock %}
{% block body %}
  <h1>Page Not Found</h1>
  <p>What you were looking for is just not there.
  <p><a href="{{ url_for('index') }}">go somewhere nice</a>
{% endblock %}
```

## 惰性载入视图

Flask 通常使用装饰器。装饰器简单易用，只要把 URL 放在相应的函数的前面就可以了。 但是这种方式有一个缺点：使用装饰器的代码必须预先导入，否则 Flask 就无法真正找到 你的函数。

当你必须快速导入应用时，这就会成为一个问题。在 Google App Engine 或其他系统中， 必须快速导入应用。因此，如果你的应用存在这个问题，那么必须使用集中 URL 映射。

[add_url_rule()](http://dormousehole.readthedocs.org/en/latest/api.html#flask.Flask.add_url_rule) 函数用于集中 URL 映射，与使用装饰器不同的是你 需要一个设置应用所有 URL 的专门文件。

### 转换为集中 URL 映射

假设有如下应用:

```
from flask import Flask
app = Flask(__name__)

@app.route('/')
def index():
    pass

@app.route('/user/<username>')
def user(username):
    pass

```

为了集中映射，我们创建一个不使用装饰器的文件（ views.py ）:

```
def index():
    pass

def user(username):
    pass
```

在另一个文件中集中映射函数与 URL:

```
from flask import Flask
from yourapplication import views
app = Flask(__name__)
app.add_url_rule('/', view_func=views.index)
app.add_url_rule('/user/<username>', view_func=views.user)
```

### 延迟载入

至此，我们只是把视图与路由分离，但是模块还是预先载入了。理想的方式是按需载入 视图。下面我们使用一个类似函数的辅助类来实现按需载入:

```
from werkzeug import import_string, cached_property

class LazyView(object):

    def __init__(self, import_name):
        self.__module__, self.__name__ = import_name.rsplit('.', 1)
        self.import_name = import_name

    @cached_property
    def view(self):
        return import_string(self.import_name)

    def __call__(self, *args, **kwargs):
        return self.view(*args, **kwargs)
```

上例中最重要的是正确设置 __module__ 和 __name__ ，它被用于在不提供 URL 规则 的情况下正确命名 URL 规则。

然后可以这样集中定义 URL 规则:

```
from flask import Flask
from yourapplication.helpers import LazyView
app = Flask(__name__)
app.add_url_rule('/',
                 view_func=LazyView('yourapplication.views.index'))
app.add_url_rule('/user/<username>',
                 view_func=LazyView('yourapplication.views.user'))
```

还可以进一步优化代码：写一个函数调用 [add_url_rule()](http://dormousehole.readthedocs.org/en/latest/api.html#flask.Flask.add_url_rule) ，加上 应用前缀和点符号:

```
def url(url_rule, import_name, **options):
    view = LazyView('yourapplication.' + import_name)
    app.add_url_rule(url_rule, view_func=view, **options)

url('/', 'views.index')
url('/user/<username>', 'views.user')
```

有一件事情要牢记：请求前和请求后处理器必须在第一个请求前导入。

其余的装饰器可以同样用上述方法改写。


## 在 Flask 中使用 MongoKit

现在使用文档型数据库来取代关系型数据库已越来越常见。本方案展示如何使用 MongoKit ,它是一个用于操作 MongoDB 的文档映射库。

本方案需要一个运行中的 MongoDB 服务器和已安装好的 MongoKit 库。

使用 MongoKit 有两种常用的方法，下面逐一说明：

### 声明

声明是 MongoKit 的缺省行为。这个思路来自于 Django 或 SQLAlchemy 的声明。

下面是一个示例 app.py 模块:

```
from flask import Flask
from mongokit import Connection, Document

# configuration
MONGODB_HOST = 'localhost'
MONGODB_PORT = 27017

# create the little application object
app = Flask(__name__)
app.config.from_object(__name__)

# connect to the database
connection = Connection(app.config['MONGODB_HOST'],
                        app.config['MONGODB_PORT'])
```

如果要定义模型，那么只要继承 MongoKit 的 Document 类就行了。如果你已经读过 SQLAlchemy 方案，那么可以会奇怪这里为什么没有使用会话，甚至没有定义一个 init_db 函数。一方面是因为 MongoKit 没有类似会话在东西。有时候这样会多写一点 代码，但会使它的速度更快。另一方面是因为 MongoDB 是无模式的。这就意味着可以在 插入数据的时候修改数据结构。 MongoKit 也是无模式的，但会执行一些验证，以确保 数据的完整性。

以下是一个示例文档（把示例内容也放入 app.py ）:

```
def max_length(length):
    def validate(value):
        if len(value) <= length:
            return True
        raise Exception('%s must be at most %s characters long' % length)
    return validate

class User(Document):
    structure = {
        'name': unicode,
        'email': unicode,
    }
    validators = {
        'name': max_length(50),
        'email': max_length(120)
    }
    use_dot_notation = True
    def __repr__(self):
        return '<User %r>' % (self.name)
```

# 在当前连接中注册用户文档
connection.register([User])
上例展示如何定义模式（命名结构）和字符串最大长度验证器。上例中还使用了一个 MongoKit 中特殊的 use_dot_notation 功能。缺省情况下， MongoKit 的运作方式和 Python 的字典类似。但是如果 use_dot_notation 设置为 True ，那么就可几乎像 其他 ORM 一样使用点符号来分隔属性。

可以像下面这样把条目插入数据库中：

```
>>> from yourapplication.database import connection
>>> from yourapplication.models import User
>>> collection = connection['test'].users
>>> user = collection.User()
>>> user['name'] = u'admin'
>>> user['email'] = u'admin@localhost'
>>> user.save()
```

注意， MongoKit 对于列类型的使用是比较严格的。对于 name 和 email 列，你都 不能使用 str 类型，应当使用 unicode 。

查询非常简单：

```
>>> list(collection.User.find())
[<User u'admin'>]
>>> collection.User.find_one({'name': u'admin'})
<User u'admin'>
```

### PyMongo 兼容层

如果你只需要使用 PyMongo ，也可以使用 MongoKit 。在这种方式下可以获得最佳的 性能。注意，以下示例中，没有 MongoKit 与 Flask 整合的内容，整合的方式参见上文:

```
from MongoKit import Connection

connection = Connection()
```

使用 insert 方法可以插入数据。但首先必须先得到一个连接。这个连接类似于 SQL 界 的表。

```
>>> collection = connection['test'].users
>>> user = {'name': u'admin', 'email': u'admin@localhost'}
>>> collection.insert(user)
```

MongoKit 会自动提交。

直接使用集合查询数据库：

```
>>> list(collection.find())
[{u'_id': ObjectId('4c271729e13823182f000000'), u'name': u'admin', u'email': u'admin@localhost'}]
>>> collection.find_one({'name': u'admin'})
{u'_id': ObjectId('4c271729e13823182f000000'), u'name': u'admin', u'email': u'admin@localhost'}
```

查询结果为类字典对象：

```
>>> r = collection.find_one({'name': u'admin'})
>>> r['email']
u'admin@localhost'
```

关于 MongoKit 的更多信息，请移步其[官方网站](https://github.com/namlook/mongokit) 。


## 添加一个页面图标

一个“页面图标”是浏览器在标签或书签中使用的图标，它可以给你的网站加上一个唯一 的标示，方便区别于其他网站。

那么如何给一个 Flask 应用添加一个页面图标呢？首先，显而易见的，需要一个图标。 图标应当是 16 X 16 像素的 ICO 格式文件。这不是规定的，但却是一个所有浏览器都 支持的事实上的标准。把 ICO 文件命名为 favicon.ico 并放入静态 文件目录 中。

现在我们要让浏览器能够找到你的图标，正确的做法是在你的 HTML 中添加一个链接。 示例：

```
<link rel="shortcut icon" href="{{ url_for('static', filename='favicon.ico') }}">
```

对于大多数浏览器来说，这样就完成任务了，但是一些老古董不支持这个标准。老的标准 是把名为“ favicon.ico ”的图标放在服务器的根目录下。如果你的应用不是挂接在域的 根目录下，那么你需要定义网页服务器在根目录下提供这个图标，否则就无计可施了。 如果你的应用位于根目录下，那么你可以简单地进行重定向:

```
app.add_url_rule('/favicon.ico',
                 redirect_to=url_for('static', filename='favicon.ico'))
```

如果想要保存额外的重定向请求，那么还可以使用 [send_from_directory()](http://dormousehole.readthedocs.org/en/latest/api.html#flask.send_from_directory) 函数来写一个视图:

```
import os
from flask import send_from_directory

@app.route('/favicon.ico')
def favicon():
    return send_from_directory(os.path.join(app.root_path, 'static'),
                               'favicon.ico', mimetype='image/vnd.microsoft.icon')
```

上例中的 MIME 类型可以省略，浏览器会自动猜测类型。但是我们在例子中明确定义了， 省去了额外的猜测，反正这个类型是不变的。

上例会通过你的应用来提供图标，如果可能的话，最好配置你的专用服务器来提供图标， 配置方法参见网页服务器的文档。

另见
Wikipedia 上的[页面图标](http://en.wikipedia.org/wiki/Favicon)词条

## 流内容

有时候你会需要把大量数据传送到客户端，不在内存中保存这些数据。当你想把运行中产生 的数据不经过文件系统，而是直接发送给客户端时，应当怎么做呢？

答案是使用生成器和直接响应。

### 基本用法

下面是一个在运行中产生大量 CSV 数据的基本视图函数。其技巧是调用一个内联函数生成 数据，把这个函数传递给一个响应对象:

```
from flask import Response

@app.route('/large.csv')
def generate_large_csv():
    def generate():
        for row in iter_all_rows():
            yield ','.join(row) + '\n'
    return Response(generate(), mimetype='text/csv')
```

每个 yield 表达式被直接传送给浏览器。注意，有一些 WSGI 中间件可能会打断流 内容，因此在使用分析器或者其他工具的调试环境中要小心一些。

### 模板中的流内容

Jinja2 模板引擎也支持分片渲染模板。这个功能不是直接被 Flask 支持的，因为它太 特殊了，但是你可以方便地自已来做:

```
from flask import Response

def stream_template(template_name, **context):
    app.update_template_context(context)
    t = app.jinja_env.get_template(template_name)
    rv = t.stream(context)
    rv.enable_buffering(5)
    return rv

@app.route('/my-large-page.html')
def render_large_template():
    rows = iter_all_rows()
    return Response(stream_template('the_template.html', rows=rows))
```

上例的技巧是从 Jinja2 环境中获得应用的模板对象，并调用 stream() 来代替 render() ，返回 一个流对象来代替一个字符串。由于我们绕过了 Flask 的模板渲染函数使用了模板对象 本身，因此我们需要调用 [update_template_context()](http://dormousehole.readthedocs.org/en/latest/api.html#flask.Flask.update_template_context) ，以确保 更新被渲染的内容。这样，模板遍历流内容。由于每次产生内容后，服务器都会把内容 发送给客户端，因此可能需要缓存来保存内容。我们使用了 rv.enable_buffering(size) 来进行缓存。 5 是一个比较明智的缺省值。

### 环境中的流内容

New in version 0.9.

注意，当你生成流内容时，请求环境已经在函数执行时消失了。 Flask 0.9 为你提供了 一点帮助，让你可以在生成器运行期间保持请求环境:

```
from flask import stream_with_context, request, Response

@app.route('/stream')
def streamed_response():
    def generate():
        yield 'Hello '
        yield request.args['name']
        yield '!'
    return Response(stream_with_context(generate()))
```

如果没有使用 [stream_with_context()](http://dormousehole.readthedocs.org/en/latest/api.html#flask.stream_with_context) 函数，那么就会引发 [RuntimeError](http://docs.python.org/dev/library/exceptions.html#RuntimeError) 错误。

## 延迟的请求回调

Flask 的设计思路之一是：响应对象创建后被传递给一串回调函数，这些回调函数可以修改 或替换响应对象。当请求处理开始的时候，响应对象还没有被创建。响应对象是由一个视图 函数或者系统中的其他组件按需创建的。

但是当响应对象还没有创建时，我们如何修改响应对象呢？比如在一个请求前函数中，我们 需要根据响应对象设置一个 cookie 。

通常我们选择避开这种情形。例如可以尝试把应用逻辑移动到请求后函数中。但是，有时候 这个方法让人不爽，或者让代码变得很丑陋。

变通的方法是把一堆回调函数贴到 g 对象上，并且在请求结束时调用这些 回调函数。这样你就可以在应用的任意地方延迟回调函数的执行。

### 装饰器

下面的装饰器是一个关键，它在 g 对象上注册一个函数列表:

```
from flask import g

def after_this_request(f):
    if not hasattr(g, 'after_request_callbacks'):
        g.after_request_callbacks = []
    g.after_request_callbacks.append(f)
    return f
```

### 调用延迟的回调函数

至此，通过使用 after_this_request 装饰器，使得函数在请求结束时可以被调用。现在 我们来实现这个调用过程。我们把这些函数注册为 after_request() 回调函数:

```
@app.after_request
def call_after_request_callbacks(response):
    for callback in getattr(g, 'after_request_callbacks', ()):
        callback(response)
    return response
```

### 一个实例

现在我们可以方便地随时随地为特定请求注册一个函数，让这个函数在请求结束时被调用。 例如，你可以在请求前函数中把用户的当前语言记录到 cookie 中:

```
from flask import request

@app.before_request
def detect_user_language():
    language = request.cookies.get('user_lang')
    if language is None:
        language = guess_language_from_request()
        @after_this_request
        def remember_language(response):
            response.set_cookie('user_lang', language)
    g.language = language
 Logo
Table Of Contents
```

## 添加 HTTP 方法重载

一些 HTTP 代理不支持所有 HTTP 方法或者不支持一些较新的 HTTP 方法（例如 PACTH ）。在这种情况下，可以通过使用完全相反的协议，用一种 HTTP 方法来“代理”另一种 HTTP 方法。

实现的思路是让客户端发送一个 HTTP POST 请求，并设置 X-HTTP-Method-Override 头部为需要的 HTTP 方法（例如 PATCH ）。

通过 HTTP 中间件可以方便的实现:

```
class HTTPMethodOverrideMiddleware(object):
    allowed_methods = frozenset([
        'GET',
        'HEAD',
        'POST',
        'DELETE',
        'PUT',
        'PATCH',
        'OPTIONS'
    ])
    bodyless_methods = frozenset(['GET', 'HEAD', 'OPTIONS', 'DELETE'])

    def __init__(self, app):
        self.app = app

    def __call__(self, environ, start_response):
        method = environ.get('HTTP_X_HTTP_METHOD_OVERRIDE', '').upper()
        if method in self.allowed_methods:
            method = method.encode('ascii', 'replace')
            environ['REQUEST_METHOD'] = method
        if method in self.bodyless_methods:
            environ['CONTENT_LENGTH'] = '0'
        return self.app(environ, start_response)
```

通过以下代码就可以与 Flask 一同工作了:

```
from flask import Flask

app = Flask(__name__)
app.wsgi_app = HTTPMethodOverrideMiddleware(app.wsgi_app)
```

## 请求内容校验

请求数据会由不同的代码来处理或者预处理。例如 JSON 数据和表单数据都来源于已经 读取并处理的请求对象，但是它们的处理代码是不同的。这样，当需要校验进来的请求 数据时就会遇到麻烦。因此，有时候就有必要使用一些 API 。

幸运的是可以通过包装输入流来方便地改变这种情况。

下面的例子演示在 WSGI 环境下读取和储存输入数据，得到数据的 SHA1 校验:

```
import hashlib

class ChecksumCalcStream(object):

    def __init__(self, stream):
        self._stream = stream
        self._hash = hashlib.sha1()

    def read(self, bytes):
        rv = self._stream.read(bytes)
        self._hash.update(rv)
        return rv

    def readline(self, size_hint):
        rv = self._stream.readline(size_hint)
        self._hash.update(rv)
        return rv

def generate_checksum(request):
    env = request.environ
    stream = ChecksumCalcStream(env['wsgi.input'])
    env['wsgi.input'] = stream
    return stream._hash
```

要使用上面的类，你只要在请求开始消耗数据之前钩接要计算的流就可以了。（按：小心 操作 request.form 或类似东西。例如 before_request_handlers 就应当小心不 要操作。）

用法示例:

```
@app.route('/special-api', methods=['POST'])
def special_api():
    hash = generate_checksum(request)
    # Accessing this parses the input stream
    files = request.files
    # At this point the hash is fully constructed.
    checksum = hash.hexdigest()
    return 'Hash was: %s' % checksum
```

## 基于 Celery 的后台任务

Celery 是一个 Python 编写的是一个异步任务队列/基于分布式消息传递的作业队列。 以前它有一个 Flask 的集成，但是从版本 3 开始，它进行了一些内部的重构，已经 不需要这个集成了。本文主要说明如何在 Flask 中正确使用 Celery 。本文假设你 已经阅读过了其官方文档中的 Celery 入门

### 安装 Celery

Celery 在 Python 包索引（ PyPI ）上榜上有名，因此可以使用 pip 或 easy_install 之类标准的 Python 工具来安装:

```
$ pip install celery
```

### 配置 Celery

你首先需要有一个 Celery 实例，这个实例称为 celery 应用。其地位就相当于 Flask 中 Flask 一样。这个实例被用作所有 Celery 相关事务的入口，例如创建 任务、管理工人等等。因此它必须可以被其他模块导入。

例如，你可以把它放在一个 tasks 模块中。这样不需要重新配置，你就可以使用 tasks 的子类，增加 Flask 应用环境的支持，并钩接 Flask 的配置。

只要如下这样就可以在 Falsk 中使用 Celery 了:

```
from celery import Celery

def make_celery(app):
    celery = Celery(app.import_name, broker=app.config['CELERY_BROKER_URL'])
    celery.conf.update(app.config)
    TaskBase = celery.Task
    class ContextTask(TaskBase):
        abstract = True
        def __call__(self, *args, **kwargs):
            with app.app_context():
                return TaskBase.__call__(self, *args, **kwargs)
    celery.Task = ContextTask
    return celery
```

这个函数创建了一个新的 Celery 对象，使用了应用配置中的 broker ，并从 Flask 配置 中升级了 Celery 的其余配置。然后创建了一个任务子类，在一个应用环境中包装了任务 执行。

### 最小的例子

基于上文，以下是一个在 Flask 中使用 Celery 的最小例子:

```
from flask import Flask

app = Flask(__name__)
app.config.update(
    CELERY_BROKER_URL='redis://localhost:6379',
    CELERY_RESULT_BACKEND='redis://localhost:6379'
)
celery = make_celery(app)


@celery.task()
def add_together(a, b):
    return a + b
```

这个任务现在可以在后台调用了：

```
>>> result = add_together.delay(23, 42)
>>> result.wait()
65
```

### 运行 Celery 工人

至此，如果你已经按上文一步一步执行，你会失望地发现你的 .wait() 不会真正 返回。这是因为你还没有运行 celery 。你可以这样以工人方式运行 celery:

```
$ celery -A your_application worker
```

把 your_application 字符串替换为你创建 celery 对像的应用包或模块。


*? Copyright 2013, Armin Ronacher. Created using [Sphinx](http://sphinx.pocoo.org/).*