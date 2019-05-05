.. meta::
   :description: Python REST API Framework to effortlessly build and deploy full featured, highly customizable RESTful Web Services.

.. title:: Python REST API Framework: Eve, the Simple Way to REST.

Eve. 通向 REST 的简单方式
===========================

版本 |version|.

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

-----

Eve 是一个为人类设计的 :doc:`open source <license>` Python REST API 框架。它允许不费吹灰之力构建和部署高度定制化，全特性的 RESTful Web 服务。

Eve 由 Flask_ 和 Cerberus_ 支持，原生支持 MongoDB_ 数据存储。对 SQL, Elasticsearch 和 Neo4js 后端的支持是由社区 extensions_ 提供的。

代码库在 Python 2.7, 3.4+, 和 PyPy 中被完全测试过。

.. 注意:: *高度* 推荐使用 **Python 3** 而不是 Python 2。如果今天你发现你自己 *还* 在生产中使用 Python 2，请考虑升级你的应用程序和基础设施。

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

为 Eve 提供资金
-----------
Eve REST 框架是一个开源的合作资助项目。如果你在做生意，并在一个可以创造利润的产品中使用 Eve，那么赞助 Eve 开发是很有商业思维的：它确保你产品所依赖的项目保持在健康和活跃维护状态。如果 Eve 对你的工作和私人项目提供过帮助，也欢迎个人用户作出长期的承诺或一次性的捐赠。每一个注册将产生一个有特殊意义的强大作用力，使 Eve 成为可能。

要加入支持者行列，瞧瞧 `Eve campaign on Patreon`_。

.. _demo:

现场演示
---------
看看 `live demo`_。如果使用一个浏览器，你会得到 XML 返回。对于浏览器中的 JSON，你可能想安装 Postman_ 或类似的扩展，然后设置 ``Accept`` 请求头到 ``application/json``。如果你是一个 CLI 用户 (你应该是)，``curl`` 会是你的朋友。`source code`_ 会向你展示通过 Eve 运行一个 API 是多么简单。你也会发现针对所有通用使用场景 (GET, POST, PATCH, DELETE 和更多) 的 `usage examples`_。也有一个简单的 `client app`_ 可用。

.. toctree::
    :hidden:

    foreword
    rest_api_for_humans
    install
    quickstart
    features
    config
    validation
    authentication
    funding
    tutorials/index
    snippets/index
    extensions
    contributing
    support
    updates
    authors
    license
    changelog

.. 注意::
   此文档正在持续开发中。请参考侧边栏的链接获取更多信息。


.. _python-eve.org: http://python-eve.org
.. _`Eve Demo instructions`: http://github.com/pyeve/eve-demo#readme
.. _`live demo`: https://eve-demo.herokuapp.com/people
.. _`source code`: https://github.com/pyeve/eve-demo
.. _`usage examples`: https://github.com/pyeve/eve-demo#readme
.. _`client app`: https://github.com/pyeve/eve-demo-client
.. _Postman: https://www.getpostman.com
.. _Flask: http://flask.pocoo.org/
.. _eve-sqlalchemy: https://github.com/RedTurtle/eve-sqlalchemy
.. _MongoDB: https://mongodb.org
.. _Redis: http://redis.io
.. _Cerberus: http://python-cerberus.org
.. _events: https://github.com/pyeve/events
.. _extensions: http://python-eve.org/extensions
.. _`Eve campaign on Patreon`: https://www.patreon.com/nicolaiarocci
