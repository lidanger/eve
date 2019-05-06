RESTful 账户管理
==========================
.. 提示:: 请注意

    这篇教程假定你已经阅读了 :ref:`quickstart` 和 :ref:`auth` 指南。

Except for the relatively rare occurence of open (and generally read-only) public
APIs, most services are only accessible to authenticated users.  A common
pattern is that users create their account on a website or with a mobile
application.  Once they have an account, they are allowed to consume one or more
APIs. This is the model followed by most social networks and service providers
(Twitter, Facebook, Netflix, etc.) So how do you, the service provider, manage
to create, edit and delete accounts while using the same API that is being
consumed by the accounts themselves?

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

1. Make an endpoint available for all account management activities
   (``/accounts``).
2. Secure the endpoint, so that it is only accessible to clients
   that we control: our own website, mobile apps with account
   management capabilities, etc.
3. Make sure that all other API endpoints are only accessible to authenticated
   accounts (created by means of the above mentioned endpoint).
4. Allow authenticated users to only access resources created by themselves.

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
        # the standard account entry point is defined as
        # '/accounts/<ObjectId>'. We define  an additional read-only entry
        # point accessible at '/accounts/<username>'.
        'additional_lookup': {
            'url': 'regex("[\w]+")',
            'field': 'username',
        },

        # We also disable endpoint caching as we don't want client apps to
        # cache account data.
        'cache_control': '',
        'cache_expires': 0,

        # Finally, let's add the schema definition for this endpoint.
        'schema': schema,
    }

We defined an additional read-only entry point at ``/accounts/<username>``.
This isn't really a necessity, but it can come in handy to easily verify if
a username has been taken already, or to retrieve an account without knowing
its ``ObjectId`` beforehand. Of course, both pieces of information can also be
found by querying the resource endpoint (``/accounts?where={"username":
"johndoe"}``), but then we would need to parse the response payload, whereas by
hitting our new endpoint with a GET request we will obtain the bare account
data, or a ``404 Not Found`` if the account does not exist.

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
                # use Eve's own db driver; no additional connections/resources are used
                accounts = app.data.driver.db['accounts']
                account = accounts.find_one({'username': username})
                return account and \
                    bcrypt.hashpw(password, account['password']) == account['password']


    if __name__ == '__main__':
        app = Eve(auth=BCryptAuth)
        app.run()

Thus, only the ``superuser`` account will be allowed to consume the
``accounts`` endpoint, while standard authentication logic will apply to all
other endpoints. Our mobile app (say) will add accounts by hitting the endpoint
with simple POST requests, of course authenticating itself as a `superuser` by
means of the `Authorization` header. The script assumes that stored passwords
are encrypted with `bcrypt` (storing passwords as plain text is *never* a good
idea). See :ref:`basic` for an alternative, faster but less secure SHA1/MAC
example.

2b. 用户角色访问控制
'''''''''''''''''''''''''''''
Hard-coding usernames and passwords might very well do the job, but it is
hardly the best approach that we can take here. What if another `superurser`
account needs access to the endpoint? Updating the script each time
a privileged user joins the ranks does not seem appropriate (it isn't).
Fortunately, the :ref:`roleaccess` feature can help us here. You see where we
are going with this: the idea is that only accounts with `superuser` and
`admin` roles will be granted access to the endpoint.

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
        # the standard account entry point is defined as
        # '/accounts/<ObjectId>'. We define  an additional read-only entry
        # point accessible at '/accounts/<username>'.
        'additional_lookup': {
            'url': 'regex("[\w]+")',
            'field': 'username',
        },

        # We also disable endpoint caching as we don't want client apps to
        # cache account data.
        'cache_control': '',
        'cache_expires': 0,

        # Only allow superusers and admins.
        'allowed_roles': ['superuser', 'admin'],

        # Finally, let's add the schema definition for this endpoint.
        'schema': schema,
    }

最后，轮到重写我们的身份验证类。

.. code-block:: python

    from eve import Eve
    from eve.auth import BasicAuth
    from werkzeug.security import check_password_hash


    class RolesAuth(BasicAuth):
        def check_auth(self, username, password, allowed_roles, resource, method):
            # use Eve's own db driver; no additional connections/resources are used
            accounts = app.data.driver.db['accounts']
            lookup = {'username': username}
            if allowed_roles:
                # only retrieve a user if his roles match ``allowed_roles``
                lookup['roles'] = {'$in': allowed_roles}
            account = accounts.find_one(lookup)
            return account and check_password_hash(account['password'], password)


    if __name__ == '__main__':
        app = Eve(auth=RolesAuth)
        app.run()

What the above snippet does is secure all API endpoints with role-base access
control. It is, in fact, the same snippet seen in :ref:`roleaccess`. This
technique allows us to keep the code untouched as we add more `superuser` or
`admin` accounts (and we'll probably be adding them by accessing our very own
API). Also, should the need arise, we could easily restrict access to more
endpoints just by updating the settings file, again without touching the
authentication class.

3. 保护其他 API 终结点
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
This will be quick, as both the `hard-coding` and the `role-based` access
control approaches above effectively secure all API endpoints already.  Passing
an authentication class to the ``Eve`` object enables authentication for the
whole API: every time an endpoint is hit with a request, the class instance is
invoked.

Of course, you can still fine-tune security, for example by allowing public
access to certain endpoints, or to certain HTTP methods. See :ref:`auth` for
more details.

4. 只允许访问账户资源
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Most of the time when you allow Authenticated users to store data, you only
want them to access their own data. This can be convenientely achieved by
using the :ref:`user-restricted` feature. When enabled, each stored document is
associated with the account that created it. This allows the API to transparently
serve only account-created documents on all kind of requests: read, edit, delete
and of course create.

要激活这个特性，我们只需要做两件事情:

1. Configure the name of the field that will be used to store the owner of the
   document;
2. Set the document owner on each incoming POST request.


由于我们希望对我们所有的 API 终结点启用这项特性，我们只要通过设置一个正确
的 ``AUTH_FIELD`` 只来更新我们的 ``settings.py`` 文件就行了:

::

    # Name of the field used to store the owner of each document
    AUTH_FIELD = 'user_id'


然后，我们希望更新我们的身份验证来正确更新字段的值:

.. code-block:: python
   :emphasize-lines: 15-17


    from eve import Eve
    from eve.auth import BasicAuth
    from werkzeug.security import check_password_hash


    class RolesAuth(BasicAuth):
        def check_auth(self, username, password, allowed_roles, resource, method):
            # use Eve's own db driver; no additional connections/resources are used
            accounts = app.data.driver.db['accounts']
            lookup = {'username': username}
            if allowed_roles:
                # only retrieve a user if his roles match ``allowed_roles``
                lookup['roles'] = {'$in': allowed_roles}
            account = accounts.find_one(lookup)
            # set 'AUTH_FIELD' value to the account's ObjectId
            # (instead of _Id, you might want to use ID_FIELD)
            self.set_request_auth_value(account['_id'])
            return account and check_password_hash(account['password'], password)


    if __name__ == '__main__':
        app = Eve(auth=RolesAuth)
        app.run()

This is all we need to do. Now when a client hits say the ``/invoices``
endpoint with a GET request, it will only be served with invoices created by
its own account. The same will happen with DELETE and PATCH, making it
impossible for an authenticated user to accidentally retrieve, edit or delete
other people's data.

使用令牌身份验证的账户
----------------------------------
As seen in :ref:`token`, token authentication is just a specialized version of
Basic Authentication. It is actually executed as a standard Basic
Authentication request where the value of the *username* field is used for
the token, and the password field is not provided (if included, it is ignored).

Consequently, handling accounts with Token Authentication is very similar to
what we saw in :ref:`accounts_basic`, but there's one little caveat: tokens
need to be generated and stored along with the account, and eventually returned
to the client.

根据这个，让我们回顾我们更新的任务列表:

1. Make an endpoint available for all account management activities
   (``/accounts``).
2. Secure the endpoint so that it is only accessible to clients (tokens) that
   we control.
3. On account creation, generate and store its token.
4. Optionally, return the new token with the response.
5. Make sure that all other API endpoints are only accessible to authenticated
   tokens.
6. Allow authenticated users to only access resources created by themselves

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
        # the standard account entry point is defined as
        # '/accounts/<ObjectId>'. We define  an additional read-only entry
        # point accessible at '/accounts/<username>'.
        'additional_lookup': {
            'url': 'regex("[\w]+")',
            'field': 'username',
        },

        # We also disable endpoint caching as we don't want client apps to
        # cache account data.
        'cache_control': '',
        'cache_expires': 0,

        # Only allow superusers and admins.
        'allowed_roles': ['superuser', 'admin'],

        # Finally, let's add the schema definition for this endpoint.
        'schema': schema,
    }

最终，这是我们的启动脚本，当然了，这一次使用了一个 ``TokenAuth`` 子类:

.. code-block:: python

    from eve import Eve
    from eve.auth import TokenAuth


    class RolesAuth(TokenAuth):
        def check_auth(self, token,  allowed_roles, resource, method):
            # use Eve's own db driver; no additional connections/resources are used
            accounts = app.data.driver.db['accounts']
            lookup = {'token': token}
            if allowed_roles:
                # only retrieve a user if his roles match ``allowed_roles``
                lookup['roles'] = {'$in': allowed_roles}
            account = accounts.find_one(lookup)
            return account


    if __name__ == '__main__':
        app = Eve(auth=RolesAuth)
        app.run()

3. 在账户创建中构建自定义令牌
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
The code above has a problem: it won't authenticate anybody, as we aren't
generating any token yet. Consequently, clients aren't getting their auth tokens
back so they don't really know how to authenticate. Let's fix that by using the
awesome :ref:`eventhooks` feature.  We'll update our launch script by
registering a callback function that will be called when a new account is about
to be stored to the database.

.. code-block:: python
   :emphasize-lines: 3-4,19-24,29

    from eve import Eve
    from eve.auth import TokenAuth
    import random
    import string


    class RolesAuth(TokenAuth):
        def check_auth(self, token,  allowed_roles, resource, method):
            # use Eve's own db driver; no additional connections/resources are used
            accounts = app.data.driver.db['accounts']
            lookup = {'token': token}
            if allowed_roles:
                # only retrieve a user if his roles match ``allowed_roles``
                lookup['roles'] = {'$in': allowed_roles}
            account = accounts.find_one(lookup)
            return account


    def add_token(documents):
        # Don't use this in production:
        # You should at least make sure that the token is unique.
        for document in documents:
            document["token"] = (''.join(random.choice(string.ascii_uppercase)
                                         for x in range(10)))


    if __name__ == '__main__':
        app = Eve(auth=RolesAuth)
        app.on_insert_accounts += add_token
        app.run()

As you can see, we are subscribing to the ``on_insert`` event of the `accounts`
endpoint with our ``add_token`` function. This callback will receive
`documents` as an argument, which is a list of validated documents accepted for
database insertion. We simply add (or replace in the unlikely case that the
request contained it already) a token to every document, and we're done! For
more information on callbacks, see `Event Hooks`_.

4. 通过响应返回令牌
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Optionally, you might want to return the tokens with the response. Truth be
told, this isn't a very good idea. You generally want to send access
information out-of-band, with an email for example. However we're assuming that
we are on SSL, and there are cases where sending the auth token just makes
sense, like when the client is a mobile application and we want the user to use
the service right away.

Normally, only automatically handled fields (``ID_FIELD``, ``LAST_UPDATED``,
``DATE_CREATED``, ``ETAG``) are included with POST response payloads.
Fortunately, there's a setting which allows us to inject additional fields in
responses, and that is ``EXTRA_RESPONSE_FIELDS``, with its endpoint-level
equivalent, ``extra_response_fields``. All we need to do is update our endpoint
definition accordingly:

.. code-block:: python
   :emphasize-lines: 19

    accounts = {
        # the standard account entry point is defined as
        # '/accounts/<ObjectId>'. We define  an additional read-only entry
        # point accessible at '/accounts/<username>'.
        'additional_lookup': {
            'url': 'regex("[\w]+")',
            'field': 'username',
        },

        # We also disable endpoint caching as we don't want client apps to
        # cache account data.
        'cache_control': '',
        'cache_expires': 0,

        # Only allow superusers and admins.
        'allowed_roles': ['superuser', 'admin'],

        # Allow 'token' to be returned with POST responses
        'extra_response_fields': ['token'],

        # Finally, let's add the schema definition for this endpoint.
        'schema': schema,
    }

From now on responses to POST requests aimed at the ``/accounts`` endpoint
will include the newly generated auth token, allowing the client to consume
other API endpoints right away.

5. 保护其他 API 终结点
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
As we've seen before, passing an authentication class to the ``Eve`` object
enables authentication for all API endpoints. Again, you can still fine-tune
security by allowing public access to certain endpoints or to certain HTTP
methods. See :ref:`auth` for more details.

6. 只允许访问账户资源
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
This is achieved with the :ref:`user-restricted` feature, as seen in
:ref:`accounts_basic`. You might want to store the user token as your
``AUTH_FIELD`` value, but if you want user tokens to be easily revocable, then
your best option is to use the account unique id for this.

基本 vs 令牌: 最后的斟酌
------------------------------------
Despite being a little more tricky to set up on the server side, Token
Authentication offers significant advantages. First, you don't have passwords
stored on the client and  being sent over the wire with every request. If
you're sending your tokens out-of-band, and you're on SSL/TLS, that's quite
a lot of additional security.

.. _SSL/TLS: http://en.wikipedia.org/wiki/Transport_Layer_Security
.. _`Event Hooks`: http://python-eve.org/features.html#event-hooks
