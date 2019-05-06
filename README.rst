Eve
====
.. image:: https://img.shields.io/pypi/v/eve.svg?style=flat-square
    :target: https://pypi.org/project/eve

.. image:: https://img.shields.io/travis/pyeve/eve.svg?branch=master&style=flat-square
    :target: https://travis-ci.org/pyeve/eve

.. image:: https://img.shields.io/pypi/pyversions/eve.svg?style=flat-square
    :target: https://pypi.org/project/eve

.. image:: https://img.shields.io/badge/license-BSD-blue.svg?style=flat-square
    :target: https://en.wikipedia.org/wiki/BSD_License

.. image:: https://img.shields.io/badge/code%20style-black-000000.svg
    :target: https://github.com/ambv/black

Eve 是一个为人类设计的开源 Python REST API 框架。它可以让你不费吹灰之力构建和部署高定制化，全特性的 RESTful Web 服务。Eve 提供对 MongoDB 的原生支持，并通过社区扩展支持 SQL 后端。

Eve 是简单的
-------------
.. code-block:: python

    from eve import Eve

    app = Eve()
    app.run()

现在 API 已经激活，随时准备被调用:

.. code-block:: console

    $ curl -i http://example.com/people
    HTTP/1.1 200 OK

要让你的 API 上线，你只需要一个数据库，一个配置文件(默认 ``settings.py``) 和一个启动脚本。总体上你会发现，配置和微调你的 API 是个非常简单的过程。

`去 Eve 网站 <http://python-eve.org/> 看看`_

特性
--------
* Emphasis on REST
* 全部 CRUD 操作
* 可定制的资源终结点
* 可定制的，多数据项终结点
* 过滤和排序
* 分页
* HATEOAS
* JSON 和 XML 渲染
* 条件化请求
* 数据完整性和并发控制
* 批量插入
* 数据验证
* 可扩展的数据验证
* 资源级缓存控制
* API 版本控制
* 文档版本控制
* 身份验证
* CORS 跨域资源共享
* JSONP
* 默认只读
* 默认值
* 预定义的数据库过滤器
* 投影
* 内嵌的资源序列化
* 事件钩子
* 限速
* 自定义 ID 字段
* 文件存储
* GeoJSON
* 内部资源
* 加强版日志记录
* 操作日志
* MongoDB 聚合框架
* MongoDB 和 SQL 支持
* 由 Flask 提供支持

资金提供
-------
Eve REST 框架是一个开源的合作资助项目。如果你在做生意，并在一个可以创造利润的产品中使用 Eve，那么赞助 Eve 开发是很有商业思维的：它确保你产品所依赖的项目保持在健康和活跃维护状态。如果 Eve 对你的工作和私人项目提供过帮助，也欢迎个人用户作出长期的承诺或一次性的捐赠。

每一个注册将产生一个有特殊意义的强大作用力，使 Eve 成为可能。要了解更多，请瞧瞧我们的 `funding page`_。

许可证
-------
Eve 是一个 `Nicola Iarocci`_ 开源项目，基于 `BSD 许可证 <https://github.com/pyeve/eve/blob/master/LICENSE>`_ 分发.

.. _`Nicola Iarocci`: http://nicolaiarocci.com
.. _`funding page`: http://python-eve.org/funding
