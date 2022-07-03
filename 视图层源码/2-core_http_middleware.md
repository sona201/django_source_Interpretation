## MIDDLEWARE/get_response

5.2 django处理http请求源码

```python
    def __call__(self, environ, start_response):
        set_script_prefix(get_script_name(environ))
        signals.request_started.send(sender=self.__class__, environ=environ)
        # WSGIRequest对象 -> HttpRequest类
        request = self.request_class(environ)
        # response是什么
        response = self.get_response(request)


    def get_response(self, request):
        """Return an HttpResponse object for the given HttpRequest."""
        # Setup default url resolver for this thread
        set_urlconf(settings.ROOT_URLCONF)
        response = self._middleware_chain(request)
        response._resource_closers.append(request.close)
        if response.status_code >= 400:
            log_response(
                "%s: %s",
                response.reason_phrase,
                request.path,
                response=response,
                request=request,
            )
        return response
```

研究 django.http.request.py
研究 django.http.response.py
self._middleware_chain(request)
他的上一步是 self.load_middleware()
在实例化 WSGIHandler 的时候就有 self.load_middleware()

```python
class WSGIHandler(base.BaseHandler):
    request_class = WSGIRequest

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.load_middleware()

    def __call__(self, environ, start_response):
        set_script_prefix(get_script_name(environ))
        signals.request_started.send(sender=self.__class__, environ=environ)
        # WSGIRequest对象 -> HttpRequest类
        request = self.request_class(environ)
        # response是什么
        response = self.get_response(request)
```

研究url 分发
```python
set_urlconf(settings.ROOT_URLCONF)
```


大的议题
1. request response
2. callback, callback_args, callback_kwargs = self.resolve_request(request)
3. 视图层的问题, as_view


```python
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]
```


```python
class WSGIHandler(base.BaseHandler):
    request_class = WSGIRequest

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        # 初始化中间件进行层层包裹
        self.load_middleware()

# 将 settings.MIDDLEWARE
#            reversed
#            import_string


# handler <-> self._get_response 方法
# handler = warp(XFrameOptionsMiddleware(self._get_response))
# handler = warp(MessageMiddleware(XFrameOptionsMiddleware(self._get_response)))
# handler() -> 1. MessageMiddleware.process_request()
#              2. XFrameOptionsMiddleware.process_request()
#              3. self._get_response
#              4. XFrameOptionsMiddleware.process_response()
#              5. MessageMiddleware.process_response()
#
#  client  ->  1  ->  2  ->  3  ->  request
#                                    view  ->  self._get_response
#  client  <-  1  <-  2  <-  3  <-  response
```


```
SecurityMiddleware(SessionMiddleware(CommonMiddleware(CsrfViewMiddleware(AuthenticationMiddleware(MessageMiddleware(XFrameOptionsMiddleware(self._get_response)))))))
                   SessionMiddleware(CommonMiddleware(CsrfViewMiddleware(AuthenticationMiddleware(MessageMiddleware(XFrameOptionsMiddleware(self._get_response))))))
                                     CommonMiddleware(CsrfViewMiddleware(AuthenticationMiddleware(MessageMiddleware(XFrameOptionsMiddleware(self._get_response)))))
                                                      CsrfViewMiddleware(AuthenticationMiddleware(MessageMiddleware(XFrameOptionsMiddleware(self._get_response))))
                                                                         AuthenticationMiddleware(MessageMiddleware(XFrameOptionsMiddleware(self._get_response)))
                                                                                                  MessageMiddleware(XFrameOptionsMiddleware(self._get_response))
                                                                                                                    XFrameOptionsMiddleware(self._get_response)
                                                                                                                                            self._get_response
```

变成一个链式调用, 请求进来时, 最外层先执行

请求进来WSGIHandler() 实例化后, 调用 __call__

```python
class WSGIHandler(base.BaseHandler):
    request_class = WSGIRequest

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.load_middleware()

    def __call__(self, environ, start_response):
        set_script_prefix(get_script_name(environ))
        signals.request_started.send(sender=self.__class__, environ=environ)
        # WSGIRequest对象 -> HttpRequest类
        request = self.request_class(environ)
        # response是什么
        response = self.get_response(request)
```

然后调用

```python
response = self._middleware_chain(request)
```

会执行调用链
SecurityMiddleware(SessionMiddleware(CommonMiddleware(CsrfViewMiddleware(AuthenticationMiddleware(MessageMiddleware(XFrameOptionsMiddleware(self._get_response)))))))

因为继承了 MiddlewareMixin, 调用触发 __call__

```python
class MiddlewareMixin:
    def __call__(self, request):
        # Exit out to async mode, if needed
        if asyncio.iscoroutinefunction(self.get_response):
            return self.__acall__(request)
        response = None
        if hasattr(self, "process_request"):
            response = self.process_request(request)
        # 当response 为空时,会执行self.get_response(request), 这个又会调用下一个函数的self.get_response(request),是一个递归
        response = response or self.get_response(request)
        if hasattr(self, "process_response"):
            response = self.process_response(request, response)
        return response
```

会递归执行
```python
response = response or self.get_response(request)
```
然后才执行
```python
self.process_response(request, response)
```

```python
class BaseHandler:
    _view_middleware = None
    _template_response_middleware = None
    _exception_middleware = None
    _middleware_chain = None

    
    def _get_response(self, request):
        """
        Resolve and call the view, then apply view, exception, and
        template_response middleware. This method is everything that happens
        inside the request/response middleware.
        """
        response = None
        # 核心内容
        callback, callback_args, callback_kwargs = self.resolve_request(request)

        # Apply view middleware
        for middleware_method in self._view_middleware:
            response = middleware_method(
                request, callback, callback_args, callback_kwargs
            )
            if response:
                break

        if response is None:
            wrapped_callback = self.make_view_atomic(callback)
            # If it is an asynchronous view, run it in a subthread.
            if asyncio.iscoroutinefunction(wrapped_callback):
                wrapped_callback = async_to_sync(wrapped_callback)
            try:
                # 自定义的视图函数处理    WSGIRequest对象
                response = wrapped_callback(request, *callback_args, **callback_kwargs)
            except Exception as e:
                response = self.process_exception_by_middleware(e, request)
                if response is None:
                    raise
```