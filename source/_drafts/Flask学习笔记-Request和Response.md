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

###### JSON编码格式
```python
@app.route('/app/login',methods = ['POST'])
def app_login():
    if request.method == 'POST':
        username = request.get_json().get('username')
        password = request.get_json().get('password')
```

## Response
