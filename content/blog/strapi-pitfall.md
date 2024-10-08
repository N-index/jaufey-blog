---
title: "Strapi 踩坑"
date: 2024-08-23T11:05:41+08:00
lastmod: 2024-08-23T11:05:41+08:00
draft: false
tags: ["CMS", "Strapi"]
---

[Strapi](https://strapi.io/) 是一个无头 CMS，总体用下来70分，本文列一下使用 Strapi 遇到的问题。  
版本是：V4，太不巧了，在项目完成后，刚好出了 v5 版本

## 优点
1. 业务人员：
   - admin 面板中的主体功能、基本操作都没有大问题，设计简洁大方，流程上手简单。
2. 开发人员：
   - 目录结构清晰，架构比较好
   - 扩展功能方便，编写插件方便
   - 其他 SSR 框架支持比较好，比如有 Nuxt Strapi 这种 module
   - 生态还算不错，虽然没有特别要用的 plugin 就是了。
   - 总的来说，文档也还不错，中度使用没问题。至少有些长久遗留的问题会在他们论坛会被骂，能被我索引到。


## 坑
1. 数据迁移费老大劲。

   Data Import 过慢，15Kb/s 的导入速度对于 2G 多的压缩包来说，根本传不完，Export 时关闭压缩和加密都没有效果，要不是不知道它的 Import 操作的传输 Upload 文件夹部分有什么副作用，我直接 Copy 文件夹过去算了。后续要不是 Data Transfer 在数次失败后传输速度突然飙到几百K并顺利完成了，那数据迁移工作算是功亏一篑彻底失败掉地上了，毕竟项目至少得在 CI/CD 测试环境中跑起来一次才好备份迁移啊。

2. Upload 功能不成熟。
    - 在 Docker 中，尽量不要往项目运行时的代码目录下写东西，但大部分 CMS 的资源 LocalProvider 有写文件的操作，然而在 Upload 插件中没有办法配置文件目录到其他指定目录，导致 CI 需要做其他软链操作来确保服务的无状态。
    - 存文件思路小学生水平：Media Library 中的逻辑文件夹未反应到文件系统中。很难想象一个正儿八经的 CMS 会把所有图片、视频、文档放到同一个文件夹下，后续文件数量上来后 fs 操作得有多慢。这点根本比不上 Wordpress 和 MediaWiki，前者是按日期分文件夹，后者是按文件名 Hash 分文件夹，而 Strapi 像是单人维护的练手项目一样（虽然事实上可能确实是）。
    - 插件接口文档不全。我想在后台的 db lifecycles 中使用 upload 的相关功能来为当前的 entity （比如 blog）自动添加 cover, 但是文档中没有写怎么用，只有一个 [Upload single file from an API controller](https://docs.strapi.io/dev-docs/plugins/upload#upload-single-file-from-an-api-controller) 的 demo 。findOne 和 findMany 有没有、要传什么参数都不知道，经过调试才实现功能。我要做的操作也不算多么小众仅仅是在后台操作文件而已。

3. Upload API 不成熟。

   关于 Media Library 的 Upload 相关接口不太成熟，像是不得已才暴露出来几个查文件、改文件的接口来应付社区。
    - [https://docs.strapi.io/dev-docs/plugins/upload#endpoints](https://docs.strapi.io/dev-docs/plugins/upload#endpoints)
    使用这里面的 API 上传文件时，不能声明目录，而是统一放到他们硬编码的 /api-upload 中去，十分草台离谱。
    - REST 没有访问目录结构的API，非得用只能去后台网页上扒管理员接口下来，自己维护 folder 及其 path 或 id 信息。
    - REST 获取文件列表的时候，不能筛选目录结构，导致从其他系统迁移文章到 Strapi 的过程中，原本不同目录下的同名图片替换成了首个同名图片。

4. 部署配置耍小孩玩。

   `config/admin.js` 中的 admin 面板地址，及 `config/server.js` 的配置过程非常不顺利。
   
   场景：因为 Strapi 很可能是作为其他系统的一个子系统来使用，所以路由地址也很可能是 https://localhost/strapi/admin，前端流量由 Nginx 来分流。但是你即便配了 PublicUrl 或 Admin 面板的 URL，Strapi Admin 面板中的其他请求仍然是从根地址发起。  
    一开始还试图妄想通过测试这些配置项搭配出可行无误的地址，但这几个配置项像是黑盒一样按下葫芦浮起瓢，走起路来摇摇晃晃撒汤带水，后续看到社区有同样问题有维护者说他们的接口地址是硬编码的，并且甩锅提倡使用者应该通过前置代理去分流什么的，导致我是直接放弃配置了，Strapi 的所有东西全部放到子域名中去。[这个帖子](https://feedback.strapi.io/developer-experience/p/configure-strapi-backend-url-after-build) 有人直接开骂了。

5. 自定义接口时内部耦合，文档缺失，找不到最佳实践，Entity Service API 和 Query Engine API 内部耦合高。

   首先我知道大部分情况下，Entity Service 是可以 cover 自定义功能的，只有 cover 不住了才需要使用 Query Engine。
   
   场景：客户端 REST API findOne 试图通过 Slug 访问资源时，需要重写 findOne 查询的逻辑，因为原始 findOne 是通过 id 查的。然而搜到的这个[解决方案](https://www.youtube.com/watch?v=qp-g8SUfreI&t=18s)纯粹是糊弄人，上来就 Query Engine 直接用 `where slug = xxx` 来查，一想就有问题，因为官方的 findOne 中的 Entity Service 内部肯定调用了 Query Engine，中间处理了各种 filter、populate 等参数到 Query Engine 的 Normalize，如果用户手动去调用 Query Engine，那就根本不知道如何让 Query Engine 的这些参数生效。Entity Service 的这些参数也不能照搬给 Query Engine，因为这两套 API 有很多相似但又不相同的地方，一层一层点到源码里面才知道 Entity Service findOne 调用 Query Engine 时写死了 `where id = xxx` 。
   
    后来还是索性自己重写 findOne 的逻辑，在Entity Service基础上筛出所有记录，并且限制 filter=slug，limit=1，总觉得很脏。刚才看了下官方 foodAdvisor 这个 Demo 项目，他们在 next 中获取单篇 blog 也是根据 slug 筛出了所有文章，然后取了第一篇，想必他们也知道这个问题了，那还不赶快优化😒。

 6. REST API 的 filter 中的 $ne 针对 Boolean 的字段不太严谨。假如有一个 nullable 字段，当我通过 $ne = true 过滤的时候，应该同时过滤出 null 或 false 的 entities，但实际上却只过滤出了 false 的 entities，我仍要通过 $or 来组合出想要的结果。
