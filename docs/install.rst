.. _install:

安装
============
这部分文档覆盖 Eve 的安装。使用任何软件包的第一步都是让它恰当地安装。

使用 `pip <http://www.pip-installer.org/>`_，安装 Eve 很简单:

.. code-block:: console

    $ pip install eve

开发版本
--------------------
Eve 在 GitHub 的开发很活跃，代码 `在 <https://github.com/pyeve/eve> 总是可用的`_。如果你想使用开发版本的 Eve，有两种方式：你可以让 `pip` 下载开发版本，或者你可以让它执行一个 git checkout。
不论哪种方式，都推荐使用 virtualenv。

在一个新的 virtualenv 中获取 git checkout 并在开发模式下运行。

.. code-block:: console

    $ git clone http://github.com/pyeve/eve.git
    在 ~/dev/eve/.git/ 初始化空 Git 库

    $ cd eve
    $ virtualenv venv
    新的 python 可执行文件在 venv/bin/python

    $ . venv/bin/activate
    $ python setup.py install
    ...
    完成对 Eve 依赖项的处理

这样会下载依赖项，并在 virtualenv 中激活 git head 为当前版本。接着你需要做的就是运行 ``git pull origin`` 来更新到最新版本。

要想只获取开发版本而不用 git，这样做:

.. code-block:: console

    $ mkdir eve
    $ cd eve
    $ virtualenv venv
    $ . venv/bin/activate
    新的 python 可执行文件在 venv/bin/python

    $ pip install git+git://github.com/pyeve/eve.git
    ...
    清理...

这样就完成了!
