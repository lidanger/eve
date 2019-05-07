RESTful 账户管理
==========================
.. 提示:: 请注意

    这篇教程假定你已经阅读了 :ref:`quickstart` 和 :ref:`auth` 指南。

除了相对稀有的事情，开放的 (并且通常只读) 公共 API，大多数服务只有授权的用户可以访问。
一个通用模式是，用户在一个网站或移动应用上创建他们自己的账户。一旦他们有了账户，他们就
被允许使用一个或更多 API。这是大多数社交网络和服务提供者 (Twitter, Facebook, Netflix
等等) 使用的模型。因此，你，服务提供者，如何使用正在被账户它们自己使用的同一个 API 管理
创建，编辑和删除账户等活动?

在下面的段落中，我们将看到一对可能的账户管理实现，都密集使用很多 Eve 特性，诸如
:ref:`endpointsec`, :ref:`roleaccess`, :ref:`user-restricted`,
:ref:`eventhooks`。

我们假定 SSL/TLS 是启用的，这意味着我们的传输层是加密的，使得 :ref:`basic` 
和 :ref:`token` 都是保护 API 终结点的有效选项。

比如说，我们在升级我们在 :ref:`quickstart` 教程中定义的 API。

.. _accounts_basic:

使用基本身份验证的账户
-----------------------------------
我们的任务是如下:

1. 制作一个可以进行所有账户管理活动 (``/accounts``) 的终结点。
2. 保护终结点，这样，它只能由我们控制的客户端: 我们自己的带有账户管理能力的网站，
   移动应用等等，访问。
3. 确保所有其他 API 终结点只能由授权的账户 (借助上面提到的终结点创建的) 访问。
4. 使授权的用户只能访问他们自己创建的资源。

1. ``/accounts`` 终结点
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
账户管理终结点与任何其他 API 终结点没什么不同，它只是一个在我们的配置文件中声明的问题。
让我们先声明资源模式。

::

        schema =  {
            'username': {
                'type': 'string',
                'required': True,
                'unique': True,
                },
            'password': {
                'type': 'string',
                'required': True,
            },
        },

然后，让我们定义终结点。

::

    accounts = {
        # 标准的账户入口点定义为 '/accounts/<ObjectId>'。我们再定义一个额外的
        # 可以在 '/accounts/<username>' 访问的只读入口点.
        'additional_lookup': {
            'url': 'regex("[\w]+")',
            'field': 'username',
        },

        # 我们也禁用终结点缓存，因为我们不希望客户端应用程序缓存账户数据.
        'cache_control': '',
        'cache_expires': 0,

        # 最后，让我们为这个终结点添加模式定义.
        'schema': schema,
    }

我们在 ``/accounts/<username>`` 定义一个额外的只读入口点。这不是真的必要，但是它
可以为轻松验证一个用户名是否已经被拿走带来便利，或者在事先不知道 ``ObjectId`` 
的情况下获取一个账户。当然，两片信息也都可以通过查询资源终结点 
(``/accounts?where={"username":"johndoe"}``) 找到，但是接着我们需要解析响应
载体，尽管通过一个对我们新终结点的 GET 请求，我们将获得裸账户数据，或者，
如果账户不存在，将得到一个 ``404 Not Found``。

一旦终结点已经配置好了，我们需要将它添加到 API 域:

::

    DOMAIN['accounts'] = accounts


2. 保护 ``/accounts/`` 终结点
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
2a. Hard-coding our way in
''''''''''''''''''''''''''
保护终结点可以通过只允许众所周知的 `superusers` 操作它来实现。我们定义在启用脚本中
的身份验证类，可以被硬编码以处理实际情况:

.. code-block:: python

    import bcrypt
    from eve import Eve
    from eve.auth import BasicAuth


    class BCryptAuth(BasicAuth):
        def check_auth(self, username, password, allowed_roles, resource, method):
            if resource == 'accounts':
                return username == 'superuser' and password == 'password'
            else:
                # 使用 Eve 自己的数据库驱动; 没有使用额外的连接/资源
                accounts = app.data.driver.db['accounts']
                account = accounts.find_one({'username': username})
                return account and \
                    bcrypt.hashpw(password, account['password']) == account['password']


    if __name__ == '__main__':
        app = Eve(auth=BCryptAuth)
        app.run()

因此，只有 ``superuser`` 账户被允许使用 ``accounts`` 终结点，而标准身份验证逻辑将应用
于所有其他终结点。我们的移动应用 (好吧) 将通过简单的 POST 请求终结点添加账户，当然，
通过 `Authorization` 头证实它自己为一个 `superuser`。这个脚本假定，保存的密码经过
`bcrypt` (将密码保存为普通文本 *从来都不是* 一个好主意) 加密。查看 :ref:`basic` 获取
一个可供替代的，更快但更不安全的 SHA1/MAC 示例。

2b. 用户角色访问控制
'''''''''''''''''''''''''''''
硬编码的用户名和密码可能会很好完成工作，但是它很难说是我们在这里可以使用的最好方式。如果
另一个 `superurser` 账户需要访问终结点怎么办? 每次一个有特权的用户加入序列就更新脚本
看起来并不合适 (确实不合适)。幸运的是，:ref:`roleaccess` 特性可以在这里帮助我们。通过
这个: 想法是，只有带 `superuser` 和 `admin` 角色的账户才被准许访问终结点，你可以看到
我们要去哪里。

让我们从更新我们的资源模式开始。

.. code-block:: python
   :emphasize-lines: 10-14

        schema =  {
            'username': {
                'type': 'string',
                'required': True,
                },
            'password': {
                'type': 'string',
                'required': True,
            },
            'roles': {
                'type': 'list',
                'allowed': ['user', 'superuser', 'admin'],
                'required': True,
            }
        },

我们只添加一个新的 ``roles`` 字段，它是一个需要的列表。从现在开始，在账户创建过程中，
一个或更多角色将必须被赋值。

现在，我们只需要限制 `superuser` 和 `admin` 账户对终结点的权限，因此，让我们对应更新
终结点定义。

.. code-block:: python
   :emphasize-lines: 16

    accounts = {
        # 标准账户入口点定义为 '/accounts/<ObjectId>'。我们再定义一个额外的可以
        # 在 '/accounts/<username>' 访问的只读入口点.
        'additional_lookup': {
            'url': 'regex("[\w]+")',
            'field': 'username',
        },

        # 我们也禁用终结点缓存，因为我们不希望客户端应用程序缓存账户数据.
        'cache_control': '',
        'cache_expires': 0,

        # 只允许超级用户和管理员.
        'allowed_roles': ['superuser', 'admin'],

        # 最后，让我们为这个终结点添加模式定义.
        'schema': schema,
    }

最后，轮到重写我们的身份验证类。

.. code-block:: python

    from eve import Eve
    from eve.auth import BasicAuth
    from werkzeug.security import check_password_hash


    class RolesAuth(BasicAuth):
        def check_auth(self, username, password, allowed_roles, resource, method):
            # 使用 Eve 自己的数据库驱动; 没有使用额外的连接/资源
            accounts = app.data.driver.db['accounts']
            lookup = {'username': username}
            if allowed_roles:
                # 只获取角色匹配 ``allowed_roles`` 的用户
                lookup['roles'] = {'$in': allowed_roles}
            account = accounts.find_one(lookup)
            return account and check_password_hash(account['password'], password)


    if __name__ == '__main__':
        app = Eve(auth=RolesAuth)
        app.run()

上面的片段做的是使用基于角色的访问控制保护所有 API 终结点。事实上，它是在 :ref:`roleaccess`
中看到的同一个片段。这项技术使我们可以保持代码原封不动，因为我们添加了更多 `superuser` 或
`admin` 账户 (并且我们很可能即将通过访问我们自己的 API 添加它们)。此外，如果需要的话，我们
只要更新配置文件可以轻易限制对更多终结点的访问，又一次不涉及身份验证类。

3. 保护其他 API 终结点
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
这是很快的，因为上面的 `hard-coding` 和 `role-based` 访问控制方式事实上已经在
保护所有 API 终结点了。向 ``Eve`` 对象传递一个身份验证类启用对整个 API 的身份验证: 
每次当一个终结点遇到一个请求时，类实例就会被调用。

当然，你仍然可以微调保护措施，例如，通过允许对特定终结点或者特定 HTTP 方法的公共访问。
查看 :ref:`auth` 获取更多详情。

4. 只允许访问账户资源
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
大多数时间，当你允许认证的用户存储数据时，你只希望他们访问他们自己的数据。这可以通过使用
:ref:`user-restricted` 特性顺利实现。启用时，每个保存的文档都关联到创建它的账户。这使
API 可以只为所有类型的请求: 读取, 编辑, 删除，当然还有创建，透明提供账户创建的文档。

要激活这个特性，我们只需要做两件事情:

1. 配置用于存储文档拥有者的字段名称;
2. 为每一个收到的 POST 请求设置文档拥有者。


由于我们希望对我们所有的 API 终结点启用这项特性，我们只要通过设置一个正确
的 ``AUTH_FIELD`` 只来更新我们的 ``settings.py`` 文件就行了:

::

    # 用于保存每个文档所有者的字段名称
    AUTH_FIELD = 'user_id'


然后，我们希望更新我们的身份验证来正确更新字段的值:

.. code-block:: python
   :emphasize-lines: 15-17


    from eve import Eve
    from eve.auth import BasicAuth
    from werkzeug.security import check_password_hash


    class RolesAuth(BasicAuth):
        def check_auth(self, username, password, allowed_roles, resource, method):
            # 使用 Eve 自己的数据库驱动; 没有使用额外的连接/资源
            accounts = app.data.driver.db['accounts']
            lookup = {'username': username}
            if allowed_roles:
                # 只获取角色匹配 ``allowed_roles`` 的用户
                lookup['roles'] = {'$in': allowed_roles}
            account = accounts.find_one(lookup)
            # 设置 'AUTH_FIELD' 值为账户的 ObjectId (你可能希望使用 ID_FIELD，而不是 _Id)
            self.set_request_auth_value(account['_id'])
            return account and check_password_hash(account['password'], password)


    if __name__ == '__main__':
        app = Eve(auth=RolesAuth)
        app.run()

这就是所有我们需要做的。限制，当一个客户端通过 GET 请求访问 ``/invoices`` 终结点时，
它只会被提供它自己的账户创建的 invoices。DELETE 和 PATCH 也会发生一样的事，使一个认证
的用户不可能偶然获取，编辑或删除其他人的数据。

使用令牌身份验证的账户
----------------------------------
就像 :ref:`token` 看到的，令牌身份验证只是一个专用版本的基本身份验证。事实上，它也被当作
一个标准基本身份验证请求来执行，*username* 字段值用作令牌，而密码字段未提供 (就算包括了
也会被忽略)。

因此，通过令牌身份验证处理账户与我们在 :ref:`accounts_basic` 中看到的非常类似，但是
有一个小警告: 令牌需要伴随账户生成和保存，并最终被返回给客户端。

根据这个，让我们回顾我们更新的任务列表:

1. 制作一个所有账户管理活动都可用的终结点 (``/accounts``)。
2. 保护终结点，这样它只能被我们控制的客户端 (令牌) 访问。
3. 账户创建时，生成和存储它的令牌。
4. 可选，通过响应返回新的令牌。
5. 确保所有其他 API 终结点只能被证实的令牌访问。
6. 使证实的用户只能访问他们自己创建的资源。

1. ``/accounts/`` 终结点
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
这跟我们在 :ref:`accounts_basic` 中做的没有什么不同。我们只需要添加 `token` 字段到
我们的模式中:

.. code-block:: python
   :emphasize-lines: 16-19

        schema =  {
            'username': {
                'type': 'string',
                'required': True,
                'unique': True,
                },
            'password': {
                'type': 'string',
                'required': True,
            },
            'roles': {
                'type': 'list',
                'allowed': ['user', 'superuser', 'admin'],
                'required': True,
            },
            'token': {
                'type': 'string',
                'required': True,
            }
        }

2. 保护 ``/accounts/`` 终结点
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
我们在上一步为 `accounts` 模式定义了 `roles` 字段。我们也需要定义终结点，确保
我们发送了允许的用户角色。

.. code-block:: python
   :emphasize-lines: 16

    accounts = {
        # 标准账户入口点定义为 '/accounts/<ObjectId>'。我们再定义一个额外的
        # 可以在 '/accounts/<username>' 访问的只读入口点.
        'additional_lookup': {
            'url': 'regex("[\w]+")',
            'field': 'username',
        },

        # 我们也禁用终结点缓存，因为我们不希望客户端应用程序缓存账户数据.
        'cache_control': '',
        'cache_expires': 0,

        # 只允许超级用户和管理员.
        'allowed_roles': ['superuser', 'admin'],

        # 最后，让我们为这个终结点添加模式定义.
        'schema': schema,
    }

最终，这是我们的启动脚本，当然了，这一次使用了一个 ``TokenAuth`` 子类:

.. code-block:: python

    from eve import Eve
    from eve.auth import TokenAuth


    class RolesAuth(TokenAuth):
        def check_auth(self, token,  allowed_roles, resource, method):
            # 使用 Eve 自己的数据库驱动; 没有使用额外的连接/资源
            accounts = app.data.driver.db['accounts']
            lookup = {'token': token}
            if allowed_roles:
                # 只获取角色匹配 ``allowed_roles`` 的用户
                lookup['roles'] = {'$in': allowed_roles}
            account = accounts.find_one(lookup)
            return account


    if __name__ == '__main__':
        app = Eve(auth=RolesAuth)
        app.run()

3. 在账户创建中构建自定义令牌
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
上面的代码有一个问题: 它不会证明任何人，因为我们还没有生成任何令牌。因此，客户端取不回
它们的认证令牌，因此，它们并不真正知道如何证明。让我们通过使用令人惊叹的 :ref:`eventhooks`
特性来修复它。我们将通过注册一个回调函数来更新我们的启动脚本，这个函数在一个新账户即将
被保存到数据库时调用。

.. code-block:: python
   :emphasize-lines: 3-4,19-24,29

    from eve import Eve
    from eve.auth import TokenAuth
    import random
    import string


    class RolesAuth(TokenAuth):
        def check_auth(self, token,  allowed_roles, resource, method):
            # 使用 Eve 自己的数据库驱动; 没有使用额外的连接/资源
            accounts = app.data.driver.db['accounts']
            lookup = {'token': token}
            if allowed_roles:
                # 只获取角色匹配 ``allowed_roles`` 的用户
                lookup['roles'] = {'$in': allowed_roles}
            account = accounts.find_one(lookup)
            return account


    def add_token(documents):
        # 不要在生产环境使用:
        # 你应该至少确保令牌使唯一的.
        for document in documents:
            document["token"] = (''.join(random.choice(string.ascii_uppercase)
                                         for x in range(10)))


    if __name__ == '__main__':
        app = Eve(auth=RolesAuth)
        app.on_insert_accounts += add_token
        app.run()

就像你可以看到的，我们在通过我们的 ``add_token`` 函数订阅 `accounts` 终结点的
``on_insert`` 事件。这个回调将收到作为参数的 `documents`，它是一个被数据库插入过程
接受的验证过的文档列表。我们只要为每个文档添加 (或者在不太可能，即请求已经包含的情况下，
替换) 一个令牌，然后就完成了! 要获取更多关于回调函数的信息，参考 `Event Hooks`_。

4. 通过响应返回令牌
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
也许，你可能希望通过响应返回令牌。说实话，这不是一个很好的主意。你通常希望通过，例如
一封电子邮件，发送权限信息 out-of-band。但是，假定我们在使用 SSL，有些场景发送认证
令牌能说得通，像客户端是移动应用，而我们希望用户马上使用服务。

正常情况下，只包括自动处理的字段 (``ID_FIELD``, ``LAST_UPDATED``, ``DATE_CREATED``, 
``ETAG``) 在 POST 响应载体中。幸运的是，有一个设置允许我们注入额外的字段到响应中，
那就是 ``EXTRA_RESPONSE_FIELDS``，以及它的终结点级别的对应词，
``extra_response_fields``。所有我们需要做的是对应更新我们的终结点定义:

.. code-block:: python
   :emphasize-lines: 19

    accounts = {
        # 标准账户入口点定义为 '/accounts/<ObjectId>'。我们再定义一个额外的
        # 可以在 '/accounts/<username>' 访问的只读入口点.
        'additional_lookup': {
            'url': 'regex("[\w]+")',
            'field': 'username',
        },

        # 我们也禁用终结点缓存，因为我们不希望客户端应用程序缓存账户数据.
        'cache_control': '',
        'cache_expires': 0,

        # 只允许超级用户和管理员.
        'allowed_roles': ['superuser', 'admin'],

        # 允许 'token' 通过 POST 响应返回
        'extra_response_fields': ['token'],

        # 最后，让我们为这个终结点添加模式定义.
        'schema': schema,
    }

从现在开始，针对 ``/accounts`` 终结点的 POST 请求的响应，将包括新生成的认证令牌，
使客户端可以立即使用其他 API 终结点。

5. 保护其他 API 终结点
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
像我们以前已经看到的，传递一个身份验证类到 ``Eve`` 对象启用对所有 API 终结点的
身份验证。再说一次，你仍然可以通过允许对特定终结点或特定 HTTP 方法的公共访问来微调
保护措施。查看 :ref:`auth` 获取更多详情。

6. 只允许访问账户资源
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
这个通过 :ref:`user-restricted` 特性实现，像 :ref:`accounts_basic` 看到的那样。
你可能希望将用户令牌存储为你的 ``AUTH_FIELD`` 值，但是，如果你希望用户令牌可以被
轻松废止，那么你的最佳选项是为此使用账户唯一 id。

基本 vs 令牌: 最后的斟酌
------------------------------------
尽管，要在服务端建立起来有点难搞，令牌身份验证提供显著的优势。首先，你不需要将密码
保存在客户端，每次请求都通过线路发送它。如果你在发送你的令牌 out-of-band，而你使用
SSL/TLS，, 那是相对大的额外保障。

.. _SSL/TLS: http://en.wikipedia.org/wiki/Transport_Layer_Security
.. _`Event Hooks`: http://python-eve.org/features.html#event-hooks
