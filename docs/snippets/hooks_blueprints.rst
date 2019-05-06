使用来自你的 Blueprint 的 Eve 事件钩子
=========================================
by Pau Freixes

使用 Flask Blueprints_ 帮助我们建立不适合作为典型的 Eve 资源的新终结点，以扩展我们
的扩展我们的 Eve 应用程序。将这些终结点脱离 Eve 的范围允许我们编写特有的代码以处理
特有的状况。

在 Blueprint 的上下文中， 我们可以预料到 Eve 特性不可用，但那常常不是实情。
我们可以继续使用大量特性，诸如，:ref:`eve event-hooks`_。

下面的片段显示 ``users`` 模块如何包含一个 blueprint，它执行一些自定义动作，然后
使用 ``users_deleted`` 信号通知和唤醒所有注册到 Eve 应用程序的回调函数。

.. code-block:: python

    from flask import Blueprint, current_app as app

    blueprint = Blueprint('prefix_uri', __name__)

    @blueprint.route('/users/<username>', methods=['DELETE'])
    def del_user(username):
        # some specific code goes here
        # ...

        # call Eve-hooks consumers for this  event
        getattr(app, "users_deleted")(username)

下面的片段显示，blueprint 是如何被绑定到我们的 Eve 主应用程序，以及特有的
``set_username_as_none`` 函数是如何被注册以确保在每次一个用户被删除时被调用，
使用 Eve 事件来恰当地更新 MongoDB 集合。

.. code-block:: python

    from eve import Eve
    from users import blueprint
    from flask import current_app, request

    def set_username_as_none(username):
        resource = request.endpoint.split('|')[0]
        return  current_app.data.driver.db[resource].update(
            {"user" : username},
            {"$set": {"user": None}},
            multi=True
        )

    app = Eve()
    # 将 blueprint 注册到 Eve 主应用程序
    app.register_blueprint(blueprint)
    # 绑定回调函数，这样在每个用户删除时它就会被调用
    app.users_deleted += set_username_as_none
    app.run()

.. _Blueprints: http://flask.pocoo.org/docs/blueprints/
.. _`eve event-hooks`: http://python-eve.org/features.html#event-hooks
