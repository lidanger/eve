.. _auth:

身份验证和授权
================================
安全保护介绍
------------------------
身份验证是系统借以安全鉴别它们的用户的机制。Eve 支持几种身份验证模式: 基本身份验证，
令牌身份验证，HMAC 身份验证。`OAuth2 集成`_ 很容易完成。

授权是系统用来确定一个特别（验证过的）的用户应该有什么级别的权限来访问系统控制的资源
的机制。在 Eve 中，你可以现在对所有或者只是其中一些 API 终结点的访问。你可以保护一些
HTTP 动词，而让其他的继续开放。例如，你可以允许公开的只读访问，而限制数据项创建和编辑
只能由授权的用户进行。你也可以通过检查方法参数允许用于特定请求的 ``GET`` 访问和
其他 ``POST`` 访问。同时也有对基于角色的权限控制的支持。

安全是定制化非常重要的其中一个区域。这就是为什么给你提供屈指可数的几个基本验证类的原因。
它们实现基本的身份验证机制，必须被子类化以实现授权逻辑。不管你选择了哪一个身份验证模式，
你需要在你的子类中做的仅有的事是重载 ``check_auth()`` 方法。

全局身份验证
---------------------
要为你的 API 启用身份验证，只需要在应用程序实例化过程中传递自定义的身份验证类。
在我们的例子中，我们将使用 ``BasicAuth`` 基类，它实现了 :ref:`basic` 模式:

.. code-block:: python

    from eve.auth import BasicAuth

    class MyBasicAuth(BasicAuth):
        def check_auth(self, username, password, allowed_roles, resource,
                       method):
            return username == 'admin' and password == 'secret'

    app = Eve(auth=MyBasicAuth)
    app.run()

你所有的 API 终结点现在都是受保护的，这意味着，为了使用 API，一个客户端需要提供
正确的证书:

.. code-block:: console

    $ curl -i http://example.com
    HTTP/1.1 401 UNAUTHORIZED
    Please provide proper credentials.

    $ curl -H "Authorization: Basic YWRtaW46c2VjcmV0" -i http://example.com
    HTTP/1.1 200 OK

默认情况下，通过所有 HTTP 动词（方法）对所有终结点的访问都是受限的，事实上锁定了整个 API。

但是如果你的授权逻辑更复杂，而你只希望保护一些终结点或根据使用的终结点的不同应用不同
的逻辑，怎么办? 你只需要在你的身份验证类中添加逻辑就可以脱离困境，或许是像这样的东西:

.. code-block:: python

    class MyBasicAuth(BasicAuth):
        def check_auth(self, username, password, allowed_roles, resource, method):
            if resource in ('zipcodes', 'countries'):
                # 'zipcodes' 和 'countries' 是公共的
                return True
            else:
                # 所有其他资源都是受保护的
                return username == 'admin' and password == 'secret'

如果需要，这个途径也允许将请求 ``方法`` 列入考虑范围，例如，允许每个人进行 ``GET`` 
请求，而对编辑 (``POST``, ``PUT``, ``PATCH``, ``DELETE``) 进行强制验证。

终结点级别身份验证
-----------------------------
上面看到的 *one class to bind them all* 途径很可能对大多数场景都不错，但一旦授权
逻辑变得更复杂，就很容易变成复杂的难以管理的代码，在处理安全问题时你真的不希望的东西。

如果我们可以使用轻松应用于选定的终结点的专用认证类会不会不错? 这样，上面看到的，传递给
Eve 构造器的全局级别的认证类，仍然对所有终结点起作用，除了那些需要不同授权逻辑的。
或者，事实上，我们甚至可以选择 *不* 提供全局认证类，让所有终结点成为公共的，除了那些
我们希望保护的。通过一个像这样的系统，我们甚至可以选择让一些终结点由，怎么说呢，基本身
份验证保护，而其它的由令牌，或 HMAC 身份验证保护!

好了，结果，事实上，通过在定义 API :ref:`domain <domain>` 时简单启用资源基本的
``身份验证``配置，这是可能的。

.. code-block:: python

    DOMAIN = {
        'people': {
            'authentication': MySuperCoolAuth,
            ...
            },
        'invoices': ...
        }

这就行了。`people` 终结点限制将使用 ``MySuperCoolAuth`` 类进行身份验证，而如果提供
了的话，``invoices`` 终结点将使用一般用途的认证类，否则，它只会被开放给公众。

有其他的特性和选项可以用于在你的认证类中减少复杂性，特别是 (但不止是) 当使用全局级别
认证系统时。让我们回顾一些它们。

全局终结点安全
------------------------
你可能想要一个公共只读的 API，只有授权的用户可以写入，编辑和删除。你可以通过使用
``PUBLIC_METHODS`` 和 ``PUBLIC_ITEM_METHODS`` :ref:`global settings <global>`
来实现。将下面的添加到你的 `settings.py`:

::

    PUBLIC_METHODS = ['GET']
    PUBLIC_ITEM_METHODS = ['GET']

运行你 API。POST, PATCH 和 DELETE 仍然受限，而在所有终结点 GET 都是公共可用的。
``PUBLIC_METHODS`` 涉及像 ``/people`` 这样的资源终结点，而 ``PUBLIC_ITEM_METHODS``
涉及像 ``/people/id`` 这样的单个数据项。

.. _endpointsec:

自定义终结点安全
------------------------
假设，你希望只有特定的资源允许公共读权限。你通过在资源级别声明公共方法来实现，当声明
API :ref:`domain <domain>` 时:

.. code-block:: python

    DOMAIN = {
        'people': {
            'public_methods': ['GET'],
            'public_item_methods': ['GET'],
            },
        }

请意识到，存在时，:ref:`resource settings <local>` 重载全局配置。你可以使用这个
作为你的优势。假设你希望对所有终结点授予读取权限，只有 ``/invoices`` 除外。你首先
打开所有终结点的读取权限:

::

    PUBLIC_METHODS = ['GET']
    PUBLIC_ITEM_METHODS = ['GET']

然后保护私有终结点:

::

    DOMAIN = {
        'invoices': {
            'public_methods': [],
            'public_item_methods': [],
            }
        }

事实上让 `invoices` 成为一个受限的资源。

.. _basic:

基本身份验证
--------------------
``eve.auth.BasicAuth`` 类允许实现级别身份验证 (RFC2617)。为了实现自定义的身份验证，它
应该被子类化。

使用 bcrypt 的基本身份验证
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
通过 bcrypt_ 编码密码是好主意。它以牺牲性能为代价，但那正是核心问题，因为慢速编码意味着
对穷举攻击的非常好的抵抗力。对于更快的 (也更不安全) 替代品，参考更下面的 SHA1/MAC 片段。

这段脚本假定用户账户存储在一个 `accounts` MongoDB 集合中，而且密码作为 bcrypt hash 值
存储的。所有 API 资源/方法都受到保护，除非它们明确定义为公共的。


.. 提示:: 请注意

    你需要安装 `py-bcrypt` 来让这个可以工作。

.. code-block:: python


    # -*- coding: utf-8 -*-

    """
        Auth-BCrypt
        ~~~~~~~~~~~

        通过基本身份验证 (RFC2617) 保护一个 Eve 支持的 API.

        你将需要安装 py-bcrypt: ``pip install py-bcrypt``

        这个由 Nicola Iarocci 提供的片段可以自由用于任何你喜欢的东西。将它视为公共领域.
    """

    import bcrypt
    from eve import Eve
    from eve.auth import BasicAuth
    from flask import current_app as app

    class BCryptAuth(BasicAuth):
        def check_auth(self, username, password, allowed_roles, resource, method):
            # 使用 Eve 自己的数据库驱动; 未使用额外的连接/资源
            accounts = app.data.driver.db['accounts']
            account = accounts.find_one({'username': username})
            return account and \
                bcrypt.hashpw(password, account['password']) == account['password']


    if __name__ == '__main__':
        app = Eve(auth=BCryptAuth)
        app.run()

使用 SHA1/HMAC 的基本身份验证
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
这段脚本假定，用户账户存储在一个 `accounts` MongoDB 集合，并且密码是作为 SHA1/HMAC hashe
值存储的。所有 API 资源/方法都会受到保护，除非它们明确定义为公共的。

.. code-block:: python

    # -*- coding: utf-8 -*-

    """
        Auth-SHA1/HMAC
        ~~~~~~~~~~~~~~

        通过基本身份验证 (RFC2617) 保护一个 Eve 支持的 API.

        由于我们正在使用 werkzeug，我们不需要任何额外的输入 (werkzeug 是 Flask/Eve 的
        必备条件之一).

        这个由 Nicola Iarocci 提供的片段可以自由用于任何你喜欢的东西。将它视为公共领域.
    """

    from eve import Eve
    from eve.auth import BasicAuth
    from werkzeug.security import check_password_hash
    from flask import current_app as app

    class Sha1Auth(BasicAuth):
        def check_auth(self, username, password, allowed_roles, resource, method):
            # 使用 Eve 自己的数据库驱动; 未使用额外的连接/资源
            accounts = app.data.driver.db['accounts']
            account = accounts.find_one({'username': username})
            return account and \
                check_password_hash(account['password'], password)


    if __name__ == '__main__':
        app = Eve(auth=Sha1Auth)
        app.run()

.. _token:

基于令牌的身份验证
--------------------------
基于令牌的身份验证可以被考虑为基本身份验证的一个专用版本。身份验证头标签将包含认证令牌作为
用户名，而没有密码。

这段脚本假定，用户账户存储在一个 `accounts` MongoDB 集合中。所有 API 资源/方法都会受到保护，
除非它们明确定义为公共的 (通过对一些配置做手脚，你可以开放一个或多个资源以及方法为公共权限，
参考文档)。

.. code-block:: python

    # -*- coding: utf-8 -*-

    """
        Auth-Token
        ~~~~~~~~~~

        通过基于令牌的身份验证保护一个 Eve 支持的 API.

        这个由 Nicola Iarocci 提供的片段可以自由用于任何你喜欢的东西。将它视为公共领域.
    """

    from eve import Eve
    from eve.auth import TokenAuth
    from flask import current_app as app

    class TokenAuth(TokenAuth):
        def check_auth(self, token, allowed_roles, resource, method):
            """由于这个示例的目的，实现尽可能简单。一个 '真正的' 令牌应该很可能包含一个
            用户名/密码混合的 hash，然后应该通过存储在数据库中的账户数据验证.
            """
            # 使用 Eve 自己的数据库驱动; 未使用额外的连接/资源
            accounts = app.data.driver.db['accounts']
            return accounts.find_one({'token': token})


    if __name__ == '__main__':
        app = Eve(auth=TokenAuth)
        app.run()

HMAC 身份验证
-------------------
``eve.auth.HMACAuth`` 类允许自定义的，类似于 Amazon S3 的，HMAC (Hash Message 
Authentication Code) 身份验证，基本上，这是一个在 `Authorization` 头附近构造的非常安全的
自定义授权模式。

HMAC 身份验证工作原理
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
服务器通过一些 out-of-band 技术(例如，服务向客户端发送一封包含用户 id 和私钥的电子邮件) 
为客户端提供一个用户 id 和一个私钥。客户端将使用提供的私钥对所有请求进行签名。

当客户端希望发送一个请求时，它构建完整的请求，然后，使用私钥，计算出一个完整消息体 (有时候
也一些消息头) 的 hash 值。

下一步，客户端将计算出的 hash 值和它的 userid 添加到身份验证头中的消息:

::

    Authorization: johndoe:uCMfSzkjue+HSDygYB5aEg==

并发送到服务。服务从消息头中获取 userid，并在它自己的数据库中搜索他个用户的私钥。接着，它
使用私钥计算整个消息体 (以及选择的头) 的 hash 值来生成它的 hash 值。如果客户端发送的 hash 
值与服务器计算的匹配，那么服务器就知道是被真正的客户端发送的，没有被任何方式篡改。

的确，唯一难办的部分是和用户分享一个密钥并保证安全。那就是为什么有些服务允许生成生存时间
有限的共享密钥，这样你可以将密钥交给第三方来临时代表自己。这也是为什么私钥通常通过 
out-of-band 通道 (常常是一个网页，或者像上面说的，一封邮件或传统的普通纸张) 提供的原因。

``eve.auth.HMACAuth`` 类也支持访问角色。

HMAC 示例
~~~~~~~~~~~~
下面的片段也可以在 Eve `repository`_ 的 `examples/security` 文件夹中找到。

.. code-block:: python

    from eve import Eve
    from eve.auth import HMACAuth
    from flask import current_app as app
    from hashlib import sha1
    import hmac


    class HMACAuth(HMACAuth):
        def check_auth(self, userid, hmac_hash, headers, data, allowed_roles,
                       resource, method):
            # 使用 Eve 自己的数据库驱动; 未使用额外的连接/资源
            accounts = app.data.driver.db['accounts']
            user = accounts.find_one({'userid': userid})
            if user:
                secret_key = user['secret_key']
            # 在这个实现中，我们只对请求数据 hash，忽略头.
            return user and \
                hmac.new(str(secret_key), str(data), sha1).hexdigest() == \
                    hmac_hash


    if __name__ == '__main__':
        app = Eve(auth=HMACAuth)
        app.run()

.. _roleaccess:

基于角色的访问控制
-------------------------
上面的代码片段故意忽略 ``allowed_roles`` 参数。你可以使用这个参数来限制已认证用户的权限，
这些用户当然，也已经被分配了一个专门的角色。

首先，你将使用新的 ``ALLOWED_ROLES`` 和 ``ALLOWED_ITEM_ROLES`` :ref:`global
settings <global>` (或者对应的 ``allowed_roles`` 和 ``allowed_item_roles``
:ref:`resource settings <local>`)。

::

    ALLOWED_ROLES = ['admin']

然后，你的子类将通过好好使用上述 ``allowed_roles`` 参数来实现授权逻辑。

下面的片段假定，用户账户存储在一个 `accounts` MongoDB 集合中，并且密码作为 SHA1/HMAC 
hash 值存储，并且用户角色存储在一个 'roles' 数组中。所有 API 资源/方法都受到包含，除非
它们明确声明为公共的。

.. code-block:: python

    # -*- coding: utf-8 -*-

    """
        Auth-SHA1/HMAC-Roles
        ~~~~~~~~~~~~~~~~~~~~

        通过基本身份验证 (RFC2617) 和 用户角色保护一个 Eve 支持的 API.

        由于我们正在使用 werkzeug，我们不需要任何额外的输入 (werkzeug 是 Flask/Eve 的
        必备条件之一).

        这个由 Nicola Iarocci 提供的片段可以自由用于任何你喜欢的东西。将它视为公共领域.
    """

    from eve import Eve
    from eve.auth import BasicAuth
    from werkzeug.security import check_password_hash
    from flask import current_app as app

    class RolesAuth(BasicAuth):
        def check_auth(self, username, password, allowed_roles, resource, method):
            # 使用 Eve 自己的数据库驱动; 未使用额外的连接/资源
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

.. _user-restricted:

用户限制的资源访问
-------------------------------
当这项特性启用时，每一个存储的文档都被关联到创建它的账户。这使 API 处理各种请求时：
读取，编辑，删除，当然还有创建，可以只透明提供账户创建的文档。要让这个正常工作，需要启用
用户身份验证。

在全局级别，这项特性通过设置 ``AUTH_FIELD`` 启用，而在本地 (在终结点级别) 则通过设置
``auth_field``。这些属性定义了用于存储创建文档的用户 id 的字段名称。因此，例如，通过
设置 ``AUTH_FIELD`` 为 ``user_id``，你事实上 (并且对用户来说很明显) 为每个保存的文档添加
了一个 ``user_id`` 字段。然后，这将被用于获取/编辑/删除用户保存的文档。

但是，你如何设置 ``auth_field`` 的值? 通过调用 ``set_request_auth_value()`` 类方法。
让我们修改我们上面的 BCrypt-authentication 例子:

.. code-block:: python
   :emphasize-lines: 25-28

    # -*- coding: utf-8 -*-

    """
        Auth-BCrypt
        ~~~~~~~~~~~

        通过基本身份验证 (RFC2617) 保护一个 Eve 支持的 API.

        你将需要安装 py-bcrypt: ``pip install py-bcrypt``

        这个由 Nicola Iarocci 提供的片段可以自由用于任何你喜欢的东西。将它视为公共领域.
    """

    import bcrypt
    from eve import Eve
    from eve.auth import BasicAuth


    class BCryptAuth(BasicAuth):
        def check_auth(self, username, password, allowed_roles, resource, method):
            # 使用 Eve 自己的数据库驱动; 未使用额外的连接/资源
            accounts = app.data.driver.db['accounts']
            account = accounts.find_one({'username': username})
            # 设置 'auth_field' 值为账户的 ObjectId (你可能希望使用 ID_FIELD 而不是 _id)
            if account and '_id' in account:
                self.set_request_auth_value(account['_id'])
            return account and \
                bcrypt.hashpw(password, account['password']) == account['password']


    if __name__ == '__main__':
        app = Eve(auth=BCryptAuth)
        app.run()

.. _authdrivendb:

身份验证驱动的数据库访问
---------------------------
自定义的身份验证类也可以设置处理活动请求时应该使用的数据库。

正常情况下，你或者为整个 API 使用单个数据库，或者通过设置 ``mongo_prefix`` 为需要的值 
(参考 :ref:`local`) 来配置每个终结点使用哪个数据库。

但是，你可以基于活动令牌，用户或客户端来选择目标数据库。如果你的使用场景包括用户专用的
数据库实例，这样很方便。所有你需要做的是在对请求认证时，调用 ``set_mongo_prefix()`` 方法。

一个微不足道的的实例就是:

.. code-block:: python

    from eve.auth import BasicAuth

    class MyBasicAuth(BasicAuth):
        def check_auth(self, username, password, allowed_roles, resource, method):
            if username == 'user1':
                self.set_mongo_prefix('MONGO1')
            elif username == 'user2':
                self.set_mongo_prefix('MONGO2')
            else:
                # 为所有其他用户提供来自默认数据库的数据.
                self.set_mongo_prefix(None)
            return username is not None and password == 'secret'

    app = Eve(auth=MyBasicAuth)
    app.run()

上面的类将为 ``user1`` 提供来自配置项前缀为 ``settings.py`` 中的 ``MONGO1`` 的数据库
的数据。``user2`` 和 ``MONGO2`` 也会发送同样的事，而所有其他用户由默认数据库提供服务。

由于通过 ``set_mongo_prefix()`` 设置的值对默认和终结点级别的 ``mongo_prefix`` 设置
有优先权，这里发生的是，两个用户将总是由它们预定的数据库来提供服务，不管终结点的最终配置
是什么。

OAuth2 集成
------------------
由于你全程控制授权过程，让 Eve 集成 OAuth2 很简单。阅读这个页面中描述的主题时请放轻松，
然后调头回到 `Eve-OAuth2`_ ，一个利用 `Flask-Sentinel`_ 演示如何使用 OAuth2 来保护
你的 API 的示例项目。

.. 提示:: 请主意

    这个页面中的片段也可以在 Eve `repository`_ 的 `examples/security` 文件夹中找到。

.. _`repository`: https://github.com/pyeve/eve
.. _bcrypt: http://en.wikipedia.org/wiki/Bcrypt
.. _`Eve-OAuth2`: https://github.com/pyeve/eve-oauth2
.. _`Flask-Sentinel`: https://github.com/pyeve/flask-sentinel
