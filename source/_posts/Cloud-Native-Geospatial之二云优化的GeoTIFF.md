---
title: Cloud Native Geospatial之二云优化的GeoTIFF
date: 2019-03-22 15:57:19
tags: Cloud Native GIS
categories: Technology
---

这篇文章是Cloud Native GeoSpatial系列文章的第二篇。本系列文章主要探寻如何使用原生云环境构建地理空间系统和工作流程。本文将深入研究地理空间云原生领域最重要的技术：云优化的GeoTIFF。

云优化的GeoTIFF（Cloud Optimized GeoTIFF，简称COG）是一种特殊格式化的GeoTiFF文件，它借助HTTP的新特性[Byte Serving](https://en.wikipedia.org/wiki/Byte_serving)实现。[cogeo.org](http://cogeo.org/)提供了更多详细信息。Byte Serving是一种提供在线视频或音频流文件的技术，能够跳过文件前面或者后面的某些内容。你可以告知服务器从某个特定的位置开始获取文件，而并不需要下载全部的多媒体文件。COG格式以同样的方式工作，允许用户访问一个栅格文件中他们所需要的那一部分。

任意跳转到一个栅格文件中任何所需的部分，这能够创造很多新的工作流程，因为数据能够像视频一样成为数据流，而不需要全部通过网络传输。在地理空间领域，用户可以在线访问Web瓦片，但是做实际的分析仍然需要原始删个文件。这意味着要获取一个几百兆或者更大的文件需要较长的下载时间。这因为原始的删个文件在云中分布式部署，但是他们不能够表现为流格式。所以用户在处理和可视化云中文件的时候，必须全部下载文件。

COG格式始于Amazon、Planet Labs、MapBox、ESRI和USGS之间的合作，为了以可访问的形式把Landsat档案放到AWS中。GeoTIFF是一种广泛应用的影像文件格式，但同时在Landsat-pds的邮件列表中对如何以最佳格式使用流式格式传输数据有着广泛的讨论。好的存储格式能使合作伙伴在他们现有的工作流程中利用Amazon Web Services中的归档数据，而不需要进行复制和重复处理这些数据。一旦这种云原生的模式建立，采用这种架构的软件将会用同样的方式使用数据，这将显著的提高数据利用。

有文献给出了针对云工作流程优化GeoTIFF的最佳实践，GDAL的实现包括了明确的文档和发布在GDALWiKi的性能测试结果。Planet Labs通过处理管道将所有数据的生产逐渐转向了云优化的GeoTIFF，并借此和它的合作伙伴FarmShots和Santiago&Cintra一起重新构建起特定领域的应用。

{% asset_img  Farmshots.png Farmshots applying an NDVI to Cloud Optimized GeoTIFF’s pulled in dynamically from Planet  %}

在产业界方面，领先的开源项目如DGAL、QGIS和GeoServer已经可以读取这种格式（通过QGIS和GeoServer的高级配置）。DigitalGlobe目前正在将他们的IDAHO系统迁移到使用COG格式，同时利用它处理大量的数据。OpenAerialMap的架构就是将用户上传的数据转换为Web可读的GeoTIFF格式并且直接从这些数据中流式传输他们的图层。GeoTrellis项目在短期开发计划中将对COG进行支持，另外一些项目包括相似的空间集群计算处理系统也有意支持它。

{% asset_img  OpenAerialMap.png A COG of drone imagery from St. Maarten served up as tiles on OpenAerialMap.org  %}

{% asset_img  COG.png The same COG streamed directly from OpenAerialMap’s S3 bucket into QGIS, saving substantial download time.  %}

这些新的线上处理系统强调利用云原生地理空间架构。这种系统能同时地在上百个甚至几千个计算机中处理影像，在几秒内就能返回原来需要几天甚至数周的分析结果。尽管有这些现代化的有点，因为核心的标准是GeoTIFF，任何软件都可以读取数据，甚至是旧的桌面应用程序。

尽管核心概念十分简单——将图像移至线上并通过流式传输，这却是构建云原生空间系统的基础。数据在云中，许多软件系统更接近数据端运行，已达到获取数据价值的目的，不需要额外的下载和存储成本。对此生态系统更全面的研究将在后续中介绍，但是核心的COG格式和原理是根本，用户可以将他们的精力花在接近实时的工作来探寻数据价值，而不用依靠少数具备空间分析专业能力的专家。

尽管云优化的GeoTIFF是一种相对新的格式，但是向后兼容性和相对容易的实现使得这种格式引人注意，这些组织的目的是鼓励更多的软件和数据提供商接收它。如果你感兴趣想了解更多的东西，请访问cogeo.org。