.. _config:

配置
=============
一般来说，Eve 配置最好通过配置文件来进行。配置文件它们自己实际上是 Python 文件。但是，Eve 会首先给与基于字典的设置优先权，然后它会尝试找到在 :envvar:`EVE_SETTINGS` 环境变量 (如果设置了的话) 中定义的一个文件，最后它会尝试找到 `settings.py` 或一个文件名通过 `settings` 参数传递到构造函数的文件。

使用文件配置
------------------------
启动时，如果 `settings` 参数在构造函数中被忽略，Eve 会尝试寻找名为 `settings.py` 的文件，先是在应用程序文件夹，然后是应用程序的其中一个子文件夹。你可以选择一个其他的文件名/路径，只要在你实例化应用程序时将它作为参数传递。如果文件路径时相对的，Eve 会尝试递归查找你的 `sys.path` 的其中一个文件夹，因此，你必须确保你的应用程序根路径被附加到它上面了。这很有用，例如，在测试环境，当配置文件不需要被放在你的应用程序的根路径下时。

.. code-block:: python

    from eve import Eve

    app = Eve(settings='my_settings.py')
    app.run()

使用一个字典配置
-------------------------------
或者，你可以选择提供一个配置字典。不想通过配置文件配置 Eve，基于字典的途径只会使用你自己的值更新 Eve 的默认设置，而不是重写所有设置。

.. code-block:: python

    from eve import Eve

    my_settings = {
        'MONGO_HOST': 'localhost',
        'MONGO_PORT': 27017,
        'MONGO_DBNAME': 'the_db_name',
        'DOMAIN': {'contacts': {}}
    }

    app = Eve(settings=my_settings)
    app.run()

开发 / 生产
------------------------
大多数应用程序需要不止一个配置。至少，应该有针对生产和开发阶段的服务器的单独配置。最简单的处理这个的方式是使用一个默认的总是被加载并作为版本控制的一部分的配置，和一个单独的根据需要重载前值的配置。

这是为什么你可以通过 :envvar:`EVE_SETTINGS` 环境变量指向的文件重载或扩展配置的主要原因。开发/本地配置应该被存储在 `settings.py` 中，然后，在开发环境，你可以定义 EVE_SETTINGS=/path/to/production_setting.py，然后就结束了。

虽然如此，还是有很多可选的方式来处理开发/生产。使用 Python 模块来配置是非常方便的，因为它们允许各种不错的玩法，像能天衣无缝地在本地和生产系统上启动同一个 API，根据需要连接到恰当地数据库实例。考虑以下直接从 :ref:`demo` 拿来地示例:

::

    # 我们希望同时在本地和 Heroku 上无缝运行，因此:
    if os.environ.get('PORT'):
        # 我们驻在 Heroku 上! 使用 MongoHQ 沙盒作为我们的后端.
        MONGO_HOST = 'alex.mongohq.com'
        MONGO_PORT = 10047
        MONGO_USERNAME = '<user>'
        MONGO_PASSWORD = '<pw>'
        MONGO_DBNAME = '<dbname>'
    else:
        # 在本地机器上运行。让我们只使用本地 mongod 实例.

        # 请注意，MONGO_HOST 和 MONGO_PORT 可以被很好地省略，因为它们已经默认为一个本地 'mongod' 实例.
        MONGO_HOST = 'localhost'
        MONGO_PORT = 27017
        MONGO_USERNAME = 'user'
        MONGO_PASSWORD = 'user'
        MONGO_DBNAME = 'apitest'

.. _global:

全局设置
--------------------
除了定义一般的 API 行为，大多数全局设置配置项都用于定义标准终结点规则集合，稍后配置单个终结点时也可以进行微调。全局设置配置项总是大写。

.. tabularcolumns:: |p{6.5cm}|p{8.5cm}|

=================================== =========================================
``URL_PREFIX``                      用于所有 API 终结点的 URL 前缀。与 ``API_VERSION`` 
                                    结合，用于构建 API 终结点 (例如，``api`` 会被
                                    渲染到 ``/api/<endpoint>``)。默认为 ``''``。

``API_VERSION``                     API 版本。与 ``URL_PREFIX`` 结合，用于构建 API 
                                    终结点 (例如，``v1`` 会被渲染到 ``/v1/<endpoint>``)。
                                    默认为 ``''``。

``ALLOWED_FILTERS``                 允许过滤的字段列表。这个列表中的项按等级工作。
                                    这意味着，举个例子，如果 ``ALLOWED_FILTERS`` 包含
                                    ``'dict.sub_dict.foo``, ``'dict.sub_dict'``
                                    或 ``'dict'`` 中的一个，那么 ``'dict.sub_dict.foo'``
                                    上的过滤是允许的。相反，如果 ``ALLOWED_FILTERS``
                                    包含 ``'dict'``， ``'dict'`` 上的过滤是允许的。
                                    可以被设置为 ``[]`` (不允许任何过滤器)，或
                                    ``['*']`` (每个字段都允许过滤)。默认为 ``['*']``。

                                    *请注意:* 如果担心 API 破坏或 DB DoS 攻击，那么
                                    全局禁用过滤器，并在本地级别把有效的列为白名单是可行的。

``VALIDATE_FILTERS``                是否通过资源模式验证过滤器。无效的过滤器会抛出一个异常。
                                    默认为 ``False``.

                                    警告: 对涉及自定义规则或类型的过滤器
                                    表达式的验证可能会对性能造成很大影响。
                                    这是一个例子，例如，带有 ``data_relation`` 规则
                                    的字段。考虑从过滤器中排除任务繁重的字段 (参考
                                    ``ALLOWED_FILTERS``)。

``SORTING``                         如果 ``GET`` 请求支持排序，``True``，否则，``False``。
                                    可以被资源配置重载。默认为 ``True``。

``PAGINATION``                      如果 ``GET`` 请求启用了分页，``True``，否则，``False``。
                                    可以被资源配置重载。默认为 ``True``。

``PAGINATION_LIMIT``                QUERY_MAX_RESULTS 查询参数允许的最大值。
                                    超出限制的值会被悄悄替换为这个值。
                                    你希望在性能和传输大小之间做一个明智的妥协。
                                    默认为 50.

``PAGINATION_DEFAULT``              QUERY_MAX_RESULTS 的默认值。默认为 25。

``OPTIMIZE_PAGINATION_FOR_SPEED``   要改善分页性能设置这个为 ``True``。当激活优化时，
                                    在数据库中没有进行在大型集合很慢的计数操作。
                                    这确实有些后果。首先，不会返回文档计数。
                                    第二，``HATEOAS`` 时精度很低的: 没有最后一页链接可用，
                                    而下一页链接总是包含的，即使在最后一页。
                                    在大型集合中，打开这项特性可以大幅改善性能。
                                    默认为 ``False`` (性能更慢;
                                    包含文档计数; 精确的 ``HATEOAS``)。

``QUERY_WHERE``                     用于过滤查询参数的键。默认为 ``where``。

``QUERY_SORT``                      用于排序查询参数的键。默认为 ``sort``。

``QUERY_PROJECTION``                用于投影查询参数的键。默认为 ``projection``。

``QUERY_PAGE``                      用于分页查询参数的键。默认为 ``page``。

``QUERY_MAX_RESULTS``               用于最大结果数目的查询参数的键。默认为 ``max_results``。

``QUERY_EMBEDDED``                  用于内嵌查询参数的键。默认为 ``embedded``。

``QUERY_AGGREGATION``               用于聚合查询参数的键。默认为 ``aggregate``。

``DATE_FORMAT``                     一个用于解析和渲染时间值的 Python 日期格式。
                                    处理请求时，匹配的 JSON 字符串会被解析和存储为
                                    ``datetime`` 值。在响应中，``datetime`` 值会被
                                    渲染为这个格式的 JSON 字符串。
                                    默认为 RFC1123 (上一个为 RFC 822) 标准
                                    ``a, %d %b %Y %H:%M:%S GMT``
                                    ("Tue, 02 Apr 2013 10:29:13 GMT")。

``RESOURCE_METHODS``                资源终结点支持的一组 HTTP 方法。允许的值: ``GET``,
                                    ``POST``, ``DELETE``。``POST`` 用于插入。
                                    ``DELETE`` 将删除*所有*资源终结点 (谨慎启用)。
                                    可以被资源配置重载。默认为 ``['GET']``。

``PUBLIC_METHODS``                  资源终结点支持的一组 HTTP 方法，即使启用了 :ref:`auth`，也保持开放公共权限。
                                    可以被资源配置重载。默认为 ``[]``。

``ITEM_METHODS``                    数据项终结点支持的一组 HTTP 方法。允许的值: ``GET``,
                                    ``PATCH``, ``PUT`` 和 ``DELETE``。``PATCH`` 或者，对于不支持 PATCH 的客户端，使用 ``X-HTTP-Method-Override`` 头部标记的 ``POST``，被用于数据项更新；``DELETE`` for item deletion. 可以被资源配置重载。默认为 ``['GET']``。

``PUBLIC_ITEM_METHODS``             数据项终结点支持的一组 HTTP 方法，当启用 :ref:`auth` 时，保持开放公共权限。
                                    可以被资源配置重载。默认为 ``[]``。

``ALLOWED_ROLES``                   资源终结点允许的一组 `roles`。可以被资源配置重载。
                                    查看 :ref:`auth` 获取更多信息。默认为 ``[]``。

``ALLOWED_READ_ROLES``              带有 GET 和 OPTIONS 方法的资源终结点允许的一组 `roles`。
                                    可以被资源配置重载。查看 :ref:`auth` 获取更多信息。
                                    默认为 ``[]``。

``ALLOWED_WRITE_ROLES``             带有 POST，PUT 和 DELETE 方法的终结点允许的一组 `roles`。
                                    可以被资源配置重载。查看 :ref:`auth` 获取更多信息。
                                    默认为 ``[]``.

``ALLOWED_ITEM_ROLES``              数据项终结点允许的一组 `roles`。
                                    查看 :ref:`auth` 获取更多信息。
                                    可以被资源配置重载。默认为 ``[]``.

``ALLOWED_ITEM_READ_ROLES``         带有 GET 和 OPTIONS 方法的数据项终结点允许的一组 `roles`。
                                    查看 :ref:`auth` 获取更多信息。
                                    可以被资源配置重载。默认为 ``[]``.

``ALLOWED_ITEM_WRITE_ROLES``        带有 POST，PUT 和 DELETE 方法的数据项终结点允许的一组 `roles`。
                                    查看 :ref:`auth` 获取更多信息。
                                    可以被资源配置重载。默认为 ``[]``.

``ALLOW_OVERRIDE_HTTP_METHOD``      全局启用 / 禁用使用 ``X-HTTP-METHOD-OVERRIDE`` 头
                                    重载发送方法的可能性。

``CACHE_CONTROL``                   ``Cache-Control`` 头字段值，用于处理 ``GET``
                                    请求 (例如，``max-age=20,must-revalidate``)。
                                    如果你不希望在 API 响应中包含缓存指令，请留空。 
                                    可以被资源配置重载。默认为 ``''``。

``CACHE_EXPIRES``                   ``Expires`` 头字段值 (单位，秒)，用于处理 ``GET``
                                    请求。如果设置为非零值，总是会包含头，无视
                                    ``CACHE_CONTROL`` 的配置。 
                                    可以被资源配置重载。默认为 0。

``X_DOMAINS``                       CORS (跨域资源共享) 支持。允许 API 维护者
                                    指定哪个域允许执行 CORS 请求。允许的值为: 
                                    ``None``, 域列表，或对一个完全开放的 API ``'*'``
                                    默认为 ``None``.

``X_DOMAINS_RE``                    与 ``X_DOMAINS`` 相同，除了允许一个正则表达式列表。
                                    这对带有动态子域的网站很有用。确保正确 anchor 和
                                    escape 正则表达式。忽略无效的正则表达式
                                    (such as ``'*'``)。默认为 ``None``。

``X_HEADERS``                       CORS (跨域资源共享) 支持。允许 API 维护者
                                    指定哪个头可以通过 CORS 请求发送。
                                    允许的值为: ``None`` 或头名称列表。
                                    默认为 ``None``。

``X_EXPOSE_HEADERS``                CORS (跨域资源共享) 支持。允许 API 维护者指定
                                    哪个头被暴露在 CORS 响应中。
                                    允许的值为: ``None`` 或头名称列表。
                                    默认为 ``None``。

``X_ALLOW_CREDENTIALS``             CORS (跨域资源共享) 支持。允许 API 维护者指定
                                    客户端是否可以发送 cookies。仅有的允许的值为: 
                                    ``True``，任何其他值都会被忽略。默认为 ``None``。

``X_MAX_AGE``                       CORS (跨域资源共享) 支持。允许为访问控制允许头设置最大
                                    年龄。默认为 21600。


``LAST_UPDATED``                    用于记录文档的最后更新日期的字段名称。
                                    这个字段由 Eve 自动处理。默认为 ``_updated``。

``DATE_CREATED``                    用于记录文档创建日期的字段名称。
                                    这个字段由 Eve 自动处理。默认为 ``_created``。

``ID_FIELD``                        用于在数据库中唯一标识数据项资源的字段名称。
                                    你希望这个字段在数据库中被恰当地索引。
                                    可以被资源配置重载。默认为 ``_id``。

``ITEM_LOOKUP``                     如果数据项终结点应该是跨 API普遍可用的 ``True``，
                                    否则 ``False``。可以被资源配置重载。默认为 ``True``。

``ITEM_LOOKUP_FIELD``               用于搜索数据项资源的文档字段。
                                    可以被资源配置重载。默认为 ``ID_FIELD``。

``ITEM_URL``                        用于创建默认数据项终结点 URL 的 URL 规则。
                                    可以被资源配置重载。默认为
                                    ``regex("[a-f0-9]{24}")``，即 MongoDB 的
                                    标准 ``Object_Id`` 格式。

``ITEM_TITLE``                      由于构建数据项引用的标题，在 XML 和 JSON 响应中
                                    都有效。默认为资源名称去除复数形式 's'，如果存在的话。
                                    可以而且很可能会在配置单个资源终结点时被重载。

``AUTH_FIELD``                      启用 :ref:`user-restricted`。当此特性启用时，
                                    用户只能 read/update/delete 他们自己创建的数据项资源。
                                    关键字包含字段的真实名称，用于存储创建数据项资源的用户 id。
                                    可以被资源配置重载。默认为 ``None``，即禁用特性。

``ALLOW_UNKNOWN``                   为 ``True`` 时，这个选项会允许任意的，未知字段插入到任何 API 终结点。
                                    谨慎使用。查看 :ref:`unknown` 获取更多信息。
                                    默认为 ``False``。

``PROJECTION``                      为 ``True`` 时，这个选项启用 :ref:`projections` 特性。
                                    可以被资源配置重载。默认为 ``True``。

``EMBEDDING``                       为 ``True`` 时，这个选项启用 :ref:`embedded_docs` 特性。
                                    默认为 ``True``。

``BANDWIDTH_SAVER``                 为 ``True`` 时，POST, PUT, 和 PATCH 响应
                                    只会返回自动处理了的字段和 ``EXTRA_RESPONSE_FIELDS``。
                                    为 ``False`` 时，整个文档都会被发送。
                                    默认为 ``True``。

``EXTRA_RESPONSE_FIELDS``           允许配置一个额外的应该在每个 POST 响应中提供的文档字段列表。
                                    正常情况下，只有自动处理的字段 (``ID_FIELD``,
                                    ``LAST_UPDATED``, ``DATE_CREATED``, ``ETAG``) 
                                    被包含在响应载体中。可以被资源配置重载。
                                    默认为 ``[]``, 实际上就是禁用这项特性。

``RATE_LIMIT_GET``                  一个表示对 GET 请求速度限制的元组。元组的
                                    第一个元素时允许的请求数目，而第二个是以秒为单位
                                    的时间窗口。例如 ``(300, 60 * 15)`` 将设置
                                    一个上限，每 15 分钟 300 个请求。默认为 ``None``。

``RATE_LIMIT_POST``                 一个表示对 POST 请求速度限制的元组。元组的
                                    第一个元素时允许的请求数目，而第二个是以秒为单位
                                    的时间窗口。例如 ``(300, 60 * 15)`` 将设置
                                    一个上限，每 15 分钟 300 个请求。默认为 ``None``。

``RATE_LIMIT_PATCH``                一个表示对 PATCH 请求速度限制的元组。元组的
                                    第一个元素时允许的请求数目，而第二个是以秒为单位
                                    的时间窗口。例如 ``(300, 60 * 15)`` 将设置
                                    一个上限，每 15 分钟 300 个请求。默认为 ``None``。

``RATE_LIMIT_DELETE``               一个表示对 DELETE 请求速度限制的元组。元组的
                                    第一个元素时允许的请求数目，而第二个是以秒为单位
                                    的时间窗口。例如 ``(300, 60 * 15)`` 将设置
                                    一个上限，每 15 分钟 300 个请求。默认为 ``None``。

``DEBUG``                           要启用调试模式，``True``，否则 ``False``。

``ERROR``                           允许定制错误代码字段。默认为 ``_error``。

``HATEOAS``                         为 ``False`` 时，这个选项禁用 :ref:`hateoas_feature`。默认为 ``True``。

``ISSUES``                          允许定制问题字段 field。默认为 ``_issues``。

``STATUS``                          允许定制状态字段。默认为 ``_status``。

``STATUS_OK``                       数据验证成功时的状态消息。默认为 ``OK``。

``STATUS_ERR``                      数据验证失败时的状态消息。默认为 ``ERR``。

``ITEMS``                           允许定制数据项字段。默认为 ``_items``。

``META``                            允许定制元数据字段。默认为 ``_meta``。

``INFO``                            字符串值，通过给定的 INFO 名称在 Eve 主页 (suggested
                                    value ``_info``) 包含一个信息节。信息节将包含 
                                    Eve 服务器版本和 API 版本 (API_VERSION，
                                    如果设置了的话)。否则 ``None``，如果你不想暴露
                                    任何服务器信息的话。默认为 ``None``。

``LINKS``                           允许定制链接字段。默认为 ``_links``。

``ETAG``                            允许定制 etag 字段。默认为 ``_etag``。

``IF_MATCH``                        要启用并发控制 ``True``，否则 ``False``。
                                    默认为 ``True``。参考 :ref:`concurrency`。

``ENFORCE_IF_MATCH``                要在启用时一直强制并发控制 ``True``，否则 ``False``。
                                    默认为 ``True``。参考 :ref:`concurrency`。

``RENDERERS``                       允许改变启用的渲染器。默认为
                                    ``['eve.render.JSONRenderer', 'eve.render.XMLRenderer']``。

``JSON_SORT_KEYS``                  要启用 JSON 键排序 ``True``，否则 ``False``。
                                    默认为 ``False``。

``JSON_REQUEST_CONTENT_TYPES``      支持 JSON 内容类型。在你需要支持厂家特定的 json
                                    类型时很有用。请注意: 响应仍会携带标准的
                                    ``application/json`` 类型时很有用。
                                    默认为 ``['application/json']``。

``VALIDATION_ERROR_STATUS``         用于验证错误的 HTTP 状态代码。默认为 ``422``。

``VERSIONING``                      为 ``True`` 时， 启用文档版本控制。
                                    可以被资源配置重载。默认为 ``False``。

``VERSIONS``                        添加到基本集合名称的后缀，用于创建存储文档版本的
                                    影子集合的名称。默认为 ``_versions``。当启用
                                    ``VERSIONING`` 时，会为一个，诸如，数据源为
                                    ``myresource`` 的资源创建一个集合
                                    ``myresource_versions``。

``VERSION_PARAM``                   URL 查询参数，用于访问指定版本的文档。默认为
                                    ``version``。忽略这个参数以获取最新版本的文档，
                                    或者使用 ``?version=all`` 来获取所有版本文档的列表。
                                    只对单个数据项终结点有效。

``VERSION``                         用于存储文档版本号的字段。默认为 ``_version``。

``LATEST_VERSION``                  用于存储文档最新版本号的字段。默认为 ``_latest_version``。

``VERSION_ID_SUFFIX``               在影子集合中，用于存储文档 id 的字段。
                                    默认为 ``_document``。如果 ``ID_FIELD`` 被设置
                                    为 ``_id``，文档 id 会被存储在字段
                                    ``_id_document`` 中。

``MONGO_URI``                       `MongoDB URI`_，用在不喜欢使用其他配置变量的情况下。

``MONGO_HOST``                      MongoDB 服务器地址。默认为 ``localhost``。

``MONGO_PORT``                      MongoDB 端口。默认为 ``27017``。

``MONGO_USERNAME``                  MongoDB 用户名。

``MONGO_PASSWORD``                  MongoDB 密码。

``MONGO_DBNAME``                    MongoDB 数据库名称。

``MONGO_OPTIONS``                   要传递给 MongoClient 类 ``__init__`` 的 MongoDB 关键字参数。
                                    默认为 ``{'connect': True, 'tz_aware': True, 'appname': 'flask_app_name'}``。
                                    查看 `PyMongo mongo_client`_ 作为参考。

``MONGO_AUTH_SOURCE``               MongoDB 身份验证数据库。默认为 ``None``。

``MONGO_AUTH_MECHANISM``            MongoDB 身份验证机制。参考 `PyMongo Authentication Mechanisms`_。
                                    默认为 ``None``。

``MONGO_AUTH_MECHANISM_PROPERTIES`` 指定 MongoDB 额外身份验证机制属性，如果需要的话。
                                    默认为 ``None``。

``MONGO_QUERY_BLACKLIST``           一组不允许用在资源过滤器 (``?where=``) 中的 Mongo 查询运算符。
                                    默认为 ``['$where', '$regex']``。

                                    Mongo JavaScript 运算符默认是禁用的，
                                    因为它们可以被用作注入攻击的载体。
                                    Javascript 查询也比较慢，通常可以轻易被
                                    (非常丰富) Mongo 查询语言取代。

``MONGO_WRITE_CONCERN``             一个定义 MongoDB 写关注的配置的字典。支持
                                    所有标准写入关注配置 (w, wtimeout, j, fsync)。
                                    默认为 ``{'w': 1}``，意味着 '进行常规已确认写入'
                                    (这也是 Mongo 的默认项)。

                                    请意识到，设置 'w' 为值 2 或 更大值，需要激活复制机制，
                                    不然你会得到 500 错误 (写入仍然会发生；
                                    Mongo 只是无法检查，它是否正在写入到多个服务器。)。

                                    可以在终结点 (Mongo 集合) 级别被重载。
                                    参考下面的 ``mongo_write_concern``。

``DOMAIN``                          一个保存 API 域定义的字典。参考 `域配置`_。

``EXTENDED_MEDIA_INFO``             转自文件上传驱动的一组属性。

``RETURN_MEDIA_AS_BASE64_STRING``   控制终结点响应中的媒体类型的 embedding。
                                    当你有其他获取二进制的方式
                                    (like custom Flask endpoints) 但仍然希望客户端
                                    可以 POST/PATCH 它的时候很有用。默认为 ``True``.

``RETURN_MEDIA_AS_URL``             要启用将媒体文件保存在一个专用的媒体节点的话，设置它为 ``True``。
                                    默认为 ``False``。

``MEDIA_BASE_URL``                  激活 ``RETURN_MEDIA_AS_URL`` 时，使用的基准 URL。
                                    结合 ``MEDIA_ENDPOINT`` 和 ``MEDIA_URL`` 决定
                                    了媒体文件返回的 URL。如果为 ``None``，也就是默认值，
                                    会使用 API 基准地址作为替代。

``MEDIA_ENDPOINT``                  媒体终结点，当 ``RETURN_MEDIA_AS_URL`` 启用时使用。
                                    默认为 ``media``。

``MEDIA_URL``                       在专用的媒体终结点提供的文件 url 格式。
                                    默认为 ``regex("[a-f0-9]{24}")``。

``MULTIPART_FORM_FIELDS_AS_JSON``   如果你在将你的资源提交为 ``multipart/form-data``，
                                    所有表单数据字段都将被提交为字符串，打破任何
                                    你可能在资源字段上设置的验证规则。如果你希望
                                    将所有提交的表单数据当作 JSON 字符串，你将需要
                                    激活这个配置。在那种情况下，字段验证继续正常工作。
                                    在 :ref:`multipart` 阅读更多关于字段应该如何被格式化
                                    的信息。默认为 ``False``。

``AUTO_COLLAPSE_MULTI_KEYS``        如果设置为 ``True``，使用同一个键发送的多个值，
                                    使用 ``application/x-www-form-urlencoded`` 或
                                    ``multipart/form-data`` 提交的内容类型，
                                    将自动被转换为值列表。

                                    当和 ``AUTO_CREATE_LISTS`` 一起使用时，
                                    使用媒体字段列表成为可能。

                                    默认为 ``False``

``AUTO_CREATE_LISTS``               当为 ``list`` 类型字段提交一个非 ``list`` 类型值时，
                                    在运行测试器前，自动创建一个单元素列表。

                                    默认为 ``False``

``OPLOG``                           要启用 :ref:`oplog` 的话，设置它为 ``True``。
                                    默认为 ``False``。

``OPLOG_NAME``                      这是存储 :ref:`oplog` 的数据库集合名称。
                                    默认为 ``oplog``。

``OPLOG_METHODS``                   操作会应该被记录进 :ref:`oplog` 的 HTTP 方法列表。
                                    默认为 ``['DELETE', 'POST', 'PATCH', 'PUT']``。

``OPLOG_CHANGE_METHODS``            操作会包含变化进 :ref:`oplog` 的 HTTP 方法列表。
                                    默认为 ``['DELETE','PATCH', 'PUT']``。

``OPLOG_ENDPOINT``                  :ref:`oplog` 终结点的名称。如果这个终结点被启用，
                                    它可以像任何其他 API 终结点一样被配置。设置它为
                                    ``None`` 来禁用这个终结点。默认为 ``None``。

``OPLOG_AUDIT``                     设置为 ``True`` 以启用审计特性。当审计启用时，
                                    客户端 IP 和文档变化也会被记录到 :ref:`oplog`。
                                    默认为 ``True``。

``OPLOG_RETURN_EXTRA_FIELD``        当启用时，可选的 ``extra`` 字段将被包含在载体中，
                                    由 ``OPLOG_ENDPOINT`` 返回。默认为 ``False``。

``SCHEMA_ENDPOINT``                 :ref:`schema_endpoint` 的名称。默认为 ``None``。

``HEADER_TOTAL_COUNT``              自定义的头部，在集合 ``GET`` 请求的响应载体中
                                    包含数据项的总数。这对客户端希望知道数据项数目
                                    而不想获取响应体的 ``HEAD`` 请求很方便。
                                    一个使用场景例子是使用 ``where`` 查询获取未读的文章
                                    而不用加载文章本身。默认为 ``X-Total-Count``。

``JSONP_ARGUMENT``                  这个选项会导致响应被封装在一个 JavaScript 函数调用中，
                                    如果在请求中设置了参数的话。例如，如果你设置
                                    ``JSON_ARGUMENT = 'callback'``，那么所有对
                                    ``?callback=funcname`` 请求的响应都会被封装在一个
                                    ``funcname`` 调用中。默认为 ``None``。

``BULK_ENABLED``                    设置为 ``True`` 时启用批量插入。查看 :ref:`bulk_insert`
                                    获取更多信息。默认为 ``True``。

``SOFT_DELETE``                     设置为 ``True`` 时启用软删除。查看 :ref:`soft_delete`
                                    获取更多信息。默认为 ``False``。

``DELETED``                         文档字段，用于当 ``SOFT_DELETE`` 启用时指示一个文档是否已经被删除。
                                    默认为 ``_deleted``。

``SHOW_DELETED_PARAM``              URL 查询参数，用于包含软删除的数据项在资源级别的 GET 响应中。
                                    默认为 'show_deleted'。

``STANDARD_ERRORS``                 这是一组一个标准 API 响应会提供的 HTTP 错误代码。
                                    经典的错误响应包括一个带有真实错误代码和描述的 JSON 体，
                                    如果你希望禁用全部经典响应，设置这个为空列表。
                                    默认为 ``[400, 401, 403, 404, 405, 406, 409, 410, 412, 422, 428]``

``VALIDATION_ERROR_AS_STRING``      如果为 ``True``，即便单个字段错误也会以列表的形式返回。
                                    默认情况下，单个字段错误以字符串的形式返回，
                                    而多个字段错误被捆在一个列表中。如果你希望规格化
                                    字段错误输出，设置这项配置为 ``True``，这样你总是
                                    会得到一个文档字段列表。默认为 ``False``。

``UPSERT_ON_PUT``                   为 ``True`` 时，``PUT`` 在文档不存在时尝试创建它。
                                    URL 终结点将被用作 ``ID_FIELD`` 值 (如果 ``ID_FIELD``
                                    包含在载体中，它就会被忽略)。正常的验证规则适用。
                                    成功创建时，响应将会是一个 ``201 Created``。
                                    响应负载与你通过执行单文档 POST 到资源终结点得到
                                    将是同一个。设置为 ``False`` 来禁用这项特性，相反，
                                    这样会返回一个 ``404``。默认为 ``True``。

``MERGE_NESTED_DOCUMENTS``          如果为 ``True``，``PATCH`` 对嵌套字段的更新是
                                    融合当前的数据。如果为 ``False``，更新会重写当前的数据。
                                    默认为 ``True``。

``NORMALIZE_DOTTED_FIELDS``         如果为 ``True``，带点的字段被解析和处理为子文档字段。
                                    如果为 ``False``，带点的字段被保留为解析和处理的，
                                    载体被原封不动地传递给潜在的数据层。请注意，
                                    使用默认的 Mongo 层时，设置这个为 ``False`` 会
                                    导致错误。默认为 ``True``。

=================================== =========================================

.. _domain:

域配置
--------------------
在 Eve 术语中，一个 `domain` 是 API 结构的定义，是你设计 API，微调资源终结点和定义验证规则的区域。

``DOMAIN`` 是一个 :ref:`global configuration setting <global>`: 一个 Python 字典，键是 API 资源，而值是它们的定义。

::

    # 这里我们定义了连个 API 终结点，'people' 和 'works'，让它们的定义留空。
    DOMAIN = {
        'people': {},
        'works': {},
        }

在下面的两节中，我们将自定义 `people` 资源。

.. _local:

资源 / 数据项终结点
'''''''''''''''''''''''''
终结点自定义主要通过重载一些 :ref:`global settings <global>` 来实现，但是还有其他的唯一设置可用。资源设置总是小写。

.. tabularcolumns:: |p{6.5cm}|p{8.5cm}|

=============================== ===============================================
``url``                         终结点 URL。如果忽略的话，``DOMAIN`` 字典的资源键将被用于构建 URL。
                                作为一个例子，``contacts`` 将使 `people` 资源在
                                ``/contacts`` (而不是 ``/people``) 可用。URL 可以
                                是要多复杂有多复杂，可以嵌套关联到另一个 API 终结点
                                (你可以有一个 ``/contacts`` 终结点，接着有一个
                                ``/contacts/overseas`` 终结点。两者是彼此独立，
                                可自由配置的)。

                                你也可以使用正则表达式来创建类似子资源的终结点。
                                参考 :ref:`subresources`。

``allowed_filters``             允许使用过滤器的一组索引。这个列表中的项按等级工作。
                                这意味着，举个例子，如果 ``allowed_filters`` 包含
                                ``'dict.sub_dict.foo``, ``'dict.sub_dict'``
                                或 ``'dict'`` 中的一个，那么 ``'dict.sub_dict.foo'``
                                上的过滤是允许的。相反，如果 ``allowed_filters``
                                包含 ``'dict'``， ``'dict'`` 上的过滤是允许的。
                                可以被设置为 ``[]`` (不允许任何过滤器)，或
                                ``['*']`` (每个字段都允许过滤)。默认为 ``['*']``。

                                *请注意:* 如果担心 API 破坏或 DB DoS 攻击，那么
                                全局禁用过滤器 (参考上面的 ``ALLOWED_FILTERS``)，
                                然后再本地基本列出有效白名单是可行的方式。

``sorting``                     如果启用了排序，``True``，否则 ``False``。本地
                                重载 ``SORTING``。

``pagination``                  如果启用了分页，``True``，否则 ``False``。本地
                                重载 ``PAGINATION``。

``resource_methods``            资源终结点支持的一组 HTTP 方法。允许的值为: ``GET``, ``POST``,
                                ``DELETE``。本地重载 ``RESOURCE_METHODS``。

                                *请注意:* 如果在允许版本 0.0.5 或更早版本，
                                请使用目前不支持的 ``methods`` 关键字来代替它。

``public_methods``              资源终结点支持的一组 HTTP 方法，即使启用了 :ref:`auth`，
                                也会开放为公共权限。本地重载 ``PUBLIC_METHODS``。

``item_methods``                数据项终结点支持的一组 HTTP 方法。允许的值: ``GET``, ``PATCH``,
                                ``PUT`` 和 ``DELETE``。``PATCH`` 或者，对不支持
                                PATCH 的客户端来说，带有 ``X-HTTP-Method-Override`` 标记的 ``POST``。
                                本地重载 ``ITEM_METHODS``.

``public_item_methods``         数据项终结点支持的一组 HTTP 方法，当启用 :ref:`auth` 时， 
                                开放为公共权限。贝蒂重载 ``PUBLIC_ITEM_METHODS``。

``allowed_roles``               资源终结点允许的一组 `roles`。查看 :ref:`auth` 
                                获取更多信息。本地重载 ``ALLOWED_ROLES``。

``allowed_read_roles``          带有 GET 和 OPTIONS 方法的资源终结点允许的一组 `roles`。
                                查看 :ref:`auth` 获取更多信息。
                                本地重载 ``ALLOWED_READ_ROLES``。

``allowed_write_roles``         带有 POST, PUT 和 DELETE 方法的资源终结点允许的一组 `roles`。
                                查看 :ref:`auth` 获取更多信息。
                                本地重载 ``ALLOWED_WRITE_ROLES``。

``allowed_item_read_roles``     带有 GET 和 OPTIONS 方法的数据项终结点允许的一组 `roles`。
                                查看 :ref:`auth` 获取更多信息。
                                本地重载 ``ALLOWED_ITEM_READ_ROLES``。


``allowed_item_write_roles``    带有 PUT，PATH 和 DELETE 方法的资源终结点允许的一组 `roles`。
                                查看 :ref:`auth` 获取更多信息。
                                本地重载 ``ALLOWED_ITEM_WRITE_ROLES``。

``allowed_item_roles``          数据项终结点允许的一组 `roles`。
                                查看 :ref:`auth` 获取更多信息。
                                本地重载 ``ALLOWED_ITEM_ROLES``。

``cache_control``               处理 ``GET`` 请求时，使用的 ``Cache-Control`` 头字段值。
                                如果你不希望在 API 响应中包含缓存指令的话，留空。
                                本地重载 ``CACHE_CONTROL``。

``cache_expires``               处理 ``GET`` 请求时，使用的 ``Expires`` 头字段值 (单位，秒)。
                                如果设置为非零值，总是会包含头部，无视 ``CACHE_CONTROL`` 的设置。
                                本地重载 ``CACHE_EXPIRES``。

``id_field``                    用于在数据库中唯一标识资源数据项的字段。本地重载 ``ID_FIELD``。

``item_lookup``                 如果数据项终结点应该可用，``True``，否则 ``False``。
                                本地重载 ``ITEM_LOOKUP``。

``item_lookup_field``           搜索资源数据项时使用的字段。本地重载 ``ITEM_LOOKUP_FIELD``。

``item_url``                    用于创建数据项终结点 URL 的规则。本地重载 ``ITEM_URL``。

``resource_title``              构建资源链接 (HATEOAS) 时使用的标题。默认为资源的 ``url``。

``item_title``                  构建数据项引用时使用的标题，同时用于 XML 和 JSON 响应。
                                重载 ``ITEM_TITLE``。

``additional_lookup``           除了标准的数据项终结点，默认为 ``/<resource>/<ID_FIELD_value>``
                                外，你可以随意定义第二个只读的终结点，类似
                                ``/<resource>/<person_name>``。你通过定义一个
                                由两个项目 `field` 和 `url` 组成的字典来做到这个。
                                前者时用于搜索的字段名称。如果字段类型
                                (像资源 schema_ 定义的那样) 是字符串，那么你在 `url`
                                中放一个 URL 规则。如果是整数，那么你只需要忽略 `url`，
                                因为它可以被自动处理。参考下面的代码片段，这个特性的用法示例。

``datasource``                  明确地链接 API 资源到数据库集合。参考 `Advanced Datasource Patterns`_。

``auth_field``                  启用 :ref:`user-restricted`。当特性启用时，
                                用户只可以 read/update/delete 自己创建的资源项。
                                关键字包含字段的实际名称，用于存储创建资源项的用户标识。
                                本地重载 ``AUTH_FIELD``。

``allow_unknown``               为 ``True`` 时，这个选项将允许插入任意的未知字段到
                                终结点。谨慎使用。本地重载 ``ALLOW_UNKNOWN``。
                                查看 :ref:`unknown` 获取更多信息。默认为 ``False``。

``transparent_schema_rules``    为 ``True`` 时，这个选项禁用终结点的 :ref:`schema_validation`。

``projection``                  为 ``True`` 时，这个选项启用 :ref:`projections` 特性。
                                本地重载 ``PROJECTION``。默认为 ``True``。

``embedding``                   为 ``True`` 时，这个选项启用 :ref:`embedded_docs` 特性。
                                默认为 ``True``。

``extra_response_fields``       允许配置一组额外的应该在每个 POST 响应中提供的文档字段。
                                正常情况下，只有自动处理的字段 (``ID_FIELD``,
                                ``LAST_UPDATED``, ``DATE_CREATED``, ``ETAG``)
                                被包含在响应载体中。重载 ``EXTRA_RESPONSE_FIELDS``。

``hateoas``                     为 ``False`` 时，这个选项禁用资源的 :ref:`hateoas_feature`。
                                默认为 ``True``。

``mongo_write_concern``         一个为终结点数据源定义 MongoDB 写关注配置的字典。支持
                                所有标准写关注配置 (w, wtimeout, j, fsync)。默认为
                                ``{'w': 1}``，这意味着 '进行常规已确认写入'
                                (这也是 Mongo 的默认项)。

                                请意识到，设置 'w' 为值 2 或 更大值，需要激活复制机制，
                                不然你会得到 500 错误 (写入仍然会发生；
                                Mongo 只是无法检查，它是否正在写入到多个服务器。)。

``mongo_prefix``                允许重载默认的 ``MONGO`` 前缀，用于从配置中获取 MongoDB 设置。

                                例如，如果 ``mongo_prefix`` 设置为 ``MONGO2``，那么，
                                当处理终结点请求时，``MONGO2`` 前缀设置将被用于访问数据库。

                                这允许在每个终结点最终提供来自不同数据库/服务器的数据。

                                请查阅: :ref:`authdrivendb`。

``mongo_indexes``               允许为资源指定一组索引，在应用程序启动前创建。

                                索引被表示为一个字典，键为索引名称，值为 (field, direction) 
                                二元组列表，或一个带有 field/direction 对列表的元组
                                *以及* 表示为一个字典的索引选项，诸如，
                                ``{'index name': [('field', 1)], 'index with
                                args': ([('field', 1)], {"sparse": True})}``。

                                多个键值对用于创建复合索引。方向可以取 PyMongo 支持的任何值，
                                诸如，``ASCENDING`` = 1 和 ``DESCENDING`` = -1。
                                支持所有索引选项，诸如，``sparse``, ``min``, ``max``,
                                等等 (参考 PyMongo_ 文档)。

                                *请注意:* 记住，索引的设计，创建和维护时非常重要的任务，
                                应该小心翼翼的计划和执行。通常它也是一个非常资源密集的操作。
                                你可能因此希望脱离 API 实例化上下文，手工处理这个任务。
                                也要记住，默认情况下，任何已经存在的索引，如果定义发生变化，
                                会被删除重建。

``authentication``              一个带有用于终结点身份验证逻辑的类。如果未提供的话，
                                将使用最终的通用的身份验证类 (作为应用程序构造参数传递)。
                                要获取更多关于身份验证和授权的详细信息，参考 :ref:`auth`。
                                默认为 ``None``。

``embedded_fields``             默认启用了 :ref:`embedded_docs` 的一组字段。
                                要让这个特性工作正常，列表中的字段必须是 ``embeddable``，
                                而资源也必须激活 ``embedding``。

``query_objectid_as_string``    启用后，Mongo 解析器将避免自动转换合格的字符串为 ObjectIds。
                                这个可能在那些很少发生的情况下很有用，比如，你的数据库里
                                有值实际上可以但又不应该转换为 ObjectId 值的字符串字段。
                                它对查询和解析载体有效。默认为 ``False``。

``internal_resource``           为 ``True`` 时，这个选项使资源成为内部得。
                                在终结点上无法进行执行 HTTP 行为，但仍然可以被 Eve 
                                数据层访问。查看 :ref:`internal_resources` 获取更多信息。
                                默认为 ``False``。

``etag_ignore_fields``          不应该用于计算 ETag 值得一组字段。默认为 ``None``，
                                意思是，默认所有字段都被包含在计算中。看起来类似
                                ``['field1', 'field2', 'field3.nested_field', ...]``。

``schema``                      一个定义资源处理的实际数据结构的字典。启用数据验证。
                                参考 `模式定义`_ 。

``bulk_enabled``                为 ``True`` 时，这个选项启用这个资源的 :ref:`bulk_insert` 特性。
                                本地重载 ``BULK_ENABLED``。

``soft_delete``                 为 ``True`` 时，这个选项启用这个资源的 :ref:`soft_delete` 特性。
                                本地重载 ``SOFT_DELETE``。

``merge_nested_documents``      如果为 ``True``，``PATCH`` 对嵌套字段的更新是合并
                                当前数据。如果为 ``False``，更新将重写当前数据。
                                本地重载 ``MERGE_NESTED_DOCUMENTS``。

``normalize_dotted_fields``     如果为 ``True``，带点的字段被解析和处理为子文档字段。
                                如果为 ``False``， 带点的字段不解析不处理，
                                载体按原样直接被传递到潜在的数据层。请注意，
                                使用默认的 Mongo 层时，设置这个为 ``False`` 会导致错误。
                                默认为 ``True``。

=============================== ===============================================

这里是一个资源自定义的示例，主要通过重载全局 API 设置实现:

::

    people = {
        # 用在数据项链接中的 'title' 标签。默认为资源标题减去最后的复数形式 's'
        # (在大多数场景工作良好，但不包括 'people')
        'item_title': 'person',

        # 默认情况下，标准数据项入口点被定义为 '/people/<ObjectId>/'。
        # 我们让它原封不动，然后我们再启用一个另外的只读入口点。这样，也可以在
        # '/people/<lastname>' 执行 GET 请求.
        'additional_lookup': {
            'url': 'regex("[\w]+")',
            'field': 'lastname'
        },

        # 我们选择为这个资源重载全局缓存控制指令.
        'cache_control': 'max-age=10,must-revalidate',
        'cache_expires': 10,

        # 在这个资源终结点上我们只允许 GET 和 POST。
        'resource_methods': ['GET', 'POST'],
    }

.. _schema:

模式定义
-----------------
除非你的 API 是只读的，你很可能希望定义资源 `模式`。模式很重要，因为它们对进来的流启用合适的验证。

::

    # 'people' 模式定义
    schema = {
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
            'unique': True,
        },
        # 'role' 是一个列表，而且只能包含 'allowed' 中的值.
        'role': {
            'type': 'list',
            'allowed': ["author", "contributor", "copy"],
        },
        # 一个内嵌的 '强类型' 字典.
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

就像你看到地那样，模式键实际上是字段名称，而值是定义字段验证规则的字典。允许的验证规则有:

.. tabularcolumns:: |p{6.5cm}|p{8.5cm}|

=============================== ==============================================
``type``                        字段数据类型。可以时下面中的一项:

                                - ``string``
                                - ``boolean``
                                - ``integer``
                                - ``float``
                                - ``number`` (允许整数和浮点数值)
                                - ``datetime``
                                - ``dict``
                                - ``list``
                                - ``media``

                                如果使用了 MongoDB 数据层，那么 ``objectid``, ``dbref`` 和
                                地理数据结构也是允许的:

                                - ``objectid``
                                - ``dbref``
                                - ``point``
                                - ``multipoint``
                                - ``linestring``
                                - ``multilinestring``
                                - ``polygon``
                                - ``multipolygon``
                                - ``geometrycollection``
                                - ``decimal``

                                查看 :ref:`GeoJSON <geojson_feature>` 获取更多关于地理字段的信息。

``required``                    如果为 ``True``，, 插入时字段是必需的。

``readonly``                    如果为 ``True``，字段是只读的。

``minlength``, ``maxlength``    ``string`` 和 ``list`` 类型允许的最小和最大长度。

``min``, ``max``                ``integer``, ``float`` 和 ``number`` 类型允许的最小和最大值。

``allowed``                     ``string`` 和 ``list`` 类型允许的值列表。

``empty``                       只用于字符串字段。如果为 ``False``，值为空时验证会失败。
                                默认为 ``True``。

``items``                       定义一组在一个定长 ``list`` 中允许使用的值，
                                参考 `docs <http://docs.python-cerberus.org/en/latest/usage.html#items-list>`_。

``schema``                      ``dict`` 类型和不定长 ``list`` 类型的验证模式。
                                要获取详情和用法示例，参考 `Cerberus documentation <http://docs.python-cerberus.org/en/latest/usage.html#schema-dict>`_。

``unique``                      字段的值在集合中必须是唯一的。

                                请注意: 验证约束是在数据库中检查，而不是在文档载体自己中。
                                这导致一个有趣的偏僻场景: 在一个多文档载体中的两个或更多文档携带
                                设置 'unique' 约束的字段为相同值事件中，
                                载体会验证成功，因为在数据库中还没有副本。

                                如果这是一个问题，客户端总是能在插入时发送文档一个，
                                或者在提交载体到 API 之前，本地先验证一下。

``unique_to_user``              字段值对用户来说是唯一的。这在一个终结点启用了 :ref:`user-restricted` 
                                时很有用。规则只会验证 *用户数据*。因此这个场景中，
                                允许副本存在，只要它们被不同的用户存储。相反，
                                单个用户无法存储重复的值。

                                如果终结点上未激活 URRA，这条规则的行为就像 ``unique``。

``data_relation``               允许指定一个值必须满足用于验证的参照完整性规则。
                                它是一个字典，带有四个键:

                                - ``resource``: 引用的资源名称;
                                - ``field``: 外部资源中的字段名称;
                                - ``embeddable``: 如果客户端可以请求引用的文档通过序列化内嵌，
                                  设置为 ``True``。参考 :ref:`embedded_docs`。
                                  默认为 ``False``。
                                - ``version``: 设置为 ``True`` 来拥有一个
                                  带数据关联的 ``_version``。参考 :ref:`document_versioning`。
                                  默认为 ``False``。

``nullable``                    如果为 ``True``，字段值可以被设置为 ``None``。

``default``                     字段的默认值。当处理 POST 和 PUT 请求时，丢失的字段将被指定为配置的默认值。

                                它对 ``dict`` 和 ``list`` 类型也有起作用。
                                对后者的作用是有限，只对带有模式的列表
                                (含有未知数量元素并且每个元素都是一个 ``dict`` 的列表) 起作用。

                                ::

                                    schema = {
                                      # 简单默认值
                                      'title': {
                                        'type': 'string',
                                        'default': 'M.'
                                      },
                                      # 字典中的默认值
                                      'others': {
                                        'type': 'dict',
                                        'schema': {
                                          'code': {
                                            'type': 'integer',
                                            'default': 100
                                          }
                                        }
                                      },
                                      # 字典列表中的默认值
                                      'mylist': {
                                        'type': 'list',
                                        'schema': {
                                          'type': 'dict',
                                          'schema': {
                                            'name': {'type': 'string'},
                                            'customer': {
                                              'type': 'boolean',
                                              'default': False
                                            }
                                          }
                                        }
                                      }
                                    }

``versioning``                  为 ``True`` 时，启用文档版本控制。默认为 ``False``。

``versioned``                   如果为 ``True``，当启用 ``versioning``时，这个字段会被包含
                                在每个文档的版本历史中。默认为 ``True``。

``valueschema``                 对 ``dict`` 中所有值的验证模式。
                                字典可以含有任意必须通过给定模式验证的键和值。
                                参考 Cerberus 文档中的 `valueschema <http://docs.python-cerberus.org/en/latest/validation-rules.html#valueschema>`_。

``keyschema``                   与 ``valueschema`` 对应，验证字典中的键。
                                对 ``dict`` 中所有值的验证模式。参考 Cerberus 文档中的
                                `keyschema <http://docs.python-cerberus.org/en/latest/validation-rules.html#keyschema>`_ 。


``regex``                       如果字段值未匹配提供的正则表达式，验证失败。
                                只用于字符串字段。参考 Cerberus 文档中的 `regex <http://docs.python-cerberus.org/en/latest/validation-rules.html#regex>`_。


``dependencies``                这条规则允许一组字段必须存在，以保证目标字段通过。
                                参考 Cerberus 文档中的 `dependencies <http://docs.python-cerberus.org/en/latest/validation-rules.html#dependencies>`_ 。

``anyof``                       这条规则允许你列出多个用于验证的规则集合。
                                如果列表中的一个集合验证通过，这个字段就被认为时有效的。
                                参考 Cerberus 文档中的 `*of-rules <http://docs.python-cerberus.org/en/latest/validation-rules.html#of-rules>`_ 。

``allof``                       与 ``anyof`` 相同，除了列表中的所有规则集合都必须验证。

``noneof``                      与 ``anyof`` 相同，除了列表中的规则集合一个都不需要验证。

``oneof``                       与 ``anyof`` 相同，除了列表中只有一个规则集合可以通过验证。

``coerce``                      类型胁迫，允许你在任何验证器运行前应用一个 callable 到一个值。
                                callable 的返回值替换了文档中的新值。
                                这个可以用于在验证之前转换值或净化数据。
                                参考 Cerberus 文档中的 `value coercion <http://docs.python-cerberus.org/en/latest/normalization-rules.html#value-coercion>`_ 。

=============================== ==============================================

模式语法基于 Cerberus_，当然它可以被扩展。事实上，Eve 自己通过添加 ``unique`` 和 ``data_relation`` 关键字 ``objectid`` 数据类型以及扩展了原始的语法。要获取更多关于自定义验证和用法示例的信息，请查看 :ref:`validation`。

在 :ref:`local` 中，你自定义了 `people` 终结点。然后，在这一节，你定义了 `people` 的验证规则。限制你已经准备好更新最初在 `Domain Configuration`_ 中创建的域了:

::

    # 添加模式到 'people' 资源定义
    people['schema'] = schema
    # 更新域
    DOMAIN['people'] = people

.. _datasource:

高级数据源模式
----------------------------
``datasource`` 关键字允许明确得链接 API 资源到数据库集合。如果被忽略，那么域资源键也会被假定为数据库集合的名称。它是一个使用四个允许的键的字典:

.. tabularcolumns:: |p{6.5cm}|p{8.5cm}|

=============================== ==============================================
``source``                      资源使用的数据库集合的名称。如果忽略，资源名称也会被
                                假定为一个有效的集合名称。参考 :ref:`source`。

``filter``                      数据库查询，用于获取和验证数据。如果忽略，默认获取
                                整个集合。参考 :ref:`filter`。

``projection``                  终结点暴露的字段集合。如果忽略，默认返回所有字段到
                                客户端。参考 :ref:`projection`。

``default_sort``                从这个终结点获取的文档的默认排序。如果忽略，文档会按
                                默认的数据库次序返回。一个有效的语句是:

                                ``'datasource': {'default_sort': [('name',
                                1)]}``

                                要获取更多关于排序和过滤的信息，参考 :ref:`filters`。

``aggregation``                 聚合管道和选项。使用后，所有其他的 ``datasource`` 
                                配置会被忽略，除了 ``source``。终结点将是只读的，
                                而且数据项搜索不可用。默认为 ``None``。

                                这是一个字典，包含一个或更多以下的键:

                                - ``pipeline``. 聚合管道。语法必须匹配 PyMongo 支持的那个。要获取更多信息，参考 
                                `PyMongo Aggregation Examples`_ 和 官方的 
                                `MongoDB Aggregation Framework`_ 文档。

                                - ``options``. 聚合选项。必须是一个字典，包含一个或更多这些键:

                                    - ``allowDiskUse`` (bool)
                                    - ``maxTimeMS`` (int)
                                    - ``batchSize`` (int)
                                    - ``useCursor`` (bool)

                                如果你想改变任何 `PyMongo aggregation defaults`_，
                                你只需要设置 ``options``。

=============================== ==============================================

.. _filter:

预定义的数据库过滤器
'''''''''''''''''''''''''''
用于 API 终结点的数据库过滤器通过 ``filter`` 关键字设置。

::

    people = {
        'datasource': {
            'filter': {'username': {'$exists': True}}
            }
        }

在上面的示例中，`people` 资源的 API 终结点只会暴露和更新带有一个已存在的 `username` 字段的文档。

预定义的过滤器运行在用户查询 (使用 `where` 从句的 GET 请求) 和标准条件请求 (`If-Modified-Since`, 等等) 之上。

请注意，数据源过滤器应用于 GET, PATCH 和 DELETE 请求。如果你的资源允许 POST 请求 (文档插入)，你很可能会希望设置对应的验证规则 (在我们的例子中，'username' 应该很可能是一个需要的字段)。

.. 提示:: 静态 vs 动态过滤器

    预定义的过滤器是静态的。相反，你也可以利用 :ref:`eventhooks` 系统 (特别是 ``on_pre_<METHOD>`` hooks) 来建立动态过滤器。

.. _source:

多 API 终结点，一个数据源
''''''''''''''''''''''''''''''''''''''
多个 API 终结点e可以以同一个数据库集合为目标。例如，你可以同时设置 ``/admins`` 和 ``/users``，从数据库中的同一个 `people` 集合中读和写。

::

    people = {
        'datasource': {
            'source': 'people',
            'filter': {'userlevel': 1}
            }
        }

上面的配置只会从 `people` 集合获取，编辑和删除 `userlevel` 是 1 文档。

.. _projection:

限制 API 终结点暴露的字段集
'''''''''''''''''''''''''''''''''''''''''''''''''
默认情况下，对 GET 请求的 API 响应会包含对应的资源 schema_ 定义的所有字段。`datasource` 资源关键字 ``projection`` 配置允许你重定义字段集合。

当你希望对客户端隐藏一些*秘密字段*时，你应该使用包含性的投影配置并包含所有应该被暴露的字段。而当你希望限制默认的响应为特定的字段，但仍允许通过客户端投影访问它们时，你应该使用排他性的投影配置并排除应该被忽略的字段。

下面是一个包含性投影配置的例子:

::

    people = {
        'datasource': {
            'projection': {'username': 1}
            }
        }

上面的配置将会只 `username` 字段给 GET 请求，不管资源是否定义了 schema_。而其他字段即使通过客户端投影也 **不会** 被暴露。下面的 API 调用不会返回 `lastname` 或 `born`。

.. code-block:: console

    $ curl -i http://eve-demo.herokuapp.com/people?projection={"lastname": 1, "born": 1}
    HTTP/1.1 200 OK

你也可以从 API 响应中排除字段。但是这次，被排除的字段 **会** 暴露给客户端投影。下面是一个排他性投影配置的例子:

::

    people = {
        'datasource': {
            'projection': {'username': 0}
            }
        }

上面的配置将保护所有文档字段，除了 `username`。虽然如此，这次下面的 API 调用会返回 `username`。所以，你可以利用这个行为来提供媒体字段或其他昂贵的字段。

在大多场景中，推荐使用空的或包含性投影配置。通过包含性投影，秘密字段被小心得从服务端获取，而返回的默认字段可以由客户端的快捷功能定义。

.. code-block:: console

    $ curl -i http://eve-demo.herokuapp.com/people?projection={"username": 1}
    HTTP/1.1 200 OK


请注意，POST 和 PATCH 方法仍允许操作整个模式。这个特性会在，例如，你希望通过一项 :ref:`auth` 模式保护插入和修改而仍然公开读取权限给大家时派上用场。

.. 提示:: 请参阅

    - :ref:`projections`
    - :ref:`projection_filestorage`

.. _Cerberus: http://python-cerberus.org
.. _`MongoDB URI`: http://docs.mongodb.org/manual/reference/connection-string/#Connections-StandardConnectionStringFormat
.. _ReadPreference: http://api.mongodb.org/python/current/api/pymongo/read_preferences.html#pymongo.read_preferences.ReadPreference
.. _PyMongo: http://api.mongodb.org/python/current/api/pymongo/collection.html#pymongo.collection.Collection.create_index
.. _`PyMongo Aggregation Examples`: http://api.mongodb.org/python/current/examples/aggregation.html#aggregation-framework
.. _`MongoDB Aggregation Framework`: https://docs.mongodb.org/v3.0/applications/aggregation/
.. _`PyMongo aggregation defaults`: http://api.mongodb.org/python/current/api/pymongo/collection.html#pymongo.collection.Collection.aggregate
.. _`PyMongo Authentication Mechanisms`: https://docs.mongodb.com/v3.0/core/authentication-mechanisms/
.. _`PyMongo mongo_client`: http://api.mongodb.com/python/current/api/pymongo/mongo_client.html
