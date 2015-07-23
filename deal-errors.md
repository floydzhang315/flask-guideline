# 排除应用错误

[掌握应用错误](http://dormousehole.readthedocs.org/en/latest/errorhandling.html#application-errors) 一文所讲的是如何为生产应用设置日志和出错通知。本文要 讲的是部署中配置调试的要点和如何使用全功能的 Python 调试器深挖错误。

## 有疑问时，请手动运行

在生产环境中，配置应用时出错？如果你可以通过 shell 来访问主机，那么请首先在部署 环境中验证是否可以通过 shell 手动运行你的应用。请确保验证时使用的帐户与配置的 相同，这样可以排除用户权限引发的错误。你可以在你的生产服务器上，使用 Flask 内建 的开发服务器，并且设置 debug=True ，这样有助于找到配置问题。但是，请**只能在可控的情况下临时这样做** ，绝不能在生产时使用 debug=True 。

## 使用调试器

为了更深入的挖掘错误，追踪代码的执行， Flask 提供一个开箱即用的调试器（参见[调试模式](http://dormousehole.readthedocs.org/en/latest/quickstart.html#debug-mode) ）。如果你需要使用其他 Python 调试器，请注意调试器之间的干扰 问题。在使用你自己的调试器前要做一些参数调整：

* debug - 是否开启调试模式并捕捉异常
* use_debugger - 是否使用 Flask 内建的调试器
* use_reloader - 出现异常后是否重载或者派生进程
* debug 必须设置为 True （即必须捕获异常），另两个随便。

如果你正在使用 Aptana 或 Eclipse 排错，那么 use_debugger 和 use_reloader 都必须设置为 False 。

一个有用的配置模式如下（当然要根据你的应用调整缩进）:

```
FLASK:
    DEBUG: True
    DEBUG_WITH_APTANA: True
```

然后，在应用入口（ main.py ），修改如下:

```
if __name__ == "__main__":
    # 为了让 aptana 可以接收到错误，设置 use_debugger=False
    app = create_app(config="config.yaml")

    if app.debug: use_debugger = True
    try:
        # 如果使用其他调试器，应当关闭 Flask 的调试器。
        use_debugger = not(app.config.get('DEBUG_WITH_APTANA'))
    except:
        pass
    app.run(use_debugger=use_debugger, debug=app.debug,
            use_reloader=use_debugger, host='0.0.0.0')
```

*© Copyright 2013, Armin Ronacher. Created using [Sphinx](http://sphinx.pocoo.org/).*