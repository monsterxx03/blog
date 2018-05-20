---
title: "Migrate to Sqlalchemy"
date: 2018-05-18T16:42:31+08:00
draft: true
categories:
  - tech
tags:
  - python
  - sqlalchemy
---

最近把公司 db 层的封装代码基于 sqlalchemy 重写了一下, 记录一下.

原来的 db 层代码历史非常古老(10年以上...), 最早写代码的人早就不在了, 问题很多:

- 完全没有单元测试.
- 暴露出的接口命名很混乱, 多数是为了兼容一些历史问题.
- 里面带一套 client 端 db sharding 的逻辑, 但在新项目里完全用不到, 还导致无法做 join, 无法子查询, 很不方便.
- 老的 db 代码没有 model 层, 和 db migration 通过一种很 trick 的方式绑定在一起实现的, 导致开发时候对着代码完全无法知道数据库表结构，只能直接看数据库.

重写时候要考虑到的:

- 现有业务代码基于老 db 代码已经写了很多了, 重写不现实, 迁移到 sqlalchemy 需要封装一套完全兼容的 api, 一些没用到的混乱的老 api 趁机清理. 
- db migration 需要重新实现, 只用 alembic 不能满足需求, 后详. 
- 重写完的代码保证高测试覆盖率.

## ORM  or Core

sqlalchemy 分成两部分, 底层的 sqlalchemy core 是一套 sql 语法生成器, 通过重载 python 的 magic function 实现用比较漂亮的 python 语法来构造 sql.
ORM 部分是基于 core 做的面向对象封装.

声明 model 的时候也有两种写法, core 是直接实例化 Table 对象, orm 用的是 declarative 语法(declare 一个 class), 例子:

    # Table syntax
    User = Table("user", metadata, Column('name', Integer, primary_key=True))
    # ORM declarative syntax
    class User(Base):
        __tablename__ = 'user'
        name = Column('name', Integer, primary_key=True)

重写时候最后选择了, 用 declarative 语法声明 model, 但用 core 语法封装 sql.

为什么用 core 不用 orm:

- 原先的业务代码相当过程式, 就算底层用了 orm 封装, 向上传递的对象也得转成 dict, 没啥用
- ORM 默认在 query 的时候会加载所有 column, 虽然可以通过 defer 实现懒加载, 但这个只能定义在 model 上, 而需求是要能让上层代码控制加载哪些 column
- core 的性能比 orm style 要好不少
- 需要用到一些 MySQL 独有的语法, core 可以, orm 就不行, 比如 `insert ...on duplicate update`, `SQL_CALC_FOUND_ROWS`, `insert ignore` ...
- core 语法生成的 query statement 可以很方便的转成 raw sql, orm 的一些操作会转成什么样的 sql 很不直观.

为什么定义 model 不 定义 table

- 通过 ORM 的 model 可以通过 `User.__table__` 获得等价的 table 对象, 直接定义 model, 以后要使用 ORM 的话可以直接用.
- 所有 table 都有一些相同的 column, model 定义可以通过继承来复用.
- model 可以定义一些 derived column.

已存在的表比较多, 用 https://pypi.org/project/sqlacodegen/ 这个工具自动化生成 model 代码后手工修改.

## Simulate old api

这倒不难, 原来的 api 暴露出一组类似下面的函数:
 
- select(where, cols=None, offset=0, limit=None, order_by=None) 
- update_many(where, update_dict)
- insert_one(row)
- ...

select 里的 where 是个 dict, 会被转成 sql 的 where 部分:

    {'id': 1}  ->  `id=1`
    {'id:' {'>': 1}}  -> `id > 1`
    {'id': [1, 2]}  -> `id in (1, 2)`
    {'first_name': 'tester', 'last_name': 'x'} -> `first_name="tester" and last_name="x"`

原先的逻辑中查询条件全是 `and` 没有 `or`, 通过 sqlalchemy 实现很容易, 转换函数类似:

    def build_where(sa_model_cls, stmt, where):
        for key, value in where.items():
            col = getattr(sa_model_cls, key)
            if isinstance(value, (list, tuple)):
                stmt = stmt.where(col.in_(value))  # 这里如果传入的是 empty list, sqlalchemy 会生成一句 `col != col`
            elif isinstance(value, dict):
                for op, val in value.items():
                    # value is {operation: value} pair, operation can be any valid mysql operations: >,<,>=, <=, !=, like...
                    stmt = stmt.where(col.op(op)(val))
            else:
                stmt = stmt.where(col == value)   # if value is None, will be converted to `is NULL`
        return stmt

模拟之后老的 api 向上是透明的，调用老 api 的人完全不会 sqlalchemy 也没问题, 新的业务需要使用 join 或 orm 的话，直接拿 model class 自己去处理就行.

## DB migration based on alembic 

alembic 是 sqlalchemy 官方的 migration 工具, 但有点简陋, 不会在数据库里记录历次 migration 的内容, 只能看代码, 数据库里只有一张 `alembic_version` 表表示数据库当前处在的版本.

对 db migration 的需求:

1. migration 版本号使用递增的数字，而不是 alembic 默认的 hash id.
2. 通过修改 model 能生成对应的 schema ddl sql, 最后到 live 上运行的一定要是 raw sql, 不是 alembic 的 python 脚本.
3. 在数据库里用一张 db_revisions 表记录每次运行 migration 的时候在什么时候对什么表做了什么操作.
4. 可以重复运行一个 revision, 并检测其中哪张表已经运行过 migration 了, 直接跳过, 只 apply 没运行过的表. 原因是一个项目经常有多个分支在同时开发, 大家可能同时在修改数据库，生成了 revision 005, 但一般是对不同表的修改, 比如 branchA  修改了 User, branchB 修改了 Topic, branchA 先 merge 回 master 上线了, 所以线上已经是 revision 005, 希望 branchB 的人 merge master代码的时候没有冲突，上线 branchB 时候再运行一次 revision 005, 这回只 apply Topic 表的修改. 如果两分支同时修改了一张表，那merge 时候生成的 sql 会冲突, 人工解决冲突, 这种情况比较少.

实现:

raw sql 可以通过 alembic 的 offline 模式生成, 但有几个问题:

- 生成的语句不支持 online ddl, `ALGORITHM=INPLACE, LOCK=None` 语法.
- add column 默认总是加在最后面, developer 希望保持 column 在 class 定义时候顺序.
