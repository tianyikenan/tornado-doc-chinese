﻿运行和部署
=====================

自从 Tornado 提供了自己的 HTTP 服务器以后, 运行和部署与其它的 Python
web 框架有些不一样. 你需要为你的应用程序编写一个 ``main()`` 函数来启动
服务器, 而不是配置一个 WSGI 容器:

.. testcode::

    def main():
        app = make_app()
        app.listen(8888)
        IOLoop.current().start()

    if __name__ == '__main__':
        main()

.. testoutput::
   :hide:

配置你的操作系统或是进程管理器来开启服务器运行这个程序.
请注意增加打开文件描述符的个数是十分重要的 (来避免 "Too many open files"-的错误).
如果要增加这个限制 ( 假设要把它设置为50000 ) 你可以使用 ulimit 命令,
修改 /etc/security/limits.conf 或者在你的 supervisord 中配置 ``minfds`` .

进程和端口
~~~~~~~~~~~~~~~~~~~

由于 Python GIL (解释器全局锁), 为了更好利用多核 CPU 的性能, 将 Python
运行在多进程模式下就十分的重要. 通常最好是为每一个核心运行一个进程.

Tornado 包含一个内建的多进程模式来一次性启动. 
这需要你在 main 函数做一些微小的改变:

.. testcode::

    def main():
        app = make_app()
        server = tornado.httpserver.HTTPServer(app)
        server.bind(8888)
        server.start(0)  # forks one process per cpu
        IOLoop.current().start()

.. testoutput::
   :hide:

这是启动多线程模式的最简单方式而且它们会共享同一个端口, 虽然它也有一些限制.
首先, 每一个子进程将会有一个自己的 IOLoop, 所以在 fork 之前你需要确保不能接触
全局的 IOLoop 实例 (甚至是间接的). 第二, 在这种模型中很难做到零宕机更新. 
最后, 因为所有的进程都共享一个端口, 想要监控他们变得更难了.

对于更加复杂的部署, 建议启动独立的进程, 让每一个进程都拥有一个不同的端口.
`supervisord <http://www.supervisord.org>`_ 的 "process groups" 特性是解
决整个问题的好方法之一. 当每一个进程使用不同的端口时需要一个外部的负载均衡器,
例如 HAProxy 或者 nginx, 它们将会把内部的每个端口给外部呈现为同一个地址.


运行在负载均衡器之后
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

当运行在像 nginx 这种负载均衡器后方, 我们建议给 `.HTTPServer` 构造器中
传 ``xheaders=True`` 参数. 这将会告诉 Tornado 在头部使用 ``X-Real-IP``
来获取用户的 IP 地址而不是认为所有流量都来自于负载均衡器的 IP 地址.

这时一份原始的 nginx 配置文件, 它和我们 FriendFeed 中用到的在结构上很相似.
假设 nginx 和 Tornado 服务器运行在同一个机器上, 四个 Tornado 服务分别运行在
端口 8000-8003上::

    user nginx;
    worker_processes 1;

    error_log /var/log/nginx/error.log;
    pid /var/run/nginx.pid;

    events {
        worker_connections 1024;
        use epoll;
    }

    http {
        # Enumerate all the Tornado servers here
        upstream frontends {
            server 127.0.0.1:8000;
            server 127.0.0.1:8001;
            server 127.0.0.1:8002;
            server 127.0.0.1:8003;
        }

        include /etc/nginx/mime.types;
        default_type application/octet-stream;

        access_log /var/log/nginx/access.log;

        keepalive_timeout 65;
        proxy_read_timeout 200;
        sendfile on;
        tcp_nopush on;
        tcp_nodelay on;
        gzip on;
        gzip_min_length 1000;
        gzip_proxied any;
        gzip_types text/plain text/html text/css text/xml
                   application/x-javascript application/xml
                   application/atom+xml text/javascript;

        # Only retry if there was a communication error, not a timeout
        # on the Tornado server (to avoid propagating "queries of death"
        # to all frontends)
        proxy_next_upstream error;

        server {
            listen 80;

            # Allow file uploads
            client_max_body_size 50M;

            location ^~ /static/ {
                root /var/www;
                if ($query_string) {
                    expires max;
                }
            }
            location = /favicon.ico {
                rewrite (.*) /static/favicon.ico;
            }
            location = /robots.txt {
                rewrite (.*) /static/robots.txt;
            }

            location / {
                proxy_pass_header Server;
                proxy_set_header Host $http_host;
                proxy_redirect off;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Scheme $scheme;
                proxy_pass http://frontends;
            }
        }
    }

静态文件和文件缓存
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

你可以在你的 Tornado 应用程序中指明 ``static_path`` 设置::

    settings = {
        "static_path": os.path.join(os.path.dirname(__file__), "static"),
        "cookie_secret": "__TODO:_GENERATE_YOUR_OWN_RANDOM_VALUE_HERE__",
        "login_url": "/login",
        "xsrf_cookies": True,
    }
    application = tornado.web.Application([
        (r"/", MainHandler),
        (r"/login", LoginHandler),
        (r"/(apple-touch-icon\.png)", tornado.web.StaticFileHandler,
         dict(path=settings['static_path'])),
    ], **settings)

设置会自动将所有以 ``/static/`` 开头的请求作为静态件夹来处理.
例如, ``http://localhost:8888/static/foo.png`` 将会把 ``foo.png`` 文件
从制定的文件夹中以静态文件方式处理. 我们也自动将  ``/robots.txt`` 和 ``/favicon.ico``
设定为静态文件 (即便它们不是以 ``/static/`` 开头).

在以上的设置中, 我们明确的设置了将静态文件 ``apple-touch-icon.png``
在 `.StaticFileHandler` 根目录下, 虽然物理上在静态文件夹中. (在正则
表达式的匹配组中必须告诉 `.StaticFileHandler` 请求的文件名; 
调用处理方法时会将匹配组传递过去.) 你可以通过同样的方法从网站的根目录
提供 ``sitemap.xml`` 文件. 当然, 你也可以通过使用适当的 HTML ``<link />``
标签来避免伪造根目录的 ``apple-touch-icon.png`` 文件.

为了提高性能, 可以讲一些静态文件缓存起来, 这样浏览器就不会发送一些
可能在渲染页面时阻塞的不必要的 ``If-Modified-Since`` 或者 ``Etag`` 请求.
Tornado 支持使用 *静态内容版本(static content versioning)* 来解决这些问题.

为了使用这个特性, 在你的模版中使用 `~.RequestHandler.static_url` 
而不是在你的 HTML 文件中输入静态文件的 URL 地址来确定静态文件::

    <html>
       <head>
          <title>FriendFeed - {{ _("Home") }}</title>
       </head>
       <body>
         <div><img src="{{ static_url("images/logo.png") }}"/></div>
       </body>
     </html>

``static_url()`` 方法将会把你的相对路径翻译成像
``/static/images/logo.png?v=aae54`` 一样的绝对路径. ``v`` 参数时 ``logo.png``
内容的散列, 它的存在使得 Tornado 向浏览器发送一个缓存头部, 这将会使得缓存
被无限期使用.

``v`` 参数的值取决于文件的内容, 如果你更新了文件而且重启了服务器,
``v`` 的值将会被更新而且重新发送, 所以用户的浏览器将会自动的获取
新的文件. 如果文件没有改变, 浏览器将会继续使用原来缓存的文件而不
是再一次向服务器请求, 可以显著提高渲染性能.

在生产环境中, 你可能会想使用一些更好的静态文件服务器, 例如 `nginx <http://nginx.net/>`_.
你可以在大多数 web 服务器上通过识别版本标签来使用 ``static_url()`` .
这里是我们在 FriendFeed 中使用的相关部分的 nginx 配置::

    location /static/ {
        root /var/friendfeed/static;
        if ($query_string) {
            expires max;
        }
     }

.. _debug-mode:

Debug 模式 和 自动重新加载
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

如果你在 ``Application`` 构造器中传递了 ``debug=True`` 参数,
你的应用将会运行在调试/开发模式下. 在这种模式下, 一些为了更方便开发的
特性将会被启用 (其中每一项都可以作为独立的标记使用; 如果两个都被设置了,
独立的标记将具有更高的优先级):

* ``autoreload=True``: 应用程序将会监控源代码文件, 在改变时重新加载
  这样减少了在开发过程中手动重启服务器的需要. 然而, 一些确定的错误 (例如,
  在 import 时现语法错误) 还是会让服务器停止运行的, 而且这无法恢复.
* ``compiled_template_cache=False``: 模版将不会被缓存.
* ``static_hash_cache=False``: 静态文件散列 (通过使用
  ``static_url`` 函数) 将不会被缓存
* ``serve_traceback=True``: 当在 `.RequestHandler`
  中的异常没有被捕获时, 会产生一个错误页.

自动重新加载模式与 `.HTTPServer` 的多进程模型不能兼容.
你不能在自动重新加载模式下给  `HTTPServer.start <.TCPServer.start>` 1 (或者调用
`tornado.process.fork_processes`) 以外的参数

调试模式的自动加载特性可以以  `tornado.autoreload` 运行在一个独立的模块中.
这两个可以结合使用, 在语法错误时可以提供额外的稳定性:
设置 ``autoreload=True`` 可以在运行时检测改变,
以 ``python -m tornado.autoreload myserver.py`` 启动将会在启动时捕捉语法错误和其它错误.

重新加载将会丢失 Python 解释器的命令行参数 (例如 ``-u``)
因为它通过使用  `sys.executable` 和 `sys.argv` 重新运行了一遍 Python 程序.
此外, 改变这些变脸将会使重载错误.

在一些平台 (包括 Windows 和 Mac OSX 10.6 之前), 进程不能在原基础上更新,
所以当代码改变被检测到时, 旧的服务器关闭, 新的服务器开启.
这些操作将会使得某些集成开发环境失效.


WSGI 和 Google App Engine
~~~~~~~~~~~~~~~~~~~~~~~~~~

Tornado 在 WSGI 容器以外通常是独立运行的.
然而, 在某些环境中 (像 Google App Engine),
只允许 WSGI, 其它方式将不能在它们的服务器上运行.
这种情况下 Tornado 支持一种被限制的模式, 这种模式下不支持一步操作,
在 WSGI-only 环境下 Tornado 只能使用其功能的一个子集.
这些在 WSGI 下不能使用的的功能包括协程, ``@asynchronous`` 修饰符,
`.AsyncHTTPClient`,  ``auth`` 模块, 和 WebSockets.

你可以使用  `tornado.wsgi.WSGIAdapter` 将
一个 Tornado `.Application` 转换成 WSGI 应用程序.
在这个例子中, 让你的 WSGI 容器找到``application`` 对象:

.. testcode::

    import tornado.web
    import tornado.wsgi

    class MainHandler(tornado.web.RequestHandler):
        def get(self):
            self.write("Hello, world")

    tornado_app = tornado.web.Application([
        (r"/", MainHandler),
    ])
    application = tornado.wsgi.WSGIAdapter(tornado_app)

.. testoutput::
   :hide:

详见 `appengine example application
<https://github.com/tornadoweb/tornado/tree/stable/demos/appengine>`_ 来查看 Tornado
在构建 AppEngine 应用程序上的所有特性.
