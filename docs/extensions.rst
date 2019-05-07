Extensions
==========

欢迎来到 Eve 扩展登记处。在这里你可以找到 Eve 扩展包的一个列表。这个列表定期评审和更新。如果你为 Eve 编写了一个包，并希望它显示在这里，只要 `联系我`_ 并向我展示你的工具!

- Eve-Auth-JWT_
- Eve-Elastic_
- Eve-Healthcheck_
- Eve-Mocker_
- Eve-Mongoengine_
- Eve-Neo4j_
- Eve-OAuth2_ and Flask-Sentinel_
- Eve-SQLAlchemy_
- Eve-Swagger_
- Eve.NET_
- EveGenie_
- `REST Layer for Golang`_

Eve-Auth-JWT
------------
*by Olivier Poitrey*

Eve-Auth-JWT_ 是一个用于 Eve 的 OAuth 2 JWT 令牌验证模块。

Eve-Elastic
-----------
*by Petr Jašek*

Eve-Elastic_ 是一个针对 Eve REST 框架的 elasticsearch 数据层。
Features facets support and the generation of mapping for schema.

Eve-Healthcheck
---------------
*by LuisComS*

Eve-Healthcheck_ 是提供用于监控你的 Eve 应用程序的健康检查 urls 服务的项目。

Eve-Mocker
----------
*by Thomas Sileo*

`Eve-Mocker`_ 是一个用于 Eve 支持的 REST API 的 mocking 工具。它基于出色的 HTTPretty, 目标是当你信任一个 Eve API 时用于你的单元测试。Eve-Mocker 在 Eve 博客: `Mocking tool for Eve APIs`_ 已经成为特色。

Eve-Mongoengine
---------------
*by Stanislav Heller*

Eve-Mongoengine_ 是一个 Eve 扩展，它使 Mongoengine ORM 模型可以被用作 eve 模式。如果你在应用程序中使用 mongoengine，又十分急切地想使用 Eve，不需要用 Cerberus 格式 (DRY!) 再写一遍模式，你可以使用这个扩展，它获取你的 mongoengine 模型并在后台自动转换为 Cerberus 模式。

Eve-Neo4j
---------
*by Abraxas Biosystems*

Eve-Neo4j_ 是一个 Eve 扩展，目标是使用 Neo4j 作为后端让它的用户构建和部署高度定制化，全特性的 RESTful Web 服务。由 Eve, Py2neo, flask-neo4j 和优质的计划提供支持。

Eve-OAuth2
----------
*by Nicola Iarocci*

Eve-OAuth2_ 本质上不是一个扩展，更像是一个关于如何能把 Flask-Sentinel_ 和 OAuth2 集成到你的 API 终结点的示例。

Eve-SQLAlchemy
--------------
*by Andrew Mleczko et al.*

由 Eve, SQLAlchemy 和优质的计划提供支持，Eve-SQLALchemy_ 允许使用基于 SQL 的后端毫不费力的构建和部署高度定制化，全特性的 RESTful Web 服务。

Eve-Swagger
-----------
*by Nicola Iarocci*

Eve-Swagger_ 是一个用于 Eve 的 swagger.io 扩展。使用一个 Swagger-enabled API，你得到了可交互的文档，客户端 SDK 生成和可发现性。来自
Swagger 网站:

    Swagger 是你的 RESTfull API 的一个简单但强大的表现形式。拥有这个星球上最大的 API 驱动生态系统，成千上万的开发者正在几乎所有现代编程语言和开发环境中支持 Swagger 。使用一个 Swagger-enabled API，你得到了可交互的文档，客户端 SDK 生成和可发现性。

要获取更多信息，请参阅 `Meet Eve-Swagger`_ 文章。

Eve.NET
-------
*by Nicola Iarocci*

`Eve.NET`_ 是一个简单的用于 Eve 框架 支持的 Web 服务的 HTTP 和 REST 客户端。它同时使用 ``System.Net.HttpClient`` 和 ``Json.NET`` 来提供 .NET 平台最好的 Eve 体验。由 Eve 框架相同作者自己编写和维护，Eve.NET 作为一个编写库 (PCL) 分发，并天衣无缝得运行在 .NET4, Mono, Xamarin.iOS, Xamarin.Android, Windows Phone 8 和 Windows 8 上。我们内部使用 Eve.NET 来驱动我们的 iOS, Web 和 Windows 应用。

EveGenie
--------
*by Erin Corson and Matt Tucker*

EveGenie_ 是一个用于生成 Eve 模式的工具。它接受一个或更多资源的一个 json 文档，为你提供一个初始模式定义。

针对 Golang 的 REST 层
---------------------
如果你正在入坑 Golang，你也应该看看 `REST Layer`_。由 Olivier Poitrey 开发，一个长期 Eve 参与者和支持者。REST Layer 是一个深受出色的 Python Eve 启发的 REST API 框架。它让你自动生成一个无所不包的，自定义的，以及安全的 REST API，胜任任何后端存储，而不需要样板代码。现在你可以关注你的业务逻辑了。


.. _Eve-Healthcheck: https://github.com/ateliedocodigo/eve-healthcheck
.. _`Mocking tool for Eve APIs`: http://blog.python-eve.org/eve-mocker
.. _`Auto generate API docs`: http://blog.python-eve.org/eve-docs
.. _charlesflynn/eve-docs: https://github.com/charlesflynn/eve-docs
.. _eve-mocker: https://github.com/tsileo/eve-mocker
.. _`联系我`: mailto:eve@nicolaiarocci.com
.. _Eve-Mongoengine: https://github.com/hellerstanislav/eve-mongoengine
.. _Eve-Elastic: https://github.com/petrjasek/eve-elastic
.. _Eve.NET: https://github.com/pyeve/Eve.NET
.. _Eve-SQLAlchemy: https://github.com/RedTurtle/eve-sqlalchemy
.. _Eve-OAuth2: https://github.com/pyeve/eve-oauth2
.. _Flask-Sentinel: https://github.com/pyeve/flask-sentinel
.. _Eve-Auth-JWT: https://github.com/rs/eve-auth-jwt
.. _`REST Layer`: https://github.com/rs/rest-layer
.. _EveGenie: https://github.com/newmediadenver/evegenie
.. _Eve-Swagger: https://github.com/pyeve/eve-swagger
.. _`Meet Eve-Swagger`: http://nicolaiarocci.com/announcing-eve-swagger/
.. _Eve-Neo4j: https://github.com/Abraxas-Biosystems/eve-neo4j
