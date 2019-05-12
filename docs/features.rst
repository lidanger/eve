特性
========
下面是任何 EVE 支持的 API 可以暴露的主要特性的一个列表。这些特性中的大多数可以通过使用 Demo API (参考 :ref:`demo`) 来现场体验。

Emphasis on REST
----------------
Eve 项目的目标使提供最大可能的兼容 REST 的 API 的实现。基本的 REST_ 原则，像
*关注分离*，*无状态和分层的系统*，*可缓存*，*统一接口*，在涉及核心 API 时已经被列入
考虑了。

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
也可以自定义 URI，这样 API 终结点可以变成，``example.com/customers/overseas`` 。 
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
都包含一个 ``_links`` 节。链接提供了关于它们相对于被访问资源的 ``关系`` 的详细信息和一
个 ``标题`` 。然后，客户端可以使用关系和标题动态更新 UI，或者在不知道 API 结构的情况下
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
源，除非他们为试图编辑的资源提供最新的 ``ETag`` 。这可以防止用过时的版本覆盖项。

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

当多个文档被提交时，API 利用了 MongoDB *批量插入* 功能，这意味着不仅是只有一个
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
regex，例如 ``X_DOMAINS_RE = ['^http://sub-\d{3}\.example\.com$']``。

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
有关更多信息，请参见 :ref:`schema` 中的 ``default`` 和 ``nullable`` 关键字。

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

软删除
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
启用软删除后，对单个项和资源的 DELETE 请求的响应与对传统 “硬” 删除的响应一样。然而，
在后台，Eve 并没有从数据库中去除已删除的条目，而是将文档的 ``_deleted`` 元字段设置
为 ``true``。(``_deleted`` 字段的名称是可配置的。参见 :ref:`global`。) 当软删除
是启用的，所有请求都过滤或以其他方式帐户的 ``_deleted`` 字段。（All requests
made when soft delete is enabled filter against or otherwise account for the
``_deleted`` field.）

启用软删除时创建的所有文档都自动添加 ``_deleted`` 字段并初始化为 ``false``。在启用
软删除之前创建因此没有在数据库中定义 ``_deleted`` 字段的文档，在 API 响应数据中仍然
包含 ``_deleted: false``，这是 Eve 在响应构建期间添加的。对这些文档的 PUT 或 
PATCH 将把 ``_deleted`` 字段添加到存储的文档中，设置为 ``false``。

对软删除文档的 GET 请求的响应与丢失或 “硬” 删除文档的响应略有不同。对软删除文档的 
GET 请求仍然会以 ``404 Not Found`` 状态码作为响应，但是响应主体将包含带有 
``_deleted: true`` 的软删除文档。无论默认设置或请求的 ``embedded`` 查询参数的内容
是什么，嵌入到已删除文档中的文档都不会在响应中展开。这是为了确保 ``404`` 响应中包含
的软删除文档反映文档被删除时的状态，并且在更新嵌入文档时不会更改。

默认情况下，资源级别 GET 请求的响应中不会包含软删除项。此行为与 “硬” 删除后的请求匹配。
如果需要在响应中包含已删除的项，可以将 ``show_deleted`` 查询参数添加到请求中。
(``show_delete`` 参数名称是可配置的。参见 :ref:`global`) Eve 将响应所有文档，无论
是否已删除，由客户端解析返回的文档 ``_deleted`` 字段。``_deleted`` 字段也可以在请求
中使用 ``?where={"_deleted": true}`` 查询显式过滤，只允许返回已删除的文档。

软删除是在数据层强制执行的，这意味着应用程序代码使用 ``app.data find_one`` 和 
``app.data.find`` 方法进行的查询都将自动过滤掉软删除项。传递一个带有 
``req.show_deleted == True`` 的请求对象或在 ``_deleted`` 字段上显式筛选的查找字典
将覆盖默认筛选。

恢复软删除项
~~~~~~~~~~~~~~~~~~~~~~~~~~~~
对软删除文档的 PUT 或 PATCH 请求将恢复它，自动将数据库中的 ``_deleted`` 设置为 
``false``。不需要 (或不允许) 直接修改 ``_deleted`` 字段。例如，使用 PATCH 请求，只
有要在恢复版本中更改的字段才会被更改指定，否则将发出空请求以恢复文档的原样。对软删除
文档的写权限请求必须经过适当的授权，否则将被拒绝。

要知道，如果以前软删除文档应该恢复，最终唯一字段有机会被复制在两个不同的文件: 一个是恢
复的，另一个是可能使用相同的字段值存储而在原来 (现在已恢复) 是 “删除” 状态的。这是因为
在为新文档或更新的文档执行 `惟一` 规则时将忽略软删除的文档。


版本控制
~~~~~~~~~~
软删除一个版本化的文档将创建该文档的一个新版本，并将 ``deleted`` 设置为 ``true``。如上
所述，对已删除版本的 GET 请求将收到 ``404 Not Found`` 响应，而以前的版本将继续响应 
``200 OK``。对 ``?version=diff`` 或 ``?version=all`` 的响应将包含删除的版本，就像它
包含任何其他版本一样。

数据关系
~~~~~~~~~~~~~~
Eve ``data_relationship`` 验证器不允许引用已被软删除的文档。试图创建或更新引用软删除
文档的文档将失败，就像该文档已被硬删除一样。与软删除文档的现有数据关系仍然存在于数据库中，
但是需要对这些关系进行嵌入式文档序列化的请求将解析为空值。同样，这与硬删除文档的关系行为
相匹配。

与已删除文档版本的版本化数据关系也将无法验证，但允许与删除之前或恢复文档之后的版本的关系，
并将继续成功解析。

注意事项
~~~~~~~~~~~~~~
在应用程序中使用后禁用软删除需要数据库维护，以确保 API 保持一致。禁用软删除后，请求将不
再过滤或处理 ``_deleted`` 字段，被软删除的文档将再次在 API 上活动。因此，在禁用软删除时，
有必要执行数据迁移，删除所有带有 ``_deleted == True`` 的文档，并建议从 
``_deleted == False`` 的文档中删除 ``_deleted`` 字段。在现有应用程序中启用软删除是安
全的，并将维护从那时起删除的文档。

.. _eventhooks:

事件钩子
-----------
Pre-Request 事件钩子
~~~~~~~~~~~~~~~~~~~~~~~
当接收到 GET/HEAD、POST、PATCH、PUT、DELETE 请求时，会引发 ``on_pre_<method>`` 和
 ``on_pre_<method>_<resource>`` 事件。你可以使用多个回调函数来订阅这些事件。

.. code-block:: pycon

    >>> def pre_get_callback(resource, request, lookup):
    ...  print('A GET request on the "%s" endpoint has just been received!' % resource)

    >>> def pre_contacts_get_callback(request, lookup):
    ...  print('A GET request on the contacts endpoint has just been received!')

    >>> app = Eve()

    >>> app.on_pre_GET += pre_get_callback
    >>> app.on_pre_GET_contacts += pre_contacts_get_callback

    >>> app.run()

回调函数将接收被请求的资源，即原始的 ``flask.request`` 对象和当前查找字典作为参数 
(唯一的例外是 ``on_pre_POST`` 钩子不提供 ``lookup`` 参数)。

动态查询过滤器
^^^^^^^^^^^^^^^^^^^^^^
由于数据层将使用 ``lookup`` 字典来检索资源文档，因此开发人员可以选择修改它，以便向
查找性查询添加自定义逻辑。

.. code-block:: python

    def pre_GET(resource, request, lookup):
        # 只返回带有 'username' 字段的文档.
        lookup["username"] = {'$exists': True}

    app = Eve()

    app.on_pre_GET += pre_GET
    app.run()

在运行时更改查找字典将产生与通过配置应用 :ref:`filter` 的类似效果。但是，你只能通过
配置设置静态过滤器，而通过连接到 ``on_pre_<METHOD>`` 事件，你可以设置动态过滤器，这
允许额外的灵活性。

Post-Request 事件钩子
~~~~~~~~~~~~~~~~~~~~~~~~
当执行 GET、POST、PATCH、PUT、DELETE 方法时，``on_post_<method>`` 和 
``on_post_<method>_<resource>`` 事件同时被触发。你可以使用多个回调函数来订阅这些事件。
回调函数将接收被访问的资源，即原始的 `flask.request` 对象和响应负载。

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

数据库事件钩子的工作原理类似于请求事件钩子。这些事件在数据库操作之前和之后触发。下面
是一个如何配置事件的例子:

.. code-block:: pycon

   >>> def add_signature(resource, response):
   ...     response['SIGNATURE'] = "A %s from eve" % resource

   >>> app = Eve()
   >>> app.on_fetched_item += add_signature

你可以使用 flask 的 ``abort()`` 方法来中断数据库操作:

.. code-block:: pycon

   >>> from flask import abort

   >>> def check_update_access(resource, updates, original):
   ...     abort(403)

   >>> app = Eve()
   >>> app.on_insert_item += check_update_access

如果操作对资源和数据项都可用，就会触发响应的事件。每个动作都会触发两个事件:

- 通用的: ``on_<action_name>``
- 带有资源名称的: ``on_<action_name>_<resource_name>``

让我们来看看可用事件的概述:

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

这些是带有方法签名的 fetch 事件:

- ``on_fetched_resource(resource_name, response)``
- ``on_fetched_resource_<resource_name>(response)``
- ``on_fetched_item(resource_name, response)``
- ``on_fetched_item_<resource_name>(response)``
- ``on_fetched_diffs(resource_name, response)``
- ``on_fetched_diffs_<resource_name>(response)``

当项目刚从数据库中读取并即将发送到客户端时，将引发这些事件。注册的回调函数可以在项目
返回给客户端之前根据需要操作它们。

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

需要注意的是，对于特定的文档版本，如 ``?version=5`` 和所有带有 ``?version=all`` 
的文档版本来说，数据项 fetch 事件将与 `Document Versioning`_ 一起工作。使用
 ``?version=diffs`` 访问所有版本的 diffs 只适用于 diffs 获取事件。注意，diffs 
返回应该在回调中处理的部分文档。


Insert 事件
^^^^^^^^^^^^^

这些是带有方法签名的 insert 事件:

- ``on_insert(resource_name, items)``
- ``on_insert_<resource_name>(items)``
- ``on_inserted(resource_name, items)``
- ``on_inserted_<resource_name>(items)``

当一个 POST 请求到达 API，并且新的条目即将被存储到数据库中时，这些事件将被触发:

- ``on_insert`` 用于每一个资源终结点。
- ``on_insert_<resource_name>`` 用于指定的 `<resource_name>` 资源终结点。

回调函数可以挂接到这些事件中，以任意添加新字段或编辑现有字段。

插入项目后，触发以下两个事件:

- ``on_inserted`` 用于每一个资源终结点。
- ``on_inserted_<resource_name>`` 用于指定的 `<resource_name>` 资源终结点。

.. 警告:: 验证错误

    作为参数传递给这些事件的数据项出现在列表中。并且只发送那些通过验证的项。

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

这些事带有方法签名的 replace 事件:

- ``on_replace(resource_name, item, original)``
- ``on_replace_<resource_name>(item, original)``
- ``on_replaced(resource_name, item, original)``
- ``on_replaced_<resource_name>(item, original)``

当 PUT 请求到达 API，并且在通过验证之后将要替换某个项时，将触发以下事件:

- ``on_replace`` 用于任何资源数据项终结点。
- ``on_replace_<resource_name>`` 用于指定的资源终结点。

`item` 是即将存储的新项目。`original` 是数据库中要被替换的项。回调函数可以挂接到
这些事件中，以任意添加或更新 `item` 字段，或执行其他辅助操作。

在数据项被替换之后，将触发以下两个事件:

- ``on_replaced`` 用于任何数据项终结点。
- ``on_replaced_<resource_name>`` 用于指定的资源终结点。

Update 事件
^^^^^^^^^^^^^

这些是带有方法签名的 update 事件:

- ``on_update(resource_name, updates, original)``
- ``on_update_<resource_name>(updates, original)``
- ``on_updated(resource_name, updates, original)``
- ``on_updated_<resource_name>(updates, original)``

当 PATCH 请求到达 API，并且数据项通过验证后即将更新时，这些事件会在项目更新之前触发:

- ``on_update`` 用于任何资源终结点。
- ``on_update_<resource_name>`` 只有在 `<resource_name>` 终结点被命中时触发。

这里的 `updates` 表示应用于该项的更新，而 `original` 则表示即将更新的数据库项。回调
函数可以挂接到这些事件中，以在 `updates` 中任意添加或更新字段，或执行其他辅助操作。

数据项更新 `后`:

- ``on_updated`` 由任何资源终结点触发。.
- ``on_updated_<resource_name>`` 只有在 `<resource_name>` 终结点被命中时触发。

.. 警告:: 请注意

    请注意，``last_modified`` 和 ``etag`` 头将始终与数据库中项的状态一致 (它们不会
    更新以反映回调函数最终应用的更改)。

Delete 事件
^^^^^^^^^^^^^

这些是带有方法签名的 delete 事件:

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

当 DELETE 请求到达一个数据项终结点，以及该项目被删除 “之前”，将触发以下事件:

- ``on_delete_item`` 用于请求命中的任何资源。
- ``on_delete_item_<resource_name>`` 用于 DELETE 命中的指定 `<resource_name>` 数据项的终结点。

数据项删除`之后`，``on_deleted_item(resource_name, item)`` 和 
``on_deleted_item_<resource_name>(item)`` 将被触发。

`item` 是被删除的项。回调函数可以挂接到这些事件中来执行辅助操作。不，此时你不能任意
中止删除操作 (你可能应该看看 :ref:`validation`，或者最终完全禁用删除命令)。

资源
.........

如果你足够的勇气在资源端点 (允许失败的整个集合在一个去) 启用 DELETE 命令，那么你可以
通过连接回调函数到 ``on_delete_resource (resource_name)`` 或 
``on_delete_resource_ < resource_name >()`` 钩子来获得对这样的灾难性事件的通知。

- ``on_delete_resource_originals`` 用于请求命中的任何资源，在检索到原始文档后触发。
- ``on_delete_resource_originals_<resource_name>`` 用于 DELETE 命中的指定 `<resource_name>` 资源终结点，在检索到原始文档后触发。

注意: 考虑到查找和原始列表，为了在实际删除操作之前执行一些业务逻辑，这两个事件非常有用

.. _aggregation_hooks:

聚合事件钩子
~~~~~~~~~~~~~~~~~~~~~~~
还可以将一个或多个回调附加到聚合终结点。当要执行聚合时，将触发 ``before_aggregation`` 
事件。任何附加的回调函数都将同时接收端点名称和聚合管道作为参数。如果需要，可以更改管道。

.. code-block:: pycon

    >>> def on_aggregate(endpoint, pipeline):
    ...   pipeline.append({"$unwind": "$tags"})

    >>> app = Eve()
    >>> app.before_aggregation += on_aggregate

``after_aggregation`` 事件在执行聚合时触发。附加的回调函数可以利用此事件在文档返回
给客户端之前修改文档。

.. code-block:: pycon

   >>> def alter_documents(endpoint, documents):
   ...   for document in documents:
   ...     document['hello'] = 'well, hello!'

   >>> app = Eve()
   >>> app.after_aggregation += alter_documents

有关聚合支持的更多信息，请参见 :ref:`aggregation`


.. 警告:: 请注意

    为了提供无缝的事件处理特性，Eve 依赖于 Events_ 包。

.. _ratelimiting:

速度限制
-------------
每个用户/方法都支持 API 速率限制。您可以为每个 HTTP 方法设置请求数量和时间窗口。
如果请求限制在时间窗口内被命中，API 将以 ``429 Request limit exceeded`` 响应，
直到计时器重置。用户是由身份验证头标识的，或者 (在缺少身份验证头时) 由客户机IP标识。
当启用速率限制时，每个 API 响应都会提供适当的 ``X-RateLimit-`` 头文件。假设速率限制
设置为每 15 分钟 300 个请求，这是用户在使用单个请求到达终结点后得到的结果:

::

    X-RateLimit-Remaining: 299
    X-RateLimit-Limit: 300
    X-RateLimit-Reset: 1370940300

您可以为每个受支持的方法 (GET、POST、PATCH、DELETE) 设置不同的限制。

.. 警告:: 请注意

   速率限制在默认情况下是禁用的，而启用时需要运行一个Redis服务器。一个关于速率限制
   的教程即将发布。

自定义 ID 字段
----------------
Eve 允许扩展其标准数据类型支持。在 :ref:`custom_ids` 教程中，我们看到了如何使用 UUID 
值代替 MongoDB 默认对象作为惟一的文档标识。

文件存储
------------
媒体文件 (图片、pdf 等) 可以作为 ``media`` 文档字段上传。和往常一样，上传通过
``POST``, ``PUT`` 和 ``PATCH`` 来完成，不过使用的是 ``multipart/form-data`` 上下
文类型。

让我们假设 ``accounts`` 终结点有这样一个模式:

.. code-block:: python

    accounts = {
        'name': {'type': 'string'},
        'pic': {'type': 'media'},
        ...
    }

使用 curl，我们会这样 ``POST``:

.. code-block:: console

    $ curl -F "name=john" -F "pic=@profile.jpg" http://example.com/accounts


对于优化的性能文件，默认情况下存储在 GridFS_ 中。可以实现自定义的 ``MediaStorage`` 
类并将其传递给应用程序，以支持其他存储系统。一个名为 ``FileSystemMediaStorage`` 
的类正在开发中，并将很快包含在 Eve 包中。

由于还没有合适的开发人员指南，如果你对开发自定义存储类感兴趣，可以查看 
MediaStorage_ 源文件。

以 Base64 字符串的形式提供媒体文件
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
当一个文档被请求时，媒体文件将以 Base64 字符串的形式返回，

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

但是，如果填充了 ``EXTENDED_MEDIA_INFO`` 列表 (默认情况下没有)，那么负载格式
将会不同。此标志允许从附加元字段的驱动程序传递。例如，使用 MongoDB 驱动程序，
像 ``content_type``, ``name`` 和 ``length`` 这样的字段可以添加到这个列表中，
并从底层驱动程序传递。

当使用 ``EXTENDED_MEDIA_INFO`` 时，字段将是一个字典，而文件本身存储在 ``file``
键下，其他键是元字段。假设标志是这样设置的:

.. code-block:: python

    EXTENDED_MEDIA_INFO = ['content_type', 'name', 'length']

那么，输出将会是像这样的东西

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

对于 MongoDB，可以在 `driver documentation`_ 中找到更多字段。

如果你有其他检索媒体文件的方法 (例如自定义 Flask 端点)，那么通过设置 
``RETURN_MEDIA_AS_BASE64_STRING`` 标志为 ``False``，可以在负载中将媒体文件排除在外。
这将考虑是否使用了 ``EXTENDED_MEDIA_INFO``。

在专用终结点上提供媒体文件
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
虽然返回嵌入为 Base64 字段的文件是默认行为，但是你可以选择在专用的媒体端点上提供这些文件。
你可以通过将 ``RETURN_MEDIA_AS_URL`` 设置为 ``True`` 来实现这一点。启用此功能时，文档
字段包含对应文件的 url，这些对应文件在媒体端点上提供服务。

你可以通过更新 ``MEDIA_BASE_URL`` 和 ``MEDIA_ENDPOINT`` 设置来更改默认媒体端点 
(``media``)。假设你通过自定义的 ``MediaStorage`` 子类将图像存储在 Amazon S3 上。你可
能会这样设置你的媒体端点:

.. code-block:: python

    # 禁用默认行为
    RETURN_MEDIA_AS_BASE64_STRING = False

    # 相反，返回媒体作为 URL
    RETURN_MEDIA_AS_URL = True

    # 创建需要的媒体终结点
    MEDIA_BASE_URL = 'https://s3-us-west-2.amazonaws.com'
    MEDIA_ENDPOINT = 'media'

``MEDIA_BASE_URL`` 设置是可选的。如果没有设置值，那么在为 ``MEDIA_ENDPOINT`` 构建 
URL 时将使用 API 基本地址。

.. _partial_request:

部分媒体下载
~~~~~~~~~~~~~~~~~~~~~~~
当文件在专用端点上提供时，客户端可以请求部分下载。这使它们可以提供一些特性，比如优化的
暂停/恢复 (不需要重新启动下载)。要执行部分下载，请确保在客户端请求中添加了 ``Range`` 
标头。

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

在上面的代码片段中，我们看到 curl 请求了文件的第一个块。

.. _projection_filestorage:

利用投影优化媒体文件的处理
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
客户端和 API 维护人员可以利用 :ref:`projections` 特性将媒体字段包含/排除在响应负载之外。

假设客户端存储了一个带有图像的文档。image 字段被称为 *image*，它的类型是 ``media``。
稍后，客户端希望检索相同的文档，但是为了优化速度，而且由于已经缓存了图像，所以它不希望
下载图像和文档。它可以通过请求将字段从响应负载中删除来做到这一点:

.. code-block:: console

    $ curl -i http://example.com/people/<id>?projection={"image": 0}
    HTTP/1.1 200 OK

文档将返回除 *image* 字段外的所有字段。

此外，当为任何给定资源终结点设置 ``datasource`` 属性时，可以显式地从默认响应中排除字段 
(``media`` 类型的字段，也可以是任何其他类型的字段):

.. code-block:: python

    people = {
        'datasource': {
            'projection': {'image': 0}
        },
        ...
    }

现在，客户端必须通过发送如下请求，显式地请求包含在响应载荷中的图像字段:

.. code-block:: console

    $ curl -i http://example.com/people/<id>?projection={"image": 1}
    HTTP/1.1 200 OK

.. 警告:: 另请参阅

    - :ref:`config`
    - :ref:`datasource`

    获取有关 ``datasource`` 设置的详细信息。

.. _multipart:

关于媒体文件作为 ``multipart/form-data`` 的注意事项
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
如果你正在把媒体文件当作 ``multipart/form-data`` 上传，除了文件字段外，为了验证所有
字段的目的，所有其他字段都将被视为 ``string``。如果你已经定义了一些不同类型的资源字
段 (boolean、number、list 等)，那么这些字段的验证规则将会失败，从而阻止你成功提交资
源。

如果你仍然希望在这种情况下能够执行字段验证，你必须在设置文件中打开 
``MULTIPART_FORM_FIELDS_AS_JSON``，以便将传入的字段视为 JSON 编码的字符串，这样
仍然能够验证字段。

请注意，如果你确实打开了 ``MULTIPART_FORM_FIELDS_AS_JSON``，则必须将所有资源字段作为
正确编码的 JSON 字符串提交。

例如，``number`` 应该被提交为 ``1234`` (正如你通常所期望的那样)。``boolean`` 必须发
送为 ``true`` (注意小写字母 ``t``)。字符串 ``list`` 为 ``["abc"， "xyz"]``。最后是
一个 ``string``，这是最有可能出错的东西，你必须提交为 ``"'abc'"`` (注意，它被双引号
包围)。如果你对提交的是否是一个有效的 JSON 字符串有任何疑问，你可以尝试在 
http://jsonlint.com/ 的 JSON Validator 传递它，以确保它是正确的。

.. _media_lists:

使用媒体列表
~~~~~~~~~~~~~~~~~~~~
当使用媒体列表时，无法在默认配置中提交这些列表。启用 ``AUTO_COLLAPSE_MULTI_KEYS`` 和
``AUTO_CREATE_LISTS`` 使这成为可能。这允许在 ``multipart/form-data`` 请求中为一个键
发送多个值，并以这种方式上传文件列表。

.. _geojson_feature:

GeoJSON
-------
MongoDB 数据层支持 GeoJSON_ 格式编码的地理数据结构。所有 MongoDB_ 支持的 GeoJSON 对象
都是可用的:

    - ``Point``
    - ``Multipoint``
    - ``LineString``
    - ``MultiLineString``
    - ``Polygon``
    - ``MultiPolygon``
    - ``GeometryCollection``

所有这些对象都实现为原生 Eve 数据类型 (参见 :ref:`schema`)，因此它们都要经过适当的验证。

在下面的示例中，我们通过添加 Point_ 类型的 ``location`` 字段来扩展 `people` 终结点。

.. code-block:: javascript

    people = {
    	...
        'location': {
            'type': 'point'
        },
        ...
    }

存储联系人及其位置非常简单:

.. code-block:: console

    $ curl -d '[{"firstname": "barack", "lastname": "obama", "location": {"type":"Point","coordinates":[100.0,10.0]}}]' -H 'Content-Type: application/json'  http://127.0.0.1:5000/people
    HTTP/1.1 201 OK

Eve 还支持 GeoJSON 的 ``Feature`` 和 ``FeatureCollection`` 对象，这些对象在 
MongoDB_ 文档中没有明确提到。GeoJSON 规范允许对象包含任意数量的成员 (名称/值对)。
Eve 验证的实现更加严格，只允许两个成员。通过将 ``ALLOW_CUSTOM_FIELDS_IN_GEOJSON`` 
设置为 ``True``，可以禁用此限制。

查询 GeoJSON 数据
~~~~~~~~~~~~~~~~~~~~~
一般来说，所有 MongoDB 的 `geospatial query operators`_  及其相关的几何说明符都是支
持的。在这个例子中，我们使用 `$near`_ 操作符来查询所有居住在距离某个点 1000 米以内的
联系人:

::

    ?where={"location": {"$near": {"$geometry": {"type":"Point", "coordinates": [10.0, 20.0]}, "$maxDistance": 1000}}}

有关地理查询的详细信息，请参阅 MongoDB 文档。

.. _internal_resources:

内部资源
------------------
默认情况下，对 home 终结点的请求的响应将包含所有资源。然而，``internal_resource`` 
设置关键字允许你将端点设置为内部的，仅用于内部数据操作: 不能对它进行 HTTP 调用，而
且它将被排除在 ``HATEOAS`` 链接之外。

一个记录系统中发生的所有插入的机制的用法示例，可以用于审计或通知系统。首先，我们定义
一个 ``internal_transaction`` 终结点，它被标记为 ``internal_resource``:

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


现在，如果我们访问主端点而且启用了 ``HATEOAS``，我们将不会得到列出的 
``internal-transactions`` (通过 HTTP 访问终结点将返回 ``404``)。我们可以使用
数据层访问我们的秘密端点。像这样:

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

我承认这个例子是最基本的，但希望它能让人明白这一点。

.. _logging:

增强的日志记录
----------------
可以通过默认的应用程序日志记录器记录许多事件。标准的 `LogRecord attributes`_ 通过
几个请求属性被扩展了:

.. tabularcolumns:: |p{6.5cm}|p{8.5cm}|

=================================== =========================================
``clientip``                        执行请求的客户端的 IP 地址。

``url``                             完整的请求 URL，包括最终的查询参数。

``method``                          请求方法 (``POST``, ``GET``, 等等)

=================================== =========================================


您可以在将日志记录到文件或任何其他目的地时使用这些字段。

回调函数也可以利用内置日志记录器。下面的示例将应用程序事件记录到文件中，并在每次调用自
定义函数时记录自定义消息。

.. code-block:: python

    import logging

    from eve import Eve

    def log_every_get(resource, request, payload):
        # 自定义的 INFO 级别消息被发送到日志文件
        app.logger.info('We just answered to a GET request!')

    app = Eve()
    app.on_post_GET += log_every_get

    if __name__ == '__main__':

        # 启用记录日志到 'app.log' 文件
        handler = logging.FileHandler('app.log')

        # 设置一个自定义的日志格式，并添加请求元数据到每一行日志
        handler.setFormatter(logging.Formatter(
            '%(asctime)s %(levelname)s: %(message)s '
            '[in %(filename)s:%(lineno)d] -- ip: %(clientip)s, '
            'url: %(url)s, method:%(method)s'))

        # 默认的日志级别设置为 WARNING，因此我们必须将日志级别显式设置为 INFO，
        # 以获取自定义消息的日志记录.
        app.logger.setLevel(logging.INFO)

        # 将处理程序添加到默认的应用程序日志记录器
        app.logger.addHandler(handler)

        # 现在开始吧
        app.run()


目前，只有 MongoDB 层以及 ``POST``, ``PATCH`` 和 ``PUT`` 方法引发的异常才会被记录下
来。我们的想法是在将来添加一些 ``INFO`` 和 ``DEBUG`` 级别的事件。

.. _oplog:

操作日志
--------------
OpLog 是一个 API 范围的日志，记录所有编辑操作。每个 ``POST``, ``PATCH`` ``PUT`` 和 
``DELETE`` 操作都可以记录到 oplog 中。oplog 的核心只是一个服务器日志。不同之处在于，
它可以公开为只读终结点，从而允许客户端像查询任何其他 API 终结点一样查询它。

每个 oplog 条目都包含关于文档和操作的信息:

- 已执行的操作
- 文档的唯一 ID
- 更新日期
- 创建日期
- 资源终结点 URL
- 用户令牌，如果整个终结点启用了 :ref:`user-restricted` 的话
- 可选的自定义数据

与任何其他 API 维护的文档一样，oplog 条目也公开:

- 条目 ID
- ETag
- HATEOAS 字段，如果启用的话。

如果启用了 ``OPLOG_AUDIT``，则还会公开以下条目:

- 客户端 IP
- 用户名或令牌，如果可用
- 应用到文档上的修改 (对于 ``DELETE`` 来说，包含整个文档)。

一个典型的 oplog 条目是这样的:

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

为了节省一点空间 (至少在 MongoDB 上)，字段名被缩短了:

- ``o`` 代表已执行的操作
- ``r`` 代表资源终结点
- ``i`` 代表文档 id
- ``ip`` 是 IP
- ``u`` 代表用户 (或令牌)
- ``c`` 代表发生的变化
- ``extra`` 是一个可选字段，你可以使用它来存储自定义数据

``_created`` 和 ``_updated`` 是相对于目标文档的，这在很多场景中都很方便 (比如当 
oplog 对客户端可用时，后面会详细介绍)。

请注意，在默认情况下，``c`` (更改) 字段不包括在 ``POST`` 操作中。如果希望在每次插入
时都包含整个文档，可以将 ``POST`` 添加到 ``OPLOG_CHANGE_METHODS`` 设置中 (参见 
:ref:`global`)。

oplog 是如何操作的?
~~~~~~~~~~~~~~~~~~~~~~~~~~
OpLog 有 7 个设置:

- ``OPLOG`` 打开和关闭 OPLOG 特性。默认为 ``False``。
- ``OPLOG_NAME`` 是数据库 oplog 集合的名称。默认为 ``oplog``。
- ``OPLOG_METHODS`` 是要记录的 HTTP 方法列表。默认为所有方法。
- ``OPLOG_ENDPOINT`` 是终结点名称。默认为 ``None``。
- ``OPLOG_AUDIT`` 如果启用，IP 地址和更改也会被记录下来。默认为 ``True``。
- ``OPLOG_CHANGE_METHODS`` 确定哪些方法将记录更改。默认为 ['PATCH', 'PUT', 'DELETE']。
- ``OPLOG_RETURN_EXTRA_FIELD`` 确定可选的 ``extra`` 字段是否应该由 ``OPLOG_ENDPOINT`` 返回。默认为 ``False``。

可以看到，oplog 特性在默认情况下是关闭的。此外，由于 ``OPLOG_ENDPOINT`` 默认为 
``None``，即使你在没有公共 oplog 终结点的情况下切换该特性，也不能使用它。你必须显式地
设置终结点名称，以便向公众公开你的 oplog。

Oplog 终结点
~~~~~~~~~~~~~~~~~~
由于 oplog 终结点只是一个标准 API 终结点，所以你可以定制它。这允许配置自定义身份验
证 (你可能希望此资源仅用于管理目的) 或任何其他有用的设置。

请注意，虽然你可以更改它的大多数设置，但终结点始终是只读的，因此将 ``resource_methods`` 
或 ``item_methods`` 设置为 ``['GET']`` 之外的其他值将不起任何作用。此外，除非需要自定
义，否则没有必要向域添加 oplog 条目，因为它将自动添加。

将 oplog 作为终结点公开，在有需要和彼此以及服务器保持同步的多个客户端 (例如电话、平板
电脑、web 和桌面应用程序) 的场景中可能很有用。他们可以访问 oplog 来了解自上次访问以来
发生的所有事情，而不是访问每个终结点。这是一个请求而不是多个请求。这并不总是客户端可以
采用的最佳方法。有时候，只查询某个终结点上的更改很可能会更好。只查询 oplog 以了解终结
点上发生的更改，也是可行的。

扩展 Oplog 条目
~~~~~~~~~~~~~~~~~~~~~~~
每次要更新 oplog 时，都会触发 ``on_oplog_push`` 事件。你可以将一个或多个回调函数挂到
此事件上。回调函数接收 ``resource`` 和 ``entries`` 作为参数。前者是资源名称，而后者
是即将写入磁盘的 oplog 条目列表。

你的回调函数可以向规范的 oplog 条目添加一个可选的 ``extra`` 字段。字段可以是任何类型。
在这个例子中，我们将一个自定义的字典添加到每个条目:

.. code-block:: python

    def oplog_extras(resource, entries):
        for entry in entries:
            entry['extra'] = {'myfield': 'myvalue'}

    app = Eve()

    app.on_oplog_push += oplog_extras
    app.run()

请注意，除非你显式地将 ``OPLOG_RETURN_EXTRA_FIELD`` 设置为 ``True``，否则 
``OPLOG_ENDPOINT`` 不会返回 ``extra`` 字段。

.. 注意::

    你在使用 MongoDB 吗? 考虑让 oplog 成为一个 `capped collection`_。另外，如果
    你想知道，是的，Eve oplog 显然是受到了很棒的 `Replica Set Oplog`_ 的启发。

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
内置了对 `MongoDB Aggregation Framework`_ 的支持。在下面的示例中 (取自 PyMongo)，我
们将执行一个简单的聚合，计算整个集合中每个标记在标记数组中出现的次数。为了实现这一点，
我们需要向管道传递三个操作。首先，我们需要展开标记数组，然后按标记分组并对它们求和，最
后按计数排序。

由于 python 字典不维护顺序，你应该在需要显式排序的地方使用 ``SON`` 或集合 
``OrderedDict``，如 ``$sort``:

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

上面的管道是静态的。你可以选择允许动态管道，这样客户端将直接影响聚合结果。让我们稍微
更新一下管道:

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

如你所见，``count`` 字段现在将会对 ``$value`` 的值求和，该值将由客户端在执行请求时设置:

::

    $ curl -i http://example.com/posts?aggregate={"$value": 2}

上面的请求将导致在服务器上执行聚合，并配置一个 `count`` 字段，就像它是一个静态 
``{"$sum": 2}`` 一样。客户端只需添加 ``aggregate`` 查询参数，然后传递一个含有字段/值对
的字典。与所有其他关键字一样，你可以将 ``aggregate`` 更改为你喜欢的关键字，只需在配置中
设置 ``QUERY_AGGREGATION`` 即可。

你还可以设置 PyMongo 本地支持的所有选项。有关聚合的更多信息，请参见 :ref:`datasource`。

您可以将 ``{}`` 传递给要忽略的字段。考虑以下管道:

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

如果执行以下请求:

::

    $ curl -i http://example.com/posts?aggregate={"$name": {"$regex": "Apple"}, "$time": {}}

管道中的阶段 ``{"$match": { "name": "$name", "time": "$time"}}`` 将作为 
``{"$match": { "name": {"$regex": "Apple"}}}`` 执行。而对于以下请求:

::

    $ curl -i http://example.com/posts?aggregate={"$name": {}, "$time": {}}

管道中的阶段 ``{"$match": { "name": "$name", "time": "$time"}}`` 将被完全跳过。

上面的请求将忽略 ``"count": {"$sum": "$value"}}``。可以将一个自定义回调函数附加到
``before_aggregation`` 和 ``after_aggregation`` 事件钩子。有关更多信息，请参见
:ref:`aggregation_hooks`。

边界
~~~~~~~~~~~
默认情况下启用了客户端分页 (``?page=2``)。目前，通过在 ``before_aggregation`` 钩
子后面，将 ``$facet`` 阶段包含的两个子管道，total_count (``$count``) 和 
paginated_results (首先是 ``$limit``，然后是 ``$skip``) 注入聚合管道的末尾，来实现
这一点。您可以通过将端点的 ``pagination`` 设置为 ``False`` 来关闭分页。请记住，禁
用分页时，每个响应都包含所有聚合结果。
只有当预期的响应负载不是很大时，禁用分页才可能是适当的 (实际上也是可取的)。

聚合终结点不支持客户端排序 (``?sort=field1``)。当然，你可以向管道中添加一个或多个
``$sort`` 阶段，就像我们在上面的示例中所做的那样。如果你确实要向管道添加 ``$sort`` 
阶段，请考虑在管道的末尾添加它。根据 MongoDB 的 ``$limit`` 文档 (link_):

    当管道中的 ``$sort`` 紧跟着 ``$limit`` 时，排序操作只维护最上面的 **n** 个结果，
    其中 **n** 是指定的限制，MongoDB 只需要在内存中存储 **n** 个数据项。

正如我们刚才看到的，分页在管道的末尾添加了一个 ``$limit`` 阶段。因此，如果启用了分页，
并且 ``$sort`` 是管道的最后一个阶段，那么最终的组合管道应该会被优化。

单个终结点不能同时提供常规结果和聚合结果。但是，由于可以设置多个终结点提供来自同一个数
据源的数据 (参见 :ref:`source`)，因此可以轻松实现类似的功能。


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
.. _`MongoDB 查询`: https://docs.mongodb.com/v3.2/reference/operator/query/
