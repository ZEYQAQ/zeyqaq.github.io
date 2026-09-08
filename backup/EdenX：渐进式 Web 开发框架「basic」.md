
注意，本文所有的EdenX可以类比为EdenX的开源版：modernjs



必要的：

框架需要nodejs，所以如果没有20 以上的nodejs，可以用nodejs版本管理工具装一个：用nvm/fnm，这里可以GitHub：https://github.com/nvm-sh/nvm https://github.com/Schniz/fnm

针对依赖，建议使用pnpm。如果只有npm，那就可以用npm装一个pnpm：npm install -g pnpm 。当然edenx也支持yarn、npm。

node_modules的镜像源可以使用bnpm（b代表byte）：

```
npm config set registry https://bnpm.byted.org
```



npx和npm一样，内置于nodejs，全程Node Package Execute，一种包执行器，本质上干的事情是去 node_modules/.bin 目录里找二进制文件并执行。

EdenX提供了@byted/create 工具来创建项目，你可以这样：

```
npx @byted/create@latest myapp
```

 EdenX会提供一个可交互的问答界面，选择「EdenX 应用 - PC Web 研发框架」 解决方案即可创建一个 EdenX 应用。

有时候你需要在一个仓库创建多个项目，那你可以用eden-monorepo，它支持一个仓库跑多个项目。不只是EdenX，字节在很多仓库有有Monorepo的概念。Monorepo（Monolithic Repository 的缩写，意为“单体仓库”）是一种软件开发和代码管理的策略：将多个不同的项目（甚至整个公司的所有代码）存放在同一个版本控制仓库（如 Git）中。

```
npm install -g @ies/eden-monorepo
#接着上面的myapp
? 请选择项目组织方式 Monorepo 项目
? 请填写子项目名称 myapp
? 请填写子项目路径(相对 apps 或 packages 目录的路径) myapp
#如果只是想创建子项目
npx @byted/create@latest --emo-sub-prj
? 请选择解决方案： EdenX 应用 - PC Web 研发框架
? 请填写子项目名称 myapp
? 请填写子项目路径(相对 apps 或 packages 目录的路径) myapp
```

创建完之后，就可以启动了，和绝大多数前端一样，只需要在项目根目录执行pnpm run dev，就可以启动了。pnpm run build就可以构建。构建产物默认生成到 dist/，目录结构如下：

```
dist
├── deploy.yml
├── html
│   └── main
│       └── index.html
├── edenx.config.json
├── nestedRoutes.json
├── route.json
├── routes-manifest.json
└── static
    ├── css
    └── js
```

edenx.config.ts，这个文件包含了EdenX框架的所有配置，所以可以通过该配置文件修改配置，覆盖 EdenX 的默认行为。



EdenX，我理解就是把打包构建路由啥的多个工具攒一块，然后命名为EdenX，当然我的理解可能不那么对，但是这个框架大概就在做这些事情。所以先从路由开始聊，

在 EdenX 应用中，每一个入口对应一个独立的页面，也对应一条服务端路由。默认情况下，EdenX 会基于目录约定来自动确定页面的入口，同时也支持通过配置项来自定义入口。嵌套路由是一种将 URL 分段与组件层次结构和数据耦合起来的路由模式。因此，使用嵌套路由时，页面的路由与 UI 结构是相呼应的。EdenX使用routes/来记录路由。routes/下的layout.tsx定义布局组件，控制所在目录下所有子路由的布局。page.tsx 为内容组件，所在目录下存在该文件时，对应的路由 URL 可访问。

所以当你访问/user/zeyqaq/profile时，

针对Pages，当有这样的文件时，

```
.
└── routes
    ├── page.tsx
    └── user
        └── page.tsx
```

就代表EdenX有两个路由：/和/user

而布局组件会比较复杂，但是长话简说就是，如果子路由的文件目录下存在 layout.tsx，上一级layout.tsx中的 「<Outlet> 」即为子路由文件目录下的 layout.tsx ，什么是outlet？简单说是 React Router v7 中提供的 API。更多可以参考https://reactrouter.com/api/components/Outlet



还有一种动态路由，它的文件目录是这样的/routers/[xxx]/page.tsx，简单来说就是被[ ]包裹的就是动态路由。举例，routes/[任意]/page.tsx 文件会转为 /:zeyqaq 路由。除了可以确切匹配的路由，其他所有没有没明确指定的都会匹配到该路由。

动态可选路由是这样的[任意$]，举例，/routes/zeyqaq/[qaq$]/page.tsx，会转为/zeyqaq/:qaq?，/zeyqaq 下的所有路由都会匹配到该路由，并且qaq 参数在代码里可以做if路由判断。

如果在某个子目录下存在 $.tsx文件，该文件会作为通配路由组件，当没有匹配的路由时，会渲染该路由组件。好的点在于，通配路由可以被添加到routes/目录下的任意子目录中。所以你可以这样为你的项目定义一个任意地方都能访问的404 页面，routes/$.tsx：

```
function Page404() {
  return <div>404 Not Found</div>;
}
export default Page404;
```



当需要为某些类型的路由，做独立的布局，或是想要将路由做归类时，可以_xx，也就是以下划线开头的文件夹，对应的目录名不会转换为实际的路由路径。举个例子，

```
.
└── routes
    ├── __auth
    │   ├── layout.tsx
    │   ├── login
    │   │   └── page.tsx
    │   └── sign
    │       └── page.tsx
    ├── layout.tsx
    └── page.tsx
```

EdenX 会生成 /login 和 /sign 两条路由，_auth/layout.tsx 组件会作为 login/page.tsx 和 sign/page.tsx 的布局组件，但 _auth 不会作为路由路径片段出现在用户访问的 URL 中。



有时候我们的url可能长这样xxx/xxx/xxx/xx/xx/xx/，同时这些路由又不存在独立的 UI 布局，如果还像上面那样配置，就会导致文件目录非常深。EdenX允许用.来分割，比如你要访问/zey/qaq/i/love/u，而那你可以这么做：

```
└── routes
    ├── zey.qaq.[who].[how].u
    │      └── page.tsx
    ├── layout.tsx
    └── page.tsx
```

访问时就会得到 UI 布局：

```
<RootLayout>
  {/* routes/zey.qaq.[who].[how].u/page.tsx */}
  <UserProfileEdit />
</RootLayout>
```



重定向呢？可以拿login举例，EdenX进页面前先做登录校验，没登录就重定向到 /login。在任意的 page.tsx 同级目录中创建，page.data.ts 文件，这个文件就是该路由的 Data Loader。在 Data Loader 中，你可以通过调用 redirect API 来完成路由的重定向。

```
import { redirect } from '@edenx/runtime/router';

export const loader = async () => {
  const user = await getUser();
  if (!user) {
    return redirect('/login');
  }
  return null;
};
```

EdenX还允许你配置预加载、路由分片处理 来提高首屏显示速度。



聊完了路由，对于渲染部分，EdenX也有较为现代的设计，EdenX支持多种渲染方式。最常见的是CSR（比如Vue、React这样的）、SSR（比如Nextjs），总的来说EdenX支持以下几种：

| 渲染模式          | 特点                                                         | 适用场景                                       |
| :---------------- | :----------------------------------------------------------- | :--------------------------------------------- |
| **CSR**           | 在浏览器端执行 JavaScript 渲染页面（常用K8S容器）            | 交互性强、对 SEO 要求不高的应用                |
| **SSR**           | 在服务端预先渲染完整 HTML 页面（常常用faas）                 | 对首屏性能和 SEO 要求高的网站                  |
| **Streaming SSR** | 边渲染边返回，更快显示初始 UI                                | 需要更快首屏感知速度的应用（**SSR 默认模式**） |
| **RSC**           | 组件在服务端渲染，减少客户端 JS 体积；数据与组件逻辑高内聚，减少状态传递 | 追求极致性能、需要减少客户端代码的项目         |
| **SSG**           | 构建时生成静态页面，可被 CDN 缓存                            | 内容相对静态的网站，如博客、文档站点           |

对于SSR而言，SEO（就比如Google对你的网站收录）是更友好的一种渲染方式，因为它是在服务器渲染好所有的内容再传给用户的浏览器。相较于CSR（在用户自己的浏览器中渲染），免去了众多交互才能被SEO搜到的限制。在EdenX里，SSR渲染也有强大的功能，比如服务器预渲染实现更快的首屏显示、流式渲染，边渲染边返回内容。

开启SSR只需要修改edenx.config.ts即可：

```
import { defineConfig } from '@edenx/app-tools';

export default defineConfig({
  server: {
    ssr: true, // 默认启用流式渲染
  },
});
```

在 EdenX 中，如果应用在 SSR 过程中出现异常，EdenX 会自动降级到 CSR 模式，并在 CSR 重新发起数据请求，保证页面能够正常展示。同时，EdenX 支持将服务器端渲染（SSR）结果进行缓存，减少服务器每次请求时的计算和渲染时间，从而加速页面加载速度，提高用户体验。同时，缓存也能降低服务端负载，节省计算资源，提高用户访问速度。开启方法是创建server/cache.ts

```
import type { CacheOption } from '@edenx/server-runtime';

export const cacheOption: CacheOption = {
  maxAge: 500, // ms
  staleWhileRevalidate: 1000, // ms
};
```

开启之后，denX 将通过响应头x-render-cache 来标识当前请求的缓存状态。下面是一个响应示例：

```
< x-render-cache: hit

注：
hit	缓存命中，返回缓存内容
stale	缓存命中，但数据陈旧，返回缓存内容的同时做重新渲染
expired	缓存命中，重新渲染后返回渲染结果
miss	缓存未命中
```



针对BFF，我们常有一种造假数据来验证可用性的方法，叫做Mock。对于EdenX项目，当config/mock目录下存在 index.ts 时自动开启mock功能。

举例编写一个mock：config/mock/index.ts

```
export default {
  /* 属性为具体的 method 和 请求 url，值为 object 或 array 作为请求的结果 */
  'GET /api/getInfo': { data: [1, 2, 3, 4] },

  /* method 默认为 GET */
  '/api/getExample': { id: 1 },

  /* 可以使用自定义函数根据请求动态返回数据, req & res 都是 Node.js HTTP 原生对象 */
  'POST /api/addInfo': (req, res, next) => {
    res.setHeader('Access-Control-Allow-Origin', '*');
    res.end('200');
  },
};
```

代码中访问http://localhost:8080/api/getInfo 时，接口会返回 JSON 格式数据：{ "data": [1, 2, 3, 4] }。

而EdenX还可以快速造Mock数据，可以在 config/mock/index.js中自主引入 [Mock.js](https://github.com/nuysoft/Mock/wiki/Getting-Started) 等库生成随机数据，例如：

```
const Mock = require('mockjs');

module.exports = {
  '/api/getInfo': Mock.mock({
    'data|1-10': [{ name: '@cname' }],
  }) /* => {data: [{name: "董霞"}, {name: "魏敏"},  {name: "石磊"}} */,
};
```

类比Chrome的弱网模式，EdenX也为mock提供了弱网响应。如果写了mock，但是不想启用，也可以设置config/mock/index.ts

```
export const config = {
  enable: false
}
```



E2E 测试。这里将e2e不只是因为EdenX有这个功能，更是因为e2e在流水线中也有常见的应用。先讲一下什么是e2e，端到端测试（E2E Testing）是从用户视角验证整个系统业务流程是否正常的测试。 模拟真实用户操作（如点击、输入、跳转），验证整个软件系统（包括前端、后端、数据库、第三方服务、网络等）协同工作的完整流程是否正常。例如，在电商系统中，E2E 测试会走完“搜索商品 -> 加入购物车 -> 提交订单 -> 支付 -> 生成物流信息”的全过程。

EdenX使用Playwright框架来做E2E。

在流水线中，一个e2e Job会起多个docker容器来验证各个功能，所以常常以一台虚拟机为基准，起多个docker（而不是起一个docker镜像后再起docker）。



EdenX 提供了对环境变量的支持，包含内置的环境变量和自定义的环境变量。内置的环境变量包含ASSET_PREFIX表示当前资源文件的路径前缀、NODE_ENV表示当前的执行环境、EDENX_ENV手动设置当前的执行环境、EDENX_TARGET区分 SSR 与 CSR 环境。对于自定义环境变量则可以在pnpm前放入变量

```
ZEYQAQ=123 QAQ=456 pnpm run dev
```

或者在项目根目录创建 .env文件，并添加自定义环境变量，这些环境变量会默认添加到启动项目的 Node.js 进程中，例如：

```
zeyqaq=123
qaq=456
```



如何将前端内容上传CDN？

打开字节云-SCM-配置-打开静态资源上传 CDN开关，并且选择静态资源打包目录为output_resource/。

除此之外不需要再操作其他内容，EdenX 中已经支持从环境变量中自动获取 SCM 上的配置并根据线上版本或线下版本选择合适的域名，拼接成项目预期的 CDN 地址。通常情况无需再配置 [output.assetPrefix](https://edenx.bytedance.net/configure/app/output/asset-prefix.html)。



[deploy.yml](https://cloud.bytedance.net/docs/goofy/docs/63db9b5c7df7d2021d05ec26/667d43f74bcd440301c6ba4e?source=search&from=search_bytecloud&x-resource-account=public) 文件是 Goofy Deploy 默认使用的标准文件协议。默认情况下，执行deploy 命令时，EdenX 会在output 目录下自动生成该文件。默认生成的deploy.yml 文件中会包含部署的元信息（如服务端路由信息），这样开发者就无需在框架和平台分别配置。如需修改路由信息，可以使用 [server.routes](https://edenx.bytedance.net/configure/app/server/routes.html) 进行调整。

但在有些场景下，你可能需要修改其他部署元信息（例如，部署后 server 使用的 Node.js 版本）。此时，你可以在项目根目录添加一个自定义的deploy.yml 文件。执行deploy 命令时，EdenX 会将项目中的deploy.yml 与框架的默认元信息合并，并生成最终的deploy.yml 文件，保存在output/deploy.yml 中。