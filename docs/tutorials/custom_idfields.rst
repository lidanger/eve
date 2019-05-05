.. _custom_ids:

处理自定义 ID 字段
=========================

当来到单个文档终结点时，在大多数情况下，除了定义父资源节点外，你不用做任何事情。因此，比如说，你配置了一个 ``/invoices`` 终结点，它允许客户端查询潜在的 `invoices` 数据库集合。``/invoices/<ObjectId>`` 终结点将被框架设为可用，并被客户端用于获取和/或编辑单个文档。默认情况下，当 ``ID_FIELD`` 字段是 ``ObjectId`` 类型时，Eve 可以天衣无缝地提供这个特性。

但是，你可能有唯一标识不是一个 ``ObjectId`` 的结合，而你仍然希望单个文档终结点正确运行。不要担心，它是可以实现的，只需要一点小修补。

处理 ``UUID`` 字段
------------------------
在这篇教程中，我们将考虑一个方案，在这个方案中，我们数据库集合 (invoices) 中的一个使用 UUID 字段作为唯一标识。我们希望我们的 API 暴露一个文档终结点，类似 ``/invoices/uuid``，翻译过来像:

``/invoices/48c00ee9-4dbe-413f-9fc3-d5f12a91de1c``.

这些是我们需要按着做的步骤:

1. 精心制作一个自定义的能将 UUID 序列化为字符串的 JSONEncoder，并将它传递给我们的 Eve 应用程序。
2. 对新的 ``uuid`` 数据类型添加支持，这样我们就可以正确地验证收到的 uuid 值。
3. 配置我们的 invoices 终结点，这样 Eve 才知道如何正确解析 UUID urls。

自定义 JSONEncoder
~~~~~~~~~~~~~~~~~~
Eve 的默认 JSON 序列化器完美胜任普通数据类型的序列化，像 ``datetime`` (序列化为一个 RFC1123 字符串，像 ``Sat, 23 Feb 1985
12:00:00 GMT``) 和 ``ObjectId`` 值 (也序列化为字符串)。

由于我们正在添加对未知数据类型的支持，我们也需要告诉我们的 Eve 实例如何正确地序列化它。这就像继承一个标准的 ``JSONEncoder`` 一样简单，或者，甚至更简单。由于继承自 Eve 自己的 ``BaseJSONEncoder``，我们的自定义序列化器会保留 Eve 所有的序列化逻辑:

.. code-block:: python

    from eve.io.base import BaseJSONEncoder
    from uuid import UUID

    class UUIDEncoder(BaseJSONEncoder):
        """ JSONEconder 子类，由 json 渲染函数使用。
        它不同于 BaseJSONEoncoder，因为它也可以处理 UUID 编码的地址
        """

        def default(self, obj):
            if isinstance(obj, UUID):
                return str(obj)
            else:
                # delegate rendering to base class method (the base class
                # will properly render ObjectIds, datetimes, etc.)
                return super(UUIDEncoder, self).default(obj)


``UUID`` 验证
~~~~~~~~~~~~~~~~~~~
默认情况下，Eve 为每一个新插入的文档创建一个唯一标识，当然了，是 ``ObjectId`` 类型。在这个终结点上这不是我们希望发生的。在这里，我们希望客户端自己提供唯一索引，而且我们也希望验证它们是 UUID 类型。为了实现这个，我们首先需要扩展我们的数据验证层 (查看 :ref:`validation` 获取关于自定义验证的详细信息):

.. code-block:: python

    from eve.io.mongo import Validator
    from uuid import UUID

    class UUIDValidator(Validator):
        """
        扩展基本的 mongo 验证器，添加对 uuid 数据类型的支持
        """
        def _validate_type_uuid(self, value):
            try:
                UUID(value)
            except ValueError:
                pass

``UUID`` URLs
~~~~~~~~~~~~~
现在，Eve 已经能胜任渲染和验证 UUID 值了，但是它仍然不知道哪个资源将使用这些特性。我们也需要设置 ``item_url``，这样 uuid 格式的 urls 可以被正确地解析。让我们选择我们的 ``settings.py`` 模块并更新对应的 API 域:

.. code-block:: python

    invoices = {
        # 这个资源项终结点 (/invoices/<id>) 将匹配一个 UUID regex.
        'item_url': 'regex("[a-f0-9]{8}-?[a-f0-9]{4}-?4[a-f0-9]{3}-?[89ab][a-f0-9]{3}-?[a-f0-9]{12}")',
        'schema': {
            # 将我们的 _id 字段设置为我们的自定义 uuid 类型.
            '_id': {'type': 'uuid'},
        },
    }

    DOMAIN = {
        'invoices': invoices
    }

如果你所有的 API 资源都要支持 uuid 作为唯一文档标识，那你可能只需要设置全局的 ``ITEM_URL`` 为 uuid regex，避免每一个资源终结点都要设置。

将 ``UUID`` juice 传递给 Eve
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
现在，所有丢失的碎片都找到了，我们只需要告诉 Eve 如何使用它们就行了。Eve 需要在构建 URL 映射时了解新数据类型，因此当我们实例化应用程序时，我们需要在刚开始就传递我们的自定义类:

.. code-block:: python

    app = Eve(json_encoder=UUIDEncoder, validator=UUIDValidator)


记住，如果你正在使用自定义 ``ID_FIELD`` 值，那么你不应该依赖 MongoDB (和 Eve) 来为你自动生成 ``ID_FIELD``。你应该像这样传递值:

::

    POST
    {"name":"bill", "_id":"48c00ee9-4dbe-413f-9fc3-d5f12a91de1c"}

.. _`custom url converters`: http://werkzeug.pocoo.org/docs/routing/#custom-converters
.. _Flask: http://flask.pocoo.org/
.. _Werkzeug: http://werkzeug.pocoo.org/
