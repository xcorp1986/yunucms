# AGENTS.md

## 适用范围

- 本文件适用于仓库根目录及其全部子目录；若子目录以后新增更具体的 `AGENTS.md`，以更深层文件为准。
- 这是一个传统 PHP 单体项目。修改前先沿真实请求链路阅读代码，不要套用现代 ThinkPHP 或前后端分离项目的默认假设。

## 项目概览

- 项目为 YUNUCMS 1.1.6，基于内置的 ThinkPHP 5.0 框架。
- `index.php` 定义 `APP_PATH`、`CONF_PATH` 后加载 `system/start.php`；Web 根目录就是仓库根目录。
- 根 `composer.json` 当前没有运行时依赖，ThinkPHP 源码直接保存在 `system/`。不要假设存在完整的 `vendor/` 或现代 Composer 启动流程。
- 框架历史最低要求是 PHP 5.4；当前开发机的 PHP 版本不代表生产基线。除非任务明确升级运行环境，否则保持现有 ThinkPHP 5 API 和兼容语法。
- 项目依赖 MySQL。首次安装使用 `data/install.sql`，并由安装器写入 `config/database.php`、`config/extra/sys.php` 和 `data/install.lock`。
- 若 `data/install.lock` 不存在，前台会跳转到安装流程。不要为了跳过安装而手工伪造锁文件。

## 请求入口与模块

- `index.php`：统一 Web 入口；`router.php` 为 PHP 内置服务器转发脚本。
- `yunu.php`：根据 `sys.url_model` 跳转到后台登录地址。
- `app/index/`：桌面端前台，包括栏目、内容、搜索、标签、表单和安装流程。
- `app/wap/`：移动端前台，与 `index` 模块存在大量平行实现。
- `app/admin/`：后台管理、登录、角色权限、内容/栏目/模型/表单、配置、上传和数据维护。
- `app/api/`：JSON/JSONP 接口；`Master` 和 `V1` 通过 `api_<名称>` 方法分发请求。
- `.htaccess`、`nginx.txt`：前台、移动端和后台的伪静态规则。

典型请求链路为：

1. Web 服务器将请求交给 `index.php`。
2. ThinkPHP 根据模块、控制器和操作分发请求。
3. 前台/WAP 的 `Common` 处理安装状态、地区分站、域名、会话、主题和移动端跳转。
4. 后台的 `Common` 处理登录态、角色权限、菜单和公共视图数据。
5. 控制器通过模型或 `db()` / `Db::name()` 读取数据，再渲染主题模板或返回 JSON。

## 目录职责

- `app/*/controller/`：请求编排和输入处理。
- `app/*/model/`：栏目、内容、地区、表单等数据访问与领域逻辑。
- `app/common/validate/`：后台表单验证规则。
- `app/common.php`、`app/*/common.php`：全局及模块级函数；修改前搜索全部调用方。
- `app/index/taglib/Yunu.php`、`app/wap/taglib/Yunu.php`：主题模板使用的自定义标签库。
- `app/admin/view/`：后台 Think 模板。
- `template/<主题>/index/`、`template/<主题>/wap/`：桌面端和移动端主题；当前默认主题为 `default`。
- `statics/`：后台静态资源、编辑器和公共前端资源。
- `app/extend/`：随项目内置的第三方库和扩展，包括 PHPMailer、七牛、图片处理等。
- `system/`：内置 ThinkPHP 框架源码。
- `config/`：全局、模块、数据库、路由及站点运行配置。
- `data/install.sql`：安装基线结构和初始数据。
- `caches/`：运行时日志、缓存和模板编译产物。
- `uploads/`：上传文件；仓库内已有示例资源，不要无目的批量删除或重写。

## 关键领域约束

- 内容不是只存于 `content` 表。`content.mid` / `content.vid` 会关联 `diymodel`、`diyfield` 以及 `diy_<模型表名>`；修改内容增删改或模型迁移时必须检查两侧数据。
- 自定义表单涉及 `diyform`、`diyfield`、`formcon` 和动态 `form_<表名>`，不能只改后台页面。
- 地区分站同时影响域名识别、会话、URL 生成、栏目/内容筛选以及桌面端/WAP 跳转。
- 主题路径来自 `config('sys.theme_style')`。模板文件名和控制器动作遵循现有 ThinkPHP 模板约定。
- URL 由控制器/模型/公共函数/标签库生成，并受 `sys.url_model`、`.htaccess` 和 `nginx.txt` 共同约束。调整 URL 时要同步核对这些位置。
- `config/extra/sys.php` 和 `config/database.php` 可能被安装器或后台直接改写，应视为运行配置，不要用只适合手工排版的结构破坏其写回流程。

## 修改原则

1. 按“理解 → 验证 → 修改 → 再验证 → 纠错”闭环执行，不停留在计划或静态猜测。
2. 先运行 `git status --short`，确认并保留用户已有改动；只触碰当前任务需要的文件。
3. 优先复用现有控制器、模型、验证器、公共函数和 ThinkPHP 5 写法，不为单一场景新增抽象或依赖。
4. 修改桌面端前台前，检查 WAP 是否有对应实现；需要保持一致时同步修改并分别验证。
5. 修改 `Master` 或 `V1` 的公共 API 前，检查另一个控制器是否也要同步，保持已有字段、状态码和 JSON/JSONP 行为兼容。
6. 除非任务明确要求修复框架或升级第三方库，不修改 `system/`，也不批量格式化 `app/extend/`、`statics/` 内的第三方文件。
7. 不把数据库账号、密码、密钥、令牌、域名、端口或环境专属值写死到新增代码；沿用 `config()` 和现有配置分层。
8. 不随意扩大后台免认证列表、上传类型、跨域范围或文件写权限。权限相关修改必须覆盖未登录、无权限和管理员路径。
9. 修改数据库结构时同步更新 `data/install.sql`，并说明既有实例的迁移方式；不要只让新安装可用。
10. 修复问题要落在共享根因处，但避免顺手重构无关遗留代码。

## 本地运行与验证

启动前先确认 PHP 版本、所需扩展、数据库配置和 `data/install.lock`。开发服务器可在仓库根目录运行：

```bash
php -S 127.0.0.1:8000 router.php
```

按改动范围执行最小但充分的验证：

```bash
php -l path/to/changed.php
git diff --check
git status --short
```

- 每个修改过的 PHP 文件都要执行 `php -l`。
- 前台改动至少验证首页及受影响的栏目、内容、搜索、标签或表单路径。
- WAP 改动验证 `/m/` 或移动域名对应路径，并检查地区分站场景。
- 后台改动验证登录态、权限拒绝、目标操作和返回 JSON；不能只验证超级管理员成功路径。
- API 改动验证成功、无数据、非法参数及 JSONP（若保留）响应结构。
- URL 改动同时用动态 URL 和目标伪静态模式验证，并核对 Apache/Nginx 规则。
- 安装、配置写回、上传、备份或升级流程会修改本地文件或外部状态，测试前先备份并明确影响范围。
- 当前没有可直接运行的应用级自动化测试。`system/phpunit.xml` 属于内置框架的旧测试配置，且对应测试目录和开发依赖不在当前仓库中；未实际恢复并执行前，不得声称 PHPUnit 已通过。

## Git 与交付

- 提交前检查完整差异，移除调试输出、临时文件、缓存、日志和本地运行配置。
- 不提交 `caches/` 产物、`data/install.lock`、真实数据库凭据或未经要求的新上传文件。
- 保持小步、单一目的提交，只暂存本次任务文件，不混入用户的其他改动。
- 提交信息使用中文 Conventional Commits，例如 `docs: 补充项目协作说明`、`fix: 修复移动端栏目跳转`。
- 不使用会覆盖用户改动的破坏性 Git 命令；需要回退或清理时先确认影响范围。
