.. _validation:

数据验证
===============
数据验证被提供为开箱即用的模式。你的配置包含对每一个 API 管理的资源的模式定义。发送到 API 即将被插入/更新的数据会被模式验证，如果验证通过，一个资源只会被更新。

.. code-block:: console

    $ curl -d '[{"firstname": "bill", "lastname": "clinton"}, {"firstname": "mitt", "lastname": "romney"}]' -H 'Content-Type: application/json' http://eve-demo.herokuapp.com/people
    HTTP/1.1 201 OK

响应将对请求中提供的每一项包含一个成功/错误状态:

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

在上面的例子中，第一个文档未验证通过，因此整个请求已经被拒绝。

当所有文档通过验证并被正确地插入后，响应状态是 ``201 Created``。如果任何一个文档验证失败，响应状态是 ``422 Unprocessable Entity``，或者任何其他通过 ``VALIDATION_ERROR_STATUS`` 配置定义的错误代码。

要获取如何定义文档模式和标准验证规则的信息，参考 :ref:`schema`。

扩展数据验证
-------------------------
数据验证基于 Cerberus_ 验证系统，因此它是可扩展的。确切地说，Eve 的 MongoDB 数据层自己就扩展了 Cerberus 验证，在标准规则的基础上实现了 ``unique`` 和 ``data_relation`` 约束，``ObjectId`` 数据类型和 ``decimal128``。

.. _custom_validation_rules:

自定义验证规则
------------------------
假设在你指定和非常特殊的使用场景，一个特定的值只能被表达为一个奇数。你决定为我们的验证模式添加对一些 ``isodd`` 规则的支持。这是实现的办法:

.. code-block:: python

    from eve.io.mongo import Validator

    class MyValidator(Validator):
        def _validate_isodd(self, isodd, field, value):
            if isodd and not bool(value & 1):
                self._error(field, "Value must be an odd number")

    app = Eve(validator=MyValidator)

    if __name__ == '__main__':
        app.run()

通过继承 Mongo 验证器基类然后添加一个自定义的 ``_validate_<rulename>`` 方法，你扩展了可用的 :ref:`schema` 语法，而现在，新的自定义规则 ``isodd`` 在你的模式中就可用了。你现在可以做某些事情，像:

.. code-block:: python

    'schema': {
        'oddity': {
            'isodd': True,
            'type': 'integer'
          }
    }

Cerberus 和 Eve 也提供 `function-based validation`_ 和 `type coercion`_，对基于类的自定义验证的轻量级替代品。

自定义数据类型
-----------------
你也可以通过简单添加 ``_validate_type_<typename>`` 方法到你的子类来添加一个新数据类型。考虑如下来自 Eve 源代码的片段。

.. code-block:: python

    def _validate_type_objectid(self, value):
        """ Enables validation for `objectid` schema attribute.

        :param value: field value.
        """
        if isinstance(value, ObjectId):
            return True

这个方法在你的模式用启用对 MongoDB ``ObjectId`` 类型的支持，允许像这个的东西:

.. code-block:: python

    'schema': {
        'owner': {
            'type': 'objectid',
            'required': True,
        },
    }

你也可以在 `source code`_ 中查看 Eve 自定义验证，你会发现更多高级使用场景，诸如 ``unique`` 和 ``data_relation`` 约束的实现。

有关更多信息

.. 注意::

    我们只是划破了数据验证的表面而已。请确保查看 Cerberus_ 文档来获取可用验证规则和数据类型的一个完整列表。

    也要注意所需的 Cerberus 定在版本 0.9.2，它仍然支持 ``validate_update`` 方法用于 ``PATCH`` 请求。Eve 0.8 版本升级到 Cerberus 1.0+ 已经在计划中。

.. _unknown:

允许 Unknown
--------------------
正常情况下，你不希望客户端注入未知字段到你的文档中。但是，可能存在希望这么做的状况。在开发周期中，例如，或者当你正在处理非常多样的数据。毕竟，非强制规范化的信息是 MongoDB 和 很多其他 NoSQL 数据存储的一项卖点。

在 Eve 中，你可以通过设置 ``ALLOW_UNKNOWN`` 选项为 ``True`` 来实现这个。一旦这个选项启用，匹配模式的字段将被正常验证，而未知字段将被无差错地悄悄存储。你也可以通过设置 ``allow_unknown`` 本地选项只为特定终结点启用这项特性。

考虑下面的域:

.. code-block:: python

    DOMAIN: {
        'people': {
            'allow_unknown': True,
            'schema': {
                'firstname': {'type': 'string'},
                }
            }
        }

正常情况下，你只可以添加 (POST) 或编辑 (PATCH) `firstnames` 到 ``/people`` 终结点。但是，由于 ``allow_unknown`` 已经启用，甚至像这样一个有效负载也会被接受:

.. code-block:: console

    $ curl -d '[{"firstname": "bill", "lastname": "clinton"}, {"firstname": "bill", "age":70}]' -H 'Content-Type: application/json' http://eve-demo.herokuapp.com/people
    HTTP/1.1 201 OK

.. 提示:: 请注意

    要十分小心地使用这项特性。也要意识到，当这个选项启用时，客户端实际上将有能力通过 PATCH (编辑) `添加` 字段。

``ALLOW_UNKNOWN`` 对只读 API 或需要返回在潜在数据库中找到的整个文档的终结点也很有用。在这个方案中，你不希望动用验证模式。对整个 API 来说，只要设置 ``ALLOW_UNKNOWN`` 为 ``True``，然后在每个终结点设置 ``schema: {}``。对单个终结点来说，使用 ``allow_unknown: True`` 代替。

.. _schema_validation:

模式验证
-----------------

默认情况下，模式被验证以确保它们遵从记录在 :ref:`schema` 中的结构。

要处理非标准的模式，请为模式中使用的非遵从键添加 :ref:`custom_validation_rules`。

.. _Cerberus: http://python-cerberus.org
.. _`source code`: https://github.com/pyeve/eve/blob/master/eve/io/mongo/validation.py
.. _`function-based validation`: http://docs.python-cerberus.org/en/latest/customize.html#function-validator
.. _`type coercion`: http://docs.python-cerberus.org/en/latest/usage.html#type-coercion
