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

``ALLOWED_FILTERS``                 List of fields on which filtering is allowed.
                                    Entries in this list work in a hierarchical
                                    way. This means that, for instance, filtering
                                    on ``'dict.sub_dict.foo'`` is allowed if
                                    ``ALLOWED_FILTERS`` contains any of
                                    ``'dict.sub_dict.foo``, ``'dict.sub_dict'``
                                    or ``'dict'``. Instead filtering on
                                    ``'dict'`` is allowed if ``ALLOWED_FILTERS``
                                    contains ``'dict'``.
                                    Can be set to ``[]`` (no filters allowed)
                                    or ``['*']`` (filters allowed on every
                                    field). Unless your API is comprised of
                                    just one endpoint, this global setting
                                    should be used as an on/off switch,
                                    delegating explicit whitelisting at the
                                    local level (see ``allowed_filters``
                                    below). Defaults to ``['*']``.

                                    *Please note:* If API scraping or DB DoS
                                    attacks are a concern, then globally
                                    disabling filters and whitelisting valid
                                    ones at the local level is the way to go.

``VALIDATE_FILTERS``                Whether to validate the filters against the
                                    resource schema. Invalid filters will throw
                                    an exception. Defaults to ``False``.

                                    Word of caution: validation on filter
                                    expressions involving fields with custom
                                    rules or types might have a considerable
                                    impact on performance. This is the case,
                                    for example, with ``data_relation``-rule
                                    fields. Consider excluding heavy-duty
                                    fields from filters (see
                                    ``ALLOWED_FILTERS``).

``SORTING``                         如果 ``GET`` 请求支持排序，``True``，否则，``False``。
                                    可以被资源配置重载。默认为 ``True``。

``PAGINATION``                      如果 ``GET`` 请求启用了分页，``True``，否则，``False``。
                                    可以被资源配置重载。默认为 ``True``。

``PAGINATION_LIMIT``                Maximum value allowed for QUERY_MAX_RESULTS
                                    query parameter. Values exceeding the
                                    limit will be silently replaced with this
                                    value. You want to aim for a reasonable
                                    compromise between performance and transfer
                                    size. Defaults to 50.

``PAGINATION_DEFAULT``              QUERY_MAX_RESULTS 的默认值。默认为 25。

``OPTIMIZE_PAGINATION_FOR_SPEED``   Set this to ``True`` to improve pagination
                                    performance. When optimization is active no
                                    count operation, which can be slow on large
                                    collections, is performed on the database.
                                    This does have a few consequences.
                                    Firstly, no document count is returned.
                                    Secondly, ``HATEOAS`` is less accurate: no
                                    last page link is available, and next page
                                    link is always included, even on last page.
                                    On big collections, switching this feature
                                    on can greatly improve performance.
                                    Defaults to ``False`` (slower performance;
                                    document count included; accurate
                                    ``HATEOAS``).

``QUERY_WHERE``                     Key for the filters query parameter. Defaults to ``where``.

``QUERY_SORT``                      Key for the sort query parameter. Defaults to ``sort``.

``QUERY_PROJECTION``                Key for the projections query parameter. Defaults to ``projection``.

``QUERY_PAGE``                      Key for the pages query parameter. Defaults to ``page``.

``QUERY_MAX_RESULTS``               Key for the max results query parameter. Defaults to ``max_results``.

``QUERY_EMBEDDED``                  Key for the embedding query parameter. Defaults to ``embedded``.

``QUERY_AGGREGATION``               Key for the aggregation query parameter.
                                    Defaults to ``aggregate``.

``DATE_FORMAT``                     A Python date format used to parse and render
                                    datetime values. When serving requests,
                                    matching JSON strings will be parsed and
                                    stored as ``datetime`` values. In
                                    responses, ``datetime`` values will be
                                    rendered as JSON strings using this format.
                                    Defaults to the RFC1123 (ex RFC 822)
                                    standard ``a, %d %b %Y %H:%M:%S GMT``
                                    ("Tue, 02 Apr 2013 10:29:13 GMT").

``RESOURCE_METHODS``                A list of HTTP methods supported at resource
                                    endpoints. Allowed values: ``GET``,
                                    ``POST``, ``DELETE``. ``POST`` is used for
                                    insertions. ``DELETE`` will delete *all*
                                    resource contents (enable with caution).
                                    Can be overridden by resource settings.
                                    Defaults to ``['GET']``.

``PUBLIC_METHODS``                  资源终结点支持的 HTTP 方法的列表，开放到公开访问，甚至启用了 :ref:`auth`。
                                    可以被资源配置重载。默认为 ``[]``。

``ITEM_METHODS``                    A list of HTTP methods supported at item
                                    endpoints. Allowed values: ``GET``,
                                    ``PATCH``, ``PUT`` and ``DELETE``. ``PATCH``
                                    or, for clients not supporting PATCH,
                                    ``POST`` with the ``X-HTTP-Method-Override``
                                    header tag, is used for item updates;
                                    ``DELETE`` for item deletion. Can be
                                    overridden by resource settings. Defaults to
                                    ``['GET']``.

``PUBLIC_ITEM_METHODS``             A list of HTTP methods supported at item
                                    endpoints, left open to public access when
                                    when :ref:`auth` is enabled. Can be
                                    overridden by resource settings. Defaults
                                    to ``[]``.

``ALLOWED_ROLES``                   A list of allowed `roles` for resource
                                    endpoints. Can be overridden by resource
                                    settings. See :ref:`auth` for more
                                    information. Defaults to ``[]``.

``ALLOWED_READ_ROLES``              A list of allowed `roles` for resource
                                    endpoints with GET and OPTIONS methods.
                                    Can be overridden by resource
                                    settings. See :ref:`auth` for more
                                    information. Defaults to ``[]``.

``ALLOWED_WRITE_ROLES``             A list of allowed `roles` for resource
                                    endpoints with POST, PUT and DELETE
                                    methods. Can be overridden by resource
                                    settings. See :ref:`auth` for more
                                    information. Defaults to ``[]``.

``ALLOWED_ITEM_ROLES``              A list of allowed `roles` for item endpoints.
                                    See :ref:`auth` for more information. Can
                                    be overridden by resource settings.
                                    Defaults to ``[]``.

``ALLOWED_ITEM_READ_ROLES``         A list of allowed `roles` for item endpoints
                                    with GET and OPTIONS methods.
                                    See :ref:`auth` for more information. Can
                                    be overridden by resource settings.
                                    Defaults to ``[]``.

``ALLOWED_ITEM_WRITE_ROLES``        A list of allowed `roles` for item endpoints
                                    with PUT, PATCH and DELETE methods.
                                    See :ref:`auth` for more information. Can
                                    be overridden by resource settings.
                                    Defaults to ``[]``.

``ALLOW_OVERRIDE_HTTP_METHOD``      Enables / Disables global the possibility
                                    to override the sent method with a header
                                    ``X-HTTP-METHOD-OVERRIDE``.

``CACHE_CONTROL``                   Value of the ``Cache-Control`` header field
                                    used when serving ``GET`` requests (e.g.,
                                    ``max-age=20,must-revalidate``). Leave
                                    empty if you don't want to include cache
                                    directives with API responses. Can be
                                    overridden by resource settings. Defaults
                                    to ``''``.

``CACHE_EXPIRES``                   Value (in seconds) of the ``Expires`` header
                                    field used when serving ``GET`` requests.
                                    If set to a non-zero value, the header will
                                    always be included, regardless of the
                                    setting of ``CACHE_CONTROL``. Can be
                                    overridden by resource settings. Defaults
                                    to 0.

``X_DOMAINS``                       CORS (Cross-Origin Resource Sharing) support.
                                    Allows API maintainers to specify which
                                    domains are allowed to perform CORS
                                    requests. Allowed values are: ``None``,
                                    a list of domains, or ``'*'`` for
                                    a wide-open API. Defaults to ``None``.

``X_DOMAINS_RE``                    The same setting as ``X_DOMAINS``, but a list
                                    of regexes is allowed. This is useful for
                                    websites with dynamic ranges of
                                    subdomains. Make sure to properly anchor and
                                    escape the regexes. Invalid
                                    regexes (such as ``'*'``) are ignored.
                                    Defaults to ``None``.

``X_HEADERS``                       CORS (Cross-Origin Resource Sharing) support.
                                    Allows API maintainers to specify which
                                    headers are allowed to be sent with CORS
                                    requests. Allowed values are: ``None`` or
                                    a list of headers names. Defaults to
                                    ``None``.

``X_EXPOSE_HEADERS``                CORS (Cross-Origin Resource Sharing) support.
                                    Allows API maintainers to specify which
                                    headers are exposed within a CORS response.
                                    Allowed values are: ``None`` or
                                    a list of headers names. Defaults to
                                    ``None``.

``X_ALLOW_CREDENTIALS``             CORS (Cross-Origin Resource Sharing) support.
                                    Allows API maintainers to specify if cookies can
                                    be sent by clients.
                                    The only allowed value is: ``True``, any other
                                    will be ignored. Defaults to
                                    ``None``.

``X_MAX_AGE``                       CORS (Cross-Origin Resource Sharing)
                                    support. Allows to set max age for the
                                    access control allow header. Defaults to
                                    21600.


``LAST_UPDATED``                    Name of the field used to record a document's
                                    last update date. This field is
                                    automatically handled by Eve. Defaults to
                                    ``_updated``.

``DATE_CREATED``                    Name for the field used to record a document
                                    creation date. This field is automatically
                                    handled by Eve. Defaults to ``_created``.

``ID_FIELD``                        Name of the field used to uniquely identify
                                    resource items within the database. You
                                    want this field to be properly indexed on
                                    the database. Can be overridden by resource
                                    settings. Defaults to ``_id``.

``ITEM_LOOKUP``                     ``True`` if item endpoints should be generally
                                    available across the API, ``False``
                                    otherwise. Can be overridden by resource
                                    settings. Defaults to ``True``.

``ITEM_LOOKUP_FIELD``               Document field used when looking up a resource
                                    item. Can be overridden by resource
                                    settings. Defaults to ``ID_FIELD``.

``ITEM_URL``                        URL rule used to construct default item
                                    endpoint URLs. Can be overridden by
                                    resource settings. Defaults
                                    ``regex("[a-f0-9]{24}")`` which is MongoDB
                                    standard ``Object_Id`` format.

``ITEM_TITLE``                      Title to be used when building item references,
                                    both in XML and JSON responses. Defaults to
                                    resource name, with the plural 's' stripped
                                    if present. Can and most likely will be
                                    overridden when configuring single resource
                                    endpoints.

``AUTH_FIELD``                      Enables :ref:`user-restricted`. When the
                                    feature is enabled, users can only
                                    read/update/delete resource items created
                                    by themselves. The keyword contains the
                                    actual name of the field used to store the
                                    id of the user who created the resource
                                    item. Can be overridden by resource
                                    settings. Defaults to ``None``, which
                                    disables the feature.

``ALLOW_UNKNOWN``                   When ``True``, this option will allow insertion
                                    of arbitrary, unknown fields to any API
                                    endpoint. Use with caution. See
                                    :ref:`unknown` for more information.
                                    Defaults to ``False``.

``PROJECTION``                      When ``True``, this option enables the
                                    :ref:`projections` feature. Can be
                                    overridden by resource settings. Defaults
                                    to ``True``.

``EMBEDDING``                       When ``True``, this option enables the
                                    :ref:`embedded_docs` feature. Defaults to
                                    ``True``.

``BANDWIDTH_SAVER``                 When ``True``, POST, PUT, and PATCH responses
                                    only return automatically handled fields
                                    and ``EXTRA_RESPONSE_FIELDS``. When
                                    ``False``, the entire document will be
                                    sent. Defaults to ``True``.

``EXTRA_RESPONSE_FIELDS``           Allows to configure a list of additional
                                    document fields that should be provided
                                    with every POST response. Normally only
                                    automatically handled fields (``ID_FIELD``,
                                    ``LAST_UPDATED``, ``DATE_CREATED``,
                                    ``ETAG``) are included in response
                                    payloads. Can be overridden by resource
                                    settings. Defaults to ``[]``, effectively
                                    disabling the feature.

``RATE_LIMIT_GET``                  A tuple expressing the rate limit on GET
                                    requests. The first element of the tuple is
                                    the number of requests allowed, while the
                                    second is the time window in seconds. For
                                    example, ``(300, 60 * 15)`` would set
                                    a limit of 300 requests every 15 minutes.
                                    Defaults to ``None``.

``RATE_LIMIT_POST``                 A tuple expressing the rate limit on POST
                                    requests. The first element of the tuple is
                                    the number of requests allowed, while the
                                    second is the time window in seconds. For
                                    example ``(300, 60 * 15)`` would set
                                    a limit of 300 requests every 15 minutes.
                                    Defaults to ``None``.

``RATE_LIMIT_PATCH``                A tuple expressing the rate limit on PATCH
                                    requests. The first element of the tuple is
                                    the number of requests allowed, while the
                                    second is the time window in seconds. For
                                    example ``(300, 60 * 15)`` would set
                                    a limit of 300 requests every 15 minutes.
                                    Defaults to ``None``.

``RATE_LIMIT_DELETE``               A tuple expressing the rate limit on DELETE
                                    requests. The first element of the tuple is
                                    the number of requests allowed, while the
                                    second is the time window in seconds. For
                                    example ``(300, 60 * 15)`` would set
                                    a limit of 300 requests every 15 minutes. Defaults to
                                    ``None``.

``DEBUG``                           ``True`` to enable Debug Mode, ``False``
                                    otherwise.

``ERROR``                           Allows to customize the error_code field. Defaults
                                    to ``_error``.

``HATEOAS``                         When ``False``, this option disables
                                    :ref:`hateoas_feature`. Defaults to ``True``.

``ISSUES``                          Allows to customize the issues field. Defaults
                                    to ``_issues``.

``STATUS``                          Allows to customize the status field. Defaults
                                    to ``_status``.

``STATUS_OK``                       Status message returned when data validation is
                                    successful. Defaults to ``OK``.

``STATUS_ERR``                      Status message returned when data validation
                                    failed. Defaults to ``ERR``.

``ITEMS``                           Allows to customize the items field. Defaults
                                    to ``_items``.

``META``                            Allows to customize the meta field. Defaults
                                    to ``_meta``

``INFO``                            String value to include an info section, with the
                                    given INFO name, at the Eve homepage (suggested
                                    value ``_info``). The info section will include
                                    Eve server version and API version (API_VERSION,
                                    if set).  ``None`` otherwise, if you do not want
                                    to expose any server info. Defaults to ``None``.

``LINKS``                           Allows to customize the links field. Defaults
                                    to ``_links``.

``ETAG``                            Allows to customize the etag field. Defaults
                                    to ``_etag``.

``IF_MATCH``                        ``True`` to enable concurrency control, ``False``
                                    otherwise. Defaults to ``True``. See
                                    :ref:`concurrency`.

``ENFORCE_IF_MATCH``                ``True`` to always enforce concurrency control when
                                    it is enabled, ``False`` otherwise. Defaults to
                                    ``True``. See :ref:`concurrency`.

``RENDERERS``                       Allows to change enabled renderers. Defaults to
                                    ``['eve.render.JSONRenderer', 'eve.render.XMLRenderer']``.

``JSON_SORT_KEYS``                  ``True`` to enable JSON key sorting, ``False``
                                    otherwise. Defaults to ``False``.

``JSON_REQUEST_CONTENT_TYPES``      Supported JSON content types. Useful when
                                    you need support for vendor-specific json
                                    types. Please note: responses will still
                                    carry the standard ``application/json``
                                    type. Defaults to ``['application/json']``.

``VALIDATION_ERROR_STATUS``         The HTTP status code to use for validation errors.
                                    Defaults to ``422``.

``VERSIONING``                      Enabled documents version control when
                                    ``True``. Can be overridden by resource
                                    settings. Defaults to ``False``.

``VERSIONS``                        Suffix added to the name of the primary
                                    collection to create the name of the shadow
                                    collection to store document versions.
                                    Defaults to ``_versions``. When
                                    ``VERSIONING`` is enabled , a collection
                                    such as ``myresource_versions`` would be
                                    created for a resource with a datasource of
                                    ``myresource``.

``VERSION_PARAM``                   The URL query parameter used to access the
                                    specific version of a document. Defaults to
                                    ``version``. Omit this parameter to get the
                                    latest version of a document or use
                                    `?version=all`` to get a list of all
                                    version of the document. Only valid for
                                    individual item endpoints.

``VERSION``                         Field used to store the version number of a
                                    document. Defaults to ``_version``.

``LATEST_VERSION``                  Field used to store the latest version number
                                    of a document. Defaults to
                                    ``_latest_version``.

``VERSION_ID_SUFFIX``               Used in the shadow collection to store the
                                    document id. Defaults to ``_document``. If
                                    ``ID_FIELD`` is set to ``_id``, the
                                    document id will be stored in field
                                    ``_id_document``.

``MONGO_URI``                       A `MongoDB URI`_ which is used in preference
                                    of the other configuration variables.

``MONGO_HOST``                      MongoDB server address. Defaults to ``localhost``.

``MONGO_PORT``                      MongoDB port. Defaults to ``27017``.

``MONGO_USERNAME``                  MongoDB user name.

``MONGO_PASSWORD``                  MongoDB password.

``MONGO_DBNAME``                    MongoDB database name.

``MONGO_OPTIONS``                   MongoDB keyword arguments to passed to
                                    MongoClient class ``__init__``.
                                    Defaults to ``{'connect': True, 'tz_aware': True, 'appname': 'flask_app_name'}``.
                                    See `PyMongo mongo_client`_ for reference.

``MONGO_AUTH_SOURCE``               MongoDB authorization database. Defaults to ``None``.

``MONGO_AUTH_MECHANISM``            MongoDB authentication mechanism.
                                    See `PyMongo Authentication Mechanisms`_.
                                    Defaults to ``None``.

``MONGO_AUTH_MECHANISM_PROPERTIES`` Specify MongoDB extra authentication mechanism properties
                                    if required. Defaults to ``None``.

``MONGO_QUERY_BLACKLIST``           A list of Mongo query operators that are not
                                    allowed to be used in resource filters
                                    (``?where=``). Defaults to ``['$where',
                                    '$regex']``.

                                    Mongo JavaScript operators are disabled by
                                    default, as they might be used as vectors
                                    for injection attacks. Javascript queries
                                    also tend to be slow and generally can be
                                    easily replaced with the (very rich) Mongo
                                    query dialect.

``MONGO_WRITE_CONCERN``             A dictionary defining MongoDB write concern
                                    settings. All standard write concern
                                    settings (w, wtimeout, j, fsync) are
                                    supported. Defaults to ``{'w': 1}``, which
                                    means 'do regular acknowledged writes'
                                    (this is also the Mongo default).

                                    Please be aware that setting 'w' to a value of
                                    2 or greater requires replication to be
                                    active or you will be getting 500 errors
                                    (the write will still happen; Mongo will
                                    just be unable to check that it's being
                                    written to multiple servers).

                                    Can be overridden at endpoint (Mongo
                                    collection) level. See
                                    ``mongo_write_concern`` below.

``DOMAIN``                          A dict holding the API domain definition.
                                    See `Domain Configuration`_.

``EXTENDED_MEDIA_INFO``             A list of properties to forward from the file upload
                                    driver.

``RETURN_MEDIA_AS_BASE64_STRING``   Controls the embedding of the media type in
                                    the endpoint response. This is useful when
                                    you have other means of getting the binary
                                    (like custom Flask endpoints) but still
                                    want clients to be able to POST/PATCH it.
                                    Defaults to ``True``.

``RETURN_MEDIA_AS_URL``             Set it to ``True`` to enable serving media
                                    files at a dedicated media endpoint.
                                    Defaults to ``False``.

``MEDIA_BASE_URL``                  Base URL to be used when
                                    ``RETURN_MEDIA_AS_URL`` is active. Combined
                                    with ``MEDIA_ENDPOINT`` and ``MEDIA_URL``
                                    dictates the URL returned for media files.
                                    If ``None``, which is the default value,
                                    the API base address will be used instead.

``MEDIA_ENDPOINT``                  The media endpoint to be used when
                                    ``RETURN_MEDIA_AS_URL`` is enabled.
                                    Defaults to ``media``.

``MEDIA_URL``                       Format of a file url served at the
                                    dedicated media endpoints. Defaults to
                                    ``regex("[a-f0-9]{24}")``.

``MULTIPART_FORM_FIELDS_AS_JSON``   In case you are submitting your resource as
                                    ``multipart/form-data`` all form data fields
                                    will be submitted as strings, breaking any
                                    validation rules you might have on the
                                    resource fields. If you want to treat all
                                    submitted form data as JSON strings you will
                                    have to activate this setting. In that case
                                    field validation will continue working
                                    correctly. Read more about how the fields
                                    should be formatted at
                                    :ref:`multipart`. Defaults to ``False``.

``AUTO_COLLAPSE_MULTI_KEYS``        If set to ``True``, multiple values sent
                                    with the same key, submitted using the
                                    ``application/x-www-form-urlencoded`` or
                                    ``multipart/form-data`` content types,
                                    will automatically be converted to a list of
                                    values.

                                    When using this together with
                                    ``AUTO_CREATE_LISTS`` it becomes possible
                                    to use lists of media fields.

                                    Defaults to ``False``

``AUTO_CREATE_LISTS``               When submitting a non ``list`` type value
                                    for a field with type ``list``,
                                    automatically create a one element list
                                    before running the validators.

                                    Defaults to ``False``

``OPLOG``                           Set it to ``True`` to enable the :ref:`oplog`.
                                    Defaults to ``False``.

``OPLOG_NAME``                      This is the name of the database collection
                                    where the :ref:`oplog` is stored. Defaults
                                    to ``oplog``.

``OPLOG_METHODS``                   List of HTTP methods which operations
                                    should be logged in the :ref:`oplog`.
                                    Defaults to ``['DELETE', 'POST', 'PATCH',
                                    'PUT']``.

``OPLOG_CHANGE_METHODS``            List of HTTP methods which operations
                                    will include changes into the :ref:`oplog` entry.
                                    Defaults to ``['DELETE','PATCH', 'PUT']``.

``OPLOG_ENDPOINT``                  Name of the :ref:`oplog` endpoint. If the
                                    endpoint is enabled it can be configured
                                    like any other API endpoint. Set it to
                                    ``None`` to disable the endpoint. Defaults
                                    to ``None``.

``OPLOG_AUDIT``                     Set it to ``True`` to enable the audit
                                    feature. When audit is enabled client IP
                                    and document changes are also logged to the
                                    :ref:`oplog`. Defaults to ``True``.

``OPLOG_RETURN_EXTRA_FIELD``        When enabled, the optional ``extra`` field
                                    will be included in the payload returned by
                                    the ``OPLOG_ENDPOINT``. Defaults to
                                    ``False``.

``SCHEMA_ENDPOINT``                 Name of the :ref:`schema_endpoint`. Defaults
                                    to ``None``.

``HEADER_TOTAL_COUNT``              Custom header containing total count of
                                    items in response payloads for collection
                                    ``GET`` requests. This is handy for ``HEAD``
                                    requests when client wants to know items
                                    count without retrieving response body.
                                    An example use case is to get the count
                                    of unread posts using ``where`` query without
                                    loading posts themselves. Defaults to
                                    ``X-Total-Count``.

``JSONP_ARGUMENT``                  This option will cause the response to be
                                    wrapped in a JavaScript function call if
                                    the argument is set in the request. For
                                    example if you set ``JSON_ARGUMENT
                                    = 'callback'``, then all responses to
                                    ``?callback=funcname`` requests will be
                                    wrapped in a ``funcname`` call. Defaults to
                                    ``None``.

``BULK_ENABLED``                    Enables bulk insert when set to ``True``.
                                    See :ref:`bulk_insert` for more
                                    information. Defaults to ``True``.

``SOFT_DELETE``                     Enables soft delete when set to ``True``.
                                    See :ref:`soft_delete` for more
                                    information. Defaults to ``False``.

``DELETED``                         Field name used to indicate if a document
                                    has been deleted when ``SOFT_DELETE``
                                    is enabled. Defaults to ``_deleted``.

``SHOW_DELETED_PARAM``              The URL query parameter used to include
                                    soft deleted items in resource level GET
                                    responses. Defaults to 'show_deleted'.

``STANDARD_ERRORS``                 This is a list of HTTP error codes for
                                    which a standard API response will be
                                    provided. Canonical error response includes
                                    a JSON body with actual error code and
                                    description. Set this to an empty list if
                                    you want to disable canonical responses
                                    altogether. Defaults to ``[400, 401, 403,
                                    404, 405, 406, 409, 410, 412, 422, 428]``

``VALIDATION_ERROR_AS_STRING``      If ``True`` even single field errors will
                                    be returned in a list. By default single
                                    field errors are returned as strings while
                                    multiple field errors are bundled in a
                                    list. If you want to standardize the field
                                    errors output, set this setting to ``True``
                                    and you will always get a list of field
                                    issues. Defaults to ``False``.

``UPSERT_ON_PUT``                   ``PUT`` attempts to create a document if it
                                    does not exist. The URL endpoint will be
                                    used as ``ID_FIELD`` value (if ``ID_FIELD``
                                    is included with the payload, it will be
                                    ignored). Normal validation rules apply.
                                    The response will be a ``201 Created`` on
                                    successful creation. Response payload will
                                    be identical the one you would get by
                                    performing a single document POST to the
                                    resource endpoint. Set to ``False`` to
                                    disable this feature, and a ``404`` will be
                                    returned instead. Defaults to ``True``.

``MERGE_NESTED_DOCUMENTS``          If ``True``, updates to nested fields are
                                    merged with the current data on ``PATCH``.
                                    If ``False``, the updates overwrite the
                                    current data. Defaults to ``True``.

``NORMALIZE_DOTTED_FIELDS``         If ``True``, dotted fields are parsed
                                    and processed as subdocument fields. If
                                    ``False``, dotted fields are left unparsed
                                    and unprocessed, and the payload is passed
                                    to the underlying data-layer as-is. Please
                                    note that with the default Mongo layer,
                                    setting this to ``False`` will result in an
                                    error. Defaults to ``True``.

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
``url``                         The endpoint URL. If omitted the resource key
                                of the ``DOMAIN`` dict will be used to build
                                the URL. As an example, ``contacts`` would make
                                the `people` resource available at
                                ``/contacts`` (instead of ``/people``). URL can
                                be as complex as needed and can be nested
                                relative to another API endpoint (you can have
                                a ``/contacts`` endpoint and then
                                a ``/contacts/overseas`` endpoint. Both are
                                independent of each other and freely
                                configurable).

                                You can also use regexes to setup
                                subresource-like endpoints. See
                                :ref:`subresources`.

``allowed_filters``             List of fields on which filtering is allowed.
                                Entries in this list work in a hierarchical
                                way. This means that, for instance, filtering
                                on ``'dict.sub_dict.foo'`` is allowed if
                                ``allowed_filters`` contains any of
                                ``'dict.sub_dict.foo``, ``'dict.sub_dict'``
                                or ``'dict'``. Instead filtering on
                                ``'dict'`` is allowed if ``allowed_filters``
                                contains ``'dict'``.
                                Can be set to ``[]`` (no filters allowed), or
                                ``['*']`` (fields allowed on every field).
                                Defaults to ``['*']``.

                                *Please note:* If API scraping or DB DoS
                                attacks are a concern, then globally disabling
                                filters (see ``ALLOWED_FILTERS`` above) and
                                then whitelisting valid ones at the local level
                                is the way to go.

``sorting``                     如果启用了排序，``True``，否则 ``False``。本地
                                重载 ``SORTING``。

``pagination``                  如果启用了分页，``True``，否则 ``False``。本地
                                重载 ``PAGINATION``。

``resource_methods``            A list of HTTP methods supported at resource
                                endpoint. Allowed values: ``GET``, ``POST``,
                                ``DELETE``. Locally overrides
                                ``RESOURCE_METHODS``.

                                *Please note:* if you're running version 0.0.5
                                or earlier use the now unsupported ``methods``
                                keyword instead.

``public_methods``              A list of HTTP methods supported at resource
                                endpoint, open to public access even when
                                :ref:`auth` is enabled. Locally overrides
                                ``PUBLIC_METHODS``.

``item_methods``                A list of HTTP methods supported at item
                                endpoint. Allowed values: ``GET``, ``PATCH``,
                                ``PUT`` and ``DELETE``. ``PATCH`` or, for
                                clients not supporting PATCH, ``POST`` with
                                the ``X-HTTP-Method-Override`` header tag.
                                Locally overrides ``ITEM_METHODS``.

``public_item_methods``         A list of HTTP methods supported at item
                                endpoint, left open to public access when
                                :ref:`auth` is enabled. Locally overrides
                                ``PUBLIC_ITEM_METHODS``.

``allowed_roles``               A list of allowed `roles` for resource
                                endpoint. See :ref:`auth` for more
                                information. Locally overrides
                                ``ALLOWED_ROLES``.

``allowed_read_roles``          A list of allowed `roles` for resource
                                endpoint with GET and OPTIONS methods.
                                See :ref:`auth` for more
                                information. Locally overrides
                                ``ALLOWED_READ_ROLES``.

``allowed_write_roles``         A list of allowed `roles` for resource
                                endpoint with POST, PUT and DELETE.
                                See :ref:`auth` for more
                                information. Locally overrides
                                ``ALLOWED_WRITE_ROLES``.

``allowed_item_read_roles``     A list of allowed `roles` for item endpoint
                                with GET and OPTIONS methods.
                                See :ref:`auth` for more information.
                                Locally overrides ``ALLOWED_ITEM_READ_ROLES``.


``allowed_item_write_roles``    A list of allowed `roles` for item endpoint
                                with PUT, PATH and DELETE methods.
                                See :ref:`auth` for more information.
                                Locally overrides ``ALLOWED_ITEM_WRITE_ROLES``.

``allowed_item_roles``          A list of allowed `roles` for item endpoint.
                                See :ref:`auth` for more information.
                                Locally overrides ``ALLOWED_ITEM_ROLES``.

``cache_control``               Value of the ``Cache-Control`` header field
                                used when serving ``GET`` requests. Leave empty
                                if you don't want to include cache directives
                                with API responses. Locally overrides
                                ``CACHE_CONTROL``.

``cache_expires``               Value (in seconds) of the ``Expires`` header
                                field used when serving ``GET`` requests. If
                                set to a non-zero value, the header will
                                always be included, regardless of the setting
                                of ``CACHE_CONTROL``. Locally overrides
                                ``CACHE_EXPIRES``.

``id_field``                    Field used to uniquely identify resource items
                                within the database. Locally overrides
                                ``ID_FIELD``.

``item_lookup``                 ``True`` if item endpoint should be available,
                                ``False`` otherwise. Locally overrides
                                ``ITEM_LOOKUP``.

``item_lookup_field``           Field used when looking up a resource
                                item. Locally overrides ``ITEM_LOOKUP_FIELD``.

``item_url``                    用于创建数据项终结点 URL 的规则。本地重载 ``ITEM_URL``。

``resource_title``              Title used when building resource links
                                (HATEOAS). Defaults to resource's ``url``.

``item_title``                  Title to be used when building item references,
                                both in XML and JSON responses. Overrides
                                ``ITEM_TITLE``.

``additional_lookup``           Besides the standard item endpoint which
                                defaults to ``/<resource>/<ID_FIELD_value>``,
                                you can optionally define a secondary,
                                read-only, endpoint like
                                ``/<resource>/<person_name>``. You do so by
                                defining a dictionary comprised of two items
                                `field` and `url`. The former is the name of
                                the field used for the lookup. If the field
                                type (as defined in the resource schema_) is
                                a string, then you put a URL rule in `url`.  If
                                it is an integer, then you just omit `url`, as
                                it is automatically handled.  See the code
                                snippet below for an usage example of this
                                feature.

``datasource``                  明确地链接 API 资源到数据库集合。参考 `Advanced Datasource
                                Patterns`_。

``auth_field``                  Enables :ref:`user-restricted`. When the
                                feature is enabled, users can only
                                read/update/delete resource items created by
                                themselves. The keyword contains the actual
                                name of the field used to store the id of
                                the user who created the resource item. Locally
                                overrides ``AUTH_FIELD``.

``allow_unknown``               When ``True``, this option will allow insertion
                                of arbitrary, unknown fields to the endpoint.
                                Use with caution. Locally overrides
                                ``ALLOW_UNKNOWN``. See :ref:`unknown` for more
                                information. Defaults to ``False``.

``transparent_schema_rules``    When ``True``, this option disables
                                :ref:`schema_validation` for the endpoint.

``projection``                  When ``True``, this option enables the
                                :ref:`projections` feature. Locally overrides
                                ``PROJECTION``. Defaults to ``True``.

``embedding``                   When ``True`` this option enables the
                                :ref:`embedded_docs` feature. Defaults to
                                ``True``.

``extra_response_fields``       Allows to configure a list of additional
                                document fields that should be provided with
                                every POST response. Normally only
                                automatically handled fields (``ID_FIELD``,
                                ``LAST_UPDATED``, ``DATE_CREATED``, ``ETAG``)
                                are included in response payloads. Overrides
                                ``EXTRA_RESPONSE_FIELDS``.

``hateoas``                     When ``False``, this option disables
                                :ref:`hateoas_feature` for the resource.
                                Defaults to ``True``.

``mongo_write_concern``         A dictionary defining MongoDB write concern
                                settings for the endpoint datasource. All
                                standard write concern settings (w, wtimeout, j,
                                fsync) are supported. Defaults to ``{'w': 1}``
                                which means 'do regular acknowledged writes'
                                (this is also the Mongo default.)

                                Please be aware that setting 'w' to a value of
                                2 or greater requires replication to be active
                                or you will be getting 500 errors (the write
                                will still happen; Mongo will just be unable
                                to check that it's being written to multiple
                                servers.)

``mongo_prefix``                Allows overriding of the default ``MONGO``
                                prefix, which is used when retrieving MongoDB
                                settings from configuration.

                                For example if ``mongo_prefix`` is set to
                                ``MONGO2`` then, when serving requests for the
                                endpoint, ``MONGO2`` prefixed settings will
                                be used to access the database.

                                This allows for eventually serving data from
                                a different database/server at every endpoint.

                                See also: :ref:`authdrivendb`.

``mongo_indexes``               Allows to specify a set of indexes to be
                                created for this resource before the app is
                                launched.

                                Indexes are expressed as a dict where keys are
                                index names and values are either a list of
                                tuples of (field, direction) pairs, or
                                a tuple with a list of field/direction pairs
                                *and* index options expressed as a dict, such
                                as ``{'index name': [('field', 1)], 'index with
                                args': ([('field', 1)], {"sparse": True})}``.

                                Multiple pairs are used to create compound
                                indexes. Direction takes all kind of values
                                supported by PyMongo, such as ``ASCENDING``
                                = 1 and ``DESCENDING`` = -1. All index options
                                such as ``sparse``, ``min``, ``max``,
                                etc. are supported (see PyMongo_ documentation.)

                                *Please note:* keep in mind that index design,
                                creation and maintenance is a very important
                                task and should be planned and executed with
                                great care. Usually it is also a very resource
                                intensive operation. You might therefore want
                                to handle this task manually, out of the
                                context of API instantiation. Also remember
                                that, by default, any already exsistent index
                                for which the definition has been changed, will
                                be dropped and re-created.

``authentication``              A class with the authorization logic for the
                                endpoint. If not provided the eventual
                                general purpose auth class (passed as
                                application constructor argument) will be used.
                                For details on authentication and authorization
                                see :ref:`auth`.  Defaults to ``None``,

``embedded_fields``             A list of fields for which :ref:`embedded_docs`
                                is enabled by default. For this feature to work
                                properly fields in the list must be
                                ``embeddable``, and ``embedding`` must be
                                active for the resource.

``query_objectid_as_string``    When enabled the Mongo parser will avoid
                                automatically casting electable strings to
                                ObjectIds. This can be useful in those rare
                                occurrences where you have string fields in the
                                database whose values can actually be casted to
                                ObjectId values, but shouldn't. It effects
                                queries (``?where=``) and parsing of payloads.
                                Defaults to ``False``.

``internal_resource``           When ``True``, this option makes the resource
                                internal. No HTTP action can be performed on
                                the endpoint, which is still accessible from
                                the Eve data layer. See
                                :ref:`internal_resources` for more
                                information. Defaults to ``False``.

``etag_ignore_fields``          List of fields that
                                should not be used to compute the ETag value.
                                Defaults to ``None`` which means that by
                                default all fields are included in the computation.
                                It looks like ``['field1', 'field2',
                                'field3.nested_field', ...]``.

``schema``                      A dict defining the actual data structure being
                                handled by the resource. Enables data
                                validation. See `Schema Definition`_.

``bulk_enabled``                When ``True`` this option enables the
                                :ref:`bulk_insert` feature for this resource.
                                Locally overrides ``BULK_ENABLED``.

``soft_delete``                 When ``True`` this option enables the
                                :ref:`soft_delete` feature for this resource.
                                Locally overrides ``SOFT_DELETE``.

``merge_nested_documents``      If ``True``, updates to nested fields are
                                merged with the current data on ``PATCH``.
                                If ``False``, the updates overwrite the
                                current data. Locally overrides
                                ``MERGE_NESTED_DOCUMENTS``.
``normalize_dotted_fields``     If ``True``, dotted fields are parsed and
                                processed as subdocument fields. If ``False``,
                                dotted fields are left unparsed and
                                unprocessed, and the payload is passed to the
                                underlying data-layer as-is. Please note that
                                with the default Mongo layer, setting this to
                                ``False`` will result in an error. Defaults to
                                ``True``.

=============================== ===============================================

这里是一个资源自定义的示例，主要通过重载全局 API 设置实现:

::

    people = {
        # 'title' tag used in item links. Defaults to the resource title minus
        # the final, plural 's' (works fine in most cases but not for 'people')
        'item_title': 'person',

        # by default, the standard item entry point is defined as
        # '/people/<ObjectId>/'. We leave it untouched, and we also enable an
        # additional read-only entry point. This way consumers can also perform
        # GET requests at '/people/<lastname>'.
        'additional_lookup': {
            'url': 'regex("[\w]+")',
            'field': 'lastname'
        },

        # We choose to override global cache-control directives for this resource.
        'cache_control': 'max-age=10,must-revalidate',
        'cache_expires': 10,

        # we only allow GET and POST at this resource endpoint.
        'resource_methods': ['GET', 'POST'],
    }

.. _schema:

模式定义
-----------------
除非你的 API 是只读的，你很可能希望定义资源 `模式`。模式很重要，因为它们对进来的流启用合适的验证。

::

    # 'people' schema definition
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
        # 'role' is a list, and can only contain values from 'allowed'.
        'role': {
            'type': 'list',
            'allowed': ["author", "contributor", "copy"],
        },
        # An embedded 'strongly-typed' dictionary.
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
``type``                        Field data type. Can be one of the following:

                                - ``string``
                                - ``boolean``
                                - ``integer``
                                - ``float``
                                - ``number`` (integer and float values allowed)
                                - ``datetime``
                                - ``dict``
                                - ``list``
                                - ``media``

                                If the MongoDB data layer is used then
                                ``objectid``, ``dbref`` and geographic data
                                structures are also allowed:

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

                                See :ref:`GeoJSON <geojson_feature>` for more
                                information geo fields.

``required``                    If ``True``, the field is mandatory on
                                insertion.

``readonly``                    If ``True``, the field is readonly.

``minlength``, ``maxlength``    Minimum and maximum length allowed for
                                ``string`` and ``list`` types.

``min``, ``max``                Minimum and maximum values allowed for
                                ``integer``, ``float`` and ``number`` types.

``allowed``                     List of allowed values for ``string`` and
                                ``list`` types.

``empty``                       Only applies to string fields. If ``False``,
                                validation will fail if the value is empty.
                                Defaults to ``True``.

``items``                       Defines a list of values allowed in a ``list``
                                of fixed length, see `docs <http://docs.python-cerberus.org/en/latest/usage.html#items-list>`_.

``schema``                      Validation schema for ``dict`` types and
                                arbitrary length ``list`` types. For details
                                and usage examples, see `Cerberus documentation <http://docs.python-cerberus.org/en/latest/usage.html#schema-dict>`_.

``unique``                      The value of the field must be unique within
                                the collection.

                                Please note: validation constraints are checked
                                against the database, and not between the
                                payload documents themselves. This causes an
                                interesting corner case: in the event of
                                a multiple documents payload where two or more
                                documents carry the same value for a field
                                where the 'unique' constraint is set, the
                                payload will validate successfully, as there
                                are no duplicates in the database (yet).

                                If this is an issue, the client can always send
                                the documents one at a time for insertion, or
                                validate locally before submitting the payload
                                to the API.

``unique_to_user``              The field value is unique to the user. This is
                                useful when :ref:`user-restricted` is
                                enabled on an endpoint. The rule will be
                                validated against *user data only*. So in this
                                scenario duplicates are allowed as long as they
                                are stored by different users. Conversely,
                                a single user cannot store duplicate values.

                                If URRA is not active on the endpoint, this
                                rule behaves like ``unique``

``data_relation``               Allows to specify a referential integrity rule
                                that the value must satisfy in order to
                                validate. It is a dict with four keys:

                                - ``resource``: the name of the resource being referenced;
                                - ``field``: the field name in the foreign resource;
                                - ``embeddable``: set to ``True`` if clients can
                                  request the referenced document to be embedded
                                  with the serialization. See :ref:`embedded_docs`. Defaults to ``False``.
                                - ``version``: set to ``True`` to require a
                                  ``_version`` with the data relation. See :ref:`document_versioning`.
                                  Defaults to ``False``.

``nullable``                    If ``True``, the field value can be set to
                                ``None``.

``default``                     The default value for the field. When serving
                                POST and PUT requests, missing fields will be
                                assigned the configured default values.

                                It works also for types ``dict`` and ``list``.
                                The latter is restricted and works only for
                                lists with schemas (list with a random number
                                of elements and each element being a ``dict``)

                                ::

                                    schema = {
                                      # Simple default
                                      'title': {
                                        'type': 'string',
                                        'default': 'M.'
                                      },
                                      # Default in a dict
                                      'others': {
                                        'type': 'dict',
                                        'schema': {
                                          'code': {
                                            'type': 'integer',
                                            'default': 100
                                          }
                                        }
                                      },
                                      # Default in a list of dicts
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

``versioning``                  Enabled documents version control when ``True``.
                                Defaults to ``False``.

``versioned``                   If ``True``, this field will be included in the
                                versioned history of each document when
                                ``versioning`` is enabled. Defaults to ``True``.

``valueschema``                 Validation schema for all values of a ``dict``.
                                The dict can have arbitrary keys, the values
                                for all of which must validate with given
                                schema. See `valueschema <http://docs.python-cerberus.org/en/latest/validation-rules.html#valueschema>`_ in Cerberus docs.

``keyschema``                   This is the counterpart to ``valueschema`` that
                                validates the keys of a dict.   Validation
                                schema for all values of a ``dict``. See
                                `keyschema <http://docs.python-cerberus.org/en/latest/validation-rules.html#keyschema>`_ in Cerberus docs.


``regex``                       Validation will fail if field value does not
                                match the provided regex rule. Only applies to
                                string fields. See `regex <http://docs.python-cerberus.org/en/latest/validation-rules.html#regex>`_ in Cerberus docs.


``dependencies``                This rule allows a list of fields that must be
                                present in order for the target field to be
                                allowed. See `dependencies <http://docs.python-cerberus.org/en/latest/validation-rules.html#dependencies>`_  in Cerberus docs.

``anyof``                       This rule allows you to list multiple sets of
                                rules to validate against. The field will be
                                considered valid if it validates against one
                                set in the list. See `*of-rules <http://docs.python-cerberus.org/en/latest/validation-rules.html#of-rules>`_ in Cerberus docs.

``allof``                       Same as ``anyof``, except that all rule
                                collections in the list must validate.

``noneof``                      Same as ``anyof``, except that it requires no
                                rule collections in the list to validate.

``oneof``                       Same as ``anyof``, except that only one rule
                                collections in the list can validate.

``coerce``                      Type coercion allows you to apply a callable to
                                a value before any other validators run. The
                                return value of the callable replaces the new
                                value in the document. This can be used to
                                convert values or sanitize data before it is
                                validated. See `value coercion <http://docs.python-cerberus.org/en/latest/normalization-rules.html#value-coercion>`_ in Cerberus docs.

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

                                - ``pipeline``. 聚合管道。语法必须匹配 PyMongo 
                                支持的那个。要获取更多信息，参考 
                                `PyMongo Aggregation Examples`_ 和 官方的 
                                `MongoDB Aggregation Framework`_ 文档。

                                - ``options``. 聚合选项。必须是一个字典，包含
                                一个或更多这些键:

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
