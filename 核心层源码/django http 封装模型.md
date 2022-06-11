django 封装模型采用 wsgiref 模块
```
from wsgiref.simple_server import make_server


def routers():
    # URLConf 配置
    urlpatterns = (
        ('/book', foo),
        ('/web', bar),
    )
    return urlpatterns


def foo(x):
    return [b'<h1>Hello, book</h1>']


def bar(x):
    return [b'<h1>Hello, web</h1>']


def application(environ, start_response):
    start_response('200 OK', [('Content-Type', 'text/html')])

    urlpatterns = routers()
    path = environ['PATH_INFO']  # http://127.0.0.1:8000/book
    func = None
    for item in urlpatterns:
        if item[0] == path:
            func = item[1]
            break
    if func:
        return func(environ)
    else:
        return ['<h1>404</h1>'.encode('utf-8')


httpd = make_server('127.0.0.1', 8000, application)

print('Serving HTTP on port 8000...')
httpd.serve_forever()
```

这里启动了一个server, 做了一个路由分发, 函数处理
