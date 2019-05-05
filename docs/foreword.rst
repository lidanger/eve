.. _foreword:

前言
========

开始使用 Eve 前请阅读这篇文章。这篇文章希望回答一些关于项目目的和目标，以及什么时候你应该或不应该使用它的问题。

信条
----------
你有数据存储在某个地方，而你想通过一个 RESTfull Web API 将它暴露给你的用户。Eve 可以让你这样做的工具。

Eve 提供一个健壮的，特性丰富的，REST 为中心的 API 实现，而你只需要设置你的 API 配置和行为，连上你的数据源，然后就完事了。查看 :doc:`features` 获取一个 Eve 支持的 API 的可用特性列表。你可能也想看看 :doc:`rest_api_for_humans` slide deck。

API 配置存储在一个标准 Python 模块 (默认为 ``settings.py``) 中，它使自定义成为十分微不足道的的任务。也可以通过给 Eve 引擎提供自定义对象，扩展一些关键特性，即 :ref:`auth`, :ref:`validation` 和数据访问。

一点背景
----------------
在 `Gestionale Amica <http://gestionaleamica.com>`_ 中，我们已经在努力创建一个全特性、Python 驱动的 RESTfull Web API。我们学习了关于 REST 最佳模式的很多东西，并有机会让 Python 知名的 web 能力进入测试。接着，在 EuroPython 2012，我有机会分享我们所学到的。我的讲话引起很多人的兴趣，甚至几个月过去了，每天幻灯片上仍然收到很多提示。我保持接收电子邮件请求源代码示例和其他东西。毕竟，每个面向 web 的开发者未来都会遇到一个 REST API，而近来谁又不会呢？

因此，我想，也许我可以取得所有权，关闭代码 (代码名称为 'Adam') 并重构它 "一点点"，这样它可以满足更广泛的使用用场景。然后我可以将它发布为一个开源项目。好吧，它变得比所说的稍微复杂了点，但最终就成这样了，当然它叫 Eve。

REST, Flask 与 MongoDB
-----------------------
来自我的 EuroPython 演讲的幻灯片，*Developing RESTful Web APIs with Flask and
MongoDB* 是 `在线可用的`_。你可能想看看并理解特定的设计决定是为什么以及如何产生的，特别是对于 REST 的实现。

BSD 许可证
-----------
今天你发现大量开源项目都是 GPL 许可。而 GPL 有它的时代和地位，它应该最可能不会成为你为你的下一个开源项目寻求的许可。

基于 GPL 发布的项目无法用于任何商业产品，除非这个产品本身也是作为开源提供。

MIT, BSD, ISC, 和 Apache 2 许可很好的对 GPL 的替代品，它们允许你的开源软件自由用于有所有权的闭源软件。

Eve 基于 BSD 许可的条款发布。参考 :ref:`license`.

.. _在线可用的: https://speakerdeck.com/u/nicola/p/developing-restful-web-apis-with-python-flask-and-mongodb
