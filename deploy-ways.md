# 部署方式

Flask 应用可以采用多种方式部署。在开发时，你可以使用内置的服务器，但是在生产环境 下你就应当选择功能完整的服务器。下面为你提供几个可用的选择。

除了下面提到的服务器之外，如果你使用了其他的 WSGI 服务器，那么请阅读其文档中与 使用 WSGI 应用相关的部分。因为 Flask 应用对象的实质就是一个 WSGI 应用。

如果需要快速体验，请参阅《快速上手》中的[部署到一个网络服务器](http://dormousehole.readthedocs.org/en/latest/quickstart.html#quickstart-deployment) 。

## mod_wsgi (Apache)

如果你正在使用 [Apache](http://httpd.apache.org/) 网络服务器，那么建议使用 [mod_wsgi](http://code.google.com/p/modwsgi/) 。

#### 小心

请务必把 app.run() 放在 if __name__ == '__main__': 内部或者放在单独 的文件中，这样可以保证它不会被调用。因为，每调用一次就会开启一个本地 WSGI 服务器。当我们使用 mod_wsgi 部署应用时，不需要使用本地服务器。

### 安装 mod_wsgi

可以使用包管理器或编译的方式安装 mod_wsgi 。在 UNIX 系统中如何使用源代码安装 请阅读 [mod_wsgi 安装介绍](http://code.google.com/p/modwsgi/wiki/QuickInstallationGuide) 。

如果你使用的是 Ubuntu/Debian ，那么可以使用如下命令安装：

```
# apt-get install libapache2-mod-wsgi
```

在 FreeBSD 系统中，可以通过编译 www/mod_wsgi port 或使用 pkg_add 来安装 mod_wsgi ：

```
# pkg_add -r mod_wsgi
```

如果你使用 pkgsrc ，那么可以通过编译 www/ap2-wsgi 包来安装 mod_wsgi 。

如果你遇到子进程段错误的话，不要理它，重启服务器就可以了。

### 创建一个 .wsgi 文件

为了运行应用，你需要一个 yourapplication.wsgi 文件。这个文件包含 mod_wsgi 开始时需要运行的代码，通过代码可以获得应用对象。文件中的 application 对象就 是以后要使用的应用。

对于大多数应用来说，文件包含以下内容就可以了:

```
from yourapplication import app as application
```

如果你的应用没有创建函数，只是一个独立的实例，那么可以直接把实例导入为 application 。

把文件放在一个以后可以找得到的地方（例如 /var/www/yourapplication ），并确保 yourapplication 和所有需要使用的库都位于 pythonpath 中。如果你不想在整个系统 中安装，建议使用 [virtual python](http://pypi.python.org/pypi/virtualenv) 实例。请记住，最好把应用安装到虚拟环境中。 有一个可选项是在 .wsgi 文件中，在导入前加入路径:

```
import sys
sys.path.insert(0, '/path/to/the/application')
```

### 配置 Apache

最后一件事是为你的应用创建一个 Apache 配置文件。基于安全原因，在下例中我们告诉 mod_wsgi 使用另外一个用户运行应用：

```
<VirtualHost *>
    ServerName example.com

    WSGIDaemonProcess yourapplication user=user1 group=group1 threads=5
    WSGIScriptAlias / /var/www/yourapplication/yourapplication.wsgi

    <Directory /var/www/yourapplication>
        WSGIProcessGroup yourapplication
        WSGIApplicationGroup %{GLOBAL}
        Order deny,allow
        Allow from all
    </Directory>
</VirtualHost>
```

注意： WSGIDaemonProcess 在 Windows 中不会被执行， 使用上面的配置 Apache 会拒绝 运行。在 Windows 系统下，请使用下面内容：

```
<VirtualHost *>
        ServerName example.com
        WSGIScriptAlias / C:\yourdir\yourapp.wsgi
        <Directory C:\yourdir>
                Order deny,allow
                Allow from all
        </Directory>
</VirtualHost>
```

更多信息参见 [mod_wsgi 维基](http://code.google.com/p/modwsgi/wiki/) 。

### 故障排除

如果你的应用无法运行，请按以下指导排除故障：

**问题**： 应用无法运行，出错记录显示 SystemExit ignored
应用文件中有 app.run() 调用，但没有放在 if __name__ == '__main__': 块内。要么把这个调用放入块内，要么把它放在一个单独的 run.py 文件中。
**问题**： 权限错误
有可以是因为使用了错误的用户运行应用。请检查用户及其所在的组 （ WSGIDaemonProcess 的 user 和 group 参数）是否有权限访问应用 文件夹。
**问题**： 打印时应用歇菜
请记住 mod_wsgi 不允许使用 [sys.stdout](http://docs.python.org/dev/library/sys.html#sys.stdout) 和 [sys.stderr](http://docs.python.org/dev/library/sys.html#sys.stderr) 。把 WSGIRestrictStdout 设置为 off 可以去掉这个保护：

```
WSGIRestrictStdout Off
```

或者你可以在 .wsgi 文件中把标准输出替换为其他的流:

```
import sys
sys.stdout = sys.stderr
```

**问题**： 访问资源时遇到 IO 错误
你的应用可能是一个独立的 .py 文件，且你把它符号连接到了 site-packages 文件夹。这样是不对的，你应当要么把文件夹放到 pythonpath 中，要么把你的应用 转换为一个包。

产生这种错误的原因是对于非安装包来说，模块的文件名用于定位资源，如果使用 符号连接的话就会定位到错误的文件名。

### 支持自动重载

为了辅助部署工具，你可以激活自动重载。这样，一旦 .wsgi 文件有所变动， mod_wsgi 就会自动重新转入所有守护进程。

在 Directory 一节中加入以下指令就可以实现自动重载：

```
WSGIScriptReloading On
```

### 使用虚拟环境

使用虚拟环境的优点是不必全局安装应用所需要的依赖，这样我们就可以更好地按照自己 的需要进行控制。如果要在虚拟环境下使用 mod_wsgi ，那么我们要对 .wsgi 略作 改变。

在你的 .wsgi 文件顶部加入下列内容:

```
activate_this = '/path/to/env/bin/activate_this.py'
execfile(activate_this, dict(__file__=activate_this))
```

以上代码会根据虚拟环境的设置载入相关路径。请记住路径必须是绝对路径。



## 独立 WSGI 容器

有一些用 Python 写的流行服务器可以容纳 WSGI 应用，提供 HTTP 服务。这些服务器是 独立运行的，你可以把代理从你的网络服务器指向它们。如果遇到问题，请阅读[代理设置](http://dormousehole.readthedocs.org/en/latest/deploying/wsgi-standalone.html#deploying-proxy-setups)一节。

### Gunicorn

[Gunicorn](http://gunicorn.org/) ‘Green Unicorn’ 是一个 UNIX 下的 WSGI HTTP 服务器，它是一个 移植自 Ruby 的 Unicorn 项目的 pre-fork worker 模型。它既支持 [eventlet](http://eventlet.net/) ， 也支持 [greenlet](http://codespeak.net/py/0.9.2/greenlet.html) 。在 Gunicorn 上运行 Flask 应用非常简单:

```
gunicorn myproject:app
```

Gunicorn 提供许多命令行参数，可以使用 gunicorn -h 来获得帮助。下面的例子 使用 4 worker 进程（ -w 4 ）来运行 Flask 应用，绑定到 localhost 的 4000 端口（ -b 127.0.0.1:4000 ）:

```
gunicorn -w 4 -b 127.0.0.1:4000 myproject:app
```

### Tornado

[Tornado](http://www.tornadoweb.org/) 是构建 [FriendFeed](http://friendfeed.com/) 的服务器和工具的开源版本，具有良好的伸缩性，非 阻塞性。得益于其非阻塞的方式和对 epoll 的运用，它可以同步处理数以千计的独立 连接，因此 Tornado 是实时 Web 服务的一个理想框架。用它来服务 Flask 是小事一桩:

```
from tornado.wsgi import WSGIContainer
from tornado.httpserver import HTTPServer
from toranado.ioloop import IOLoop
from yourapplication import app

http_server = HTTPServer(WSGIContainer(app))
http_server.listen(5000)
IOLoop.instance().start()
```

### Gevent

[Gevent](http://www.gevent.org/) 是一个 Python 并发网络库，它使用了基于 libevent 事件循环的 greenlet 来提供一个高级同步 API:

```
from gevent.wsgi import WSGIServer
from yourapplication import app

http_server = WSGIServer(('', 5000), app)
http_server.serve_forever()

```

### Twisted Web

[Twisted Web](https://twistedmatrix.com/trac/wiki/TwistedWeb) 是一个 [Twisted](https://twistedmatrix.com/) 自带的网络服务器，是一个成熟的、异步的、事件 驱动的网络库。 Twisted Web 带有一个标准的 WSGI 容器，该容器可以使用 twistd 工具运行命令行来控制:

```
twistd web --wsgi myproject.app
```

这个命令会运行一个名为 app 的 Flask 应用，其模块名为 myproject 。

与 twistd 工具一样， Twisted Web 支持许多标记和选项。更多信息参见 twistd -h 和 twistd web -h 。例如下面命令在前台运行一个来自 myproject 的应用， 端口为 8080:

```
twistd -n web --port 8080 --wsgi myproject.app
```

### 代理设置

如果你要在一个 HTTP 代理后面在上述服务器上运行应用，那么必须重写一些头部才行。 通常在 WSGI 环境中经常会出现问题的有两个变量： REMOTE_ADDR 和 HTTP_HOST 。 你可以通过设置你的 httpd 来传递这些头部，或者在中间件中修正这些问题。 Werkzeug 带有一个修复工具可以用于常用的设置，但是你可能需要为特定的设置编写你 自己的 WSGI 中间件。

下面是一个简单的 nginx 配置，代理目标是 localhost 8000 端口提供的服务，设置了 适当的头部：

```
server {
    listen 80;

    server_name _;

    access_log  /var/log/nginx/access.log;
    error_log  /var/log/nginx/error.log;

    location / {
        proxy_pass         http://127.0.0.1:8000/;
        proxy_redirect     off;

        proxy_set_header   Host             $host;
        proxy_set_header   X-Real-IP        $remote_addr;
        proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
    }
}
```

如果你的 httpd 无法提供这些头部，那么最常用的设置是调用 X-Forwarded-Host 定义 的主机和 X-Forwarded-For 定义的远程地址:

```
from werkzeug.contrib.fixers import ProxyFix
app.wsgi_app = ProxyFix(app.wsgi_app)
```

#### 头部可信问题

请注意，在非代理情况下使用这个中间件是有安全问题的，因为它会盲目信任恶意 客户端发来的头部。

如果你要根据另一个头部来重写一个头部，那么可以像下例一样使用修复工具:

```
class CustomProxyFix(object):

    def __init__(self, app):
        self.app = app

    def __call__(self, environ, start_response):
        host = environ.get('HTTP_X_FHOST', '')
        if host:
            environ['HTTP_HOST'] = host
        return self.app(environ, start_response)

app.wsgi_app = Cus

tomProxyFix(app.wsgi_app)
```

## uWSGI

uWSGI 也是部署 Flask 的途径之一,类似的部署途径还有 nginx 、 lighttpd 和 cherokee 。其他部署途径的信息参见 FastCGI 和 独立 WSGI 容器 。使用 uWSGI 协议来部署 WSGI 应用的先决条件是 需要一个 uWSGI 服务器。 uWSGI 既是一个协议也是一个服务器。如果作为一个服务器， 它可以服务于 uWSGI 、 FastCGI 和 HTTP 协议。

最流行的 uWSGI 服务器是 uwsgi ，本文将使用它来举例，请先安装它。

#### 小心

请务必把 app.run() 放在 if __name__ == '__main__': 内部或者放在单独 的文件中，这样可以保证它不会被调用。因为，每调用一次就会开启一个本地 WSGI 服务器。当我们使用 uWSGI 部署应用时，不需要使用本地服务器。

### 使用 uwsgi 启动你的应用

uwsgi 是基于 python 模块中的 WSGI 调用的。假设 Flask 应用名称为 myapp.py ， 可以使用以下命令：

```
$ uwsgi -s /tmp/uwsgi.sock --module myapp --callable app
```

或者这个命令也行：

```
$ uwsgi -s /tmp/uwsgi.sock -w myapp:app
```

### 配置 nginx

一个 nginx 的基本 uWSGI 配置如下:

```
location = /yourapplication { rewrite ^ /yourapplication/; }
location /yourapplication { try_files $uri @yourapplication; }
location @yourapplication {
  include uwsgi_params;
  uwsgi_param SCRIPT_NAME /yourapplication;
  uwsgi_modifier1 30;
  uwsgi_pass unix:/tmp/uwsgi.sock;
}
```

这个配置把应用绑定到 /yourapplication 。如果你想要在根 URL 下运行应用非常 简单，因为你不必指出 WSGI PATH_INFO 或让 uwsgi 修改器来使用它:

```
location / { try_files $uri @yourapplication; }
location @yourapplication {
    include uwsgi_params;
    uwsgi_pass unix:/tmp/uwsgi.sock;
}
```


## FastCGI

FastCGI 是部署 Flask 的途径之一,类似的部署途径还有 nginx 、 lighttpd 和 cherokee 。其他部署途径的信息参见 uWSGI 和 独立 WSGI 容器 。本文讲述的是使用 FastCGI 部署，因此先决条件 是要有一个 FastCGI 服务器。 flup 最流行的 FastCGI 服务器之一，我们将会在本文 中使用它。在阅读下文之前先安装好 flup 。

####小心
请务必把 app.run() 放在 if __name__ == '__main__': 内部或者放在单独 的文件中，这样可以保证它不会被调用。因为，每调用一次就会开启一个本地 WSGI 服务器。当我们使用 FastCGI 部署应用时，不需要使用本地服务器。

### 创建一个 .fcgi 文件

首先你必须创建 FastCGI 服务器配置文件，我们把它命名为 yourapplication.fcgi:

```
#!/usr/bin/python
from flup.server.fcgi import WSGIServer
from yourapplication import app

if __name__ == '__main__':
    WSGIServer(app).run()
```

如果使用的是 Apache ，那么使用了这个文件之后就可以正常工作了。但是如果使用的是 nginx 或老版本的 lighttpd ，那么需要显式地把接口传递给 FastCGI 服务器，即把接口 的路径传递给 WSGIServer:

```
WSGIServer(application, bindAddress='/path/to/fcgi.sock').run()
```

这个路径必须与服务器配置中定义的路径一致。

把这个 yourapplication.fcgi 文件放在一个以后可以找得到的地方，最好是 /var/www/yourapplication 或类似的地方。

为了让服务器可以执行这个文件，请给文件加上执行位，确保这个文件可以执行：

```
# chmod +x /var/www/yourapplication/yourapplication.fcgi
```

### 配置 Apache

上面的例子对于基本的 Apache 部署已经够用了，但是你的 .fcgi 文件会暴露在应用的 URL 中，比如 example.com/yourapplication.fcgi/news/ 。有多种方法可以避免出现这 中情况。一个较好的方法是使用 ScriptAlias 配置指令:

```
<VirtualHost *>
    ServerName example.com
    ScriptAlias / /path/to/yourapplication.fcgi/
</VirtualHost>
```

如果你无法设置 ScriptAlias ，比如你使用的是一个共享的网络主机，那么你可以使用 WSGI 中间件把 yourapplication.fcgi 从 URL 中删除。你可以这样设置 .htaccess:

```
<IfModule mod_fcgid.c>
   AddHandler fcgid-script .fcgi
   <Files ~ (\.fcgi)>
       SetHandler fcgid-script
       Options +FollowSymLinks +ExecCGI
   </Files>
</IfModule>

<IfModule mod_rewrite.c>
   Options +FollowSymlinks
   RewriteEngine On
   RewriteBase /
   RewriteCond %{REQUEST_FILENAME} !-f
   RewriteRule ^(.*)$ yourapplication.fcgi/$1 [QSA,L]
</IfModule>
```

设置 yourapplication.fcgi:

```
#!/usr/bin/python
#: optional path to your local python site-packages folder
import sys
sys.path.insert(0, '<your_local_path>/lib/python2.6/site-packages')

from flup.server.fcgi import WSGIServer
from yourapplication import app

class ScriptNameStripper(object):
   def __init__(self, app):
       self.app = app

   def __call__(self, environ, start_response):
       environ['SCRIPT_NAME'] = ''
       return self.app(environ, start_response)

app = ScriptNameStripper(app)

if __name__ == '__main__':
    WSGIServer(app).run()
```

### 配置 lighttpd

一个 lighttpd 的基本 FastCGI 配置如下:

```
fastcgi.server = ("/yourapplication.fcgi" =>
    ((
        "socket" => "/tmp/yourapplication-fcgi.sock",
        "bin-path" => "/var/www/yourapplication/yourapplication.fcgi",
        "check-local" => "disable",
        "max-procs" => 1
    ))
)

alias.url = (
    "/static/" => "/path/to/your/static"
)

url.rewrite-once = (
    "^(/static($|/.*))$" => "$1",
    "^(/.*)$" => "/yourapplication.fcgi$1"
```

请记住启用 FastCGI 、 alias 和 rewrite 模块。以上配置把应用绑定到 /yourapplication 。如果你想要让应用在根 URL 下运行，那么必须使用 LighttpdCGIRootFix 中间件来解决一个 lighttpd 缺陷。

请确保只有应用在根 URL 下运行时才使用上述中间件。更多信息请阅读 FastCGI 和 Python （注意，已经不再需要把一个接口显式传递给 run() 了）。

### 配置 nginx

在 nginx 上安装 FastCGI 应用有一些特殊，因为缺省情况下不传递 FastCGI 参数。

一个 nginx 的基本 FastCGI 配置如下:

```
location = /yourapplication { rewrite ^ /yourapplication/ last; }
location /yourapplication { try_files $uri @yourapplication; }
location @yourapplication {
    include fastcgi_params;
    fastcgi_split_path_info ^(/yourapplication)(.*)$;
    fastcgi_param PATH_INFO $fastcgi_path_info;
    fastcgi_param SCRIPT_NAME $fastcgi_script_name;
    fastcgi_pass unix:/tmp/yourapplication-fcgi.sock;
}
```

这个配置把应用绑定到 /yourapplication 。如果你想要在根 URL 下运行应用非常 简单，因为你不必指出如何计算出 PATH_INFO 和 SCRIPT_NAME:

```
location / { try_files $uri @yourapplication; }
location @yourapplication {
    include fastcgi_params;
    fastcgi_param PATH_INFO $fastcgi_script_name;
    fastcgi_param SCRIPT_NAME "";
    fastcgi_pass unix:/tmp/yourapplication-fcgi.sock;
}
```

### 运行 FastCGI 进程

Nginx 和其他服务器不会载入 FastCGI 应用，你必须自己载入。 [Supervisor 可以管理 FastCGI 进程](http://supervisord.org/configuration.html#fcgi-program-x-section-settings)。 在启动时你可以使用其他 FastCGI 进程管理器或写一个脚本来运行 .fcgi 文件，例如 使用一个 SysV init.d 脚本。如果是临时使用，你可以在一个 GNU screen 中运行 .fcgi 脚本。运行细节参见 man screen ，同时请注意这是一个手动启动方法， 不会在系统重启时自动启动:

```
$ screen
$ /var/www/yourapplication/yourapplication.fcgi
```

### 调试

在大多数服务器上， FastCGI 部署难以调试。通常服务器日志只会告诉你类似 “ premature end of headers ”的内容。为了调试应用，查找出错的原因，你必须切换 到正确的用户并手动执行应用。

下例假设你的应用是 application.fcgi ，且你的网络服务用户为 www-data:

```
$ su www-data
$ cd /var/www/yourapplication
$ python application.fcgi
Traceback (most recent call last):
  File "yourapplication.fcgi", line 4, in <module>
ImportError: No module named yourapplication
```

上面的出错信息表示 “yourapplication” 不在 python 路径中。原因可能有：

* 使用了相对路径。在当前工作路径下路径出错。
* 当前网络服务器设置未正确设置环境变量。
* 使用了不同的 python 解释器。

## CGI

如果其他的部署方式都不管用，那么就只能使用 CGI 了。 CGI 适应于所有主流服务器， 但是其性能稍弱。

这也是在 Google 的 App Engine 使用 Flask 应用的方法，其执行方式类似于 CGI 环境。

#### 小心

请务必把 app.run() 放在 if __name__ == '__main__': 内部或者放在单独 的文件中，这样可以保证它不会被调用。因为，每调用一次就会开启一个本地 WSGI 服务器。当我们使用 CGI 或 App Engine 部署应用时，不需要使用本地服务器。

在使用 CGI 时，你还必须确保代码中不包含任何 print 语句，或者 sys.stdout 被重载，不会写入 HTTP 响应中。

### 创建一个 .cgi 文件

首先，你需要创建 CGI 应用文件。我们把它命名为 yourapplication.cgi:

```
#!/usr/bin/python
from wsgiref.handlers import CGIHandler
from yourapplication import app

CGIHandler().run(app)
```

### 服务器设置

设置服务器通常有两种方法。一种是把 .cgi 复制为 cgi-bin （并且使用 mod_rewrite 或其他类似东西来改写 URL ）；另一种是把服务器直接指向文件。

例如，如果使用 Apache ，那么可以把如下内容放入配置中：

```
ScriptAlias /app /path/to/the/application.cgi
```

在共享的网络服务器上，你可能无法变动 Apache 配置。在这种情况下，你可以使用你 的公共目录中的 .htaccess 文件。但是 ScriptAlias 指令会失效：

```
RewriteEngine On
RewriteCond %{REQUEST_FILENAME} !-f # Don't interfere with static files
RewriteRule ^(.*)$ /path/to/the/application.cgi/$1 [L]
```

更多信息参见你所使用的服务器的文档。

*© Copyright 2013, Armin Ronacher. Created using [Sphinx](http://sphinx.pocoo.org/).*