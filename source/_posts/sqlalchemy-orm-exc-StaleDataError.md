---
layout: layout
title: sqlalchemy.orm.exc.StaleDataError
date: 2019-03-21 13:14:11
tags:
---
今天在对数据库做数据删除操作时遇到了这么个错误:  
`sqlalchemy.orm.exc.StaleDataError:DELETE statement on table 'xxx' expected to delete x row(s); Only x 0 were matched`  

数据结构如下:  
```
A, B, C 三张表 + B_C 中间表
A.Bs (one to many)
A.Cs (one to many)
B.C (many to many)
C.B (many to many)
```
<!-- more -->
B 对 C 的引用是这么写的:  
```
cs = db.relationship('C', secondary=B_C)
```
另外也写了 C 对 B 的引用:
```
bs = db.relationship('B', secondary=B_C)
```
我删除的数据是 A, 而 A 对 B,C 都是 级联删除关系.  

抛出错误后, 错误最后一行发生在这里:  
```
  File "/Users/siu/.virtualenvs/project_name/lib/python3.7/site-packages/sqlalchemy/orm/dependency.py", line 1200, in _run_crud
    result.rowcount,
```

进去看了一下, 打断点, 发现这个抛出错误所在的 `_run_crud` 方法跑了两遍, 其中 `result.rowcount` 第一次是有值, 而且值是准确的, 第二次是0
```python
    def _run_crud(
        self, uowcommit, secondary_insert, secondary_update, secondary_delete
    ):
        connection = uowcommit.transaction.connection(self.mapper)

        if secondary_delete:
            associationrow = secondary_delete[0]
            statement = self.secondary.delete(
                sql.and_(
                    *[
                        c == sql.bindparam(c.key, type_=c.type)
                        for c in self.secondary.c
                        if c.key in associationrow
                    ]
                )
            )
            result = connection.execute(statement, secondary_delete)

            if (
                result.supports_sane_multi_rowcount()
            ) and result.rowcount != len(secondary_delete):
                raise exc.StaleDataError(
                    "DELETE statement on table '%s' expected to delete "
                    "%d row(s); Only %d were matched."
                    % (
                        self.secondary.description,
                        len(secondary_delete),
                        result.rowcount,
                    )
                )
```

我猜想是不是因为 同时在 B/C 中写了对 C/B 的引用, 于是, 改了一下   
```
# 注释其中一个
# cs = db.relationship('C', secondary=B_C)
bs = db.relationship('B', secondary=B_C, backref=db.backref('cs'))
```

于是, 顺利删除了 A.  
___
  

### 另外也记录一下:
#### 在声明 many to many / one to many / one to one 字段时, 赋值 lazy 为 `'dynamic'`, 如:  
```python3
bs = db.relationship('B', secondary=B_C, backref=db.backref('cs'), lazy='dynamic')
```
在获取 `bs = c.bs` 时, `bs` 不是一个 B 的集合(数组), 而是一个 `Query` 对象.
#### 同时, 也可以为被 backref 的字段添加动态引用属性:
```python3
bs = db.relationship('B', secondary=B_C, backref=db.backref('cs', lazy='dynamic'), lazy='dynamic')
```
这样 `cs = b.cs`, `cs` 也是一个 `Query` 对象, 而不是 C 的集合.
如要获取到 C 的集合, 就要通过查询来获取 `cs = b.cs.all()` 

推荐一篇关于sqlalchemy数据关系的教程:  
[SQLAlchemy ORM教程之三：Relationship](https://www.jianshu.com/p/9771b0a3e589)
