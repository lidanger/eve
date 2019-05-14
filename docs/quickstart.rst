.. _quickstart:

快速入门
==========

渴望入门？让此页面对 Eve 作一个首次介绍。

先决条件
-------------
- 你已经安装了 Eve。如果没有，掉头转到 :ref:`install` 节。
- 安装_ 了 MongoDB。
- MongoDB 的一个实例正在 运行_ 。

一个最小的应用程序
---------------------

一个最小的 Eve 应用程序看起来像这个样子::

    from eve import Eve
    app = Eve()

    if __name__ == '__main__':
        app.run()

保存为 run.py 就行了。下一步，创建一个新文本文件，加入如下内容:

::

    DOMAIN = {'people': {}}

在存储 run.py 的同一文件夹下保存为 settings.py。这是 Eve 的配置文件，一个标准 Python 模块，它告诉 Eve 你的 API 由单个可访问的资源构成，``people``。

现在，准备启动你的 API。

.. code-block:: console

    $ python run.py
     * Running on http://127.0.0.1:5000/

现在，你可以使用 API 了:

.. code-block:: console

    $ curl -i http://127.0.0.1:5000
    HTTP/1.0 200 OK
    Content-Type: application/json
    Content-Length: 82
    Server: Eve/0.0.5-dev Werkzeug/0.8.3 Python/2.7.3
    Date: Wed, 27 Mar 2013 16:06:44 GMT

恭喜，你的 GET 请求得到一个不错的响应返回。让我们看看这个载体（payload）:

::

    {
      "_links": {
        "child": [
          {
            "href": "people",
            "title": "people"
          }
        ]
      }
    }

API 入口点符合 :ref:`hateoas_feature` 规范并提供关于 API 资源可用性的信息。在我们的例子中，只有一个子资源可用，那就是 ``people``。

现在试试请求 ``people``:

.. code-block:: console

    $ curl http://127.0.0.1:5000/people

::

    {
      "_items": [],
      "_links": {
        "self": {
          "href": "people",
          "title": "people"
        },
        "parent": {
          "href": "/",
          "title": "home"
        }
      }
    }

这一次我们也得到一个 ``_items`` 列表。``_links`` 是对获取到的资源的关联，这样你得到一个父资源 (主页) 和资源自己的链接。如果你从 pymongo 得到一个超时错误，请检查先决条件时候满足。很可能发生的情况是 ``mongod`` 服务器进程未运行。

默认情况下，Eve API 是只读的:

.. code-block:: console

    $ curl -X DELETE http://127.0.0.1:5000/people
    <!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
    <title>405 Method Not Allowed</title>
    <h1>Method Not Allowed</h1>
    <p>The method DELETE is not allowed for the requested URL.</p>

由于我们没有在 settings.py 中提供任何数据库详情，Eve 没有关于 ``people`` 集合 (甚至很可能不存在) 真正内容的线索，只能天衣无缝地提供一个空资源，因为我们不想让 API 用户失望。

数据库插曲（Interlude）
------------------
让我们通过添加如下行到 settings.py 连接到一个数据库:

::

    # 让我们仅仅使用本地 mongod 实例。按需要的编辑。

    # 请注意，MONGO_HOST 就 MONGO_PORT 可以很好的省略，因为它们已经默认为本地 'mongod' 实例。
    MONGO_HOST = 'localhost'
    MONGO_PORT = 27017

    # 如果你的数据库未启用认证，跳过这些。但是实际上应该是需要的。
    MONGO_USERNAME = '<your username>'
    MONGO_PASSWORD = '<your password>'
    MONGO_AUTH_SOURCE = 'admin'  # 在 --auth 模式启用的情况下需要

    MONGO_DBNAME = 'apitest'

由于 MongoDB 的 *laziness*，我们并不需要真正创建数据库集合。实际上我们甚至不需要创建数据库：对一个空/不存在的数据库的 GET 请求会被精确的对待 (``200 OK`` 对一个空集合); DELETE/PATCH/PUT 会收到一个合适的响应 (``404 Not Found``)，而 POST 请求会根据需要创建数据库和集合。
但是，这样一个自动管理的数据库性能很差，因为它缺少索引和各种优化。

一个更复杂的应用程序
--------------------------
目前为止，我们的 API 是只读的了。让我们启用完整的 CRUD 操作:

::

    # 启用对资源/集合的读 (GET)，插入 (POST) 和 DELETE (如果你忽略这一行，API 将默认为 ['GET'] 并对终结点提供只读访问)。
    RESOURCE_METHODS = ['GET', 'POST', 'DELETE']

    # 启用对单个数据项的读 (GET)，编辑 (PATCH)，替代 (PUT) 和删除 (默认为只读的数据项访问)。
    ITEM_METHODS = ['GET', 'PATCH', 'PUT', 'DELETE']

``RESOURCE_METHODS`` 列出了资源终结点 (``/people``) 允许的方法，而 ``ITEM_METHODS`` 列出了数据项终结点 (``/people/<ObjectId>``) 启用的方法。这两个设置都是全局作用域，适用于所有的终结点。然后，你可以在单个终结点级别启用或禁用 HTTP 方法，就像我们很快会看到的样子。

由于我们正在启用编辑，我们也项启用恰当的数据验证。
让我们为我们的 ``people`` 资源定义一个模式。

::

    schema = {
        # 模式定义，基于 Cerberus 语法。找 Cerberus 项目 (https://github.com/pyeve/cerberus) 获取详细信息。
        'firstname': {
            'type': 'string',
            'minlength': 1,
            'maxlength': 10,
        },
        'lastname': {
            'type': 'string',
            'minlength': 1,
            'maxlength': 15,
            'required': True,
            # 这才叫硬性约束! 由于演示 'lastname' 是一个 API 入口点，所以我么需要它是唯一的。
            'unique': True,
        },
        # 'role' 是一个列表，只能包含 'allowed' 中的值。
        'role': {
            'type': 'list',
            'allowed': ["author", "contributor", "copy"],
        },
        # 一个内嵌的 'strongly-typed' 字典。
        'location': {
            'type': 'dict',
            'schema': {
                'address': {'type': 'string'},
                'city': {'type': 'string'}
            },
        },
        'born': {
            'type': 'datetime',
        },
    }

更多关于验证的信息，请参考 :ref:`validation`。

现在，比如说，我们想进一步自定义 ``people`` 终结点。我们想：

- 设置项标题为 ``person``
- 在 ``/people/<lastname>`` 添加另一个 :ref:`custom item endpoint <custom_item_endpoints>`
- 重写默认的 :ref:`cache control directives <cache_control>`
- 对 ``/people`` 终结点禁用 DELETE (全局启用)

这里是我们更新后的 settings.py 文件中完整的 ``people`` 定义看起来的样子:

::

    people = {
        # 'title' 标签用于数据项链接中。默认为资源标题减去最后的复数形式 's' (大多数情况下工作良好，除了 'people')
        'item_title': 'person',

        # 默认标准数据项入口点被定义为 '/people/<ObjectId>'。我们让它原封不动，再启用一个另外的只读入口点。这样适用者也可以通过 '/people/<lastname>' 执行 GET 请求。
        'additional_lookup': {
            'url': 'regex("[\w]+")',
            'field': 'lastname'
        },

        # 我们选择重写对这个资源的全局缓存控制指令。
        'cache_control': 'max-age=10,must-revalidate',
        'cache_expires': 10,

        # 大多数全局设置都可以再资源即便被重写
        'resource_methods': ['GET', 'POST'],

        'schema': schema
    }

最后我们更新我们的域定义：

::

    DOMAIN = {
        'people': people,
    }

保存 settings.py，启动 run.py。现在我们可以在 ``people`` 终结点插入文档:

.. code-block:: console

    $ curl -d '[{"firstname": "barack", "lastname": "obama"}, {"firstname": "mitt", "lastname": "romney"}]' -H 'Content-Type: application/json'  http://127.0.0.1:5000/people
    HTTP/1.0 201 OK

我们也可以更细和删除数据项 (但不能删除整个资源，因为我们禁用了它)。我们也可以对新的 ``lastname`` 终结点执行 GET 请求:

.. code-block:: console

    $ curl -i http://127.0.0.1:5000/people/obama
    HTTP/1.0 200 OK
    Etag: 28995829ee85d69c4c18d597a0f68ae606a266cc
    Last-Modified: Wed, 21 Nov 2012 16:04:56 GMT
    Cache-Control: 'max-age=10,must-revalidate'
    Expires: 10
    ...

.. code-block:: javascript

    {
        "firstname": "barack",
        "lastname": "obama",
        "_id": "50acfba938345b0978fccad7"
        "updated": "Wed, 21 Nov 2012 16:04:56 GMT",
        "created": "Wed, 21 Nov 2012 16:04:56 GMT",
        "_links": {
            "self": {"href": "people/50acfba938345b0978fccad7", "title": "person"},
            "parent": {"href": "/", "title": "home"},
            "collection": {"href": "people", "title": "people"}
        }
    }

缓存指令和项标题符合我们的新设置。查看 :doc:`features` 获取可用特性和更多用法示例的完整列表。

.. 注意::
    所有的示例和代码片段都来源于 :ref:`demo`，它是一个完整的实用性 API，可以用于在现场实例或本地实例进行独立的试验(你可以实用样例客户端应用来输入数据或者重置数据库)。

.. _`安装`: http://docs.mongodb.org/manual/installation/
.. _运行: http://docs.mongodb.org/manual/tutorial/manage-mongodb-processes/
