特性
========
下面是任何 EVE 支持的 API 可以暴露的主要特性的一个列表。这些特性中的大多数可以通过使用 Demo API (参考 :ref:`demo`) 来现场体验。

Emphasis on REST
----------------
Eve 项目的目标使提供最大可能的兼容 REST 的 API 的实现。基本的 REST_ 原则，像
*separation of concerns*, *stateless and layered system*, *cacheability*, 
*uniform interface*，在涉及核心 API 时已经被列入考虑了。

全部 CRUD 操作
-----------------------------
API 可以支持所有 CRUD_ 操作。在相同的 API 中，你可以在一个端点上访问只读资源，在另
一个端点上访问完整的的可编辑资源。下表是 Eve 对基于 REST 的 CRUD 的实现:

======= ========= ===================
动作     HTTP 动词 使用环境
======= ========= ===================
创建     POST      Collection
创建     PUT       Document
覆盖     PUT       Document
读取     GET, HEAD Collection/Document
更新     PATCH     Document
删除     DELETE    Collection/Document
======= ========= ===================

重写 HTTP 方法
~~~~~~~~~~~~~~~~~~~~~~~
对于不支持任何这些方法的奇怪客户机，API 是一种备用方法将欣然接受
``X-HTTP-Method-Override`` 请求。例如，不支持 ``PATCH`` 方法的客户端方法可以
使用 ``X-HTTP-Method-Override: PATCH`` 头发送 ``POST`` 请求。然后 API 将执行
一个覆盖原来的请求方法的 ``PATCH``。

.. _resource_endpoints:

自定义资源终结点
-------------------------------
默认情况下，Eve 将使已知的数据库集合作为可用资源端点 (REST 习惯用法中的持久化标识)。
因此，数据库集合 ``people`` 在 ``example.com/people`` API 终结点是可用的。但是你
也可以自定义 URI，这样 API 终结点可以变成，``example.com/customers/overseas``。 
考虑以下请求:

.. code-block:: console

    $ curl -i http://eve-demo.herokuapp.com/people
    HTTP/1.1 200 OK

响应负载看起来应该是这样的:

.. code-block:: javascript

    {
        "_items": [
            {
                "firstname": "Mark",
                "lastname": "Green",
                "born": "Sat, 23 Feb 1985 12:00:00 GMT",
                "role": ["copy", "author"],
                "location": {"city": "New York", "address": "4925 Lacross Road"},
                "_id": "50bf198338345b1c604faf31",
                "_updated": "Wed, 05 Dec 2012 09:53:07 GMT",
                "_created": "Wed, 05 Dec 2012 09:53:07 GMT",
                "_etag": "ec5e8200b8fa0596afe9ca71a87f23e71ca30e2d",
                "_links": {
                    "self": {"href": "people/50bf198338345b1c604faf31", "title": "person"},
                },
            },
            ...
        ],
        "_meta": {
            "max_results": 25,
            "total": 70,
            "page": 1
        },
        "_links": {
            "self": {"href": "people", "title": "people"},
            "parent": {"href": "/", "title": "home"}
        }
    }


``_items`` 列表包含请求数据，以及它自己的字段，每一项都提供一些重要的附加字段:

============ =================================================================
字段          说明
============ =================================================================
``_created`` 数据项创建日期。
``_updated`` 数据项上一次更新日期。
``_etag``    ETag，用于并发控制和有条件的请求。
``_id``      唯一数据项键，也是在访问单个项目端点时需要。
============ =================================================================

这些额外的字段由 API 自动处理(客户端不需要在添加/编辑资源时提供它们)。

``_meta`` 字段提供分页数据，只有当 :ref:`Pagination` 已启用 (默认设置)，并至少有
一个文档返回时才会有。``_links`` 列表提供 HATEOAS_ 指令。

.. _subresources:

子资源
~~~~~~~~~~~~~
终结点支持子资源，所以你可以有这样的东西: ``people/<contact_id>/invoices``。当为
这样的终结点设置 ``url`` 规则时，您将使用 regex 并为其分配字段名:

.. code-block:: python

    invoices = {
        'url': 'people/<regex("[a-f0-9]{24}"):contact_id>/invoices'
        ...

然后，对以下终结点执行 GET 请求:

::

    people/51f63e0838345b6dcd7eabff/invoices

会导致底层数据库查询像这样:

::

    {'contact_id': '51f63e0838345b6dcd7eabff'}

还有这一个:

::

    people/51f63e0838345b6dcd7eabff/invoices?where={"number": 10}

将会使查询像这样:

::

    {'contact_id': '51f63e0838345b6dcd7eabff', "number": 10}

请注意，在设计 API 时，大多数时候你都可以不用求助于子资源。在上面的例子中，只要简单地
公开一个客户端可以这样查询的 ``invoices`` 终结点，就可以得到相同的结果: 

::

    invoices?where={"contact_id": 51f63e0838345b6dcd7eabff}

或者

::

    invoices?where={"contact_id": 51f63e0838345b6dcd7eabff, "number": 10}

这主要是一种设计选择，但请记住，启用单个文档终结点，可能会导致性能下降。否则，这个合法
的 GET 请求:

::

    people/<contact_id>/invoices/<invoice_id>

将导致在数据库上的双字段查找。这不是理想的，也不是真正需要的，因为 ``<invoice_id>`` 
是一个惟一字段。相反，如果您有一个简单的资源端点，那么文档查找将发生在单个字段上: 

::

    invoices/<invoice_id>


支持子资源的终结点在 `DELETE`` 操作方面有一个特定的行为。对如下终结点的 ``DELETE``:

::

    people/51f63e0838345b6dcd7eabff/invoices

将导致删除所有匹配以下查询的文档:

::

    {'contact_id': '51f63e0838345b6dcd7eabff'}


因此，对于子资源终结点，只有满足端点语义的文档才会被删除。这与标准行为不同，而集合
终结点上的 delete 操作将导致删除集合中的所有文档。

另一个例子。对如下数据项终结点的 ``DELETE``:

::

    people/51f63e0838345b6dcd7eabff/invoices/1

会导致删除所有匹配如下查询的文档:

::

    {'contact_id': '51f63e0838345b6dcd7eabff', "<invoice_id>": 1}

这种行为支持典型的树结构，其中资源的 id 本身不一定是主键。


.. _custom_item_endpoints:

自定义的多数据项终结点
-------------------------------------
资源可以或不公开单个项端点。API 消费者可以获得访问 ``people``、``people/<ObjectId>`` 
和 ``people/Doe`` 的权限，但只是对 ``/works`` 的权限。当你授予对数据项终结点的访问权
时，您最多可以定义两个查找，它们都是通过 regex 定义的。第一个将是主终结点，并将匹配数
据库主键结构 (即，MongoDB 数据库中的 ``ObjectId``)。

.. code-block:: console

    $ curl -i http://eve-demo.herokuapp.com/people/521d6840c437dc0002d1203c
    HTTP/1.1 200 OK
    Etag: 28995829ee85d69c4c18d597a0f68ae606a266cc
    Last-Modified: Wed, 21 Nov 2012 16:04:56 GMT
    ...

第二个是可选和只读的，它将匹配具有惟一值的字段，因为 Eve 无论如何只取第一个匹配。

.. code-block:: console

    $ curl -i http://eve-demo.herokuapp.com/people/Doe
    HTTP/1.1 200 OK
    Etag: 28995829ee85d69c4c18d597a0f68ae606a266cc
    Last-Modified: Wed, 21 Nov 2012 16:04:56 GMT
    ...

由于我们访问的是相同的数据项，在这两种情况下，响应负载看起来都是这样的:

.. code-block:: javascript

    {
        "firstname": "John",
        "lastname": "Doe",
        "born": "Thu, 27 Aug 1970 14:37:13 GMT",
        "role": ["author"],
        "location": {"city": "Auburn", "address": "422 South Gay Street"},
        "_id": "50acfba938345b0978fccad7"
        "_updated": "Wed, 21 Nov 2012 16:04:56 GMT",
        "_created": "Wed, 21 Nov 2012 16:04:56 GMT",
        "_etag": "28995829ee85d69c4c18d597a0f68ae606a266cc",
        "_links": {
            "self": {"href": "people/50acfba938345b0978fccad7", "title": "person"},
            "parent": {"href": "/", "title": "home"},
            "collection": {"href": "people", "title": "people"}
        }
    }

可以看到，数据项终结点提供了它们自己的 HATEOAS_ 指令。

.. 警告:: 请注意

    根据 REST 规范，资源项应该只有一个惟一标识符。Eve 坚持为每个项目提供一个默认端点。
    添加辅助端点是一个需要仔细考虑的决定。

    考虑我们上面的例子。即使没有 ``people/<lastname>`` 终结点，客户端也总是可以通过
    按姓氏查询资源端点来检索人员:``people/?where={"lastname": "Doe"}``。实际上，
    整个例子是无法处置的，因为可能有很多人共享相同的姓氏，但是你应该明白了。

.. _filters:

筛选
---------
资源终结点允许使用者检索多个文档。支持查询字符串，允许过滤和排序。同时支持原生 Mongo 
查询和 Python 条件表达式。

这里我们请求所有 ``lastname`` 值是 ``Doe``的文档:

::

    http://eve-demo.herokuapp.com/people?where={"lastname": "Doe"}

使用 ``curl``，你可以这样做:

.. code-block:: console

    $ curl -i -g http://eve-demo.herokuapp.com/people?where={%22lastname%22:%20%22Doe%22}
    HTTP/1.1 200 OK

过滤嵌入的文档字段是可行的:

::

    http://eve-demo.herokuapp.com/people?where={"location.city": "San Francisco"}

日期字段也很容易查询:

::

    http://eve-demo.herokuapp.com/people?where={"born": {"$gte":"Wed, 25 Feb 1987 17:00:00 GMT"}}

日期值应该符合 RFC1123。如果需要不同的格式，可以更改 ``DATE_FORMAT`` 设置。

一般来说，你会发现大多数 `MongoDB 查询`_ “只是工作”。如果你需要，
``MONGO_QUERY_BLACKLIST`` 允许你将不需要的操作符列入黑名单。

原生 Python 语法是这样工作的:

.. code-block:: console

    $ curl -i http://eve-demo.herokuapp.com/people?where=lastname=="Doe"
    HTTP/1.1 200 OK

这两种语法都允许条件运算符和逻辑运算符 And/Or 运算符，无论它们是嵌套的还是组合的。

默认情况下，对所有文档字段都启用过滤器。但是，API 维护者可以选择禁用它们所有，并/或在
白名单中列出允许的 (参见 :ref:`global` 中的 ``ALLOWED_FILTERS``)。如果抓取或者担心
通过查询非索引字段而受到 DB DoS 攻击，那么允许过滤器的白名单就是解决方法。

您还可以使用 ``VALIDATE_FILTERING`` 系统设置(参见 :ref:`global`)，根据资源的模式
验证传入的过滤器，如果有任何过滤器无效，则拒绝应用过滤。

整齐打印
---------------
您可以通过指定一个名为 ``pretty`` 的查询参数来美化对响应的打印:

.. code-block:: console

    $ curl -i http://eve-demo.herokuapp.com/people?pretty
    HTTP/1.1 200 OK

    {
        "_items": [
            {
                "_updated": "Tue, 19 Apr 2016 08:19:00 GMT",
                "firstname": "John",
                "lastname": "Doe",
                "born": "Thu, 27 Aug 1970 14:37:13 GMT",
                "role": [
                    "author"
                ],
                "location": {
                    "city": "Auburn",
                    "address": "422 South Gay Street"
                },
                "_links": {
                    "self": {
                        "href": "people/5715e9f438345b3510d27eb8",
                        "title": "person"
                    }
                },
                "_created": "Tue, 19 Apr 2016 08:19:00 GMT",
                "_id": "5715e9f438345b3510d27eb8",
                "_etag": "86dc6b45fe7e2f41f1ca53a0e8fda81224229799"
            },
            ...
        ]
    }


排序
-------
也支持排序:

.. code-block:: console

    $ curl -i http://eve-demo.herokuapp.com/people?sort=city,-lastname
    HTTP/1.1 200 OK

将返回按 city 和 lastname (降序) 排序的文档。如你所见，如果需要反转字段的排序顺序，
只需在字段名称前加上一个减号。

MongoDB 数据层也支持原生 MongoDB 语法:

::

    http://eve-demo.herokuapp.com/people?sort=[("lastname", -1)]

翻译过来就是下面的 ``curl`` 请求:

.. code-block:: console

    $ curl -i http://eve-demo.herokuapp.com/people?sort=[(%22lastname%22,%20-1)]
    HTTP/1.1 200 OK

将返回按 lastname 降序排序的文档。

默认情况下，排序是启用的，可以同时在全局和/或资源级别禁用(参见 :ref:`global` 中的
``SORTING`` 和 :ref:`domain` 中的 ``sorting``)。还可以在每个 API 终结点上设置默认
排序 (参见 :ref:`domain` 中的 ``default_sort``)。

.. 警告:: 请注意

    始终使用双引号来包装字段名和值。使用单引号将导致 ``400 Bad Request`` 响应。

.. _pagination:

分页
----------
默认情况下，为了改善性能和保留带宽，资源分页是启用的。当使用者请求资源时，将提供与查询
匹配的前 N 个项，并通过响应提供到后续/以前页面的链接。默认和最大页面大小是可定制的，
消费者可以通过查询字符串请求特定的页面:

.. code-block:: console

    $ curl -i http://eve-demo.herokuapp.com/people?max_results=20&page=2
    HTTP/1.1 200 OK

当然，您可以混合所有可用的查询参数:

.. code-block:: console

    $ curl -i http://eve-demo.herokuapp.com/people?where={"lastname": "Doe"}&sort=[("firstname", 1)]&page=5
    HTTP/1.1 200 OK

可以禁用分页。请注意，为了清楚起见，上面的示例没有正确转义。如果使用 ``curl``，
请参考 :ref:`filters` 中提供的示例。

.. _hateoas_feature:

HATEOAS
-------
默认启用 *Hypermedia as the Engine of Application State* (HATEOAS_)。每个 GET 响应
都包含一个 ``_links`` 节。链接提供了关于它们相对于被访问资源的``关系``的详细信息和一
个``标题``。然后，客户端可以使用关系和标题动态更新 UI，或者在不知道 API 结构的情况下
导航 API。一个例子:

::

    {
        "_links": {
            "self": {
                "href": "people",
                "title": "people"
            },
            "parent": {
                "href": "/",
                "title": "home"
            },
            "next": {
                "href": "people?page=2",
                "title": "next page"
            },
            "last": {
                "href": "people?page=10",
                "title": "last page"
            }
        }
    }

对 API 主页 (API 入口点) 的 GET 请求将提供到可访问资源的链接列表。从那里，任何客户端
都可以只通过跟随每个响应提供的链接来导航 API。

HATEOAS 链接总是相对于 API 入口点，所以如果你的 API 主页面在 ``examples.com/api/v1``，
在上面例子中的 ``self`` 链接就意味着 *people* 端点位于 ``examples.com/api/v1/people``。

请注意，``next``, ``previous``, ``last`` 和 ``related`` 只有在适当的情况下才会包括在内。

禁用 HATEOAS
~~~~~~~~~~~~~~~~~
可以在 API 和/或资源级别禁用 HATEOAS。为什么要把 HATEOAS 关掉? 好吧，如果你知道你的客
户端应用程序不会使用该特性，那么你可能希望同时节省带宽和性能。

.. _rendering:

渲染
---------
Eve 响应自动呈现为 JSON (默认值) 或 XML，这取决于请求 ``Accept`` 报头。入站文档 (用于
插入和编辑) 采用 JSON 格式。

.. code-block:: console

    $ curl -H "Accept: application/xml" -i http://eve-demo.herokuapp.com
    HTTP/1.1 200 OK
    Content-Type: application/xml; charset=utf-8
    ...

.. code-block:: html

    <resource>
        <link rel="child" href="people" title="people" />
        <link rel="child" href="works" title="works" />
    </resource>

默认渲染器可以通过编辑设置文件中的 ``RENDERERS`` 值来更改。

.. code-block:: python

    RENDERERS = [
        'eve.render.JSONRenderer',
        'eve.render.XMLRenderer'
    ]

你可以通过子类化 ``eve.render.Renderer`` 来创建自己的渲染器。每个渲染器应该设置有效的
``mime`` 特性并实现 ``.render()`` 方法。请注意，必须始终启用至少一个渲染器。

.. _conditional_requests:

带条件的请求
--------------------
每个资源陈述都提供关于它最后一次更新的信息 (``Last-Modified``)，以及对陈述本身计算的
散列值 (``ETag``)。这些头使客户端可以使用 ``If-Modified-Since`` 头执行条件请求:

.. code-block:: console

    $ curl -H "If-Modified-Since: Wed, 05 Dec 2012 09:53:07 GMT" -i http://eve-demo.herokuapp.com/people/521d6840c437dc0002d1203c
    HTTP/1.1 200 OK

或者 ``If-None-Match`` 头:

.. code-block:: console

    $ curl -H "If-None-Match: 1234567890123456789012345678901234567890" -i http://eve-demo.herokuapp.com/people/521d6840c437dc0002d1203c
    HTTP/1.1 200 OK


.. _concurrency:

数据完整性和并发控制
--------------------------------------
API 响应包括一个 ``ETag`` 头，它也允许适当的并发控制。``ETag`` 是一个哈希值，表示服
务器上资源的当前状态。使用者不得编辑 (``PATCH`` 或 ``PUT``) 或删除 (``DELETE``) 资
源，除非他们为试图编辑的资源提供最新的 ``ETag``。这可以防止用过时的版本覆盖项。

考虑以下工作流程:

.. code-block:: console

    $ curl -H "Content-Type: application/json" -X PATCH -i http://eve-demo.herokuapp.com/people/521d6840c437dc0002d1203c -d '{"firstname": "ronald"}'
    HTTP/1.1 428 PRECONDITION REQUIRED

我们尝试编辑 (``PATCH``)，但我们没有为项目提供 ``ETag``，所以我们得到
``428 PRECONDITION REQUIRED`` 返回。让我们再试一次:

.. code-block:: console

    $ curl -H "If-Match: 1234567890123456789012345678901234567890" -H "Content-Type: application/json" -X PATCH -i http://eve-demo.herokuapp.com/people/521d6840c437dc0002d1203c -d '{"firstname": "ronald"}'
    HTTP/1.1 412 PRECONDITION FAILED

这次出了什么问题? 我们提供了强制的 ``If-Match`` 标头，但是它的值与当前存储在服务器上
的项的陈述计算出的 ``ETag`` 不匹配，因此我们得到了 ``412 PRECONDITION FAILED``。
又一次!

.. code-block:: console

    $ curl -H "If-Match: 80b81f314712932a4d4ea75ab0b76a4eea613012" -H "Content-Type: application/json" -X PATCH -i http://eve-demo.herokuapp.com/people/50adfa4038345b1049c88a37 -d '{"firstname": "ronald"}'
    HTTP/1.1 200 OK

终于! 响应负载看起来是这样的:

.. code-block:: javascript

    {
        "_status": "OK",
        "_updated": "Fri, 23 Nov 2012 08:11:19 GMT",
        "_id": "50adfa4038345b1049c88a37",
        "_etag": "372fbbebf54dfe61742556f17a8461ca9a6f5a11"
        "_links": {"self": "..."}
    }

这一次我们的补丁打对了，服务器返回了新的 ``ETag``。我们还得到了新的 ``_updated``
值，它最终将允许我们执行后续的 `conditional requests`_ 。

并发控制适用于所有版本方法: ``PATCH`` (edit), ``PUT`` (覆盖), ``DELETE`` (删除)。

禁用并发控制
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
如果你的使用场景需要，您可以选择完全禁用并发控制。可以通过设置 ``IF_MATCH`` 配置变量
为 ``False`` 来禁用 ETag 匹配检查 (参见 :ref:`global`)。当并发控制被禁用时，响应中
不会提供 ETag。您应该谨慎禁用此功能，因为你将开放你的 API，使其实际上面临文档被旧版本
替换的风险。另外，如果禁用了 ``ENFORCE_IF_MATCH``，ETag 匹配检查可以被客户端当成可
选的。当禁用强制性并发检查后，带有 ``If-Match`` 标头的请求将作为条件请求处理，而没有
``If-Match`` 标头的请求将不作为条件请求处理。

.. _bulk_insert:

批量插入
------------
客户可提交单个文档插入:

.. code-block:: console

    $ curl -d '{"firstname": "barack", "lastname": "obama"}' -H 'Content-Type: application/json' http://eve-demo.herokuapp.com/people
    HTTP/1.1 201 OK

在这种情况下，响应有效负载将只包含相关的文档元数据:

.. code-block:: javascript

    {
        "_status": "OK",
        "_updated": "Thu, 22 Nov 2012 15:22:27 GMT",
        "_id": "50ae43339fa12500024def5b",
        "_etag": "749093d334ebd05cf7f2b7dbfb7868605578db2c"
        "_links": {"self": {"href": "people/50ae43339fa12500024def5b", "title": "person"}}
    }

当 POST 请求返回 ``201 Created`` 时，响应中还包含了 ``Location`` 报头。它的值
是新文档的 URI。

为了减少回送的数量，客户端还可以通过一个请求提交多个文档。它所需要做的就是将文档
封装在 JSON 列表中:

.. code-block:: console

    $ curl -d '[{"firstname": "barack", "lastname": "obama"}, {"firstname": "mitt", "lastname": "romney"}]' -H 'Content-Type: application/json' http://eve-demo.herokuapp.com/people
    HTTP/1.1 201 OK

响应本身将是一个列表，带有每个文档的状态:

.. code-block:: javascript

    {
        "_status": "OK",
        "_items": [
            {
                "_status": "OK",
                "_updated": "Thu, 22 Nov 2012 15:22:27 GMT",
                "_id": "50ae43339fa12500024def5b",
                "_etag": "749093d334ebd05cf7f2b7dbfb7868605578db2c"
                "_links": {"self": {"href": "people/50ae43339fa12500024def5b", "title": "person"}}
            },
            {
                "_status": "OK",
                "_updated": "Thu, 22 Nov 2012 15:22:27 GMT",
                "_id": "50ae43339fa12500024def5c",
                "_etag": "62d356f623c7d9dc864ffa5facc47dced4ba6907"
                "_links": {"self": {"href": "people/50ae43339fa12500024def5c", "title": "person"}}
            }
        ]
    }

当多个文档被提交时，API 利用了 MongoDB *bulk insert* 功能，这意味着不仅是只有一个
请求从客户端传输到远程 API，还在 API 服务器和数据库之间执行了一个回送。

如果多文档插入成功，请记住 ``Location`` 头只返回创建的第一个文档的 URI。


数据验证
---------------
数据验证是开箱即用的。您的配置包括对 API 管理的每个资源的模式定义。发送到 API 
要被插入/更新的数据将根据模式进行验证，只有验证通过时才会更新资源。

.. code-block:: console

    $ curl -d '[{"firstname": "bill", "lastname": "clinton"}, {"firstname": "mitt", "lastname": "romney"}]' -H 'Content-Type: application/json' http://eve-demo.herokuapp.com/people
    HTTP/1.1 201 OK

响应中将包含请求中提供的每个项目的成功/错误状态:

.. code-block:: javascript

    {
        "_status": "ERR",
        "_error": "Some documents contains errors",
        "_items": [
            {
                "_status": "ERR",
                "_issues": {"lastname": "value 'clinton' not unique"}
            },
            {
                "_status": "OK",
            }
        ]
    ]

在上面的示例中，第一个文档没有通过验证，因此整个请求被拒绝。

当所有文档通过验证并正确插入时，响应状态为 ``201 Created``。如果任何文档验证失败，那么
响应状态为 ``422 Unprocessable Entity``，或由 ``VALIDATION_ERROR_STATUS`` 配置定义
的任何其他错误代码。

有关更多信息，请参见 :ref:`validation`。

扩展性的数据验证
--------------------------
数据验证基于 Cerberus_ 验证系统，因此它是可扩展的，所以你可以根据你的特定使用场景
调整它。假设你的 API 在某个字段值上只能接受奇数，你可以扩展 validation 类来验证
它。或者你希望确保 VAT 字段实际上匹配你自己国家的增值税算法，你也可以这么做。事
实上，Eve 的 MongoDB 数据层本身通过实现 ``unique`` 模式字段约束扩展了 Cerberus 
验证。有关更多信息，请参见 :ref:`validation`。

编辑一个文档 (PATCH)
--------------------------
客户端可以通过 ``PATCH`` 方法编辑文档，而 ``PUT`` 将替换它。``PATCH`` 不能删除
字段，而只能更新其值。

考虑以下模式:

.. code-block:: javascript

    'entity': {
        'name': {
            'type': 'string',
            'required': True
        },
        'contact': {
            'type': 'dict',
            'required': True,
            'schema': {
                'phone': {
                    'type': 'string',
                    'required': False,
                    'default': '1234567890'
                },
                'email': {
                    'type': 'string',
                    'required': False,
                    'default': 'abc@efg.com'
                },
            }
        }
    }


两种写法: ``{contact: {email: 'an email'}}`` 和 ``{contact.email: 'an
email'}`` 都可以用于更新 ``contact`` 子文档种 ``email`` 字段。


.. _cache_control:

资源级缓存控制
----------------------------
您可以为每个资源设置全局和单独的缓存控制指令。

.. code-block:: console

    $ curl -i http://eve-demo.herokuapp.com
    HTTP/1.1 200 OK
    Content-Type: application/json
    Content-Length: 131
    Cache-Control: max-age=20
    Expires: Tue, 22 Jan 2013 09:34:34 GMT
    Server: Eve/0.0.3 Werkzeug/0.8.3 Python/2.7.3
    Date: Tue, 22 Jan 2013 09:34:14 GMT

上面的响应包括 ``Cache-Control`` 和 ``Expires`` 头信息。这将最小化服务器上的负载，
因为启用缓存的使用者仅在真正需要时才执行资源密集型请求。

API 版本控制
--------------
我不太喜欢 API 版本控制。我相信客户端应该足够聪明，能够透明地处理 API 更新，特别是
是在 Eve 支持的 API 支持 HATEOAS_ 以后。当需要进行版本控制时，不同的 API 版本应该
是独立的实例，因为不同版本之间有许多不同之处: 缓存、URI、模式、验证等等。支持 URI 
版本控制 (http://api.example.com/v1/..)。

.. _document_versioning:

文档版本控制
-------------------
Eve 支持文档的自动版本控制。默认情况下，该设置是关闭的，但是可以全局打开，也可以为
每个资源单独配置。启用后，Eve 开始自动跟踪对文档的更改，并在检索文档时添加
``_version`` 和 ``_latest_version`` 字段。

在幕后，Eve 将文档版本存储在影子集合中，影子集合与 Eve 定义的每个主要资源的集合平行。
在正常的 POST、PUT 和 PATCH 操作期间，新文档版本会自动添加到这个集合中。当获取提供
对文档版本访问的数据项时，可以使用一个特殊的新查询参数。使用 ``?version=VERSION``
访问特定版本，使用 ``?version=all`` 访问所有版本，并使用 ``?version=diffs`` 访问
所有版本的差异。集合查询特性，如投影、分页和排序，可以与 ``all`` 和 ``diff`` 一起
工作，但排序与 ``diff`` 不行。

需要注意的是，在开启版本控制时，有一些非标准场景可能会产生意想不到的结果。特别是，
在 Eve 生成的 API 之外修改集合时，不会保存文档历史记录。此外，如果在任何时候从主文
档中删除了 ``VERSION`` 字段 (启用版本控制时不能通过 API 执行)，后续的写操作通过
``VERSION`` = 1 重新初始化 ``VERSION`` 号。此时将有多个版本的文档使用相同的版本号。
在正常实践中，可以启用 ``VERSIONING``，而不必担心任何新集合，甚至不必担心以前没有
启用版本控制的现有集合。

此外，还有文档版本特有的缓存边缘场景。特定的文档版本包含 ``_latest_version`` 字段，
当创建新文档版本时，该字段的值将发生更改。为了解释这一点，Eve 确定时间 
``_latest_version`` 是否已更改 (最后一个的时间戳) 并使用该值填充 ``Last-Modified``
标头，并检查特定文档版本查询的 ``If-Modified-Since`` 条件缓存验证器'。注意，这与
版本最后更新字段中的时间戳不同。但是，当 ``_latest_version`` 更改时，文档版本的
etag 不会更改。这导致了两种极端情况。首先，由于 Eve 无法仅从 ETag 确定客户端的
``_latest_version`` 是否为最新的，因此仅使用 ``If-None-Match`` 进行旧文档版本
缓存验证的查询将始终使其缓存失效。其次，在创建多个新版本的同一秒内获取和缓存的版本
可能会在随后的 ``GET`` 查询中收到不正确的 ``Not Modified`` 响应，原因是
``Last-Modified`` 值的分辨率为 1 秒，而静态 etag 值没有提供更改指示。这些都是非常
不可能的场景，但是希望每秒进行多次编辑的应用程序应该考虑到打破旧 ``_latest_version`` 
数据的可能性。

有关更多信息，请参见: :ref:`global` 和 :ref:`domain`。


身份验证
--------------
支持可定制的基本身份验证 (RFC-2617)、基于令牌的身份验证和基于 HMAC 的身份验证。
OAuth2 可以很容易地集成。您可以锁定整个 API，或者只是一些终结点。你还可以限制 CRUD 
命令，比如，允许打开只读访问，同时限制对授权用户的编辑、插入和删除。还支持基于角色
的访问控制。有关更多信息，请参见 :ref:`auth`。

CORS 跨域资源共享
----------------------------------
web 页面中包含的 JavaScript 可以访问 Eve 支持的 API。默认情况下是禁用的，CORS_ 使
web 页面可以使用 REST API，这通常受到大多数浏览器 “同域” 安全策略的限制。
``X_DOMAINS`` 设置允许指定允许哪些域可以执行 CORS 请求。正则表达式列表可以在
``X_DOMAINS_RE`` 中定义，这对于具有子域动态范围的网站非常有用。确保锚定并正确转义 
regex，例如 ``X_DOMAINS_RE = ['^http://sub-\d{3}\.example\.com$']`。

JSONP 支持
-------------
一般来说，当你可以启用 CORS 时，你并不想添加 JSONP:

    有人对 JSONP 提出了一些批评。跨源资源共享 (CORS) 是一种较新的从不同域中的服务器
    获取数据的方法，它解决了其中一些批评。现在所有的现代浏览器都支持 CORS，这使得它
    成为跨浏览器的可行选择 (source_)。

然而，在某些情况下，你确实需要 JSONP，比如必须支持老软件时(有人支持 IE6 吗?)

要在 Eve 中启用 JSONP，只需设置 ``JSONP_ARGUMENT``。然后，任何带有 
``JSONP_ARGUMENT`` 的有效请求都会返回一个包含该参数值的响应。例如，如果您设置 
``JSON_ARGUMENT = 'callback'``:

.. code-block:: console

    $ curl -i http://localhost:5000/?callback=hello
    hello(<JSON here>)

不含 ``callback`` 参数的请求将被当作没有 JSONP 的情况来处理。


默认只读
--------------------
如果你只需要一个只读 API，那么你可以在几分钟内启动并运行它。

默认的和可为空的值
---------------------------
字段可以有默认值和可空类型。当服务 POST (创建) 请求时，将为缺失的字段分配配置的默认值。
有关更多信息，请参见 :ref:`schema` 中的 ``default`` and ``nullable``关键字。

预定义的数据库过滤器
---------------------------
资源端点将只公开 (和更新) 匹配预定义筛选器的文档。这使多个资源端点可以无缝地针对同一个
数据库集合。典型的使用场景是假想中的，由 ``/admins`` 和 ``/users`` API 终结点使用的
数据库上的 ``people`` 集合。

.. 警告:: 另请参阅

    - :ref:`datasource`
    - :ref:`filter`

.. _projections:

投影
-----------
这项特性允许你创建集合和文档的动态视图，或者更精确地说，使用 'projection' 来决定应该
或不应该返回哪些字段。换句话说，投影是条件查询，客户端指定 API 应该返回哪些字段。

.. code-block:: console

    $ curl -i http://eve-demo.herokuapp.com/people?projection={"lastname": 1, "born": 1}
    HTTP/1.1 200 OK

上面的查询将只返回 'people' 资源中所有可用字段中的 *lastname* 和 *born*。你还可以排除字段:

.. code-block:: console

    $ curl -i http://eve-demo.herokuapp.com/people?projection={"born": 0}
    HTTP/1.1 200 OK

上面将返回除 *born* 以外的所有字段。请注意，负载中仍然包含 ID_FIELD、DATE_CREATED、
DATE_UPDATED 等关键字段。还要记住，有些数据库引擎 (包括 Mongo) 不允许混合包含和排除
选项。

.. 警告:: 另请参阅

    - :ref:`projection`
    - :ref:`projection_filestorage`

.. _embedded_docs:

内嵌资源序列化
-------------------------------
如果文档字段正在引用另一个资源中的文档，客户端可以请求将引用的文档嵌入到请求的文档中。

客户端可以通过查询参数激活基于每个请求的文档嵌入。假设你有一个这样配置的 ``emails`` 
资源:

.. code-block:: python
   :emphasize-lines: 9

    DOMAIN = {
        'emails': {
            'schema': {
                'author': {
                    'type': 'objectid',
                    'data_relation': {
                        'resource': 'users',
                        'field': '_id',
                        'embeddable': True
                    },
                },
                'subject': {'type': 'string'},
                'body': {'type': 'string'},
            }
        }

A GET 像这样: ``/emails?embedded={"author":1}``，将返回一个完全嵌入的用户文档，
而没有 ``embedded`` 参数的相同请求将只返回用户 ``ObjectId``。嵌入式资源序列化在
资源和数据项 (``/emails/<id>/?embedded={"author":1}``) 终结点都可用。

可以在全局级别 (通过将 ``EMBEDDING`` 设置为 ``True`` 或 ``False``) 和资源级别 
(通过切换 ``embedding`` 值) 启用或禁用嵌入。此外，只有 ``embeddable`` 值显式设
置为 ``True`` 的字段才允许嵌入引用的文档。

嵌入还可以处理到文档特定版本的 data_relationship，但是模式看起来有点不同。要将 
data_relationship 启用到特定版本，请将 ``'version': True`` 添加到 
data_relationship 块。你还需要将 ``type`` 更改为 ``dict``，并添加如下所示的 
``schema`` 定义。

.. code-block:: python
   :emphasize-lines: 5, 6, 11

    DOMAIN = {
        'emails': {
            'schema': {
                'author': {
                    'type': 'dict',
                    'schema': {
                        '_id': {'type': 'objectid'},
                        '_version': {'type': 'integer'}
                    },
                    'data_relation': {
                        'resource': 'users',
                        'field': '_id',
                        'embeddable': True,
                        'version': True,
                    },
                },
                'subject': {'type': 'string'},
                'body': {'type': 'string'},
            }
        }

如你所见，``'version': True`` 将 data_relationship 字段的期望值更改为具有字段名称
``data_relation['field']`` 和 ``VERSION`` 的字典。在上面的 data_relationship 定
义中使用 ``'field': '_id'``，在 Eve 配置中使用 ``VERSION = '_version'``，在这种
情况下，data_relationship 的值将是一个字典，其中的字段为 ``_id`` 和 ``_version``。

预定义的资源序列化
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
还可以为预定义的资源序列化选择一些字段。如果列出的字段是可嵌入的，并且它们实际上正在
引用其他资源中的文档 (并且对资源启用了嵌入)，那么默认情况下将嵌入所引用的文档。客户端
仍然可以选择退出默认嵌入的字段:

.. code-block:: console

    $ curl -i http://example.com/people/?embedded={"author": 0}
    HTTP/1.1 200 OK

Limitations
~~~~~~~~~~~
目前，我们通过对位于任何子文档 (嵌套的字典和列表) 的引用支持文档嵌入。例如，查询 
``/invoices/?embedded={"user.friends":1}`` 将返回一个带有 ``user`` 的文档
和它所有的 ``friends`` 嵌入，但只有当 ``user`` 是一个子文档，而 ``friends`` 是
一个引用列表 (它可以是一个字典列表，嵌套字典，等等) 时。这个特性是关于 GET 请求的
序列化。不支持对嵌入文档的 POST、PUT 或 PATCH。

默认情况下文档嵌入是启用的。

.. 警告:: 请注意

    当涉及到 MongoDB 时，嵌入式资源序列化处理的是*文档引用* (链接文档)，这与
    *嵌入式文档*不同，也受到 Eve 的支持 (参见 `MongoDB Data Model
    Design`_)。嵌入式资源序列化是一个很好的特性，它可以帮助你为客户端规范化数据
    模型。但是，在决定是否启用它时 (尤其是默认情况下)，请记住，正在查找的每个嵌入
    式资源都需要进行数据库查找，这很容易导致性能问题。

.. _soft_delete:

Soft Delete
-----------
Eve 提供了一个可选的 “软删除” 模式，在该模式中，已删除的文档继续存储在数据库中，并且
能够恢复，但在响应 API 请求时仍然作为已删除的项。默认情况下软删除是禁用的，但是可以
使用 ``SOFT_DELETE`` 配置项设置全局启用软删除，或者使用域配置 ``soft_delete`` 设置
在资源级别单独配置软删除。有关启用和配置软删除的更多信息，请参见 :ref:`global` 和 
:ref:`domain`。

当启用软删除时，附加到 ``on_delete_resource_originals`` 和 
``on_delete_resource_originals_<resource_name>`` 事件的回调将通过 ``originals`` 
参数接收已删除和未删除的文档(参见 :ref:`eventhooks`)。

行为
~~~~~~~~
With soft delete enabled, DELETE requests to individual items and resources
respond just as they do for a traditional "hard" delete. Behind the scenes,
however, Eve does not remove deleted items from the database, but instead
patches the document with a ``_deleted`` meta field set to ``true``. (The name
of the ``_deleted`` field is configurable. See :ref:`global`.) All requests
made when soft delete is enabled filter against or otherwise account for the
``_deleted`` field.

The ``_deleted`` field is automatically added and initialized to ``false`` for
all documents created while soft delete is enabled. Documents created prior to
soft delete being enabled and which therefore do not define the ``_deleted``
field in the database will still include ``_deleted: false`` in API response
data, added by Eve during response construction. PUTs or PATCHes to these
documents will add the ``_deleted`` field to the stored documents, set to
``false``.

Responses to GET requests for soft deleted documents vary slightly from
responses to missing or "hard" deleted documents. GET requests for soft deleted
documents will still respond with ``404 Not Found`` status codes, but the
response body will contain the soft deleted document with ``_deleted: true``.
Documents embedded in the deleted document will not be expanded in the
response, regardless of any default settings or the contents of the request's
``embedded`` query param. This is to ensure that soft deleted documents
included in ``404`` responses reflect the state of a document when it was
deleted, and do not to change if embedded documents are updated.

By default, resource level GET requests will not include soft deleted items in
their response. This behavior matches that of requests after a "hard" delete.
If including deleted items in the response is desired, the ``show_deleted``
query param can be added to the request. (the ``show_deleted`` param name is
configurable. See :ref:`global`) Eve will respond with all documents, deleted
or not, and it is up to the client to parse returned documents' ``_deleted``
field. The ``_deleted`` field can also be explicitly filtered against in a
request, allowing only deleted documents to be returned using a
``?where={"_deleted": true}`` query.

Soft delete is enforced in the data layer, meaning queries made by application
code using the ``app.data.find_one`` and ``app.data.find`` methods will
automatically filter out soft deleted items. Passing a request object with
``req.show_deleted == True`` or a lookup dictionary that explicitly filters on
the ``_deleted`` field will override the default filtering.

恢复软删除项
~~~~~~~~~~~~~~~~~~~~~~~~~~~~
PUT or PATCH requests made to a soft deleted document will restore it,
automatically setting ``_deleted`` to ``false`` in the database. Modifying the
``_deleted`` field directly is not necessary (or allowed). For example, using
PATCH requests, only the fields to be changed in the restored version would be
specified, or an empty request would be made to restore the document as is. The
request must be made with proper authorization for write permission to the soft
deleted document or it will be refused.

Be aware that, should a previously soft deleted document be restored, there is
a chance that an eventual unique field might end up being now duplicated in two
different documents: the restored one, and another which might have been stored
with the same field value while the original (now restored) was in 'deleted'
state. This is because soft deleted documents are ignored when validating the
`unique` rule for new or updated documents.


版本控制
~~~~~~~~~~
Soft deleting a versioned document creates a new version of that document with
``_deleted`` set to ``true``. A GET request to the deleted version will receive
a ``404 Not Found`` response as described above, while previous versions will
continue to respond with ``200 OK``. Responses to ``?version=diff`` or
``?version=all`` will include the deleted version as if it were any other.

数据关系
~~~~~~~~~~~~~~
The Eve ``data_relation`` validator will not allow references to documents that
have been soft deleted. Attempting to create or update a document with a
reference to a soft deleted document will fail just as if that document had
been hard deleted. Existing data relations to documents that are soft deleted
remain in the database, but requests requiring embedded document serialization
of those relations will resolve to a null value. Again, this matches the
behavior of relations to hard deleted documents.

Versioned data relations to a deleted document version will also fail to
validate, but relations to versions prior to deletion or after restoration of
the document are allowed and will continue to resolve successfully.

注意事项
~~~~~~~~~~~~~~
Disabling soft delete after use in an application requires database maintenance
to ensure your API remains consistent. With soft delete disabled, requests will
no longer filter against or handle the ``_deleted`` field, and documents that
were soft deleted will now be live again on your API. It is therefore necessary
when disabling soft delete to perform a data migration to remove all documents
with ``_deleted == True``, and recommended to remove the ``_deleted`` field
from documents where ``_deleted == False``. Enabling soft delete in an existing
application is safe, and will maintain documents deleted from that point on.

.. _eventhooks:

事件钩子
-----------
Pre-Request 事件钩子
~~~~~~~~~~~~~~~~~~~~~~~
When a GET/HEAD, POST, PATCH, PUT, DELETE request is received, both
a ``on_pre_<method>`` and a ``on_pre_<method>_<resource>`` event is raised.
You can subscribe to these events with multiple callback functions.

.. code-block:: pycon

    >>> def pre_get_callback(resource, request, lookup):
    ...  print('A GET request on the "%s" endpoint has just been received!' % resource)

    >>> def pre_contacts_get_callback(request, lookup):
    ...  print('A GET request on the contacts endpoint has just been received!')

    >>> app = Eve()

    >>> app.on_pre_GET += pre_get_callback
    >>> app.on_pre_GET_contacts += pre_contacts_get_callback

    >>> app.run()

Callbacks will receive the resource being requested, the original
``flask.request`` object and the current lookup dictionary as arguments (only
exception being the ``on_pre_POST`` hook which does not provide a ``lookup``
argument).

动态查询过滤器
^^^^^^^^^^^^^^^^^^^^^^
Since the ``lookup`` dictionary will be used by the data layer to retrieve
resource documents, developers may choose to alter it in order to add custom
logic to the lookup query.

.. code-block:: python

    def pre_GET(resource, request, lookup):
        # only return documents that have a 'username' field.
        lookup["username"] = {'$exists': True}

    app = Eve()

    app.on_pre_GET += pre_GET
    app.run()

Altering the lookup dictionary at runtime would have similar effects to
applying :ref:`filter` via configuration. However, you can only set static
filters via configuration whereas by hooking to the ``on_pre_<METHOD>`` events
you are allowed to set dynamic filters instead, which allows for additional
flexibility.

Post-Request 事件钩子
~~~~~~~~~~~~~~~~~~~~~~~~
When a GET, POST, PATCH, PUT, DELETE method has been executed, both
a ``on_post_<method>`` and ``on_post_<method>_<resource>`` event is raised. You
can subscribe to these events with multiple callback functions. Callbacks will
receive the resource accessed, original `flask.request` object and the response
payload.

.. code-block:: pycon

    >>> def post_get_callback(resource, request, payload):
    ...  print('A GET on the "%s" endpoint was just performed!' % resource)

    >>> def post_contacts_get_callback(request, payload):
    ...  print('A get on "contacts" was just performed!')

    >>> app = Eve()

    >>> app.on_post_GET += post_get_callback
    >>> app.on_post_GET_contacts += post_contacts_get_callback

    >>> app.run()

数据库事件钩子
~~~~~~~~~~~~~~~~~~~~

Database event hooks work like request event hooks. These events are fired
before and after a database action. Here is an example of how events are
configured:

.. code-block:: pycon

   >>> def add_signature(resource, response):
   ...     response['SIGNATURE'] = "A %s from eve" % resource

   >>> app = Eve()
   >>> app.on_fetched_item += add_signature

You may use flask's ``abort()`` to interrupt the database operation:

.. code-block:: pycon

   >>> from flask import abort

   >>> def check_update_access(resource, updates, original):
   ...     abort(403)

   >>> app = Eve()
   >>> app.on_insert_item += check_update_access

The events are fired for resources and items if the action is available for
both. And for each action two events will be fired:

- Generic: ``on_<action_name>``
- With the name of the resource: ``on_<action_name>_<resource_name>``

Let's see an overview of what events are available:

+-------+--------+------+--------------------------------------------------+
|Action |What    |When  |Event name / method signature                     |
+=======+========+======+==================================================+
|Fetch  |Resource|After || ``on_fetched_resource``                         |
|       |        |      || ``def event(resource_name, response)``          |
|       |        |      +--------------------------------------------------+
|       |        |      || ``on_fetched_resource_<resource_name>``         |
|       |        |      || ``def event(response)``                         |
|       +--------+------+--------------------------------------------------+
|       |Item    |After || ``on_fetched_item``                             |
|       |        |      || ``def event(resource_name, response)``          |
|       |        |      +--------------------------------------------------+
|       |        |      || ``on_fetched_item_<resource_name>``             |
|       |        |      || ``def event(response)``                         |
|       +--------+------+--------------------------------------------------+
|       |Diffs   |After || ``on_fetched_diffs``                            |
|       |        |      || ``def event(resource_name, response)``          |
|       |        |      +--------------------------------------------------+
|       |        |      || ``on_fetched_diffs_<resource_name>``            |
|       |        |      || ``def event(response)``                         |
+-------+--------+------+--------------------------------------------------+
|Insert |Items   |Before|| ``on_insert``                                   |
|       |        |      || ``def event(resource_name, items)``             |
|       |        |      +--------------------------------------------------+
|       |        |      || ``on_insert_<resource_name>``                   |
|       |        |      || ``def event(items)``                            |
|       |        +------+--------------------------------------------------+
|       |        |After || ``on_inserted``                                 |
|       |        |      || ``def event(resource_name, items)``             |
|       |        |      +--------------------------------------------------+
|       |        |      || ``on_inserted_<resource_name>``                 |
|       |        |      || ``def event(items)``                            |
+-------+--------+------+--------------------------------------------------+
|Replace|Item    |Before|| ``on_replace``                                  |
|       |        |      || ``def event(resource_name, item, original)``    |
|       |        |      +--------------------------------------------------+
|       |        |      || ``on_replace_<resource_name>``                  |
|       |        |      || ``def event(item, original)``                   |
|       |        +------+--------------------------------------------------+
|       |        |After || ``on_replaced``                                 |
|       |        |      || ``def event(resource_name, item, original)``    |
|       |        |      +--------------------------------------------------+
|       |        |      || ``on_replaced_<resource_name>``                 |
|       |        |      || ``def event(item, original)``                   |
+-------+--------+------+--------------------------------------------------+
|Update |Item    |Before|| ``on_update``                                   |
|       |        |      || ``def event(resource_name, updates, original)`` |
|       |        |      +--------------------------------------------------+
|       |        |      || ``on_update_<resource_name>``                   |
|       |        |      || ``def event(updates, original)``                |
|       |        +------+--------------------------------------------------+
|       |        |After || ``on_updated``                                  |
|       |        |      || ``def event(resource_name, updates, original)`` |
|       |        |      +--------------------------------------------------+
|       |        |      || ``on_updated_<resource_name>``                  |
|       |        |      || ``def event(updates, original)``                |
+-------+--------+------+--------------------------------------------------+
|Delete |Item    |Before|| ``on_delete_item``                              |
|       |        |      || ``def event(resource_name, item)``              |
|       |        |      +--------------------------------------------------+
|       |        |      || ``on_delete_item_<resource_name>``              |
|       |        |      || ``def event(item)``                             |
|       |        +------+--------------------------------------------------+
|       |        |After || ``on_deleted_item``                             |
|       |        |      || ``def event(resource_name, item)``              |
|       |        |      +--------------------------------------------------+
|       |        |      || ``on_deleted_item_<resource_name>``             |
|       |        |      || ``def event(item)``                             |
|       +--------+------+--------------------------------------------------+
|       |Resource|Before|| ``on_delete_resource``                          |
|       |        |      || ``def event(resource_name)``                    |
|       |        |      +--------------------------------------------------+
|       |        |      || ``on_delete_resource_<resource_name>``          |
|       |        |      || ``def event()``                                 |
|       |        |      +--------------------------------------------------+
|       |        |      || ``on_delete_resource_originals``                |
|       |        |      || ``def event(resource_name, originals, lookup)`` |
|       |        |      +--------------------------------------------------+
|       |        |      || ``on_delete_resource_originals_<resource_name>``|
|       |        |      || ``def event(originals, lookup)``                |
|       |        +------+--------------------------------------------------+
|       |        |After || ``on_deleted_resource``                         |
|       |        |      || ``def event(resource_name, item)``              |
|       |        |      +--------------------------------------------------+
|       |        |      || ``on_deleted_resource_<resource_name>``         |
|       |        |      || ``def event(item)``                             |
+-------+--------+------+--------------------------------------------------+



Fetch 事件
^^^^^^^^^^^^

These are the fetch events with their method signature:

- ``on_fetched_resource(resource_name, response)``
- ``on_fetched_resource_<resource_name>(response)``
- ``on_fetched_item(resource_name, response)``
- ``on_fetched_item_<resource_name>(response)``
- ``on_fetched_diffs(resource_name, response)``
- ``on_fetched_diffs_<resource_name>(response)``

They are raised when items have just been read from the database and are
about to be sent to the client. Registered callback functions can manipulate
the items as needed before they are returned to the client.

.. code-block:: pycon

    >>> def before_returning_items(resource_name, response):
    ...  print('About to return items from "%s" ' % resource_name)

    >>> def before_returning_contacts(response):
    ...  print('About to return contacts')

    >>> def before_returning_item(resource_name, response):
    ...  print('About to return an item from "%s" ' % resource_name)

    >>> def before_returning_contact(response):
    ...  print('About to return a contact')

    >>> app = Eve()
    >>> app.on_fetched_resource += before_returning_items
    >>> app.on_fetched_resource_contacts += before_returning_contacts
    >>> app.on_fetched_item += before_returning_item
    >>> app.on_fetched_item_contacts += before_returning_contact

It is important to note that item fetch events will work with `Document
Versioning`_ for specific document versions like ``?version=5`` and all
document versions with ``?version=all``. Accessing diffs of all versions
with ``?version=diffs`` will only work with the diffs fetch events. Note
that diffs returns partial documents which should be handled in the
callback.


Insert 事件
^^^^^^^^^^^^^

These are the insert events with their method signature:

- ``on_insert(resource_name, items)``
- ``on_insert_<resource_name>(items)``
- ``on_inserted(resource_name, items)``
- ``on_inserted_<resource_name>(items)``

When a POST requests hits the API and new items are about to be stored in
the database, these events are fired:

- ``on_insert`` for every resource endpoint.
- ``on_insert_<resource_name>`` for the specific `<resource_name>` resource
  endpoint.

Callback functions could hook into these events to arbitrarily add new fields
or edit existing ones.

After the items have been inserted, these two events are fired:

- ``on_inserted`` for every resource endpoint.
- ``on_inserted_<resource_name>`` for the specific `<resource_name>` resource
  endpoint.

.. admonition:: Validation errors

    Items passed to these events as arguments come in a list. And only those items
    that passed validation are sent.

示例:

.. code-block:: pycon

    >>> def before_insert(resource_name, items):
    ...  print('About to store items to "%s" ' % resource)

    >>> def after_insert_contacts(items):
    ...  print('About to store contacts')

    >>> app = Eve()
    >>> app.on_insert += before_insert
    >>> app.on_inserted_contacts += after_insert_contacts


Replace 事件
^^^^^^^^^^^^^^

These are the replace events with their method signature:

- ``on_replace(resource_name, item, original)``
- ``on_replace_<resource_name>(item, original)``
- ``on_replaced(resource_name, item, original)``
- ``on_replaced_<resource_name>(item, original)``

When a PUT request hits the API and an item is about to be replaced after
passing validation, these events are fired:

- ``on_replace`` for any resource item endpoint.
- ``on_replace_<resource_name>`` for the specific resource endpoint.

`item` is the new item which is about to be stored. `original` is the item in
the database that is being replaced. Callback functions could hook into these
events to arbitrarily add or update `item` fields, or to perform other
accessory action.

After the item has been replaced, these other two events are fired:

- ``on_replaced`` for any resource item endpoint.
- ``on_replaced_<resource_name>`` for the specific resource endpoint.

Update 事件
^^^^^^^^^^^^^

These are the update events with their method signature:

- ``on_update(resource_name, updates, original)``
- ``on_update_<resource_name>(updates, original)``
- ``on_updated(resource_name, updates, original)``
- ``on_updated_<resource_name>(updates, original)``

When a PATCH request hits the API and an item is about to be updated after
passing validation, these events are fired `before` the item is updated:

- ``on_update`` for any resource endpoint.
- ``on_update_<resource_name>`` is fired only when the `<resource_name>`
  endpoint is hit.

Here `updates` stands for updates being applied to the item and `original` is
the item in the database that is about to be updated. Callback functions
could hook into these events to arbitrarily add or update fields in
`updates`, or to perform other accessory action.

`After` the item has been updated:

- ``on_updated`` is fired for any resource endpoint.
- ``on_updated_<resource_name>`` is fired only when the `<resource_name>`
  endpoint is hit.

.. admonition:: Please note

    Please be aware that ``last_modified`` and ``etag`` headers will always be
    consistent with the state of the items on the database (they  won't be
    updated to reflect changes eventually applied by the callback functions).

Delete 事件
^^^^^^^^^^^^^

These are the delete events with their method signature:

- ``on_delete_item(resource_name, item)``
- ``on_delete_item_<resource_name>(item)``
- ``on_deleted_item(resource_name, item)``
- ``on_deleted_item_<resource_name>(item)``
- ``on_delete_resource(resource_name)``
- ``on_delete_resource_<resource_name>()``
- ``on_delete_resource_originals(originals, lookup)``
- ``on_delete_resource_originals_<resource_name>(originals, lookup)``
- ``on_deleted_resource(resource_name)``
- ``on_deleted_resource_<resource_name>()``

数据项
.....

When a DELETE request hits an item endpoint and `before` the item is deleted,
these events are fired:

- ``on_delete_item`` for any resource hit by the request.
- ``on_delete_item_<resource_name>`` for the specific `<resource_name>` item endpoint
  hit by the DELETE.

`After` the item has been deleted the ``on_deleted_item(resource_name,
item)`` and ``on_deleted_item_<resource_name>(item)`` are raised.

`item` is the item being deleted. Callback functions could hook into
these events to perform accessory actions. And no you can't arbitrarily abort
the delete operation at this point (you should probably look at
:ref:`validation`, or eventually disable the delete command altogether).

资源
.........

If you were brave enough to enable the DELETE command on resource endpoints
(allowing for wipeout of the entire collection in one go), then you can be
notified of such a disastrous occurrence by hooking a callback function to the
``on_delete_resource(resource_name)`` or
``on_delete_resource_<resource_name>()`` hooks.

- ``on_delete_resource_originals`` for any resource hit by the request after having retrieved the originals documents.
- ``on_delete_resource_originals_<resource_name>`` for the specific `<resource_name>` resource endpoint
  hit by the DELETE after having retrieved the original document.

NOTE: those two event are useful in order to perform some business
logic before the actual remove operation given the look up and the
list of originals

.. _aggregation_hooks:

聚合事件钩子
~~~~~~~~~~~~~~~~~~~~~~~
You can also attach one or more callbacks to your aggregation endpoints. The
``before_aggregation`` event is fired when an aggregation is about to be
performed. Any attached callback function will receive both the endpoint name
and the aggregation pipeline as arguments. The pipeline can then be altered if
needed.

.. code-block:: pycon

    >>> def on_aggregate(endpoint, pipeline):
    ...   pipeline.append({"$unwind": "$tags"})

    >>> app = Eve()
    >>> app.before_aggregation += on_aggregate

The ``after_aggregation`` event is fired when the aggregation has been
performed. An attached callback function could leverage this event to modify
the documents before they are returned to the client.

.. code-block:: pycon

   >>> def alter_documents(endpoint, documents):
   ...   for document in documents:
   ...     document['hello'] = 'well, hello!'

   >>> app = Eve()
   >>> app.after_aggregation += alter_documents

For more information on aggregation support, see :ref:`aggregation`


.. admonition:: Please note

    To provide seamless event handling features Eve relies on the Events_ package.

.. _ratelimiting:

速度限制
-------------
API rate limiting is supported on a per-user/method basis. You can set the
number of requests and the time window for each HTTP method. If the requests
limit is hit within the time window, the API will respond with ``429 Request
limit exceeded`` until the timer resets. Users are identified by the
Authentication header or (when missing) by the client IP. When rate limiting
is enabled, appropriate ``X-RateLimit-`` headers are provided with every API
response.  Suppose that the rate limit has been set to 300 requests every 15
minutes, this is what a user would get after hitting a endpoint with a single
request:

::

    X-RateLimit-Remaining: 299
    X-RateLimit-Limit: 300
    X-RateLimit-Reset: 1370940300

You can set different limits for each one of the supported methods (GET, POST,
PATCH, DELETE).

.. admonition:: Please Note

   Rate Limiting is disabled by default, and needs a Redis server running when
   enabled. A tutorial on Rate Limiting is forthcoming.

自定义 ID 字段
----------------
Eve 允许扩展其标准数据类型支持。在 :ref:`custom_ids` 教程中，我们看到了如何使用 UUID 
值代替 MongoDB 默认对象作为惟一的文档标识。

文件存储
------------
Media files (images, pdf, etc.) can be uploaded as ``media`` document
fields. Upload is done via ``POST``, ``PUT`` and
``PATCH`` as usual, but using the ``multipart/form-data`` content-type.

Let us assume that the ``accounts`` endpoint has a schema like this:

.. code-block:: python

    accounts = {
        'name': {'type': 'string'},
        'pic': {'type': 'media'},
        ...
    }

With curl we would ``POST`` like this:

.. code-block:: console

    $ curl -F "name=john" -F "pic=@profile.jpg" http://example.com/accounts


For optimized performance files are stored in GridFS_ by default. Custom
``MediaStorage`` classes can be implemented and passed to the application to
support alternative storage systems. A ``FileSystemMediaStorage`` class is in
the works, and will soon be included with the Eve package.

As a proper developer guide is not available yet, you can peek at the
MediaStorage_ source if you are interested in developing custom storage
classes.

以 Base64 字符串的形式提供媒体文件
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
When a document is requested media files will be returned as Base64 strings,

.. code-block:: python

    {
        '_items': [
            {
                '_updated':'Sat, 05 Apr 2014 15:52:53 GMT',
                'pic':'iVBORw0KGgoAAAANSUhEUgAAA4AAAAOACA...',
            }
        ]
        ...
   }

However, if the ``EXTENDED_MEDIA_INFO`` list is populated (it isn't by
default) the payload format will be different. This flag allows passthrough
from the driver of additional meta fields. For example, using the MongoDB
driver, fields like ``content_type``, ``name`` and ``length`` can be added to
this list and will be passed-through from the underlying driver.

When ``EXTENDED_MEDIA_INFO`` is used the field will be a dictionary
whereas the file itself is stored under the ``file`` key and other keys
are the meta fields. Suppose that the flag is set like this:

.. code-block:: python

    EXTENDED_MEDIA_INFO = ['content_type', 'name', 'length']

Then the output will be something like

.. code-block:: python

    {
        '_items': [
            {
                '_updated':'Sat, 05 Apr 2014 15:52:53 GMT',
                'pic': {
                    'file': 'iVBORw0KGgoAAAANSUhEUgAAA4AAAAOACA...',
                    'content_type': 'text/plain',
                    'name': 'test.txt',
                    'length': 8129
                }
            }
        ]
        ...
    }

For MongoDB, further fields can be found in the `driver documentation`_.

If you have other means to retrieve the media files (custom Flask endpoint for
example) then the media files can be excluded from the payload by setting to
``False`` the ``RETURN_MEDIA_AS_BASE64_STRING`` flag. This takes into account
if ``EXTENDED_MEDIA_INFO`` is used.

在专用终结点上提供媒体文件
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
While returning files embedded as Base64 fields is the default behaviour, you
can opt for serving them at a dedicated media endpoint. You achieve that by
setting ``RETURN_MEDIA_AS_URL`` to ``True``. When this feature is enabled
document fields contain urls to the correspondent files, which are served at the
media endpoint.

You can change the default media endpoint (``media``) by updating the
``MEDIA_BASE_URL`` and ``MEDIA_ENDPOINT`` setting. Suppose you are storing your
images on Amazon S3 via a custom ``MediaStorage`` subclass. You would probably
set your media endpoint like so:

.. code-block:: python

    # disable default behaviour
    RETURN_MEDIA_AS_BASE64_STRING = False

    # return media as URL instead
    RETURN_MEDIA_AS_URL = True

    # set up the desired media endpoint
    MEDIA_BASE_URL = 'https://s3-us-west-2.amazonaws.com'
    MEDIA_ENDPOINT = 'media'

Setting ``MEDIA_BASE_URL`` is optional. If no value is set, then
the API base address will be used when building the URL for ``MEDIA_ENDPOINT``.

.. _partial_request:

部分媒体下载
~~~~~~~~~~~~~~~~~~~~~~~
When files are served at a dedicated endpoint, clients can request partial
downloads. This allows them to provide features such as optimized
pause/resume (with no need to restart the download). To perform a partial
download, make sure the ``Range`` header is added the the client request.

    .. code-block:: console

        $ curl http://localhost/media/yourfile -i -H "Range: bytes=0-10"
        HTTP/1.1 206 PARTIAL CONTENT
        Date: Sun, 20 Aug 2017 14:26:42 GMT
        Content-Type: audio/mp4
        Content-Length: 11
        Connection: keep-alive
        Content-Range: bytes 0-10/23671
        Last-Modified: Sat, 19 Aug 2017 03:25:36 GMT
        Accept-Ranges: bytes

        abcdefghilm

In the snippet above, we see curl requesting the first chunk of a file.

.. _projection_filestorage:

利用投影优化媒体文件的处理
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Clients and API maintainers can exploit the :ref:`projections` feature to
include/exclude media fields from response payloads.

Suppose that a client stored a document with an image. The image field is
called *image* and it is of ``media`` type. At a later time, the client wants
to retrieve the same document but, in order to optimize for speed and since the
image is cached already, it does not want to download the image along with the
document. It can do so by requesting the field to be trimmed out of the
response payload:

.. code-block:: console

    $ curl -i http://example.com/people/<id>?projection={"image": 0}
    HTTP/1.1 200 OK

The document will be returned with all its fields except the *image* field.

Moreover, when setting the ``datasource`` property for any given resource
endpoint it is possible to explicitly exclude fields (of ``media`` type, but
also of any other type) from default responses:

.. code-block:: python

    people = {
        'datasource': {
            'projection': {'image': 0}
        },
        ...
    }

Now clients will have to explicitly request the image field to be included with
response payloads by sending requests like this one:

.. code-block:: console

    $ curl -i http://example.com/people/<id>?projection={"image": 1}
    HTTP/1.1 200 OK

.. admonition:: See also

    - :ref:`config`
    - :ref:`datasource`

    for details on the ``datasource`` setting.

.. _multipart:

关于媒体文件作为 ``multipart/form-data`` 的注意事项
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
If you are uploading media files as ``multipart/form-data`` all the
additional fields except the file fields will be treated as ``strings``
for all field validation purposes.  If you have already defined some of
the resource fields to be of different type (boolean, number, list etc)
the validation rules for these fields would fail, preventing you to
successffully submit your resource.

If you still want to be able to perform field validation in this case, you
will have to turn on ``MULTIPART_FORM_FIELDS_AS_JSON`` in your settings
file in order to treat the incoming fields as JSON encoded strings and still
be able to validate your fields.

Please note, that in case you indeed turn on ``MULTIPART_FORM_FIELDS_AS_JSON``
you will have to submit all resource fields as properly encoded JSON strings.

For example a ``number`` should be submited as ``1234`` (as you would normally
expect). A ``boolean`` will have to be send as ``true`` (note the lowercase
``t``). A ``list`` of strings as ``["abc", "xyz"]``. And finally
a ``string``, which is the thing that will most likely trip, you will have
to be submitted as ``"'abc'"`` (note that it is surrounded with double
quotes). If ever in doubt if what you are submitting is a valid JSON string
you can try passing it from the JSON Validator at http://jsonlint.com/ to
be sure that it is correct.

.. _media_lists:

使用媒体列表
~~~~~~~~~~~~~~~~~~~~
When using lists of media, there is no way to submit these in the default
configuration. Enable ``AUTO_COLLAPSE_MULTI_KEYS`` and ``AUTO_CREATE_LISTS``
to make this possible. This allows to send multiple values for one key in
``multipart/form-data`` requests and in this way upload a list of files.

.. _geojson_feature:

GeoJSON
-------
The MongoDB data layer supports geographic data structures
encoded in GeoJSON_ format. All GeoJSON objects supported by MongoDB_ are available:

    - ``Point``
    - ``Multipoint``
    - ``LineString``
    - ``MultiLineString``
    - ``Polygon``
    - ``MultiPolygon``
    - ``GeometryCollection``

All these objects are implemented as native Eve data types (see :ref:`schema`)
so they are are subject to the proper validation.

In the example below we are extending the `people` endpoint by adding
a ``location`` field of type Point_.

.. code-block:: javascript

    people = {
    	...
        'location': {
            'type': 'point'
        },
        ...
    }

Storing a contact along with its location is pretty straightforward:

.. code-block:: console

    $ curl -d '[{"firstname": "barack", "lastname": "obama", "location": {"type":"Point","coordinates":[100.0,10.0]}}]' -H 'Content-Type: application/json'  http://127.0.0.1:5000/people
    HTTP/1.1 201 OK

Eve also supports GeoJSON ``Feature`` and ``FeatureCollection`` objects, which
are not explicitely mentioned in MongoDB_ documentation. GeoJSON specification
allows object to contain any number of members (name/value pairs). Eve
validation was implemented to be more strict, allowing only two members. This
restriction can be disabled by setting ``ALLOW_CUSTOM_FIELDS_IN_GEOJSON`` to
``True``.

查询 GeoJSON 数据
~~~~~~~~~~~~~~~~~~~~~
As a general rule all MongoDB `geospatial query operators`_ and their associated
geometry specifiers are supported. In this example we are using the `$near`_
operator to query for all contacts living in a location within 1000 meters from
a certain point:

::

    ?where={"location": {"$near": {"$geometry": {"type":"Point", "coordinates": [10.0, 20.0]}, "$maxDistance": 1000}}}

Please refer to MongoDB documentation for details on geo queries.

.. _internal_resources:

内部资源
------------------
By default responses to GET requests to the home endpoint will include all the
resources. The ``internal_resource`` setting keyword, however, allows you to
make an endpoint internal, available only for internal data manipulation: no
HTTP calls can be made against it and it will be excluded from the ``HATEOAS``
links.

An usage example would be a mechanism for logging all inserts happening in
the system, something that can be used for auditing or a notification system.
First we define an ``internal_transaction`` endpoint, which is flagged as an
``internal_resource``:

.. code-block:: python
   :emphasize-lines: 10

    internal_transactions = {
        'schema': {
            'entities': {
                'type': 'list',
            },
            'original_resource': {
                'type': 'string',
            },
        },
        'internal_resource': True
    }


Now, if we access the home endpoint and ``HATEOAS`` is enabled, we won't get
the ``internal-transactions`` listed (and hitting the endpoint via HTTP will
return a ``404``.) We can use the data layer to access our secret endpoint.
Something like this:

.. code-block:: python
   :emphasize-lines: 12

    from eve import Eve

    def on_generic_inserted(self, resource, documents):
        if resource != 'internal_transactions':
            dt = datetime.now()
            transaction = {
                'entities':  [document['_id'] for document in documents],
                'original_resource': resource,
                config.LAST_UPDATED: dt,
                config.DATE_CREATED: dt,
            }
            app.data.insert('internal_transactions', [transaction])

    app = Eve()
    app.on_inserted += self.on_generic_inserted

    app.run()

I admit that this example is as rudimentary as it can get, but hopefully it
will get the point across.

.. _logging:

增强的日志记录
----------------
A number of events are available for logging via the default application
logger. The standard `LogRecord attributes`_ are extended with a few request
attributes:

.. tabularcolumns:: |p{6.5cm}|p{8.5cm}|

=================================== =========================================
``clientip``                        IP address of the client performing the
                                    request.

``url``                             Full request URL, eventual query parameters
                                    included.

``method``                          Request method (``POST``, ``GET``, etc.)

=================================== =========================================


You can use these fields when logging to a file or any other destination.

Callback functions can also take advantage of the builtin logger. The following
example logs application events to a file, and also logs custom messages every
time a custom function is invoked.

.. code-block:: python

    import logging

    from eve import Eve

    def log_every_get(resource, request, payload):
        # custom INFO-level message is sent to the log file
        app.logger.info('We just answered to a GET request!')

    app = Eve()
    app.on_post_GET += log_every_get

    if __name__ == '__main__':

        # enable logging to 'app.log' file
        handler = logging.FileHandler('app.log')

        # set a custom log format, and add request
        # metadata to each log line
        handler.setFormatter(logging.Formatter(
            '%(asctime)s %(levelname)s: %(message)s '
            '[in %(filename)s:%(lineno)d] -- ip: %(clientip)s, '
            'url: %(url)s, method:%(method)s'))

        # the default log level is set to WARNING, so
        # we have to explicitly set the logging level
        # to INFO to get our custom message logged.
        app.logger.setLevel(logging.INFO)

        # append the handler to the default application logger
        app.logger.addHandler(handler)

        # let's go
        app.run()


Currently only exceptions raised by the MongoDB layer and ``POST``, ``PATCH``
and ``PUT`` methods are logged. The idea is to also add some ``INFO`` and
possibly ``DEBUG`` level events in the future.

.. _oplog:

操作日志
--------------
The OpLog is an API-wide log of all edit operations. Every ``POST``, ``PATCH``
``PUT`` and ``DELETE`` operation can be recorded to the oplog. At its core the
oplog is simply a server log. What makes it a little bit different is that it
can be exposed as a read-only endpoint, thus allowing clients to query it as
they would with any other API endpoint.

Every oplog entry contains information about the document and the operation:

- Operation performed
- Unique ID of the document
- Update date
- Creation date
- Resource endpoint URL
- User token, if :ref:`user-restricted` is enabled for the endpoint
- Optional custom data

Like any other API-maintained document, oplog entries also expose:

- Entry ID
- ETag
- HATEOAS fields if that's enabled.

If ``OPLOG_AUDIT`` is enabled entries also expose:

- client IP
- Username or token, if available
- changes applied to the document (for ``DELETE`` the whole document is included).

A typical oplog entry looks like this:

.. code-block:: python

    {
        "o": "DELETE",
        "r": "people",
        "i": "542d118938345b614ea75b3c",
        "c": {...},
        "ip": "127.0.0.1",
        "u": "admin",
        "_updated": "Fri, 03 Oct 2014 08:16:52 GMT",
        "_created": "Fri, 03 Oct 2014 08:16:52 GMT",
        "_etag": "e17218fbca41cb0ee6a5a5933fb9ee4f4ca7e5d6"
        "_id": "542e5b7438345b6dadf95ba5",
        "_links": {...},
    }

To save a little space (at least on MongoDB) field names have been shortened:

- ``o`` stands for operation performed
- ``r`` stands for resource endpoint
- ``i`` stands for document id
- ``ip`` is the client IP
- ``u`` stands for user (or token)
- ``c`` stands for changes occurred
- ``extra`` is an optional field which you can use to store custom data

``_created`` and ``_updated`` are relative to the target document, which comes
handy in a variety of scenarios (like when the oplog is available to clients,
more on this later).

Please note that by default the ``c`` (changes) field is not included for
``POST`` operations. You can add ``POST`` to the ``OPLOG_CHANGE_METHODS``
setting (see :ref:`global`) if you wish the whole document to be included on
every insertion.

oplog 是如何操作的?
~~~~~~~~~~~~~~~~~~~~~~~~~~
Seven settings are dedicated to the OpLog:

- ``OPLOG`` switches the oplog feature on and off. Defaults to ``False``.
- ``OPLOG_NAME`` is the name of the oplog collection on the database. Defaults to ``oplog``.
- ``OPLOG_METHODS`` is a list of HTTP methods to be logged. Defaults to all of them.
- ``OPLOG_ENDPOINT`` is the endpoint name. Defaults to ``None``.
- ``OPLOG_AUDIT`` if enabled, IP addresses and changes are also logged. Defaults to ``True``.
- ``OPLOG_CHANGE_METHODS`` determines which methods will log changes. Defaults to ['PATCH', 'PUT', 'DELETE'].
- ``OPLOG_RETURN_EXTRA_FIELD`` determines if the optional ``extra`` field
  should be returned by the ``OPLOG_ENDPOINT``. Defaults to ``False``.

As you can see the oplog feature is turned off by default. Also, since
``OPLOG_ENDPOINT`` defaults to ``None``, even if you switch the feature on no
public oplog endpoint will be available. You will have to explicitly set the
endpoint name in order to expose your oplog to the public.

Oplog 终结点
~~~~~~~~~~~~~~~~~~
Since the oplog endpoint is nothing but a standard API endpoint, you can
customize it. This allows for setting up custom authentication (you might want
this resource to be only accessible for administrative purposes) or any other
useful setting.

Note that while you can change most of its settings, the endpoint will always
be read-only so setting either ``resource_methods`` or ``item_methods`` to
something other than ``['GET']`` will serve no purpose. Also, unless you need to
customize it, adding an oplog entry to the domain is not really necessary as it
will be added for you automatically.

Exposing the oplog as an endpoint could be useful in scenarios where you have
multiple clients (say phone, tablet, web and desktop apps) which need to stay
in sync with each other and the server. Instead of hitting every single
endpoint they could just access the oplog to learn all that's happened
since their last access. That’s a single request versus several. This is not
always the best approach a client could take. Sometimes it is probably better
to only query for changes on a certain endpoint. That's also possible, just
query the oplog for changes occured on that endpoint.

Extending Oplog entries
~~~~~~~~~~~~~~~~~~~~~~~
Every time the oplog is about to be updated the ``on_oplog_push`` event is fired.
You can hook one or more callback functions to this event. Callbacks receive
``resource`` and ``entries`` as arguments. The former is the resource name
while the latter is a list of oplog entries which are about to be written to
disk.

Your callback can add an optional ``extra`` field to canonical oplog entries.
The field can be of any type. In this example we are adding a custom dict to
each entry:

.. code-block:: python

    def oplog_extras(resource, entries):
        for entry in entries:
            entry['extra'] = {'myfield': 'myvalue'}

    app = Eve()

    app.on_oplog_push += oplog_extras
    app.run()

Please note that unless you explicitly set ``OPLOG_RETURN_EXTRA_FIELD`` to
``True``, the ``extra`` field will *not* be returned by the ``OPLOG_ENDPOINT``.

.. note::

    Are you on MongoDB? Consider making the oplog a `capped collection`_. Also,
    in case you are wondering yes, the Eve oplog is blatantly inspired by the
    awesome `Replica Set Oplog`_.

.. _schema_endpoint:

模式终结点
-------------------
通过启用 Eve 的模式终结点，可以将资源模式公开给 API 客户端。为此，将 
``SCHEMA_ENDPOINT`` 配置选项设置为要从中提供模式数据的 API 端点名称。启用后，Eve 将
把终结点当作一个只读资源，其中包含 JSON 编码的 Cerberus 模式定义，并按资源名称索引。
将启用资源可见性和授权设置，因此无法在模式终结点访问内部资源或请求没有读取身份验证的资
源。默认情况下，``SCHEMA_ENDPOINT`` 被设置为 ``None``。

.. _aggregation:

MongoDB 聚合框架
-----------------------------
Support for the `MongoDB Aggregation Framework`_ is built-in. In the example
below (taken from PyMongo) we’ll perform a simple aggregation to count the
number of occurrences for each tag in the tags array, across the entire
collection. To achieve this we need to pass in three operations to the
pipeline. First, we need to unwind the tags array, then group by the tags and
sum them up, finally we sort by count.

As python dictionaries don’t maintain order you should use ``SON`` or
collections ``OrderedDict`` where explicit ordering is required eg ``$sort``:

::

    posts = {
        'datasource': {
            'aggregation': {
                'pipeline': [
                    {"$unwind": "$tags"},
                    {"$group": {"_id": "$tags", "count": {"$sum": 1}}},
                    {"$sort": SON([("count", -1), ("_id", -1)])}
                ]
            }
        }
    }

The pipeline above is static. You have the option to allow for dynamic
pipelines, whereas the client will directly influence the aggregation results.
Let's update the pipeline a little bit:

::

    posts = {
        'datasource': {
            'aggregation': {
                'pipeline': [
                    {"$unwind": "$tags"},
                    {"$group": {"_id": "$tags", "count": {"$sum": "$value"}}},
                    {"$sort": SON([("count", -1), ("_id", -1)])}
                ]
            }
        }
    }

As you can see the `count` field is now going to sum the value of ``$value``,
which will be set by the client upon performing the request:

::

    $ curl -i http://example.com/posts?aggregate={"$value": 2}

The request above will cause the aggregation to be executed on the server with
a `count` field configured as if it was a static ``{"$sum": 2}``. The client
simply adds the ``aggregate`` query parameter and then passes a dictionary with
field/value pairs. Like with all other keywords, you can change ``aggregate``
to a keyword of your liking, just set ``QUERY_AGGREGATION`` in your settings.

You can also set all options natively supported by PyMongo. For more
information on aggregation see :ref:`datasource`.

You can pass ``{}`` to fields which you want to ignore. Considering the following pipelines:

::

    posts = {
        'datasource': {
            'aggregation': {
                'pipeline': [
                    {"$match": { "name": "$name", "time": "$time"}}
                    {"$unwind": "$tags"},
                    {"$group": {"_id": "$tags", "count": {"$sum": 1}}},
                ]
            }
        }
    }

If performing the following request:

::

    $ curl -i http://example.com/posts?aggregate={"$name": {"$regex": "Apple"}, "$time": {}}

The stage ``{"$match": { "name": "$name", "time": "$time"}}`` in the pipeline will be executed as ``{"$match": { "name": {"$regex": "Apple"}}}``. And for the following request:

::

    $ curl -i http://example.com/posts?aggregate={"$name": {}, "$time": {}}

The stage ``{"$match": { "name": "$name", "time": "$time"}}`` in the pipeline will be completely skipped.

The request above will ignore ``"count": {"$sum": "$value"}}``. A
Custom callback functions can be attached to the ``before_aggregation`` and ``after_aggregation`` event hooks. For more information, see :ref:`aggregation_hooks`.

Limitations
~~~~~~~~~~~
Client pagination (``?page=2``) is enabled by default. This is currently
achieved by injecting a ``$facet`` stage contianing two sub-pipelines,
total_count (``$count``) and paginated_results (``$limit`` first, then ``$skip``)
to the very end of the aggregation pipeline after the ``before_aggregation`` hook.
You can turn pagination off by setting ``pagination`` to ``False`` for the endpoint. Keep in mind that, when pagination
is disabled, all aggregation results are included with every response.
Disabling pagination might be appropriate (and actually advisable) only if the
expected response payload is not huge.

Client sorting (``?sort=field1``) is not supported at aggregation endpoints.
You can of course add one or more ``$sort`` stages to the pipeline, as we did
with the example above. If you do add a ``$sort`` stage to the pipeline,
consider adding it at the end of the pipeline. According to MongoDB's ``$limit``
documentation (link_):

    When a ``$sort`` immediately precedes a ``$limit`` in the pipeline, the
    sort operation only maintains the top **n** results as it progresses, where
    **n** is the specified limit, and MongoDB only needs to store **n** items
    in memory.

As we just saw earlier, pagination adds a ``$limit`` stage to the end of the
pipeline. So if pagination is enabled and ``$sort`` is the last stage of your
pipeline, then the resulting combined pipeline should be optimized.

A single endpoint cannot serve both regular and aggregation results. However,
since it is possible to setup multiple endpoints all serving from the same
datasource (see :ref:`source`), similar functionality can be easily achieved.


MongoDB 和 SQL 支持
------------------------
原生支持对单台或多台 MongoDB 数据库/服务器的。
SQLAlchemy 扩展提供了对 SQL 后端的支持。其他的数据层也相对容易地开发。访问
`extensions page`_ 来获取社区开发的数据层和扩展的列表。

Flask 技术支持
----------------
Eve 基于 Flask_ 微 web 框架。事实上，Eve 本身就是一个 Flask 子类，这意味着 Eve 公开
了 Flask 的所有功能和良好的细节，比如内置的开发服务器和 debugger_ ，对于 unittesting_ 
和一个 `extensive documentation`_ 的集成支持。

.. _HATEOAS: http://en.wikipedia.org/wiki/HATEOAS
.. _Cerberus: https://github.com/pyeve/cerberus
.. _REST: http://en.wikipedia.org/wiki/Representational_state_transfer
.. _CRUD: http://en.wikipedia.org/wiki/Create,_read,_update_and_delete
.. _`CORS`: http://en.wikipedia.org/wiki/Cross-origin_resource_sharing
.. _`PostgreSQL effort`: https://github.com/pyeve/eve/issues/17
.. _Flask: http://flask.pocoo.org
.. _debugger: http://flask.pocoo.org/docs/quickstart/#debug-mode
.. _unittesting: http://flask.pocoo.org/docs/testing/
.. _`extensive documentation`: http://flask.pocoo.org/docs/
.. _`this`: https://speakerdeck.com/nicola/developing-restful-web-apis-with-python-flask-and-mongodb?slide=113
.. _Events: https://github.com/pyeve/events
.. _`MongoDB Data Model Design`: http://docs.mongodb.org/manual/core/data-model-design
.. _GridFS: http://docs.mongodb.org/manual/core/gridfs/
.. _MediaStorage: https://github.com/pyeve/eve/blob/develop/eve/io/media.py
.. _`driver documentation`: http://api.mongodb.org/python/2.7rc0/api/gridfs/grid_file.html#gridfs.grid_file.GridOut
.. _GeoJSON: http://geojson.org/
.. _Point: http://geojson.org/geojson-spec.html#point
.. _MongoDB: http://docs.mongodb.org/manual/applications/geospatial-indexes/#geojson-objects
.. _`geospatial query operators`: http://docs.mongodb.org/manual/reference/operator/query-geospatial/#query-selectors
.. _$near: http://docs.mongodb.org/manual/reference/operator/query/near/#op._S_near
.. _`capped collection`: http://docs.mongodb.org/manual/core/capped-collections/
.. _`Replica Set Oplog`: http://docs.mongodb.org/manual/core/replica-set-oplog/
.. _`extensions page`: http://python-eve.org/extensions
.. _source: http://en.wikipedia.org/wiki/JSONP
.. _`LogRecord attributes`: https://docs.python.org/2/library/logging.html#logrecord-attributes
.. _`MongoDB Aggregation Framework`: https://docs.mongodb.org/v3.0/applications/aggregation/
.. _link: https://docs.mongodb.org/manual/reference/operator/aggregation/limit/
.. _`MongoDB queries`: https://docs.mongodb.com/v3.2/reference/operator/query/
