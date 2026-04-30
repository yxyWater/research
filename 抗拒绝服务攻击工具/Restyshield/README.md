# Nginx-Lua-Anti-DDoS（OpenResty）部署步骤

## 环境

- 使用 **OpenResty**（内置 Nginx + Lua，无需自行编译 Nginx+Lua）
- 项目核心脚本：`anti_ddos_challenge.lua`（需放在 Nginx 可加载的路径下）

## 1. 安装

1. 安装 **OpenResty**。

## 2. 放置 Lua 文件

1. 在 OpenResty 的 `conf` 目录下新建文件夹 **`lua`**。
2. 将 GitHub 仓库中的 **Lua 文件** 复制到该 `lua` 目录。

## 3. 编辑主配置（`nginx.conf` 或包含的配置）

在 **`http { ... }`** 块中增加：

### 共享内存（必须，否则脚本报错或功能异常）

在 `http` 内添加（名称需与脚本一致）：

```nginx
lua_shared_dict antiddos 10m;
lua_shared_dict antiddos_blocked 10m;
lua_shared_dict ddos_counter 10m;
lua_shared_dict jspuzzle_tracker 10m;
```

### 每个请求先执行反 DDoS 脚本

在 `http` 内（或仅在被保护的 `server` 内，见下文）添加，路径按实际放置位置修改：

```nginx
access_by_lua_file /path/to/openresty/conf/lua/anti_ddos_challenge.lua;
```

- 写在 **`http`**：本实例下监听 80（等）的站点都会经过该逻辑。
- 只写在某个 **`server`**：仅该站点走脚本；**`lua_shared_dict` 仍放在 `http`**，一般不必挪。

## 4. 检查并重载/启动

1. 在 **CMD 或 PowerShell** 中执行 OpenResty/Nginx 的检查与启动方式（以你本机 OpenResty 安装目录下的 `nginx.exe` 为准）：
   - 测试配置：`nginx -t`
   - 启动/重载：按你安装包说明执行（如 `nginx -s reload`）。
2. 浏览器访问 **`http://127.0.0.1`**，确认能打开页面（服务已监听 80 等端口）。

## 5. 验证 Lua 是否生效

1. 使用浏览器**无痕/隐私模式**访问 `http://127.0.0.1`。
2. 按脚本与项目说明观察是否出现**带 JavaScript 的验证页**、Cookie 等行为（与“仅显示 OpenResty 欢迎页”区分时以项目配置为准）。

## 6. 改配置后

1. 修改 `nginx` 配置或 Lua 后执行 **`nginx -t`** 确认无误，再 **`nginx -s reload`** 重载。

## 7. 白名单等业务配置

1. 在 **`anti_ddos_challenge.lua`** 或项目文档指定位置，按需要配置**白名单**等项；改完后**重载 Nginx**。

---

*说明：该方案针对应用层（L7）限流、挑战页等，不能覆盖所有 DDoS 类型。*
