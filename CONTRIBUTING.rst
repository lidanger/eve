如何参与
=================

欢迎参与! 还不熟悉基础代码? 没问题!有很多参与到开源项目的方式: 报告 bugs，帮助完善文档，传播信息，当然，还有添加新特性和补丁。

支持问题
-----------------

请，不要使用问题跟踪器来做这件事。使用以下资源中的一项来提出关于你自己代码的问题:

* 在 `Stack Overflow`_ 上提问。首先使用: ``site:stackoverflow.com eve {search term, exception message, etc.}`` 在 Google 搜索。
* `邮件列表`_ 是为了成为一项同时为开发者/贡献者和 API 维护者提供寻求帮助或请求反馈的低流量资源。
* FreeNode 上的 IRC 频道 ``#python-eve``。

.. _Stack Overflow: https://stackoverflow.com/questions/tagged/eve?sort=linked
.. _`邮件列表`: https://groups.google.com/forum/#!forum/python-eve

报告问题
----------------

- 描述你所期望发生的。
- 如果可能的话，包含一个 `最小的，完整的，可验证的例子`_ 来帮助我们识别问题。这也会帮助检查问题是否跟你自己的代码无关。
- 描述实际上发生了什么。如果出现异常的话，包含全过程跟踪回溯。
- 列出你的 Python 和 Eve 版本。如果可能的话，在代码库中检查这个问题是否已经被修复了。

.. _最小的，完整的，可验证的例子: https://stackoverflow.com/help/mcve

提交补丁
------------------

- 如果你的补丁目的是解决一个 bug，请包含测试，并解释清楚在什么条件下 bug 会出现。确保如果没有你的补丁，测试会失败。
- 启用和安装 pre-commit_ 来确保遵守了风格指南和代码检查。CI 会拒绝一个没有遵守行为准则的修改。

.. _pre-commit: https://pre-commit.com/

首次安装
~~~~~~~~~~~~~~~~

- 下载安装 `最新版本的 git`_。
- 使用你的 `用户名`_ and `电子邮件`_ 配置 git ::

        git config --global user.name 'your name'
        git config --global user.email 'your email'

- 确保你有一个 `GitHub 账户`_。
- 通过点击 `Fork`_ 按钮 Fork Eve 到你的 GitHub 账户。
- `Clone`_ 你的 GitHub fork 到本地::

        git clone https://github.com/{username}/eve
        cd eve

- 添加主代码库为 remote 以便稍后更新::

        git remote add pyeve https://github.com/pyeve/eve
        git fetch pyeve

- 创建一个 virtualenv::

        python3 -m venv env
        . env/bin/activate
        # or "env\Scripts\activate" on Windows

- 在可编辑模式安装 Eve 和开发依赖项::

        pip install -e ".[dev]"

- 安装 pre-commit_ 然后激活它的 hooks。pre-commit 是一个用于管理和维护多语言 pre-commit hooks 的框架。Eve 使用 pre-commit 来确保相同的编码风格和编码格式::

    $ pip install --user pre-commit
    $ pre-commit install

  以后，只要你一提交 pre-commit 就会运行。


.. _GitHub 账户: https://github.com/join
.. _最新版本的 git: https://git-scm.com/downloads
.. _用户名: https://help.github.com/articles/setting-your-username-in-git/
.. _电子邮件: https://help.github.com/articles/setting-your-email-in-git/
.. _Fork: https://github.com/pallets/flask/fork
.. _Clone: https://help.github.com/articles/fork-a-repo/#step-2-create-a-local-clone-of-your-fork

开始写代码
~~~~~~~~~~~~

- 创建一个分支来标识你想解决的问题 (例如 ``fix_for_#1280``)
- 使用你最喜欢的编辑器，做出修改，`committing as you go`_.
- 遵行 `PEP8`_.
- 包含覆盖你做出的任何代码变化的测试。确保如果没有你的补丁，测试会失败。`运行测试。<contributing-testsuite_>`_。
- 推送你的提交到 GitHub 并 `生成一个 pull 请求`_.
- 庆祝 🎉

.. _committing as you go: http://dont-be-afraid-to-commit.readthedocs.io/en/latest/git/commandlinegit.html#commit-your-changes
.. _PEP8: https://pep8.org/
.. _create a pull request: https://help.github.com/articles/creating-a-pull-request/

.. _contributing-testsuite:

运行测试
~~~~~~~~~~~~~~~~~

你应该在你的系统中同时安装 Python 2.7 和 3.6。现在，运行测试就像发出这个命令一样简单::

    $ tox -e linting,py27,py36

这个命令将通过的 "tox" 工具运行 Python 2.7 和 3.6 测试，也执行 "lint" 代码风格检查。

你可以为 ``tox`` 传递不同的选项。例如，要在 Python 2.7 中运行测试并传递选项到 pytest (例如，失败时进入 pdb) 来 pytest 你可以做的::

    $ tox -e py27 -- --pdb

或者只是在 Python 3.6 中运行特定测试模块::

    $ tox -e py36 -- -k TestGet

当你提交你的 pull 请求时，Travis-CI 会运行全套。全套测试需要运行很长时间，因为它测试多个 Python 和依赖项的组合。你需要安装 Python 2.7, 3.4, 3.5, 3.6 和 PyPy 来支持所有的环境。然后运行::

    tox

请注意，为了运行测试，你需要有一个 MongoDB 实例运行在本地。也要注意，为了执行 :ref:`ratelimiting` 测试，你需要一个运行中的 Redis_ 服务器。如果这两项条件中的任一个没有满足，会默默跳过限速测试。

构建文档
~~~~~~~~~~~~~~~~~
使用 Sphinx 构建 ``docs`` 文件夹下的文档::

    cd docs
    make html

在你的浏览器中打开 ``_build/html/index.html`` 查看文档。

阅读更多关于 `Sphinx <http://www.sphinx-doc.org>`_ 的信息。

生成目标
~~~~~~~~~~~~
Eve 通过各种快捷方式提供一个 ``Makefile``。它们将确认所有依赖都已经安装。

- ``make test`` 使用 ``pytest`` 运行基本测试套件
- ``make test-all`` 使用 ``tox`` 运行全部测试套件 
- ``make docs`` 构建 HTML 文档
- ``make check`` 对包进行一些检查
- ``make install-dev`` 在可编辑模式安装 Eve 和所有开发依赖项。

第一次当贡献者?
-----------------------
没问题。我们都已经在那里了。看看下一章。

不知道从哪里开始?
--------------------------
通常在基础代码周围分散着几个 TODO 注释，或许检查它们，看看是否有些想法或是否能帮助改善它们。也可以看看那些能引起你的兴趣的 `open issues`_。再者，文档怎么样? 我英语很糟糕，因此如果你的英语流利 (或者通知任何排版问题或错误)，为什么不帮助改善它呢? 在任何情况下，除了 GitHub help_ 页面，你可能想试试这个出色的 `Effective Guide to Pull Requests`_

.. _`the repository`: http://github.com/pyeve/eve
.. _AUTHORS: https://github.com/pyeve/eve/blob/master/AUTHORS
.. _`open issues`: https://github.com/pyeve/eve/issues
.. _`new issue`: https://github.com/pyeve/eve/issues/new
.. _GitHub: https://github.com/
.. _`proper format`: http://tbaggery.com/2008/04/19/a-note-about-git-commit-messages.html
.. _flake8: http://flake8.readthedocs.org/en/latest/
.. _tox: http://tox.readthedocs.org/en/latest/
.. _help: https://help.github.com/
.. _`Effective Guide to Pull Requests`: http://codeinthehole.com/writing/pull-requests-and-other-good-practices-for-teams-using-github/
.. _`fork and edit`: https://github.com/blog/844-forking-with-the-edit-button
.. _`Pull Request`: https://help.github.com/articles/creating-a-pull-request
.. _`running the tests`: http://python-eve.org/testing#running-the-tests
.. _Redis: https://redis.io
