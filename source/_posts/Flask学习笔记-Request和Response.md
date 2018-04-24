---
title: 'Flask学习笔记:Request和Response'
tags:
  - python
  - Flask
---

## Request:
###### HTTP编码格式
从 request 对象中获取POST请求参数
```python
@app.route('/app/login',methods = ['POST'])
def app_login():
    if request.method == 'POST':
        username = request.form.get('username')
        password = request.form.get('password')
```
<!-- more --> 
###### JSON编码格式
```python
@app.route('/app/login',methods = ['POST'])
def app_login():
    if request.method == 'POST':
        username = request.get_json().get('username')
        password = request.get_json().get('password')
```

## Response
###### JSON编码格式
```python
  # return json.jsonify(dict/list/tuple)
```


## 另外,通过自定义响应类, 还可以将一些繁琐的任务简单化
此处参考文章:  
##### [如何自定义Flask中的响应类（译文）](https://segmentfault.com/a/1190000003987049) 以及 [Flask设置返回json格式数据](https://www.polarxiong.com/archives/Flask%E8%AE%BE%E7%BD%AE%E8%BF%94%E5%9B%9Ejson%E6%A0%BC%E5%BC%8F.html)  



自定义 response 类
```python
from flask import Flask, Response

class MyResponse(Response): # 定义一个继承自Response的对象
    pass

app = Flask(__name__)
app.response_class = MyResponse # 告诉 app 你自定义的 response 类型

```

如果直接在返回响应体中传入 dict/list 等内容, Response 是无法解析的
```python
    return Response(dict(abc='abc', def='def'))
    # 以上这样返回响应体是会报错的:
    TypeError: 'dict' object is not callable
```

利用自定义的 response 类, 实现自动返回 JSON 响应体  
Response()传入的参数中, 所有不能处理的数据，都由 force_type() 方法尝试处理后，再决定报错或通过。
```python
class MyResponse(Response):
    @classmethod  # 重写 force_type() 类方法, 拦截并判断报错的参数类型
    def force_type(cls, rv, environ=None):
        if isinstance(rv, dict):
            rv = jsonify(rv)
        return super(MyResponse, cls).force_type(rv, environ)
```
