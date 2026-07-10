# YUNUCMS ThinkPHP 8 生产升级任务清单

> **实施要求：** 后续执行本清单时使用 `superpowers:executing-plans` 分阶段实施并在每个阶段停留验收。任务状态统一使用 `- [ ]`；只有验收证据齐全后才能改为 `- [x]`。

**目标：** 将当前内置 ThinkPHP 5.0 的 YUNUCMS 1.1.6 升级为 Composer 管理的 ThinkPHP 8.1 多应用项目，在保持非地区业务 URL、数据库、主题、后台权限和 API 契约的前提下达到可测试、可观测、可灰度、可回滚的生产标准。`yunu_area` 及地区分站能力按明确的破坏性变更完整下线；多站点由全局 middleware 将显式 Host 映射为请求级 `SiteContext`，不再由控制器或地区表识别；现有 `nginx.txt` 只作为迁移输入，其余业务 rewrite 语义必须内化为 ThinkPHP 8 命名路由。

**迁移策略：** 采用“冻结旧版行为 → 建立完整回归基线 → 搭建 TP8 多应用骨架 → 按公共层、数据层和应用逐批迁移 → 双版本契约比对 → 灰度切换”的路线。禁止直接用新框架覆盖 `system/` 后集中修错；TP5 基线通过 Git 标签/独立工作树保留，不在新代码中长期维护两套框架。

**技术基线：** PHP 8.4、ThinkPHP `^8.1`、`topthink/think-multi-app ^1.1`、`topthink/think-view ^2.0`、Guzzle `^7.13`、`overtrue/pinyin ^6.0`、PHPUnit `^13.1`、Mago `^1.43`、Composer 2、MySQL 8.0+（CI 使用 MySQL 8.4）、Unsplash 官方 JSON API、生产 URL 固定为 rewrite 模式（目标语义对应 `sys.url_model = 3`）、模板逻辑路径分隔符固定为 `/`。当前仓库样例配置实际为 `url_model = 1` 且 index/WAP `view_depr = '_'`，升级时必须显式切换并验证。实际解析版本由 `composer.lock` 固定。

## 状态与阶段门禁

本文创建时所有升级任务均未开始。每个阶段只有同时满足以下条件才可进入下一阶段：该阶段所有复选框完成；对应 PHPUnit 套件通过；Mago 无新增问题；`composer audit --locked` 无可利用漏洞；变更具备回滚路径；验收证据附在提交或变更说明中。

生产发布总门禁：

- [ ] 每个行为变更、缺陷修复、删除项和结构迁移均留有可复现的 RED→GREEN→REFACTOR 证据；没有先失败的 PHPUnit 测试不得把实现任务标为 `[x]`。
- [ ] P0/P1 业务场景与公开路由的回归矩阵覆盖率达到 100%；非地区契约无未批准差异，地区 MVC/API/标签/URL/SQL 的下线结果全部有测试。
- [ ] `vendor/bin/phpunit` 在干净测试数据库上全量通过，无 risky、warning、deprecation、skipped（明确记录的外部服务隔离项除外）。
- [ ] 应用代码总体行覆盖率不低于 80%，认证、授权、配置写回、安装升级、上传和动态表事务代码不低于 90%，且覆盖率不低于 TP5 基线。
- [ ] `vendor/bin/mago lint --semantics`、`vendor/bin/mago lint`、`vendor/bin/mago analyze`、`vendor/bin/mago format --check` 全部退出 0。
- [ ] 对第一方迁移代码执行 `vendor/bin/mago lint --ignore-baseline` 与 `vendor/bin/mago analyze --ignore-baseline` 后无错误；遗留基线只允许覆盖尚未迁移且有明确删除阶段的代码。
- [ ] `composer validate --strict`、`composer check-platform-reqs`、`composer audit --locked` 全部通过。
- [ ] 数据库连接参数只由 `.env`/部署密钥提供；第一方 PHP、migration、Command、测试 fixture 和部署模板不存在真实值或可被生产误用的 host/port/database/username/password/prefix/charset/timezone/SSL 硬编码。
- [ ] 所有保留的站内链接均为 rewrite URL；Apache、Nginx 与 PHP 内置服务器对同一路径的状态码、参数和 WAP 分发结果一致，公开页面不生成 `index.php` 或地区分站地址。
- [ ] `deploy/nginx/yunucms.conf` 和 `public/.htaccess` 不包含栏目、内容、地区、WAP、后台等业务正则；移除任意一层 Web 服务器后，ThinkPHP 路由契约仍能独立通过。
- [ ] index、WAP、admin 的模板逻辑路径全部使用 `/`；主题目录、栏目 `tpl_cover/tpl_list/tpl_show`、include 和控制器渲染调用不存在以 `_` 充当目录分隔符的旧写法。
- [ ] 所有 HTTP 请求先经多站点识别 middleware；未知 Host 返回 421、停用站点返回 503，控制器/模型/模板/API 不直接读取 `HTTP_HOST` 或自行选择站点。
- [ ] 第一方通用网络请求和拼音处理只通过已批准的 Composer 能力；不存在 `send_post()`、`url_get_contents()`、手写拼音表、散落的 cURL/远程流调用或功能重叠的依赖。
- [ ] Unsplash 配图只访问官方 API/CDN，密钥不下发客户端；图片热链、摄影师/Unsplash 署名、UTM 链接和 `download_location` 选用追踪符合官方 API Guidelines。
- [ ] 预发布环境完成全量回归、数据迁移演练、性能对比、安全验证、灰度与回滚演练。

## TDD 执行规则

- [ ] RED：每次只定义一个可观察行为，先运行最小目标命令（例如 `vendor/bin/phpunit --filter ResolveSiteMiddlewareTest`），确认测试因目标能力缺失而失败，而非测试拼写、启动、fixture 或环境错误；记录测试名、命令和关键失败信息。
- [ ] GREEN：只编写让当前失败测试通过的最小生产改动，不顺带重构邻近模块；再次运行相同目标命令并保存通过证据。
- [ ] REFACTOR：仅在 GREEN 后删除重复、改善命名或收紧结构；重跑目标测试、受影响 testsuite、Mago 和 Composer 门禁，行为发生变化时必须回到新的 RED。
- [ ] 迁移遗留行为先写 characterization/contract 测试；路由、middleware、migration、目录、模板、删除 `yunu_area`、移除 Web Install 和替换第三方库同样必须测试先行，不能以“只是重构/删除/配置”为由跳过 RED。
- [ ] 测试调用真实生产入口和公开接口，使用真实临时数据库、临时文件系统及 Composer 类；只在邮件、对象存储、Unsplash 等进程外边界使用 fake/mock，不为测试在生产类中增加专用方法，也不把“某 mock 被调用”当作唯一业务断言。
- [ ] 如果新测试首次运行即通过，先证明行为本已存在或修正测试使其覆盖缺口；不得把一个从未失败的测试作为本任务 TDD 证据。
- [ ] 覆盖率用于发现盲区，不能替代 RED→GREEN；每个小提交/PR 在说明中附最小失败命令、失败原因、通过命令和重构后回归结果。
- [ ] 外部 API 测试默认断网：用 Guzzle `MockHandler` 或受控本地 fake server 返回真实协议形状，断言应用结果、持久化和错误映射；仅独立、显式授权的预发布 smoke test 可访问真实服务。

## 官方依据

- ThinkPHP 8 官方手册：<https://doc.thinkphp.cn/v8_0/setup.html>
- ThinkPHP 8 官方框架：<https://github.com/top-think/framework/tree/8.x>
- ThinkPHP 8 官方项目骨架：<https://github.com/top-think/think/tree/8.x>
- 多应用扩展：<https://github.com/top-think/think-multi-app>
- Think 模板驱动：<https://github.com/top-think/think-view>
- PHPUnit 支持矩阵：<https://phpunit.de/supported-versions.html>
- Mago 官方仓库：<https://github.com/carthage-software/mago>
- Mago 配置与命令：<https://mago.carthage.software/main/zh/guide/configuration/>、<https://mago.carthage.software/main/zh/fundamentals/command-line-interface/>
- Guzzle：<https://github.com/guzzle/guzzle>、<https://packagist.org/packages/guzzlehttp/guzzle>
- Overtrue Pinyin：<https://github.com/overtrue/pinyin>、<https://packagist.org/packages/overtrue/pinyin>
- Unsplash API 文档与 Guidelines：<https://unsplash.com/documentation>、<https://help.unsplash.com/en/articles/2511245-unsplash-api-guidelines>

## 目标文件结构

| 路径 | 目标职责 |
| --- | --- |
| `composer.json` / `composer.lock` | PHP 8.4、TP8、测试和质量工具的唯一依赖来源 |
| `think` | 官方 CLI 入口，承载安装、迁移、缓存和运维 Command |
| `public/index.php` | Composer 自动加载与 TP8 HTTP 入口 |
| `public/router.php` / `public/.htaccess` | 本地开发与 Apache 通用前端控制器转发，不承载业务路由 |
| `deploy/nginx/yunucms.conf` | Nginx 通用生产站点配置：`public/` 根目录、静态文件、PHP-FPM、安全头与 `try_files` |
| `app/AppService.php`、`app/BaseController.php`、`app/ExceptionHandle.php`、`app/Request.php` | 官方应用服务、基础控制器、异常和请求扩展点 |
| `app/common.php`、`app/event.php`、`app/middleware.php`、`app/provider.php`、`app/service.php` | 官方根级公共函数、事件、全局中间件、容器绑定和服务注册入口 |
| `app/index/`、`app/wap/`、`app/admin/`、`app/api/` | TP8 多应用目录；内部按需包含 `controller/`、`model/`、`validate/`、`view/`、`config/`、`route/`、`middleware.php`、`provider.php` |
| `app/index/route/app.php`、`app/wap/route/app.php`、`app/admin/route/app.php`、`app/api/route/app.php` | 唯一业务路由来源及命名路由定义 |
| `app/common/` | 跨应用且确实共享的中间件、服务、模型、验证器和辅助函数 |
| `app/common/middleware/ResolveSite.php` | 校验 Host、解析站点并建立请求级上下文的唯一入口 |
| `app/common/site/SiteContext.php` | 当前站点 key、canonical URL、主题、通道及隔离前缀的只读值对象 |
| `app/common/integration/UnsplashImageService.php` | 使用共享 Guzzle 客户端完成 Unsplash 搜索/随机配图、选择追踪和错误映射的最小集成 |
| `app/common/service/PinyinSlugger.php` | 基于 `overtrue/pinyin` 的唯一拼音 slug 规则；不保留自制字符表 |
| `config/sites.php` / `.example.env` | 显式站点/域名注册表、数据库连接键及环境覆盖，不接受数据库地区记录、泛域名猜测或真实凭据 |
| `app/common/service/FormSubmissionService.php` | index、WAP、API 共用的唯一自定义表单提交实现 |
| `app/command/Install.php` / `config/console.php` | `php think yunu:install` 安装命令及注册；不提供 Web 安装入口 |
| `config/` | 官方全局配置目录：app、cache、console、cookie、database、filesystem、lang、log、middleware、route、session、view 等 |
| `route/app.php` | 仅放框架级/健康检查等全局路由；应用业务路由留在各应用 `route/` |
| `view/error/` | 根级通用 404/异常/维护模板；业务模板留在应用 view 目录 |
| `app/index/view/<theme>/<group>/<template>.html`、`app/wap/view/<theme>/<group>/<template>.html` | 多应用标准 view 目录内的主题模板，逻辑分隔符使用 `/` |
| `app/admin/view/<controller>/<action>.html` | 后台标准控制器/操作目录模板 |
| `public/static/admin/`、`public/static/themes/<theme>/<app>/` | 后台和主题 CSS、JS、字体、图片等公开静态资源 |
| `public/uploads/` | 持久化上传挂载点，禁止脚本执行；非公开文件使用存储驱动而非 Web 目录 |
| `extend/` | 仅保留无法进入 `app/` 且不适合 Composer 包的扩展；第三方库优先归 `vendor/` |
| `database/migrations/` / `database/seeders/` | 可重复、可审计、可回滚的结构迁移与测试种子 |
| `tests/Unit/` | 无框架或轻框架依赖的单元测试 |
| `tests/Unit/Bootstrap/FunctionGuardTest.php` | 全局用户函数防重定义、重复加载与同名语义冲突检查 |
| `tests/Unit/Configuration/DatabaseConfigurationTest.php` | 数据库 `.env` 必需键、类型、缺失失败、源码硬编码和日志脱敏门禁 |
| `tests/Integration/` | 数据库、缓存、会话、文件系统和外部适配器集成测试 |
| `tests/Feature/` | 前台、WAP、后台、API、安装与运维 HTTP 回归测试 |
| `tests/Feature/Command/` | 安装命令的进程退出码、输出、幂等、故障恢复和密钥脱敏测试 |
| `tests/Contract/` | TP5 与 TP8 的路由、响应、HTML 和数据副作用契约比对 |
| `tests/Contract/Route/` | `nginx.txt` 规则等价、命名路由往返、冲突优先级及跨 Web 服务器契约 |
| `tests/Contract/AreaRemovalTest.php` | 地区表、MVC、路由、API、标签、配置和 SQL 完整删除门禁 |
| `tests/Contract/TemplatePathTest.php` | `_`→`/` 模板映射、数据库引用、include、全主题编译和路径安全门禁 |
| `tests/Contract/ProjectStructureTest.php` | TP8 官方目录白名单、入口、应用文件加载和旧目录清零门禁 |
| `tests/Integration/External/UnsplashImageServiceTest.php` | Unsplash 协议、热链、署名元数据、选用追踪、限流与故障回退测试 |
| `tests/Fixtures/` / `tests/Support/` | 脱敏固定数据、快照、测试引导和辅助工具 |
| `mago.toml` / `*-baseline.toml` | PHP 8.4 语义、lint、分析、格式及遗留问题基线 |
| `.github/workflows/ci.yml` | Composer、Mago、PHPUnit、覆盖率和安全审计门禁 |
| `runtime/` | TP8 缓存、日志和编译文件，不进入 Git |

## `nginx.txt` 路由内化基线

以下矩阵区分“继续兼容”和“随地区功能下线”。保留路径使用 ThinkPHP 8 命名路由、固定控制器动作和参数约束；地区路径只在受控退役窗口返回确定状态，最终不保留地区控制器、模型、查询或 URL 生成。

| `nginx.txt` 行 | 现有公开路径族 | TP8 路由目标 |
| --- | --- | --- |
| 1–8 | 不存在文件/目录且带 `.asp`、`.aspx`、`.asa`、`.asax`、`.dll`、`.jsp`、`.cgi`、`.fcgi`、`.pl` 等后缀 | 统一进入应用 404/安全拒绝契约；Nginx 只保留与业务无关的脚本执行防护 |
| 10–11 | `/admin/<path>.html`、`/admin/<path>` | 根据后台菜单、表单和 AJAX 清单定义 `admin.*` 显式命名路由；禁止保留任意控制器/方法通配分发 |
| 12、15–17 | `/m/`、`/m/search/`、`/m/myform/`、`/m/captcha/<id>` | 保留为 `wap.home`、`wap.search`、`wap.form`、`wap.captcha` |
| 13–14、18、20、23–24、27 | WAP 地区内容、搜索、标签、栏目、分页和地区首页 | 标记为地区退役路径；一个发布周期内统一返回 410，随后删除兼容路由并返回 404，不创建 `wap.*.area` 路由 |
| 19、22 | `/m/<categoryPath>/<content>[_<cw>].html` | 保留为 `wap.content`、`wap.content.keyword` |
| 21 | `/m/tag/<title>` | 保留为 `wap.tag` |
| 25–26 | `/m/<categoryPath>/`、`/m/<categoryPath>/page/<page>` | 保留为 `wap.category`、`wap.category.page` |
| 28 | `/statics/ueditor/dialogs/.../*.html` | 真实静态文件直接返回，不建立应用路由，也不保留自我 rewrite |
| 29 | `/index.../` 引导页路径 | 先以真实生成链接和访问日志确认允许形态，再收敛为 `index.guide`；禁止照搬贪婪 `(.*)` |
| 32–35 | `/search/`、`/search/page/<page>`、`/myform/`、`/captcha/<id>` | 保留为 `index.search`、`index.search.page`、`index.form`、`index.captcha` |
| 30–31、36、39、41–42、45 | 桌面地区内容、搜索、标签、栏目、分页和地区首页 | 标记为地区退役路径；一个发布周期内统一返回 410，随后删除兼容路由并返回 404，不创建 `index.*.area` 路由 |
| 37–38 | `/<categoryPath>/<content>[_<cw>].html` | 保留为 `index.content`、`index.content.keyword` |
| 40 | `/tag/<title>` | 保留为 `index.tag` |
| 43–44 | `/<categoryPath>/`、`/<categoryPath>/page/<page>` | 保留为 `index.category`、`index.category.page` |

路由实现硬约束：

- [ ] 所有固定端点、内容端点、分页端点、栏目端点按“最具体到最宽泛”注册，并用冲突测试锁定优先级，确保 `search`、`tag`、`myform`、`captcha`、`page`、`admin`、`m` 不会被栏目路由吞掉。
- [ ] `<id>` 和 `<page>` 仅接受正整数；`<categoryPath>`、`<content>`、`<cw>`、`<title>` 明确字符集、长度、URL 解码次数和保留字，非法或二次编码输入返回确定的 404/422。
- [ ] 允许多级栏目时，由固定的栏目/内容解析控制器接收受约束的完整路径并查询栏目元数据；路由变量不得决定控制器类、方法、表名或 SQL 片段。
- [ ] 所有反向生成使用命名路由和统一站点域名；模板、模型、标签库、API、sitemap、canonical 与跳转代码不得手工拼接路径、地区上下文或读取 `url_model` 数值分支。
- [ ] 地区 URL 不得被普通栏目宽路由误识别；退役清单在一个发布周期内统一返回 410，清单外返回 404，观察期后删除兼容清单和地区子域名 DNS。

---

## 阶段 0：冻结范围、生产基线与回滚条件

### 0.1 建立不可变升级基线

**涉及：** Git 标签、升级分支、发布记录，不修改业务代码。

- [ ] 确认工作区干净，只保留用户明确授权的变更；记录 `git status --short`、当前提交、PHP/Composer/MySQL/Web 服务器版本和已加载扩展。
- [ ] 为当前可运行 TP5 版本创建带说明标签 `pre-thinkphp8`，升级工作使用 `codex/thinkphp8-upgrade` 分支；标签和分支都推送到远端后再开始编码。
- [ ] 从生产备份制作脱敏数据库与上传文件样本，禁止测试环境连接生产数据库、生产 Redis、生产对象存储或真实外部 API。
- [ ] 在独立位置恢复数据库和上传备份，验证备份可用、字符集/时区一致，并记录恢复耗时与校验结果。
- [ ] 建立两个隔离运行实例：`legacy` 固定 `pre-thinkphp8`，`candidate` 运行升级分支；写操作使用两套独立数据库副本。
- [ ] 明确回滚触发器：5xx、登录失败率、关键接口错误率、数据校验差异、P95 响应时间、队列/外部 API 异常达到阈值即停止灰度并回滚。

**验收：** 任意开发者可从标签、脱敏数据和运行说明重建 TP5 基线；回滚不依赖升级分支中的代码。

### 0.2 固化生产兼容矩阵

**产物：** `docs/upgrade/runtime-matrix.md`、`docs/upgrade/route-inventory.md`、`docs/upgrade/site-matrix.md`、`docs/upgrade/integration-inventory.md`。

- [ ] 固定 PHP 8.4 最新安全补丁为构建和生产基线，列出必需扩展：`ctype`、`curl`、`fileinfo`、`gd`、`json`、`mbstring`、`openssl`、`pdo`、`pdo_mysql`、`session`、`zip`。
- [ ] 固定数据库基线为 MySQL 8.0+、`utf8mb4`、严格 SQL 模式；CI 使用 MySQL 8.4，生产若低于该基线则先完成数据库升级。
- [ ] 导出所有前台、WAP、后台、API、现有 Web 安装入口、rewrite URL、地区子域名和移动域名，标注请求方法、认证要求、输入、状态码、响应类型和数据副作用；地区 URL/API 与 Web 安装 URL 分别列为明确下线项，其余旧动态业务 URL另列迁移期兼容跳转。
- [ ] 把 `nginx.txt` 第 1–45 行逐条录入 `route-inventory.md`，每条包含原正则、真实示例 URL、HTTP 方法、目标应用/动作、参数、状态码、TP8 路由名和兼容结论；不得用“同类规则略”留下未覆盖项。
- [ ] 从访问日志、数据库栏目 `etitle` 和全部 URL 生成函数采样多级栏目、内容关键词、中文标签及引导页真实路径，明确保留的旧 `(.*)` 实际允许层级；另导出仍有流量的地区 URL 作为退役报告，不把地区前缀加入新路由变量。
- [ ] 对比 `app/index/common.php`、`app/wap/common.php`、Category/Content Model、Yunu 标签库与 `nginx.txt`，列出所有地区 URL 生成点并删除；不得把地区化 myform/captcha 等旧缺陷转换为 TP8 兼容路由。
- [ ] 从后台菜单、模板、JS/AJAX 和控制器动作生成后台路由白名单，记录 GET/POST 等真实方法；不得把旧 `/admin/(.*)` 当作 TP8 动态控制器分发需求。
- [ ] 生成旧模板到目录模板的不可变映射表，覆盖 index/WAP 全部 HTML、模板 include、控制器隐式 `fetch()`、`tpl_file` 拼接、栏目 `tpl_cover/tpl_list/tpl_show` 数据及后台 `list_*`/`show_*`/`cover_*` 扫描器。
- [ ] 映射规则只转换结构分隔用途的首个 `_`：`index_index.html → index/index.html`、`list_product_diy.html → list/product_diy.html`、`public_header.html → public/header.html`；业务 slug 中剩余下划线和 CSS/JS/图片文件名保持不变。
- [ ] 在 `site-matrix.md` 列出每个站点的稳定 key、canonical URL、主域名/别名、桌面或 WAP 通道、主题、启用状态、Cookie 名和缓存前缀；本轮站点共享业务内容，不引入站点 CRUD 或业务表 `site_id`。
- [ ] 盘点全部 `$_SERVER['HTTP_HOST']`、Request host、`site_url`、`site_levelurl`、`wap_levelurl`、主题选择、绝对 URL、Cookie domain 和缓存 key 调用，标注改由 middleware/SiteContext 提供或删除。
- [ ] 将当前 `config/extra/sys.php` 的 `url_model = 1` 记录为迁移输入，将生产目标固定为 rewrite（旧语义值 `3`），并列出配置切换、缓存清理和回滚步骤。
- [ ] 盘点 PHPMailer、七牛、百度 AIP、验证码、图片处理、压缩、IP 库、云授权/升级等内置依赖及其调用方，给出“Composer 替换 / 保留并适配 / 下线”结论；记录包名、许可证、最新稳定版、PHP 8.4 约束、安全公告、维护活跃度和替代成本。
- [ ] 用调用图盘点 `send_post()`、`url_get_contents()`、`get_headers()`、`com\Http`、第一方 `curl_*`/远程 stream context 及其超时、TLS、重试、SSRF 和错误语义，收敛为 Guzzle 迁移矩阵；厂商 SDK 内部网络实现不复制到第一方。
- [ ] 盘点 `get_pinyin()`、`statics/pinyin.dat`、GB2312/UTF-8 转换及 CategoryModel 调用方，保存中文、ASCII、多音字、标点、空值和冲突 slug 的 characterization fixture，明确迁移到 `overtrue/pinyin` 后允许的差异。
- [ ] 把“图片生成”产品范围固定为后台内容/栏目可选择的 Unsplash 自动配图：支持按标题/SEO 关键词搜索及随机候选、方向和安全级别筛选、人工确认；不引入 AI 生成、自建图库、Unsplash 克隆、批量无人值守抓图或网页爬取。
- [ ] 在集成清单记录 Unsplash 应用状态、Access Key 配置来源、生产配额申请、允许端点、UTM source、署名展示位置、图片/摄影师元数据保存方式、禁用开关和无凭据/超限时的人工上传回退；Secret Key 不用于只读公开能力也不进入浏览器。
- [ ] 盘点所有运行时写目录和写文件行为：`caches/`、`uploads/`、`data/*.lock`、数据库备份、配置写回、模板下载和在线升级。
- [ ] 盘点定时任务、反向代理头、HTTPS 终止、域名泛解析、邮件、对象存储及外部服务的生产配置来源和超时/重试策略。
- [ ] 盘点所有数据库连接来源与运行模式，列出 host、port、database、username、password、prefix、charset、collation、timezone、SSL CA/cert/key；删除前确认旧 PHP 配置中的每个硬编码值都有 `.env` 键、示例占位和部署密钥映射。

**验收：** 路由清单与真实访问日志抽样一致；外部依赖和运行时写入不存在未归属项。

---

## 阶段 1：先建立 PHPUnit 与 Mago 基线

### 1.1 用 Composer 安装测试和质量工具

**修改：** `composer.json`、`composer.lock`、`.gitignore`。

- [ ] 将根包的 PHP 约束设为 `^8.4`，声明上述 `ext-*` 运行要求，并配置 `app\\`、`tests\\` 的 PSR-4 自动加载。
- [ ] 自动执行 `composer require --dev "phpunit/phpunit:^13.1" "carthage-software/mago:^1.43" --with-all-dependencies`，提交生成的 `composer.lock`，禁止下载 PHAR 或依赖全局工具。
- [ ] 执行 `composer install --no-interaction --prefer-dist`、`vendor/bin/phpunit --version`、`vendor/bin/mago --version`，确认干净环境可重复安装。
- [ ] 在 Composer scripts 中新增 `test:unit`、`test:integration`、`test:feature`、`test:contract`、`test`、`test:coverage`、`qa:syntax`、`qa:lint`、`qa:analyze`、`qa:format` 和聚合命令 `qa`。
- [ ] 把 `vendor/`、`runtime/`、覆盖率文件、测试缓存、测试数据库转储和本地 `.env` 加入 `.gitignore`，但保留示例配置与脱敏 fixtures。
- [ ] 执行 `composer validate --strict`、`composer check-platform-reqs`、`composer audit --locked`，处理全部错误与安全公告后再提交。

**验收命令：** `composer install && composer validate --strict && composer check-platform-reqs && composer audit --locked`，全部退出 0。

### 1.2 建立根级 PHPUnit 13 测试框架

**创建：** `phpunit.xml`、`tests/bootstrap.php`、`tests/Support/`、`.env.testing.example`。

- [ ] 配置 `unit`、`integration`、`feature`、`contract` 四个 testsuite，覆盖率只统计第一方 `app/`、`config/` 和后续新增的迁移代码。
- [ ] 测试引导强制 `APP_ENV=testing`、`APP_DEBUG=false`，拒绝生产域名、生产数据库名、非测试 Redis DB 和真实云凭据；提供两个显式测试站点及独立 Host/Cookie/cache 前缀。
- [ ] 实现数据库重建、事务隔离、固定时钟、固定随机值、文件系统临时目录和 HTTP 客户端 fake 的测试支持类。
- [ ] 将 fixture 分为 legacy 与 candidate：legacy 脱敏快照保留当前 24 表只用于 TP5 对照和地区删除审计；candidate 完全由 migration/seeder 建立，覆盖管理员/角色、栏目树、四类 DIY 内容模型、自定义表单、附件、链接、Banner 和站点配置且不创建 `yunu_area` 或地区字段。
- [ ] 增加测试环境防误操作用例：数据库名不含 `_test`、上传根目录不在临时路径、外部主机未被 fake 时立即失败；真实 Unsplash/邮件/云服务凭据出现在测试进程时直接拒绝启动。
- [ ] 运行 `vendor/bin/phpunit --testsuite unit`，确保测试框架本身可启动且没有加载 `system/phpunit.xml` 的旧 PHPUnit 4 配置。

**验收：** 新机器只需 Composer 和测试数据库即可运行测试；测试不会写入真实配置、真实上传目录或外部服务。

### 1.3 初始化 Mago 并锁定质量规则

**创建：** `mago.toml`、`lint-baseline.toml`、`analysis-baseline.toml`。

- [ ] 执行 `vendor/bin/mago init`，在 `mago.toml` 中锁定 `version = "1.43"`、`php-version = "8.4"`，schema 指向 Composer 包内的 `vendor/carthage-software/mago/schema.json`。
- [ ] 将 `app/`、`config/`、`tests/` 列为第一方源码，将 `vendor/` 列为 includes；排除 `system/`、旧 `app/extend/` 第三方副本、旧 `statics/`、新 `public/static/`、`runtime/`、`public/uploads/` 和生成文件。
- [ ] 执行 `vendor/bin/mago list-files` 并人工抽查，确保业务 PHP 和测试被包含，第三方、缓存、上传文件未被格式化或分析。
- [ ] 对旧代码先执行 `vendor/bin/mago lint --semantics`，修复 PHP 8.4 下会导致解析/运行失败的问题，不使用 baseline 隐藏语法或语义错误。
- [ ] 执行 `vendor/bin/mago lint --generate-baseline --baseline lint-baseline.toml` 与 `vendor/bin/mago analyze --generate-baseline --baseline analysis-baseline.toml`，把遗留问题与新增问题分开。
- [ ] 配置 baseline 后执行 `vendor/bin/mago lint` 和 `vendor/bin/mago analyze`，要求退出 0；迁移过程中只允许删除或修复 baseline 项，禁止重新生成 baseline 吞掉新增问题。
- [ ] 执行 `vendor/bin/mago format --dry-run` 审核影响范围；格式化修改必须放在独立提交，CI 使用 `vendor/bin/mago format --check`。

**验收：** Mago 能稳定扫描相同文件集；语义检查零错误；遗留基线可审计且后续新增问题会使 CI 失败。

### 1.4 建立 GitHub Actions 基线流水线

**创建：** `.github/workflows/ci.yml`。

- [ ] 配置 PHP 8.4、Composer 缓存、MySQL 8.4 和覆盖率驱动；CI 密钥只使用测试值。
- [ ] 使用 `composer install --no-interaction --prefer-dist` 从锁文件安装，不在 CI 中执行无约束 `composer update`。
- [ ] 分开执行 Mago format、semantic lint、lint、analyze，使一个工具失败时其他诊断仍可见；首次 Composer Mago 下载显式传入只读 `GITHUB_TOKEN`。
- [ ] 分开执行 PHPUnit unit、integration、feature、contract 和 coverage，保存 JUnit、Clover、失败页面/响应及应用日志为构建产物。
- [ ] 增加 `composer validate --strict`、`composer check-platform-reqs`、`composer audit --locked` 供应链门禁。
- [ ] 将 CI 设为合并升级分支的必需检查，禁止跳过失败检查直接合并。

**验收：** 在只有代码和 GitHub Secrets 的全新 runner 上可重复通过，不依赖开发机全局 PHP 工具。

---

## 阶段 2：在 TP5 上完成全量特征与回归测试

### 2.1 单元测试：公共函数与纯业务规则

**创建：** `tests/Unit/Common/`、`tests/Unit/Validation/`、`tests/Unit/Security/`。

- [ ] 覆盖字符串截断、URL 参数构造、桌面/WAP URL 转换、标签/面包屑、拼音、文件大小和数组递归处理函数的正常值、空值、中文、边界值；用 fixture 锁定旧 `get_pinyin()` 的实际输入输出，作为 `overtrue/pinyin` 迁移的 RED 基线而非保留旧算法的理由。
- [ ] 覆盖栏目树、父子 ID 展开、内容前后篇、动态模型表名和表单字段映射的纯业务规则；删除地区树、地区占位符和地区 URL 单元测试。
- [ ] 覆盖所有 `app/common/validate/` 规则的成功、必填缺失、类型错误、长度边界和恶意输入。
- [ ] 覆盖登录密码校验、权限 URL 标准化、回调名、文件名、扩展名、MIME、域名和重定向地址验证。
- [ ] 为时间、随机数、HTTP_HOST、客户端 IP 和移动端识别建立可注入测试替身，禁止依赖当前机器状态。
- [ ] 执行 `vendor/bin/phpunit --testsuite unit`，记录 TP5 基线用例数、断言数和覆盖率。

**验收：** 公共纯逻辑无需真实数据库或网络即可稳定复现，失败信息能定位到具体业务规则。

### 2.2 集成测试：数据库与动态表一致性

**创建：** `tests/Integration/Database/`、`tests/Integration/Model/`、`tests/Integration/Storage/`。

- [ ] 验证 `data/install.sql` 能在 MySQL 8.4 严格模式下全新安装，所有表、索引、默认值和初始管理员记录符合预期。
- [ ] 覆盖栏目、内容、Banner、区块、链接、站内链接、用户、角色、菜单、日志的增删改查与分页排序；候选 schema 明确不存在 `yunu_area`。
- [ ] 覆盖 `content` 与 `diy_<模型表名>` 的创建、更新、复制、移动、删除事务，任一侧失败时不得留下孤儿数据。
- [ ] 覆盖 `diyform`、`diyfield`、`formcon` 与 `form_<表名>` 的建表、提交、编辑、删除和字段演进。
- [ ] 在 TP5 快照上统计 `content.area`、`link.area`、`category.isarea`、`sitelink.areapre`、地区 SEO 配置和地区占位符的非默认数据量，生成删除前审计与备份校验，不把这些字段带入新 fixture。
- [ ] 覆盖数据库备份分卷、恢复、表前缀替换和字符集，所有操作只针对临时测试库。
- [ ] 覆盖本地上传、缓存、会话和锁文件的创建、读取、清理、并发冲突及权限错误。
- [ ] 执行 `vendor/bin/phpunit --testsuite integration`，每次运行前重建 fixture，运行后确认无残留表、文件和连接。

**验收：** 动态表和主表在成功/失败路径都保持一致；测试可重复运行且数据结果确定。

### 2.3 前台与 WAP HTTP 回归

**创建：** `tests/Feature/Index/`、`tests/Feature/Wap/`、`tests/Contract/Html/`、`tests/Contract/Route/`。

- [ ] 覆盖首页、引导页、栏目列表、多级栏目、内容详情、上一篇/下一篇、搜索、标签、留言/自定义表单、验证码和 404。
- [ ] 收集桌面/WAP 地区查询参数、路径和独立子域名的旧响应作为下线输入；候选版本验证无地区状态写入 session/config，一个发布周期内返回 410，随后删除兼容路由/DNS 并返回 404。
- [ ] 覆盖移动端自动跳转、`/m/` 路径、移动子域名和 rewrite 模式；旧动态 URL 只验证按既定 301/308 策略跳转到规范 rewrite URL。
- [ ] 对 `site-matrix.md` 中每个主域名/别名运行相同公开路由矩阵，断言 SiteContext、canonical、主题和 index/WAP 通道正确；未知 Host 为 421、停用站点为 503，两个测试站点的 Cookie/session/cache/绝对 URL 不串用。
- [ ] 建立 `NginxRewriteParity` 数据集：`nginx.txt` 每条业务规则至少一个正例、边界例和反例，固定请求路径、方法、控制器语义、解析参数及 TP8 命名路由；恶意脚本后缀和 UEditor 静态文件规则单独验证。
- [ ] 为每个公开命名路由执行双向契约：根据参数生成规范 rewrite URL，再请求该 URL 并断言还原同一业务参数；生成结果不得包含 `index.php`、重复编码或非规范尾斜杠。
- [ ] 覆盖路由优先级与碰撞：`search`、`tag`、`myform`、`captcha`、`page`、`admin`、`m`、普通栏目、多级栏目和 `.html` 内容互不吞并，地区退役路径不会落入普通栏目/内容动作。
- [ ] 覆盖 `<page>`/`<id>` 的零、负数、小数、超大值，路径穿越、双重 URL 编码、未知后缀、保留字栏目、中文标签、空标签和不存在的栏目/内容。
- [ ] 使用同一契约套件分别经 PHP 内置服务器、Apache 通用前端控制器规则和 Nginx `try_files` 访问，证明分发结果由应用路由决定；差异必须失败而非维护三份预期。
- [ ] 对关键页面保存规范化 HTML 快照：移除时间、随机值和 CSRF 等动态字段后比较 DOM、标题、meta、canonical、导航、链接和结构化数据。
- [ ] 为每个旧主题模板记录逻辑模板名、物理文件、调用入口和 HTML 快照；TP8 目录迁移后用新 `/` 路径渲染并比较同一快照，确保变化只来自已批准的地区区块删除。
- [ ] 对所有保留的主题标签生成结果建立 fixture，覆盖 `Yunu` 标签库中的列表、栏目、内容、Banner、区块、链接和表单标签；模板中出现 `<yunu:area>` 时测试先失败，删除标签后全部主题重新编译通过。
- [ ] 校验静态资源 URL、图片补全、分页 URL、中文 URL 编码、301/302 跳转及 Apache/Nginx 等价行为。
- [ ] 执行 `vendor/bin/phpunit --testsuite feature --filter 'Index|Wap'`，TP5 基线必须全部通过。

**验收：** TP8 候选版本后续必须与同一套状态码、跳转、DOM/SEO 和数据库副作用契约一致。

### 2.4 后台与权限完整回归

**创建：** `tests/Feature/Admin/`、`tests/Integration/Auth/`。

- [ ] 覆盖登录成功、密码错误、账号不存在、禁用账号、验证码、退出、会话过期和登录次数/IP/时间更新。
- [ ] 使用超级管理员、只读角色、内容编辑角色和无权限用户覆盖 RBAC；每个受保护动作都验证允许与拒绝路径。
- [ ] 覆盖用户、角色、菜单、栏目、内容、模型、字段、表单、Banner、区块、链接、站内链接和日志的核心 CRUD，并断言后台地区菜单/动作均不可达。
- [ ] 覆盖排序、批量状态、批量移动、批量 SEO、复制内容和动态字段保存；后台请求携带旧 `area`/`isarea`/`areapre` 字段时拒绝或忽略且不写库。
- [ ] 覆盖系统基础配置、SEO、移动端、上传、七牛、禁用词、站点地图和接口配置的读写与失败回滚。
- [ ] 覆盖上传浏览、文件/图片上传、非法扩展、伪造 MIME、双扩展名、路径穿越、超限文件和未授权上传。
- [ ] 覆盖内容/栏目编辑页的 Unsplash 配图入口：权限、搜索/随机候选、方向与安全筛选、摄影师/Unsplash 署名、人工选择、取消、无结果、禁用/无凭据/限流回退，以及保存后前台/API 的图片和署名契约。
- [ ] 覆盖数据库备份/恢复和缓存清理；在线升级只使用签名测试包和本地 fake 服务，禁止访问真实升级服务器。
- [ ] 执行 `vendor/bin/phpunit --testsuite feature --filter Admin`，保存失败响应和权限矩阵报告。

**验收：** 后台每个关键写操作均有认证、授权、成功、验证失败和存储失败用例。

### 2.5 API 契约完整回归

**创建：** `tests/Feature/Api/`、`tests/Contract/Api/`。

- [ ] 覆盖 `Master` 与 `V1` 保留的 `config`、`list`、`listmip`、`link`、`banner`、`block`、`catlist`、`nav`、`type`、`position`、`url`、`cwkeywords`、`content`、`form`；`area` 明确列为移除接口。
- [ ] 每个保留接口覆盖成功、空数据、缺失参数、非法参数、边界分页和不存在方法；旧地区筛选参数不再改变结果，`area` 接口在退役窗口返回稳定下线响应并最终 404。
- [ ] 固化 JSON 字段、类型、状态值、日期格式、URL、空数组/空对象和 HTTP Content-Type；禁止仅比较“请求成功”。
- [ ] 覆盖 JSONP 合法 callback、缺失 callback 和恶意 callback；记录真实消费者，未完成下线通知前保持兼容。
- [ ] 覆盖 CORS 预检、允许来源/方法/头和拒绝来源；TP8 不得继续无条件扩大跨域权限。
- [ ] 覆盖 API 表单写入、重复提交、限流、验证失败和数据库回滚。
- [ ] 执行 `vendor/bin/phpunit --testsuite contract --filter Api`，输出 TP5 响应快照供 TP8 比对。

**验收：** 所有公开 API 都有机器可比较的契约，TP8 差异必须经过明确审批而非静默变化。

### 2.6 安装、升级与外部集成回归

**创建：** `tests/Contract/Install/`、`tests/Feature/Operations/`、`tests/Integration/External/`。

- [ ] 固化旧 `Install` 控制器的环境检查、数据库连接/建库、SQL 导入、配置写回、默认管理员和安装锁副作用，作为 CLI 安装器的数据结果基线，而不是继续保留 Web 页面契约。
- [ ] 记录 `/index.php?s=index/install/*` 等全部 Web 安装入口及未安装自动跳转；TP8 契约明确这些入口下线并返回 404，未安装实例只允许健康检查与 CLI 安装命令。
- [ ] 覆盖邮件、七牛、百度 AIP、云授权、排名/推送和 Unsplash 的成功、超时、DNS/TLS 错误、非 2xx、畸形 JSON、限流与重试上限；全部通过受控 fake，不访问真实服务。
- [ ] 为 Unsplash 固化真实协议 fixture：搜索/随机响应、`photo.urls.*`、`user`、`links.html`、带原查询串的 `links.download_location`、`X-Ratelimit-*` 与错误数组；选择图片时断言只调用返回的 download tracking URL，不把它误作图片地址。
- [ ] 覆盖升级包下载、哈希/签名失败、Zip Slip、磁盘不足、文件权限失败、中途失败和回滚。
- [ ] 覆盖配置并发写入、特殊字符、引号/换行、空值、敏感字段脱敏和原子替换，禁止生成不可解析 PHP。
- [ ] 执行完整 `vendor/bin/phpunit`，生成 TP5 基线 JUnit、覆盖率和契约快照并存档。

**验收：** 在开始 TP8 骨架改造前，TP5 全量套件必须稳定连续通过三次。

---

## 阶段 3：建立 Composer 化 ThinkPHP 8 多应用骨架

### 3.1 自动安装 TP8 生产依赖

**修改：** `composer.json`、`composer.lock`。

- [ ] 执行 `composer require "topthink/framework:^8.1" "topthink/think-multi-app:^1.1" "topthink/think-view:^2.0" "topthink/think-filesystem:^3.0" "topthink/think-migration:^3.1" --with-all-dependencies`。
- [ ] 自动执行 `composer require "guzzlehttp/guzzle:^7.13" "overtrue/pinyin:^6.0" --with-all-dependencies`，由同一 `composer.lock` 固定实际版本；不得下载源码包到 `app/extend/` 或要求人工复制类文件。
- [ ] 以“PHP 8.4 兼容、维护活跃、许可证可接受、无未处理高危公告、功能不重叠”为直接依赖准入条件；标准库/扩展可可靠完成的 ZIP、哈希、JSON、文件与随机数能力不额外引包。
- [ ] Unsplash 使用已安装的 Guzzle 调用官方 v1 JSON API；评审 `unsplash/unsplash` 当前维护状态后若不能满足生产准入门禁则不安装，禁止为了一个 API 再叠加陈旧 SDK、OAuth 客户端和第二套 HTTP 栈。
- [ ] 使用 `composer show --direct` 审核直接依赖，使用 `composer why-not topthink/framework ^8.1` 排除冲突，禁止同时加载仓库内 `system/` 的 `think\\` 类。
- [ ] 固定 Composer scripts 的 `post-autoload-dump` 服务发现/发布流程，执行 `composer dump-autoload -o` 并验证 `app\\`、`tests\\` 和 `extend\\` 命名空间。
- [ ] 执行 `composer audit --locked`；存在当前可利用安全公告时不得以 ignore 绕过生产门禁。
- [ ] 从空 `vendor/` 执行两次 `composer install`，确认锁文件解析一致且没有未记录的交互步骤。

**验收：** `php think --version` 显示 ThinkPHP 8.1.x，框架类只来自 `vendor/topthink/`。

### 3.2 按 ThinkPHP 8 官方骨架标准化目录与多应用配置

**创建：** `public/index.php`、`public/router.php`、`think`、`config/app.php`、`config/route.php`、各应用 `config/` 与 `route/`。

- [ ] 按官方 TP8 项目骨架创建 Composer HTTP/CLI 入口，Web 文档根目录固定为 `public/`，入口不再定义 TP5 的 `APP_PATH`/`CONF_PATH` 或加载 `system/start.php`。
- [ ] 以锁定版本的 `topthink/think` 8.x 骨架为目录基线，建立根级 `app/`、`config/`、`extend/`、`public/`、`route/`、`runtime/`、`vendor/`、`view/`、`think`、`.example.env`；`database/`、`deploy/`、`docs/`、`tests/` 作为本项目明确扩展目录。
- [ ] 建立并评审逐项移动表：根 `index.php`→`public/index.php`、`router.php`→`public/router.php`、`yunu.php`→`think`/Command、`config/<应用>`→`app/<应用>/config`、`template/`→应用 `view/`、`statics/`→`public/static/`、`uploads/`→`public/uploads/`、第一方 `app/extend`→`app/common`、第三方副本→Composer；每项包含引用更新、删除时点和回滚方式。
- [ ] 保留并适配官方 `app/AppService.php`、`BaseController.php`、`ExceptionHandle.php`、`Request.php`、`common.php`、`event.php`、`middleware.php`、`provider.php`、`service.php` 扩展点；禁止另建功能重叠的 bootstrap、kernel 或自定义框架目录。
- [ ] 启用官方 `think-multi-app`，注册 `index`、`wap`、`admin`、`api` 四个应用，默认应用为 `index`，禁止直接访问内部公共应用。
- [ ] 每个应用按需使用 `controller/`、`model/`、`validate/`、`view/`、`config/`、`route/`、`middleware.php`、`provider.php`；不创建空的 repository/interface/domain 分层，跨应用复用代码放 `app/common/` 并把 `common` 加入禁止直接访问应用清单。
- [ ] 将旧 `config/index|wap|admin|api/` 迁到对应 `app/<应用>/config/`，业务 route 迁到 `app/<应用>/route/app.php`；根 `config/` 和 `route/app.php` 只保留全局配置、全局 middleware 与健康检查等框架级入口。
- [ ] 将旧 `template/` 迁到 `app/index/view/`、`app/wap/view/`，旧 `app/admin/view/` 保持多应用标准位置；公共错误页放 `view/error/`，模板 `/` 分隔规则按阶段 7.1 执行。
- [ ] 将旧 `statics/` 和主题公开资源迁到 `public/static/admin/`、`public/static/themes/`，上传挂载到 `public/uploads/`；旧 `router.php`、入口和 Web 配置只在 `public/` 保留标准版本。
- [ ] 将第一方类从 `app/extend/` 迁入 `app/common/` PSR-4 命名空间，第三方库由 Composer 进入 `vendor/`；`extend/` 仅保留有明确理由且有测试的非 Composer 扩展，不复制 vendor 包。
- [ ] 统一 Linux 大小写和 PSR-4：目录名小写、类名/文件名按 Composer 规则匹配，禁止通过 classmap 掩盖错误路径；执行 `composer dump-autoload -o --strict-psr`。
- [ ] 不注册 `Install` Web 控制器或安装路由；未安装状态返回不泄露配置的 503，运维人员只能通过 `php think yunu:install` 完成安装。
- [ ] 在 TP8 路由配置中只启用 rewrite 目标路由，所有 URL 通过命名路由生成；应用业务代码不得根据多种 `url_model` 分支继续生成动态地址。
- [ ] 将旧 `url_model` 配置迁移为明确的 rewrite 配置/特性开关；迁移完成后不再依赖数值 `1/2/3` 在运行期切换 URL 生成算法。
- [ ] 将全局配置拆分为 `app.php`、`database.php`、`cache.php`、`cookie.php`、`filesystem.php`、`log.php`、`route.php`、`session.php`、`view.php`。
- [ ] 将旧 `config/index|wap|admin|api/config.php` 映射为各应用配置，保留各自模板、返回类型、过滤器和 URL 后缀语义。
- [ ] 使用 `.env` / 环境变量承载数据库、缓存、邮件、对象存储、域名和密钥；按官方骨架提交 `.example.env`，测试专用覆盖使用 `.env.testing.example`，不提交真实值。
- [ ] `config/database.php` 仅读取并校验 `DB_HOST`、`DB_PORT`、`DB_DATABASE`、`DB_USERNAME`、`DB_PASSWORD`、`DB_PREFIX`、`DB_CHARSET`、`DB_COLLATION`、`DB_TIMEZONE` 及 SSL 键，完成必要的整数/布尔转换；除空 secret 占位外不写连接默认值，生产缺失任一必需键时启动/ready/install 立即失败。
- [ ] 在 `app/service.php` 或 `provider.php` 只注册一个共享 Guzzle 客户端配置和最小外部集成；禁止创建 HTTP interface/factory/manager 多层包装或为每个调用方各实例化客户端。
- [ ] 新增 `/health/live` 与 `/health/ready`：live 只检查进程，ready 检查数据库和关键可写目录，不泄露版本、凭据或堆栈。
- [ ] 新增 `ProjectStructureTest`：校验官方/项目扩展目录白名单、标准入口和应用加载文件存在，拒绝 `system/`、根 `index.php`/`router.php`、`template/`、`statics/`、`resources/views/`、旧模块配置目录及公网可见源码。
- [ ] 先运行只验证骨架的 PHPUnit 启动/目录测试与 Mago 全套检查，再迁移业务控制器。

**验收：** 目录与官方 TP8 骨架及多应用加载机制一致，四应用能被正确识别；旧目录清零且 Composer strict PSR 通过；健康检查可用于部署探针，旧业务尚未迁移时以明确 503/维护响应失败。

### 3.3 内化 rewrite 路由并收紧 Web 根目录

**修改/移动：** `.htaccess`、`nginx.txt`、`router.php`、`statics/`、`template/`、`uploads/`、`404.htm`、`error.htm`。

- [ ] `public/.htaccess`、`public/router.php` 与 `deploy/nginx/yunucms.conf` 只负责真实静态文件优先和其余请求转发 `public/index.php`；不得出现 area、category、show、tag、myform、wap、admin 等业务匹配。
- [ ] Nginx `root` 固定为 `public/`，`location /` 使用通用 `try_files $uri $uri/ /index.php?$query_string` 语义，只允许精确 `public/index.php` 进入 PHP-FPM，拒绝上传目录和其他路径执行 PHP，并通过 `nginx -t`。
- [ ] 在四个应用的 `route/app.php` 按本文矩阵注册显式命名路由和允许的 HTTP 方法：页面/验证码通常只读，表单提交只接受 POST，后台动作按清单限定；错误方法稳定返回 405。
- [ ] 依次注册固定端点、普通内容、普通分页、普通栏目和最终 404，使用路由约束与自动化测试证明宽路由不会遮蔽窄路由；地区路径只允许命中临时退役响应，不进入业务控制器。
- [ ] 对多级 `<categoryPath>` 使用 TP8 受约束路由变量/路由组并交给固定栏目或内容解析动作；支持旧 URL 所需的层级，但路径值绝不能决定 PHP 类、方法、表名或原生 SQL。
- [ ] 把旧 `/admin/(.*)` 展开为后台显式命名路由及中间件白名单；未盘点动作和旧 Web Install 动作返回 404，不允许自动路由兜底。
- [ ] 将 `nginx.txt` 第 1–8 行的危险后缀行为落实为应用 404/安全测试，并在 Web 服务器保留通用脚本执行防护；不存在的脚本式 URL不得暴露框架调试信息。
- [ ] 旧 `/statics/ueditor/dialogs/.../*.html` 迁到 `/static/admin/ueditor/dialogs/.../*.html` 真实静态路径，删除无效果的自我 rewrite；静态文件不存在时由应用 404，不创建 UEditor 业务控制器。
- [ ] 将后台公共静态资源移动到 `public/static/admin/`，上传目录移动到/挂载为 `public/uploads/`，禁止 PHP 执行和目录索引。
- [ ] 将主题模板与静态资源拆分为 `app/index|wap/view/<theme>/` 和 `public/static/themes/<theme>/<app>/`，更新 `__PUBLIC__` 等替换变量；不得创建非官方的 `resources/views/`。
- [ ] 将 404/异常页面放入非敏感模板目录，生产环境关闭调试和堆栈输出。
- [ ] 为旧 `/index.php?s=...`、`/index.php/<app>/...` 业务地址在应用层建立迁移期 301/308 到规范 rewrite URL 的白名单策略，不在 Nginx/Apache、页面、API 或 sitemap 中继续维护动态地址；旧 Install 地址明确排除并返回 404。
- [ ] 生成并审查 `php think route:list` 路由清单，逐项对照 `route-inventory.md`，检测缺失、重复名称、不可达路由、隐式自动路由和优先级遮蔽。
- [ ] 路由契约全量通过后删除根目录 `nginx.txt` 或移入只读升级证据目录；CI 检查生产 Nginx/Apache 配置不再含业务 rewrite 正则。
- [ ] 对静态文件缓存头、上传文件 Content-Type、CSP、X-Content-Type-Options、Referrer-Policy 和 HTTPS 重定向做生产验证。

**验收：** 只有 `public/` 可从 Web 访问；业务 URL 只由 TP8 路由定义；替换 Nginx/Apache 为任一通用前端控制器后契约仍通过；`config/`、`vendor/`、`app/`、`view/`、`database/`、测试和备份文件均无法通过 HTTP 下载。

---

## 阶段 4：迁移公共生命周期、配置与安全边界

### 4.1 用 TP8 生命周期替代 TP5 `_initialize`

**修改：** `app/index/controller/Common.php`、`app/wap/controller/Common.php`、`app/admin/controller/Common.php`、相关中间件配置。

- [ ] 先让现有生命周期回归测试在 TP8 上失败，确认失败来自 `_initialize`、控制器基类或请求 API 差异。
- [ ] 将安装状态、移动端跳转、后台认证授权分别迁移为职责单一的 TP8 中间件；删除地区/分站域名解析、`sys_area`/`sys_areainfo` session 与请求期配置污染。
- [ ] 将视图公共变量和菜单/位置信息保留在控制器公共层，不让认证中间件承担模板渲染职责。
- [ ] 替换 `request()->module()/controller()/action()`、TP5 redirect/error 跳转和隐式控制器响应，使用 TP8 Request/Response/异常接口。
- [ ] 覆盖中间件短路、异常传播、404、未安装、未登录、无权限和移动端跳转的优先级。
- [ ] 运行生命周期相关 PHPUnit、完整回归及 Mago 检查。

**验收：** 不再存在业务 `_initialize`；每个横切逻辑都有独立测试和明确执行顺序。

### 4.2 用 Middleware 实现多站点识别与请求上下文

**创建/修改：** `app/common/middleware/ResolveSite.php`、`app/common/site/SiteContext.php`、`app/middleware.php`、`config/sites.php`、`config/middleware.php`、URL/模板/缓存/会话配置及对应测试。

- [ ] 固定本轮边界：多个显式 Host/别名映射到站点上下文，共享同一业务数据库和内容；站点差异仅含 canonical URL、主题、桌面/WAP 通道、配置覆盖、Cookie 名和缓存前缀，不新增站点后台 CRUD、站点表或业务表 `site_id`。
- [ ] 定义只读 `SiteContext` 字段：`key`、`name`、`canonicalUrl`、`theme`、`channel`、`cookieName`、`cachePrefix`；值只能来自验证后的 `config/sites.php`，业务代码不得在请求过程中修改。
- [ ] `config/sites.php` 使用稳定 site key 和显式 ASCII/punycode Host 映射，启动时校验 key/Host/缓存前缀唯一、canonical Host 已登记、主题目录存在、URL 仅为 http/https；禁止 `*` 泛域名、正则猜测和从 `yunu_area` 派生站点。
- [ ] 先写 `ResolveSiteMiddlewareTest` 的失败用例，再实现 middleware：从 TP8 Request 获取已验证 Host，小写化、去端口和单个尾点后精确匹配站点；未知 Host 返回 421，停用站点返回 503，不回退默认站点。
- [ ] 只有来源地址在可信代理白名单时才接受框架解析后的 forwarded host/proto；覆盖伪造 `Host`/`X-Forwarded-Host`、多个转发值、CRLF、Unicode Host、IP Host、大小写、端口和尾点测试。
- [ ] 在 `app/middleware.php` 全局注册 `ResolveSite`，顺序位于可信代理解析之后、session/cookie/cache/主题选择和业务控制器之前；index、WAP、admin、API 共用同一实现，不在各应用复制识别逻辑。
- [ ] 将 `SiteContext` 绑定为请求级对象并附加到 Request，控制器、服务、URL 生成器和视图只读取注入上下文；禁止调用全局 `config()` 写入当前站点，防止常驻进程跨请求串站。
- [ ] 用 context 的 `channel` 选择 index/WAP 应用表现，用 `theme` 选择 `app/<应用>/view/<theme>/`，用 `canonicalUrl` 生成命名路由绝对地址；Host 不得动态决定控制器、模板任意路径或路由文件。
- [ ] 删除 Common 控制器中的 `HTTP_HOST`、`site_url`、`site_levelurl`、`wap_levelurl` 和地区子域名判断；移动端路径/域名只作为已配置站点通道，地区前缀不得被识别为站点。
- [ ] Cookie 名/domain、session namespace、应用缓存/模板缓存 key、限流 key 和日志字段全部包含 site key；两个站点相同路径/用户/缓存键不得互相读取或覆盖。
- [ ] CLI/队列没有 HTTP Host 时不得隐式取第一个站点：需要站点语义的 Command 强制 `--site=<key>`，全站任务显式遍历已启用站点；`yunu:install --check` 验证完整站点注册表。
- [ ] PHPUnit 覆盖主域名、别名、桌面/WAP 通道、canonical、未知/停用 Host、可信/不可信代理、Cookie/cache 隔离、连续不同站点请求无上下文泄漏，以及四应用获得同一 SiteContext。
- [ ] 增加源码门禁，排除第三方后第一方代码不得直接出现 `$_SERVER['HTTP_HOST']`、自行解析 `X-Forwarded-Host` 或按 Host 修改全局配置；运行定向 PHPUnit、完整回归和 Mago。

**验收：** 多站点识别只有一个全局 middleware 入口；站点上下文请求隔离且覆盖四应用、模板、URL、Cookie、缓存和日志；未知 Host、代理欺骗与跨站污染测试全部通过，功能与 `yunu_area` 无依赖。

### 4.3 迁移公共助手、配置与自动加载

**修改：** `app/common.php`、`app/*/common.php`、`app/extend/com/`、`app/extend/org/`、Composer autoload。

- [ ] 盘点 `input()`、`config()`、`db()`、`session()`、`url()`、`json()`、`jsonp()` 等助手在 TP8 的签名和返回差异，逐一以测试锁定后迁移。
- [ ] 清点 `app/common.php`、`app/*/common.php` 及保留的第一方 helper 中全部全局用户函数，记录名称、签名、调用方、加载顺序和所属应用；函数名按 PHP 大小写不敏感规则检查冲突。
- [ ] 对 `sitelink`、`getHomeurl`、`getSearchurl`、`getPosition`、`get_wapurl`、`setConfigfile`、`mysqlupdate` 等同名函数先比较语义：完全相同的合并为一个实现，不同语义的迁移为应用命名空间类/方法或改成唯一函数名，禁止用加载先后静默选择错误实现。
- [ ] 所有迁移后仍保留的全局用户函数统一使用 `if (!function_exists('函数名')) { function 函数名(...) { ... } }` 顶层保护，字符串名称与实际声明完全一致；不得用 `@`、条件 include 或关闭错误报告掩盖冲突。
- [ ] 安装页专用的 `check_env`、`check_dirfile`、`mysqlupdate`、`check_func`、`show_msg` 随 Web Install 删除；配置源码写回、旧 SQL 解析和在线升级专用函数按对应下线任务删除，不为无调用死代码增加长期兼容层。
- [ ] Composer files 自动加载顺序固定且只加载预期 helper；业务/安全关键逻辑优先使用命名空间类，避免第三方先声明同名全局函数后劫持实现。
- [ ] 新增隔离进程 PHPUnit：每个 helper 连续加载两次、以不同应用顺序组合加载，断言无 redeclare fatal、函数来源/返回语义正确且参数默认值不变。
- [ ] 新增源码守卫测试，使用 PHP tokenizer 检测第一方全局函数是否位于匹配的 `function_exists` 否定条件内；未保护的新函数使 `composer qa` 失败，类方法和闭包不得误报。
- [ ] 将有状态、涉及 I/O 或外部网络的全局函数迁移为可注入服务；纯函数保留为 Composer files 自动加载，避免为了形式创建单实现接口。
- [ ] 先以 characterization test 覆盖 `send_post()`、`url_get_contents()`、`get_headers()` 和 `com\Http` 的真实调用方，再逐个改用共享 Guzzle 客户端/官方 SDK；调用方迁移完成后删除这些全局函数和第一方手写 HTTP 包装，不保留同名兼容代理。
- [ ] 先让 `PinyinSluggerTest` 针对中文、ASCII、多音字、标点、emoji、空值及同栏目重名 slug 失败，再用 `overtrue/pinyin` 的无音调结果实现小写、连字符、去空白/标点和数据库唯一性冲突后缀规则；迁移 CategoryModel 后删除 `get_pinyin()`、`statics/pinyin.dat` 及 GB2312/iconv 自制算法。
- [ ] 将第一方 `com`/`org` 类迁移到 `app\\common` PSR-4 命名空间，修复大小写和 Linux 文件系统兼容问题。
- [ ] 替换直接 `$_SERVER` 读取为 Request 或配置注入，代理头只信任明确配置的反向代理。
- [ ] 把配置读取统一为 TP8 点号路径，移除运行期全局修改造成的请求间污染。
- [ ] 运行公共函数单元测试、所有应用启动测试、Mago analyze 与 Composer autoload 校验。

**验收：** 第一方代码不依赖 TP5 `system/helper.php`；所有保留的全局用户函数均有 `function_exists` 保护且重复/乱序加载无冲突；Mago 能解析所有第一方符号和返回类型。

### 4.4 升级认证、会话、CSRF 与 API 边界

**修改：** 后台登录/RBAC、`config/session.php`、`config/cookie.php`、API 中间件与数据库迁移。

- [ ] 为旧双重 MD5 密码实现一次性兼容校验：旧密码登录成功后立即升级为 `password_hash()`，数据库字段扩展到 255，失败不泄露账号存在性。
- [ ] 配置会话 ID 轮换、HttpOnly、Secure、SameSite、固定过期时间和退出失效；覆盖会话固定攻击测试。
- [ ] 为后台和前台写操作启用 CSRF；API 使用明确的认证/签名策略，不以关闭 CSRF 代替认证。
- [ ] 将后台免认证列表改为显式路由/中间件白名单，逐条测试；上传、批量修改和缓存清理不得匿名访问。
- [ ] 将 API CORS 从 `*` 改为配置化来源、方法与请求头白名单；预检与拒绝路径纳入回归。
- [ ] 对 JSONP 建立兼容开关和 callback 格式白名单，完成消费者迁移计划后再单独下线。
- [ ] 运行 Security、Admin Auth、API Contract 测试和完整回归。

**验收：** 旧用户无需重置密码即可平滑升级；新密码不再使用 MD5；认证、授权、CSRF、CORS 均有拒绝路径测试。

---

## 阶段 5：迁移数据库、模型与结构变更机制

### 5.1 将 `data/install.sql` 完整改造为 TP8 Migration/Seeder

**创建/删除：** 创建 `config/database.php`、`database/migrations/`、`database/seeders/`、`database/seeders/data/`；等价验证通过后删除 `data/install.sql`。

- [ ] 使用 PDO MySQL；host、port、database、username、password、prefix、charset、collation、timezone、SSL 和连接超时全部读取 `.env`/部署密钥，`config/database.php` 不硬编码生产连接参数，也不由 PHP 文件拼接写入敏感信息。
- [ ] 新增 `DatabaseConfigurationTest`：覆盖完整配置、缺失键、非法端口/charset/timezone/SSL 文件、空密码策略和日志/异常脱敏；源码守卫排除 `vendor/` 后拒绝 PHP 中出现已知真实连接值或连接参数默认常量。
- [ ] 从当前 24 张表生成不可变旧版基线清单，逐表记录字段类型/长度/符号、NULL、默认值、自增、主键/索引、引擎、字符集、排序规则和表前缀语义；目标新装 schema 明确为 23 张业务表，不包含 `yunu_area`。
- [ ] 按认证权限、内容/栏目、动态模型、表单、营销组件和日志等最小内聚批次创建有序 ThinkPHP migration；migration 只负责保留的 schema，禁止创建 `yunu_area` 或把 52 万字节原始 SQL 作为字符串继续执行。
- [ ] 将必须的 RBAC、动态模型/字段等基础记录转换为可幂等的核心 seeders；排除 3,544 条地区数据和地区权限菜单，演示栏目、内容、Banner、区块、链接和示例表单单独放入可选 `DemoSeeder`。
- [ ] 管理员记录不从旧 SQL seed：由安装 Command 创建首个管理员；测试 fixtures 使用独立测试 Seeder，不污染生产初始化数据。
- [ ] 空库安装使用 migration table 记录版本；既有 TP5 数据库先校验 24 表旧基线并建立受审计的 baseline，再运行包含地区功能删除的增量 migration，禁止重复建表或把现有业务数据当 seed 覆盖。
- [ ] 每个迁移提供前置校验、幂等保护和可回滚策略；破坏性删列/改类型延迟到灰度稳定后的独立版本。
- [ ] 为管理员密码字段、索引、`utf8mb4`、时间字段和动态表元数据建立兼容迁移。
- [ ] 为 `category.tpl_cover`、`tpl_list`、`tpl_show` 建立可逆数据迁移，只把白名单前缀 `cover_`、`list_`、`show_` 的首个下划线替换为 `/`，保留 `.html` 和后续业务下划线；未知值在预检阶段报错并阻止发布。
- [ ] 在生产数据量级副本上测量迁移锁表时间、磁盘增长和回滚时间，超过维护窗口则改为在线/分批迁移。
- [ ] 执行“空库 migrate+seed → 回滚 → 再迁移”“TP5 快照 baseline → 增量升级”两组测试，使用 `information_schema` 比较所有字段/索引属性；允许的结构差异只有已列明的 TP8 调整和地区表/字段删除。
- [ ] `php think yunu:install` 在 migration/seed 等价测试全部通过后删除 `data/install.sql`；CI 搜索并拒绝任何运行时代码、Command、测试或文档继续读取该文件。

**验收：** 空库只靠 migration、核心 seeder 和安装 Command 完成生产初始化；既有 TP5 数据库可无损建立 baseline 后升级；仓库不再包含或解析 `data/install.sql`。

### 5.2 改写 TP5 查询表达式与模型返回语义

**修改：** `app/*/model/`、直接使用 `Db::name()`/`db()` 的控制器与标签库。

- [ ] 用现有集成测试逐项迁移 `['IN', ...]`、`['LT', ...]`、多条件 `or`、`exp`、字符串 where、`page()`、`setField()`、`strict(false)` 等 TP5 查询写法。
- [ ] 所有用户可控排序、字段、表名和原生 SQL 改为白名单；值参数始终绑定，禁止拼接 SQL。
- [ ] 明确 TP8 Model/Collection 的数组转换边界，修复模板、JSON 和 `count()` 对象/数组差异。
- [ ] 明确模型时间戳、字段类型、软删除和隐藏/追加字段，确保 API 日期与字段类型保持旧契约。
- [ ] 为大列表加分页上限和必要索引，使用查询计划验证栏目、内容和后台列表无明显退化。
- [ ] 每迁移一个模型先运行对应 Integration 测试，再运行全量 PHPUnit 和 Mago。

**验收：** TP5 与 TP8 在同一 fixture 上返回相同业务结果；不存在依赖旧 Query 数组语法的第一方代码。

### 5.3 保证动态内容与表单的事务一致性

**修改：** Content、Diymodel、Diyfield、Diyform、Formcon 相关模型/服务。

- [ ] 从 `diymodel`/`diyform` 元数据生成动态表名，只允许数据库中已登记且符合命名正则的表名。
- [ ] 将 `content` 与 `diy_*`、`formcon` 与 `form_*` 的跨表写入封装为数据库事务，异常必须回滚两侧。
- [ ] 为模型切换、栏目删除、内容复制、字段增删和表单模型删除增加引用完整性检查。
- [ ] 在并发创建/更新/删除场景验证锁和唯一约束，防止重复 vid、孤儿记录和丢失更新。
- [ ] 对真实数据快照执行主表/动态表行数、孤儿记录、字段映射和抽样内容哈希校验。

**验收：** 动态表名无法被请求参数注入；故障注入下主表与动态表始终一致。

### 5.4 精简 `Myform` 控制器与模型

**修改/删除：** `app/index/controller/Myform.php`、`app/wap/controller/Myform.php`、`app/index/model/DiyformModel.php`、`app/wap/model/DiyformModel.php`；创建最小共享提交服务及对应测试。

- [ ] 用调用清单确认两个 `Myform` 控制器的提交、验证码和响应逻辑完全重复，并确认 WAP `DiyformModel::insertDiyform()` 等额外方法是否无调用；未经调用证明和测试保护不得直接删除。
- [ ] `index.form` 与 `wap.form` 指向同一个固定提交动作；若多应用隔离要求保留两个入口控制器，则控制器只负责接收请求、调用共享服务和转换 HTML/JSON 响应，不复制业务逻辑。
- [ ] 只保留一个 `Diyform` 模型负责表单定义/字段元数据查询；把输入校验、验证码、动态表事务、客户端 IP 和邮件通知从 Model 移入一个最小 `FormSubmissionService`，删除重复 WAP 模型和未使用方法。
- [ ] 提交服务只接收显式 DTO/数组和客户端上下文，不直接调用全局 `input()`、`request()` 或 `session()`；根据 `diyfield` 白名单提取字段，拒绝客户端覆盖 `fid`、`vid`、`create_time`、`update_time`、`ip`、表名等服务端字段。
- [ ] 恢复并统一 CSRF/表单 token 校验，验证码按表单配置强制校验；补充 token 缺失、伪造、重放、验证码错误、超长/数组输入、未知字段和批量提交限流测试。
- [ ] 在同一事务写入 `form_<白名单表名>` 与 `formcon`，任一步失败全部回滚；异常只记录脱敏诊断，不向访客返回 SQL、表名、路径或邮件凭据。
- [ ] 明确“数据提交成功、通知失败”的生产语义：数据提交结果不被提交后的邮件异常反转，通知失败进入有界重试/告警；该行为差异写入契约审批记录。
- [ ] 用统一结果对象生成旧版兼容的默认跳转/提示和 JSON 字段，删除 `return` 后的 `exit()`、不可达代码和控制器内邮件/数据库细节。
- [ ] 让 index、WAP、API 共用同一提交服务，覆盖三入口成功、验证失败、重复提交、事务故障、通知故障和响应契约；运行定向 PHPUnit、全量 PHPUnit 与 Mago。

**验收：** 自定义表单提交只有一个业务实现；两个公开 rewrite 路由与 API 契约通过；Model 不读取 HTTP 全局状态、不发邮件、不编排跨表事务，控制器不包含业务分支。

### 5.5 完整移除 `yunu_area` 地区功能

**删除：** `app/admin/controller/Area.php`、`app/admin/model/AreaModel.php`、`app/admin/view/area/index.html`、`app/admin/view/area/addarea.html`、`app/admin/view/area/editarea.html`、`app/index/model/AreaModel.php`、`app/wap/model/AreaModel.php` 及对应路由、菜单、权限、标签和测试。

**修改（前台/API）：** `app/common.php`、`app/index/common.php`、`app/wap/common.php`、`app/api/common.php`、`app/index/controller/Common.php`、`app/index/controller/Category.php`、`app/index/controller/Show.php`、`app/wap/controller/Common.php`、`app/wap/controller/Category.php`、`app/wap/controller/Show.php`、两端 Category/Content Model、`app/api/controller/Master.php`、`app/api/controller/V1.php`、两端 `taglib/Yunu.php`。

**修改（后台/模板）：** Admin Common/System/Api/Category/Content/Link Controller，Category/Content/Link/Sitelink Model，`app/admin/view/system/seo.html`、`app/admin/view/system/basic.html`、content/link/category/sitelink/xml 相关视图，以及默认 index/WAP 主题中的地区区块。

**修改（数据/部署）：** `config/extra/sys.php`、`database/migrations/`、`database/seeders/`、路由退役清单、DNS/发布文档；`data/install.sql` 仍按 5.1 整体淘汰，不转换其中的 `yunu_area` 结构和数据。

- [ ] 在删除前生成 `docs/upgrade/area-removal.md`：列出 3 套 Area Model、后台 Area Controller/视图、所有 `db('area')`、地区 route/API/tag、session/config、模板和 SQL 引用，并区分 Layui 弹窗 `area`、IP 地址 `area` 等无关同名字段，防止误删。
- [ ] 对生产副本导出 `yunu_area`、`content.area`、`link.area`、`category.isarea`、`sitelink.areapre`、地区 SEO 配置、RBAC 地区规则和含 `[prov]`/`[city]`/`[prov_or_city]` 内容的行数与哈希；备份校验通过后才允许破坏性 migration。
- [ ] 新装 migration 从第一版起不创建 `yunu_area`，并从 `content`、`link`、`category`、`sitelink` 排除 `area`、`isarea`、`areapre` 字段；既有库升级 migration 按“清理引用数据 → 删除字段 → 删除 `yunu_area`”顺序执行并记录影响行数。
- [ ] Seeder 删除全部地区记录及 `admin/area/*` 权限项，同时从角色 rules 中移除对应规则 ID；升级后不存在指向已删除菜单的孤儿权限。
- [ ] 删除后台 Area Controller/Model/三张视图、路由和 Common 免认证项；System SEO 页面移除分站 SEO/default area，Category/Content/Link/Sitelink 页面与控制器移除地区开关、选择器、批量地区和提交字段。
- [ ] 删除 index/wap `AreaModel`，从 Common 生命周期移除地区域名/路径解析、默认地区、`sys_area`/`sys_areainfo` session/config；Category/Content/Common URL 生成与查询只保留主站语义。
- [ ] 从 `Master`、`V1` 和 API common 删除 `api_area`、地区列表、地区筛选和地区 URL 分支；发布说明标记破坏性 API 变更，退役窗口返回稳定错误后最终 404，禁止返回伪造空地区列表掩盖客户端问题。
- [ ] 从 index/wap `Yunu` 标签库删除 `<yunu:area>` 及所有地区筛选/标题替换分支；删除 `update_str_dq()`、`top_aera()` 等仅服务地区的全局函数，并更新 function guard 清单。
- [ ] 清理默认主题中的城市分站/地区产品区块；若 `list_map.html` 失去业务内容，则删除模板并把仍引用它的栏目迁移到普通列表模板，其他模板只删除地区片段，不保留空标题或死链接。
- [ ] 删除 `seo_area`、`seo_title_area`、`seo_keywords_area`、`seo_description_area`、`seo_default_area` 等地区配置；扫描数据库 SEO/内容文本中的地区占位符，按审核后的主站文案替换，禁止线上页面遗留原始 token。
- [ ] 删除 sitemap、链接、Banner/Block、站内链接及后台 API 中的地区派生循环；保留的主站数据不得因旧 area 字段为空/非空而改变可见性。
- [ ] 地区 rewrite 在一个发布周期内只允许命中无数据库依赖的精确 410 响应，不得被普通栏目/内容路由吞掉；观察期后删除退役映射和地区子域名 DNS，地区路径与 `area` API 最终均为 404。
- [ ] 新增 `AreaRemovalTest`：断言目标数据库无 `yunu_area` 及四个地区字段、容器无 Area 类、路由/菜单/API/tag 无 area 能力、页面不写地区 session、模板不含 `<yunu:area>`、源码无 `db('area')`/`AreaModel` 引用。
- [ ] 运行 schema 升级/回滚演练、全部路由契约、Index/WAP/Admin/API/模板测试、`vendor/bin/phpunit`、Mago 四项和 `composer audit --locked`；地区删除以外的响应与数据副作用不得变化。

**验收：** 新装与升级后的数据库均不存在 `yunu_area` 及地区关联字段；代码库不存在地区 MVC、SQL、API、标签、配置、URL 生成和运行期查询；无关的 UI `area`/IP 地理字段未被误删。

---

## 阶段 6：按应用迁移并保持每批可发布

### 6.1 迁移 `index` 前台应用

**修改：** `app/index/`、`app/index/view/`、对应路由和配置。

- [ ] 先迁移首页与公共布局，使首页 TP5/TP8 契约测试通过，再迁移栏目、内容、搜索、标签和表单。
- [ ] 保持主站 SEO 标题/关键词/描述、canonical、分页、上一篇/下一篇和浏览量副作用；地区分站属于已批准删除项。
- [ ] 所有前台 URL 使用 TP8 命名路由生成 rewrite 地址，canonical、分页和内容链接不得包含 `index.php` 或地区前缀。
- [ ] 从注入的 `SiteContext` 获取 canonical URL、主题和 cache/cookie 前缀；首页、栏目、内容及 sitemap 不读取 Host、`site_url` 或请求期全局配置，逐站点运行同一契约套件。
- [ ] 将控制器内可复用数据逻辑下沉到现有模型或最小服务，禁止复制到 WAP/API。
- [ ] 每完成一个控制器运行对应 Feature/Contract、完整 Unit/Integration、Mago 和格式检查。
- [ ] `index` 全部迁移后执行桌面端路由清单 100% 冒烟与 HTML 快照比对。

**验收：** 桌面前台所有公开 URL、SEO 和核心写操作与基线一致，批准的安全修复差异有独立记录。

### 6.2 迁移 `wap` 移动应用

**修改：** `app/wap/`、`app/wap/view/`、移动路由和域名绑定。

- [ ] 对照 `index` 共享数据逻辑，复用已迁移模型/服务；仅保留移动端渲染和跳转差异。
- [ ] 迁移 `/m/`、移动子域名、自动设备跳转和移动分页；不迁移地区 URL、地区子域名或地区 session。
- [ ] 固定 WAP rewrite 规则和命名路由，确保 `/m/` 与移动子域名不会回退生成动态 URL。
- [ ] WAP 只接受 `SiteContext.channel` 决定的路径/域名表现，别名统一指向 canonical；不得在控制器再次解析 Host 或把多站点识别复制成移动端分支。
- [ ] 对首页、栏目、内容、搜索、标签、表单逐项运行 WAP Feature 与 HTML 快照。
- [ ] 验证桌面↔移动跳转不循环，代理 HTTPS、端口和查询参数不会丢失。
- [ ] 运行完整回归与 Mago，确认没有因 WAP 修改破坏 `index`。

**验收：** WAP 路由矩阵 100% 通过，移动端和桌面端共享数据结果一致。

### 6.3 迁移 `admin` 后台应用

**修改：** `app/admin/`、后台视图、路由、中间件和配置。

- [ ] 按“登录/RBAC → 基础 CRUD → 内容/动态模型 → 表单 → 配置/上传 → 运维功能”顺序迁移，每批可独立回归。
- [ ] 将输入验证统一到 TP8 Validate/表单请求，所有批量 ID、排序、状态、动态字段和文件参数都先验证再写入。
- [ ] 保持后台 JSON 字段、状态码和前端 Layui 交互契约，修复时增加回归测试而非修改前端掩盖后端错误。
- [ ] 配置写操作改为持久化配置服务或环境配置，不再直接拼接 PHP 源码；需要运行期修改的站点设置保留数据库来源。
- [ ] 后台请求同样经过 `ResolveSite`，所有生成的预览/前台链接使用当前 SiteContext；本轮不新增站点 CRUD、站点表或按站点复制业务配置页面。
- [ ] 缓存清理只清除应用 runtime 白名单目录，不接受请求路径，不删除日志/上传/配置。
- [ ] 每迁移一个后台域运行对应权限矩阵、CRUD、失败回滚、完整回归和 Mago。

**验收：** 后台所有菜单对应动作均可用；普通角色无法调用未授权接口；所有写入失败均返回稳定 JSON 且不留半成品数据。

### 6.4 迁移 `api` 应用

**修改：** `app/api/`、API 路由、中间件、响应转换器。

- [ ] 用显式路由替代依赖空控制器/`_empty` 的动态分发，路由名称与允许方法白名单化。
- [ ] 合并 `Master` 与 `V1` 的真实重复实现，只在契约证明等价后复用；保留两个入口的兼容路由。
- [ ] 使用统一响应转换器固定字段类型、日期、图片 URL、空集合和错误格式。
- [ ] API 的绝对 URL 和图片字段使用当前 SiteContext；Unsplash 热链保持 API 返回的 HTTPS URL，不再被 `uploads_add_url()` 二次拼接本站域名，并按契约输出所需署名元数据。
- [ ] 为分页、limit、orderby、callback 和内容 ID 设置边界与白名单，补充滥用/注入测试；删除地区筛选参数及实现。
- [ ] 外部消费者未完成迁移前保留 JSONP 兼容开关，并记录访问量；默认新客户端使用 JSON。
- [ ] 执行 API 全量契约、CORS、安全、负载边界和数据库副作用测试。

**验收：** API 清单中的每个端点都通过 TP5/TP8 契约对比；不存在任意方法动态调用。

---

## 阶段 7：迁移模板、标签库与主题体系

### 7.1 适配 `topthink/think-view`

**修改：** `config/view.php`、各应用视图配置、`app/index/view/`、`app/wap/view/`、`app/admin/view/`、index/WAP 渲染控制器、Admin Category 模板选择器及栏目模板 migration。

- [ ] 用锁定的 `topthink/think-view` 版本确认实际配置键后，将 index、WAP、admin 的逻辑模板分隔符统一显式设为字符串 `/`，不使用 `_` 或依赖操作系统 `DS`；配置 Think 模板驱动、`.html` 后缀、缓存、主题根目录和替换变量。
- [ ] 按映射表原子移动 index/WAP 模板：`index_*`、`search_*`、`tag_*`、`guestbook_*`、`list_*`、`show_*`、`cover_*`、`public_*` 分别进入同名目录；只拆首个结构下划线，例如 `list_product_diy.html` 必须成为 `list/product_diy.html`。
- [ ] Web Install 与地区 Area 模板按对应下线阶段直接删除，不迁移到新目录；admin 现有 `controller/action.html` 结构保留，只把逻辑标识和配置统一为 `/`。
- [ ] 将 index/WAP 的 `<include file="public:header">` 等旧 include 改为 `/` 逻辑路径（如 `public/header`），扫描 extend/layout/include 中的 `:`、反斜线和结构下划线引用并全部消除。
- [ ] 将无参 `fetch()` 的自动定位固定为 `<controller>/<action>.html`；显式渲染只传根目录相对的 `/` 逻辑模板名，删除 `$tpl_file` 绝对/相对文件系统拼接，不允许控制器直接拼接主题物理路径。
- [ ] 主题只能取自已验证的 `SiteContext.theme`，并在 `app/index|wap/view/<theme>/` 白名单根内解析；跨两个测试站点验证不同主题、相同逻辑模板名及模板缓存 key 隔离，Host/查询参数不得成为任意目录片段。
- [ ] 栏目模板值继续保留 `.html` 后缀，但只接受 `cover/<slug>.html`、`list/<slug>.html`、`show/<slug>.html`；`slug` 可含业务下划线，拒绝绝对路径、反斜线、`..`、空段、双斜线和非主题目录文件。
- [ ] 把 Admin Category 的 `getFileFolderList(..., 'list_*')`/`show_*`/`cover_*` 改为扫描对应目录并返回规范化的相对 `/` 路径；新增、编辑、批量设置均验证文件真实存在于当前主题且类型前缀匹配字段。
- [ ] 同步 migration、核心 Seeder、DemoSeeder、fixtures 和已有栏目记录中的 `tpl_cover/tpl_list/tpl_show`；升级迁移先验证全部旧值可映射，再在同一发布窗口切换文件与数据，禁止长期双读 `_` 与 `/` 两套路径。
- [ ] 新增 `TemplatePathTest` 精确断言映射：`index_index.html → index/index.html`、`index_default.html → index/default.html`、`list_article.html → list/article.html`、`list_product_diy.html → list/product_diy.html`、`show_product.html → show/product.html`、`public_header.html → public/header.html`。
- [ ] 自动枚举每个主题和应用的全部 HTML：所有栏目模板引用必须存在，每个保留模板至少可编译一次，include/extend 目标无缺失或循环，Linux 大小写路径与本地结果一致。
- [ ] 增加源码/制品门禁：主题目录不得残留 `^(index|search|tag|guestbook|list|show|cover|public)_*.html` 旧结构文件，配置不得出现 `view_depr = '_'`，栏目模板字段不得包含结构下划线或反斜线。
- [ ] 将 TP5 `assign/fetch` 调用迁移到 TP8 视图 API，显式传递模板和变量，模板缺失/非法路径返回确定的 404 或受控异常，不暴露绝对路径。
- [ ] 验证后台视图、前台多主题、WAP 主题和错误页定位；使用 HTML 快照与浏览器冒烟检查资源路径、表单 token、分页和中文输出，并确认 CLI 安装不依赖模板。

**验收：** 所有模板逻辑路径和栏目模板数据使用 `/`；旧下划线结构不存在且无兼容双读；全部主题通过编译、快照和路径穿越测试，模板源文件不对公网暴露。

### 7.2 迁移自定义 `Yunu` 标签库

**修改：** `app/index/taglib/Yunu.php`、`app/wap/taglib/Yunu.php` 或合并后的共享标签实现。

- [ ] 为每个自定义标签先建立输入属性→生成代码/渲染结果测试，再适配 TP8 `think-template` 标签接口。
- [ ] 替换标签中生成的 TP5 查询数组语法、动态 PHP 字符串和隐式全局变量。
- [ ] 抽取 index/wap 真正相同的标签数据逻辑，保留模板输出差异；不为了去重改变现有标签名称或属性。
- [ ] 对非法属性、动态排序、动态字段和空数据增加白名单/默认值测试；删除地区筛选和 `<yunu:area>` 定义。
- [ ] 运行全部主题模板编译、HTML 快照、前台/WAP Feature 和 Mago 检查。

**验收：** 当前主题使用的全部 Yunu 标签均有测试，模板编译不执行未白名单的动态代码。

---

## 阶段 8：替换或隔离内置第三方依赖

### 8.1 Composer 化外部库

**修改：** `composer.json`、`composer.lock`、`app/extend/` 及调用方。

- [ ] 为每个替换建立 ADR/清单，记录现有调用方、选择的 Composer 包、直接/传递依赖、许可证、PHP 8.4 支持、维护与安全状态、行为差异、回退方案；同一能力只保留一个包，禁止“先全装再决定”。
- [ ] 把共享 Guzzle 客户端作为第一方通用 HTTP 唯一底座：默认启用 TLS 校验、连接/总超时、受控重定向、响应/下载大小上限、明确 User-Agent、结构化异常与敏感头脱敏；调用方只能按业务需要收紧这些设置。
- [ ] 网络重试只覆盖可判定的瞬时错误并采用有界退避；非幂等 POST 默认不自动重试，除非目标协议和幂等键已验证。URL、scheme、Host、端口、DNS 解析结果和重定向目标都经过 SSRF 白名单，禁止访问 loopback、link-local、私网和云 metadata。
- [ ] 用 `overtrue/pinyin` 完成栏目拼音/slug 迁移，审核 characterization 差异及已有 URL 影响；已发布 slug 默认不批量重写，新建/编辑规则稳定且冲突可预测，删除 `get_pinyin()` 与 `statics/pinyin.dat`。
- [ ] 用 Composer 官方包替换内置 PHPMailer，保持邮件编码、附件、TLS、超时和错误日志契约。
- [ ] 用 Composer 官方七牛 SDK 替换 `app/extend/Qiniu/`，凭据来自环境，上传/删除/CDN 失败可重试但有上限。
- [ ] 核实百度 AIP 当前官方 PHP SDK 的维护和 PHP 8.4 兼容性；可用则 Composer 化，不可用则把现有 SDK 隔离为受测试适配器并禁止无界网络调用。
- [ ] 用 `ZipArchive` 替换 PclZip，使用 GD/Imagick 或受维护包替换无法通过 PHP 8.4/Mago 的图片代码。
- [ ] 评估验证码实现；优先使用与 TP8 兼容的官方包，保留现有视觉/接口契约和服务端校验测试。
- [ ] 每次 `composer require` 都使用明确主版本和 `--with-all-dependencies`，更新锁文件后运行全量测试、Mago 与 `composer audit --locked`。
- [ ] 调用方全部切换且测试通过后，删除 `send_post()`、`url_get_contents()`、`com\Http`、散落第一方 cURL/远程 stream 调用及对应内置第三方副本；不得保留兼容代理或同时加载两套同名库。
- [ ] 增加源码门禁：排除 `vendor/` 和经评审 SDK 后，第一方禁止新增 `curl_init`、远程 `file_get_contents`/`fopen`、`get_headers`、HTTP stream context、`get_pinyin` 和 `pinyin.dat`；例外必须指向有期限的 ADR。

**验收：** 第三方代码由 Composer 或明确隔离适配器管理；通用网络与拼音无手写重复实现；`app/extend/` 只保留有归属、有测试的第一方/数据资源。

### 8.2 使用 Unsplash 实现内容配图

**创建/修改：** `app/common/integration/UnsplashImageService.php`、共享 HTTP 配置、后台内容/栏目图片选择器、`content.pic`/配图元数据 migration、前台/API 图片与署名输出、对应 PHPUnit 测试。

- [ ] 按 TDD 先建立 `UnsplashImageServiceTest` 和后台 Feature 失败用例，再实现最小能力；“图片生成”在本项目中明确指从 Unsplash 官方 API 搜索或随机选取现有图片，不宣称生成原创图片。
- [ ] 只调用 `https://api.unsplash.com/` v1 的公开搜索/随机照片及返回的 `links.download_location`，请求带 `Accept-Version: v1` 和服务端 Access Key；禁止抓取 unsplash.com HTML、使用已弃用 Source 接口、拼接未记录端点或把 Access/Secret Key 下发浏览器。
- [ ] 后台入口默认用内容标题/SEO 关键词形成可编辑 query，支持搜索/随机、`landscape|portrait|squarish`、`content_filter=high`、分页和人工确认；限制 query 长度、候选数量与请求频率，不做定时批量或无人值守抓图。
- [ ] 搜索结果只接受官方响应的 `photo.urls.*` HTTPS 热链并使用受控的尺寸/质量参数；不得把 Unsplash 原图下载到 `public/uploads/`、对象存储或代理成本站图片，不得移除/替换返回 URL 的来源域名。
- [ ] 当编辑者确认图片用于内容、栏目或头图时，服务端精确请求该响应自带的完整 `links.download_location` 进行 download tracking，保留其已有查询串并安全附加认证；tracking 成功后才保存选择，失败时明确提示且不产生半保存状态。
- [ ] 迁移扩大 `content.pic` 或相关图片字段以容纳 HTTPS URL，并在同一现有业务表增加最小 JSON 配图元数据：provider、photo ID、摄影师名称/用户名、摄影师主页、Unsplash 照片页、UTM 后链接、选用时间；不新增媒体表、站点表或把凭据写入数据库。
- [ ] 前台、WAP、预览和 API 展示所选图片时同时提供可点击的摄影师与 Unsplash 署名，回链包含稳定 `utm_source`/`utm_medium=referral`；模板和 JSON 快照验证名称、链接及可访问性，普通本地上传不显示错误署名。
- [ ] 将 `UNSPLASH_ACCESS_KEY`、`UNSPLASH_UTM_SOURCE`、启用开关、超时与缓存 TTL 放入 `.example.env`/配置，真实值只由密钥系统注入；配置缺失或功能关闭时隐藏入口并保留现有人工上传，不阻断内容编辑。
- [ ] 读取 `X-Ratelimit-Limit`/`X-Ratelimit-Remaining`，对查询结果做有界短缓存且不缓存 Access Key/异常；接近配额、429、5xx、超时、DNS/TLS、空/畸形响应时无自动换供应商，返回可诊断信息并回退人工上传。
- [ ] 图片 URL、`download_location` 和回链必须来自同一已验证响应且 Host/scheme 白名单化，防止 fake/污染响应把服务端引向内网；日志只记录 request ID、Unsplash photo ID、状态和剩余配额，不记录凭据或完整认证 URL。
- [ ] PHPUnit 使用 Guzzle `MockHandler`/本地 fake 覆盖成功搜索、随机、人工选择、tracking、署名、UTM、缓存、并发选择、429/5xx/超时、恶意 URL、缺字段、无凭据和禁用回退；全量测试断网运行，真实 API 仅在显式预发布 smoke profile 下人工授权执行。
- [ ] 发布前用非生产内容验证 Unsplash 应用已满足 API Guidelines 并取得适合生产流量的状态/配额；将限流预算、禁用开关、密钥轮换、服务不可用和删除/换图流程写入运行手册。

**验收：** 内容配图只来自 Unsplash 官方 API/CDN，选择行为完成 tracking，所有展示均有合规署名；密钥、限流、SSRF、故障回退和断网测试通过，未引入第二套 HTTP 客户端或图片抓取代码。

### 8.3 收紧文件、上传与外部网络安全

- [ ] 上传同时校验大小、扩展、MIME 和真实文件内容，使用随机服务端文件名，拒绝 PHP/脚本/双扩展和路径穿越。
- [ ] 上传、模板包、升级包解压前校验总大小、文件数、目标真实路径和压缩炸弹；写入使用临时文件+原子替换。
- [ ] 所有 HTTP 客户端启用 HTTPS、证书校验、连接/读取超时、响应大小上限和结构化错误；禁止默认跟随到内网地址。
- [ ] 密钥、token、密码和用户隐私字段不进入异常页、访问日志、Mago 输出、测试快照或 CI artifacts。
- [ ] 运行上传/SSRF/Zip Slip/路径穿越/TLS 失败回归与 Mago analyze。

**验收：** 文件和网络边界的失败路径均有自动化测试，不依赖 PHP 配置碰巧阻止攻击。

---

## 阶段 9：重构安装、配置、备份和升级流程

### 9.1 用 ThinkPHP Command 完成生产安装

**创建/删除：** 创建 `app/command/Install.php`、`config/console.php`、`tests/Feature/Command/InstallCommandTest.php`；删除 `app/index/controller/Install.php`、`app/index/view/install_*.html` 及全部 Web 安装路由。

- [ ] 注册唯一安装入口 `php think yunu:install`，同时支持安全的交互模式和部署使用的 `--no-interaction`；另提供无副作用的 `--check` 只执行预检。
- [ ] 命令启动时读取 Composer platform requirements，检查 PHP 8.4、必需扩展、配置完整性、数据库连通性/版本/字符集/严格模式及必要目录权限；所有检查在 schema 或文件写入前完成。
- [ ] 数据库连接参数全部使用已加载的 `.env`/部署密钥，安装 Command 不接受另一套 `--db-host`/`--db-password` 参数，也不在 PHP 中提供默认连接；管理员非敏感参数可来自部署配置，管理员密码只从受权限保护的 secret file、密钥环境或隐藏式标准输入读取，禁止放入 shell 历史、进程列表和日志。
- [ ] 数据库及最小权限账户由部署系统预先创建；安装命令不得使用生产应用账号执行 `CREATE DATABASE`、`CREATE USER` 或 `GRANT`，只验证目标空库/受支持的升级状态。
- [ ] 通过版本化 migration/core-seeder API 安装 schema 和基础数据，不再解析 `data/install.sql`，也不调用 `mysql_*`/`mysqli_*` 或拼接 SQL；命令与后续版本升级共用同一迁移链，可选 `--with-demo-data` 只追加显式演示 Seeder。
- [ ] 创建首个管理员时要求部署提供强密码或安全随机生成一次性密码，使用当前密码哈希 API 保存，禁止 `admin/admin`；随机密码只允许在已确认的安全输出通道显示一次并强制首次登录修改。
- [ ] 设置站点 URL、rewrite 路由模式、schema/app 版本和安装完成状态；数据库与外部服务密钥继续来自 `.env`/密钥系统，命令不得生成、回写或修改 `.env`，也不得生成包含连接参数/凭据的 PHP 文件。
- [ ] `--check` 逐项验证数据库 `.env` 键、站点注册表和 Unsplash 等启用集成的配置完整性，只输出键名与通过/失败，不回显值；`/health/ready` 复用同一脱敏校验结果。
- [ ] 使用数据库 advisory lock 或等价原子互斥阻止并发安装；MySQL DDL 不假装可由单一事务回滚，依靠 migration 记录、幂等 seed 和明确的恢复点支持失败后重跑。
- [ ] 只有迁移、种子、管理员、站点配置全部成功后才原子标记安装完成；重复执行默认拒绝并返回稳定非零退出码，不提供可误覆盖生产数据的宽泛 `--force`。
- [ ] 为预检、配置、数据库、迁移、种子、管理员和并发冲突定义稳定退出码；控制台输出只含阶段、结果和可执行修复提示，异常与测试快照必须脱敏。
- [ ] 未安装 HTTP 请求返回 503，`/health/live` 可用、`/health/ready` 失败；旧 Install 路径、模板和静态安装资源均返回 404，不暴露数据库探测或写入能力。
- [ ] PHPUnit 覆盖空库成功安装、交互/非交互、`--check`、缺失配置、弱密码、数据库失败、非空未知 schema、迁移/seed 中断、权限不足、并发、重复执行、恢复重跑、特殊字符和秘密不出现在输出/日志。
- [ ] 在全新临时数据库运行 `php think yunu:install --no-interaction`，确认目标 23 表、后续 migration、核心 seed、管理员和站点配置全部完成且不存在 `yunu_area`；随后执行管理员登录、rewrite 路由冒烟、完整 PHPUnit 与 Mago，并在第二个实例验证失败恢复和重复安装拒绝。

**验收：** 新环境仅通过 Command 完成安装；仓库和 Web 路由中不存在可执行安装控制器；命令可自动化、可诊断、可安全重跑且不泄露凭据。

### 9.2 生产化备份、恢复与发布升级

- [ ] 数据备份使用数据库原生工具或经过测试的库，记录 schema 版本、校验和、压缩方式和加密状态。
- [ ] 恢复必须在隔离库先校验，再执行维护窗口切换；禁止通过普通后台请求直接覆盖生产库。
- [ ] 停用应用内远程下载并覆盖源码的在线升级路径，生产升级统一走 CI 构建的不可变制品和受控数据库迁移。
- [ ] 若保留模板/数据包更新，只允许签名制品、HTTPS 固定来源、哈希校验、临时目录和原子切换。
- [ ] 为备份失败、恢复失败、磁盘不足、权限错误、校验失败和迁移失败建立自动化测试及运行手册。

**验收：** 应用进程没有修改自身 PHP 源码的权限；代码回滚通过部署制品完成。

---

## 阶段 10：质量门禁、完整回归与差异清零

### 10.1 固化本地与 CI 命令

- [ ] `composer qa:syntax` 执行 `vendor/bin/mago lint --semantics`。
- [ ] `composer qa:lint` 执行 `vendor/bin/mago lint`。
- [ ] `composer qa:analyze` 执行 `vendor/bin/mago analyze`。
- [ ] `composer qa:format` 执行 `vendor/bin/mago format --check`，禁止 CI 使用只打印差异但退出 0 的 `--dry-run`。
- [ ] `composer qa` 包含 `FunctionGuardTest`，阻止未使用 `if (!function_exists(...))` 保护的第一方全局用户函数进入主分支。
- [ ] `composer qa` 包含 `ProjectStructureTest`、`TemplatePathTest`、`AreaRemovalTest`、`DatabaseConfigurationTest` 及源码守卫，阻止旧目录/模板分隔符、地区能力、数据库硬编码、手写 HTTP/拼音实现重新进入主分支。
- [ ] `composer test` 执行全量 PHPUnit；`composer test:coverage` 使用覆盖率驱动输出 Clover/HTML，并执行覆盖率阈值校验。
- [ ] `composer qa` 顺序执行 Composer validate/audit、Mago 四项和 PHPUnit 全量套件，任一失败立即退出非 0。
- [ ] CI/PR 模板要求每个实现提交填写 RED 命令与失败摘要、GREEN 命令、REFACTOR 后受影响套件；缺失证据的任务保持 `[ ]`，不以最终测试全绿倒推为 TDD。
- [ ] 在 README 和 `AGENTS.md` 中只记录验证过的命令、运行环境和故障排查路径。

**验收：** 开发者与 CI 使用同一组 Composer scripts，不出现“本地命令和流水线规则不同”。

### 10.2 清理 Mago 遗留基线

- [ ] 每迁移一个目录后运行 `vendor/bin/mago lint --remove-outdated-baseline-entries`，并对 analyzer baseline 做等价清理。
- [ ] 对 `app/`、`config/`、`tests/` 执行 `--ignore-baseline`，修复全部 error；warning 只允许有规则编号、原因、作用域和移除期限的最小抑制。
- [ ] 禁止对新代码使用文件级 blanket ignore，禁止重新生成 baseline 把新增错误计入历史债务。
- [ ] 对格式化变更使用独立提交，确认行为快照和数据库副作用完全一致。
- [ ] 发布前删除已迁移代码对应的所有 baseline 条目，并用 `mago inspect-baseline` 证明剩余项仅属于待删除旧源码。

**验收：** 最终删除 `system/` 和旧依赖后，lint/analyze baseline 为空或删除，第一方代码全量无 baseline 通过。

### 10.3 执行完整回归与契约差异审核

- [ ] 在同一脱敏数据快照上并行运行 TP5 legacy 与 TP8 candidate 的全部 GET 契约；写操作使用克隆数据库后比较副作用。
- [ ] 比较状态码、Location、Content-Type、Cookie、安全头、JSON、规范化 HTML、数据库行和生成文件哈希。
- [ ] 将差异分为“必须修复”“已批准安全增强”“已批准产品变更”；没有负责人和批准记录的差异视为失败。
- [ ] 连续三次从空数据库运行 `composer qa`，排除测试顺序、缓存、随机值和并发导致的偶发失败。
- [ ] 在关闭网络的测试环境运行全量套件，确认外部服务均已 fake，不存在隐藏真实调用。
- [ ] 生成最终回归报告，列出用例数、断言数、覆盖率、跳过项、耗时、Mago/Composer 结果和剩余风险。

**验收：** P0/P1 契约零未批准差异；完整回归报告可作为发布审批附件。

---

## 阶段 11：性能、安全、可观测性与故障演练

### 11.1 性能与容量回归

- [ ] 在相同机器、相同数据和相同并发下记录 TP5 与 TP8 的首页、栏目、内容、搜索、后台列表和 API 吞吐/P50/P95/P99/错误率。
- [ ] 对慢查询、N+1、模板编译、缓存命中、会话和动态表查询建立监控，使用真实查询计划验证索引。
- [ ] 执行缓存冷启动、缓存热启动、并发写入、大分页、大文件上传和数据库连接耗尽测试。
- [ ] 设定门禁：无已批准原因时 TP8 P95 不得比 TP5 退化超过 10%，错误率不得上升，资源使用不得突破生产容量预算。
- [ ] 性能修复后重跑完整 PHPUnit 与 Mago，防止用缓存/批量优化改变业务结果。

### 11.2 安全回归

- [ ] 验证 SQL 注入、XSS、CSRF、开放重定向、SSRF、路径穿越、任意上传、Zip Slip、会话固定、权限绕过和暴力登录。
- [ ] 验证生产 `APP_DEBUG=false`、错误页不泄露路径/SQL/凭据，日志自动脱敏且具有请求关联 ID。
- [ ] 对 Composer 锁文件执行审计，对运行镜像执行漏洞扫描，处理高危/严重项。
- [ ] 验证数据库/缓存/对象存储最小权限，应用账户无建用户、授权和修改自身源码权限。
- [ ] 将安全修复加入 PHPUnit 回归，确保后续不会复发。

### 11.3 可观测性与故障注入

- [ ] 统一访问日志、应用日志和错误日志字段，包含请求 ID、site key、应用、路由、状态码、耗时和脱敏用户标识；站点不存在时记录规范化 Host 哈希而非未验证原值。
- [ ] 为 5xx、登录失败、权限拒绝、外部 API、Unsplash 剩余配额/429/tracking 失败、上传、迁移和队列/定时任务建立按 site key 聚合的指标与告警。
- [ ] 验证 health live/ready、优雅停止、缓存不可用、数据库短暂中断、外部 API 超时、Unsplash 禁用/限流、磁盘只读/满等故障行为；ready 校验全部启用站点注册表与必需 `.env` 键但不输出值。
- [ ] 故障期间不得产生半写数据、无限重试或敏感异常；恢复后无需人工修改数据库即可继续服务。

**验收：** 性能、安全和故障演练报告通过发布评审，监控能在用户大量报障前识别失败。

---

## 阶段 12：预发布、灰度、回滚与收尾

### 12.1 构建不可变生产制品

- [ ] 在干净 runner 使用 `composer install --no-dev --no-interaction --prefer-dist --optimize-autoloader` 构建，不在生产主机执行 `composer update`。
- [ ] 制品包含 `composer.lock`、应用代码、公开资源、迁移和版本信息，不包含 `.env`、测试、开发工具、缓存、日志、备份或上传数据。
- [ ] 构建后执行 `composer check-platform-reqs --no-dev`、最小启动/健康检查和制品哈希签名。
- [ ] 使用同一制品依次部署预发布、灰度和全量环境，禁止每个环境重新解析依赖。

### 12.2 预发布验收与迁移演练

- [ ] 从最新生产脱敏快照恢复预发布数据库/上传，执行迁移并记录耗时、锁、校验和和回滚结果。
- [ ] 在预发布执行 Composer 审计、Mago、PHPUnit 全量、HTTP 路由冒烟、浏览器关键路径和性能/安全回归。
- [ ] 对 `site-matrix.md` 每个主域名、别名及 index/WAP 通道执行同一 rewrite/模板/canonical/Cookie/cache 路由矩阵，验证未知 Host 421、停用站点 503、连续跨站请求不泄漏上下文。
- [ ] 使用预发布 Unsplash 应用和非生产内容执行一次显式 smoke：搜索/随机候选、人工选择、download tracking、热链、署名、UTM、限流头和禁用回退均符合官方 Guidelines；不得把真实 Access Key 写入报告或构建产物。
- [ ] 由业务验收首页、内容、WAP、后台 CRUD、权限、API、表单、上传、邮件/对象存储和运维流程，并确认地区分站入口、菜单和内容区块已完整移除。
- [ ] 冻结发布窗口内的 schema/配置变更，生成最终发布步骤、负责人、观察指标和回滚命令。

### 12.3 灰度切换与回滚验证

- [ ] 迁移优先采用向后兼容 expand/contract，发布初期 TP5 与 TP8 均可读取新 schema；破坏性收缩延后。
- [ ] 先切内部/白名单流量，再按比例灰度；每一级观察完整业务周期并核对错误率、登录、写入一致性、P95 和资源。
- [ ] 灰度期间持续抽样比较 TP5/TP8 响应和数据库副作用，出现未批准差异立即停止扩容。
- [ ] 实际执行一次应用制品回滚和数据库兼容回滚，确认恢复时间满足目标且不丢失灰度期间写入。
- [ ] 全量发布后保留旧制品、数据库备份和回滚能力至稳定观察期结束。

### 12.4 删除旧框架并完成交付

- [ ] 稳定观察期结束后删除 `system/`、旧根入口、旧配置、`data/install.sql`、旧内置第三方副本和仅供 TP5 使用的兼容代码。
- [ ] 确认旧 `template/`、`statics/`、手写 HTTP/拼音实现和 `pinyin.dat` 已删除，应用模板/静态资源分别只位于标准 `app/<应用>/view/` 与 `public/static/`，所有逻辑模板路径使用 `/`。
- [ ] 删除或清空 Mago 遗留 baseline，重新执行 `composer qa`、生产制品构建和全量回归。
- [ ] 更新 `README.md`、`AGENTS.md`、部署/备份/恢复/回滚/故障排查文档，使其只描述 TP8 真实流程。
- [ ] 确认仓库无真实凭据、缓存、日志、测试产物、备份和上传临时文件；执行 `git diff --check` 与最终代码审查。
- [ ] 使用小步中文 Conventional Commits 完成收尾，生成版本标签和发布说明，列出依赖版本、迁移、已批准差异和已知限制。

**最终完成定义：** 生产流量稳定运行 TP8；旧框架不再被加载；目录与 ThinkPHP 8 官方多应用骨架一致，模板路径只用 `/`；数据库和源码不存在 `yunu_area` 相关 MVC、SQL、字段、API、标签、配置或 URL 生成；所有站点只由全局 middleware/SiteContext 识别且相互隔离；站内和 sitemap 只生成保留业务的 rewrite URL；Nginx/Apache 不含业务 rewrite；安装仅由 Command 调用 migration/seeder 完成且不存在 `data/install.sql`/Web Install；数据库连接只从 `.env`/部署密钥读取且 PHP 无硬编码；Myform 只有一个提交实现；保留的全局用户函数均受 `function_exists` 保护；通用网络与拼音由 Guzzle/`overtrue/pinyin` 提供且旧手写实现已删除；Unsplash 配图满足热链、tracking、署名、UTM、配额和密钥安全要求；所有实现任务均有 RED→GREEN→REFACTOR 证据；完整回归、Mago、Composer 审计、性能、安全、灰度和回滚证据齐全；文档与实际部署一致。
