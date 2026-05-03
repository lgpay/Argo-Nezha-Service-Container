# 固定 v0.20.13 的 Argo-Nezha / nezha-dashboard CDN 故障修复记录

本文记录 `lgpay/Argo-Nezha-Service-Container` 与 `lgpay/nezha-dashboard` 两个旧库在固定运行哪吒面板 `v0.20.13` 时，解决前端 CDN 加载失败的有效过程。

## 背景

线上面板：`https://tz.flag.tk/`

现象：面板页面可打开，但部分主题样式、图标或脚本加载异常。浏览器网络请求/页面源码中可见旧 CDN：

```text
lf6-cdn-tos.bytecdntp.com
```

该域名访问不稳定，导致以下资源在旧主题里失败：

- Font Awesome
- Vue 2.6.14
- jQuery 3.6.0
- SweetAlert2
- Semantic UI
- MDUI
- font-logos
- bootstrap-icons
- flag-icon-css 国旗资源

本项目不是升级哪吒，而是**固定使用 `v0.20.13`**，所以修复点必须落在 `v0.20.13` 的构建与下载链路上。

## 两个仓库的角色

### 1. `lgpay/nezha-dashboard`

用途：维护固定 `v0.20.13` 的定制 dashboard release 包。

关键产物：

```text
https://github.com/lgpay/nezha-dashboard/releases/download/v0.20.13/dashboard-linux-amd64.zip
https://github.com/lgpay/nezha-dashboard/releases/download/v0.20.13/dashboard-linux-arm64.zip
```

Actions workflow：

```text
.github/workflows/update-geoip.yml
```

作用：

1. checkout 固定 tag `v0.20.13`；
2. 可选更新 `pkg/geoip/geoip.db`；
3. 编译 dashboard；
4. 上传 `dashboard-linux-amd64.zip` / `dashboard-linux-arm64.zip` 到 release。

### 2. `lgpay/Argo-Nezha-Service-Container`

用途：容器/脚本部署 Argo + Nezha 服务。

关键点：启动或更新 dashboard 时，根据 `DASHBOARD_VERSION` 下载对应 dashboard release 包。

对于固定版本 `v0.20.13`，应下载 `lgpay/nezha-dashboard` 的定制 release 包，而不是原始上游包。

## 根因

`v0.20.13` 原始源码模板中仍引用：

```text
https://lf6-cdn-tos.bytecdntp.com/cdn/expire-1-y/...
```

即使 `Argo-Nezha-Service-Container` 的启动脚本里有 CDN 替换逻辑，如果下载到的 dashboard 二进制本身来自未修补的 release，页面仍可能继续输出旧 CDN URL。

因此有效修复是：

> 在 `lgpay/nezha-dashboard` 构建 `v0.20.13` release 前，对 `resource/template` 执行 CDN patch，然后重新发布 release 附件。

## 已采用的有效修复方案

### Step 1：在 `nezha-dashboard` 增加 CDN patch 脚本

新增 Python 脚本：

```text
scripts/patch-cdn.py
```

脚本处理目录：

```text
resource/template
```

替换规则：

```text
lf6-cdn-tos.bytecdntp.com -> cdn.jsdelivr.net
```

覆盖资源包括：

```text
font-awesome 6.0.0
vue 2.6.14
jquery 3.6.0
sweetalert2 11.4.4
semantic-ui 2.4.1
mdui 1.0.2
font-logos 0.17
bootstrap-icons 1.8.1
flag-icon-css 4.1.5
```

脚本会在 patch 后检查是否仍残留：

```text
lf6-cdn-tos.bytecdntp.com
```

如果仍残留则退出失败，避免构建出未修干净的 release。

### Step 2：修改 `nezha-dashboard` 的 release workflow

文件：

```text
.github/workflows/update-geoip.yml
```

核心改动：

```yaml
permissions:
  contents: write
  actions: read
```

构建流程变为：

```yaml
- name: Checkout fixed tag
  uses: actions/checkout@v4
  with:
    fetch-depth: 0
    ref: ${{ env.FIXED_TAG }}

- name: Checkout patch files
  uses: actions/checkout@v4
  with:
    ref: master
    path: patch-files

- name: Apply CDN fallback patch
  run: |
    python3 patch-files/scripts/patch-cdn.py resource/template
```

原因：workflow checkout 的是固定 tag `v0.20.13`，该 tag 本身不包含后续 master 的补丁脚本，所以需要额外 checkout master 到 `patch-files` 目录，再用其中的 patch 脚本修补固定 tag 的源码。

### Step 3：让 GeoIP 更新变成可选

原 workflow 在 `IPINFO_TOKEN` 为空时会失败，导致 release 构建无法继续。

修复为：

```yaml
- name: Fetch IPInfo GeoIP Database
  env:
    IPINFO_TOKEN: ${{ secrets.IPINFO_TOKEN }}
  run: |
    if [ -z "$IPINFO_TOKEN" ]; then
      echo "IPINFO_TOKEN is empty, keep existing pkg/geoip/geoip.db"
      exit 0
    fi

    rm -f pkg/geoip/geoip.db
    wget -qO pkg/geoip/geoip.db \
      "https://ipinfo.io/data/free/country.mmdb?token=${IPINFO_TOKEN}"
```

这样没有 GeoIP token 时不会影响 CDN 修复和 release 构建。

### Step 4：重新触发 Actions 并发布 release

成功 workflow run：

```text
https://github.com/lgpay/nezha-dashboard/actions/runs/25286763569
```

结果：

```text
completed / success
```

重新发布的 release：

```text
https://github.com/lgpay/nezha-dashboard/releases/tag/v0.20.13
```

新附件：

```text
dashboard-linux-amd64.zip
dashboard-linux-arm64.zip
```

验证新 amd64 包：

```text
bytecdntp 0
jsdelivr 40
```

说明新 dashboard 二进制中已无：

```text
lf6-cdn-tos.bytecdntp.com
```

## Argo-Nezha-Service-Container 侧需要注意的点

`Argo-Nezha-Service-Container` 的有效职责是**拉取正确的 dashboard release 包**。

固定 `v0.20.13` 时，下载源应该指向：

```text
lgpay/nezha-dashboard/releases/download/v0.20.13/dashboard-linux-$ARCH.zip
```

如果脚本中指向不存在的 tag，例如：

```text
v0.20.13-lts
```

而 `lgpay/nezha-dashboard` 没有该 release，则会触发 fallback，可能下载到其它项目或未修补的包，导致 CDN 问题再次出现。

因此建议统一使用真实存在并已修补的 release：

```text
v0.20.13
```

检查位置：

```text
init.sh
dashboard.sh
template/backup.sh
```

重点搜索：

```bash
grep -R -n "v0.20.13-lts\|nap0o/nezha-dashboard\|lgpay/nezha-dashboard\|dashboard-linux" \
  init.sh dashboard.sh template README.md README_EN.md
```

## 验证方法

### 1. 验证 release 包内部没有旧 CDN

```bash
tmp=$(mktemp -d)
cd "$tmp"
curl -L -sS -o dashboard-linux-amd64.zip \
  https://github.com/lgpay/nezha-dashboard/releases/download/v0.20.13/dashboard-linux-amd64.zip

python3 - <<'PY'
import zipfile
with zipfile.ZipFile('dashboard-linux-amd64.zip') as z:
    for name in z.namelist():
        data = z.read(name)
        print('bytecdntp', data.count(b'lf6-cdn-tos.bytecdntp.com'))
        print('jsdelivr', data.count(b'cdn.jsdelivr.net'))
PY
```

期望：

```text
bytecdntp 0
jsdelivr > 0
```

### 2. 验证线上页面

```bash
curl -k -L -s https://tz.flag.tk/ | grep -o 'lf6-cdn-tos.bytecdntp.com\|cdn.jsdelivr.net' | sort | uniq -c
```

期望：不再出现 `lf6-cdn-tos.bytecdntp.com`。

### 3. 验证容器/服务已重新拉取新包

如果容器启动时会自动下载 dashboard，重启容器即可：

```bash
docker restart <container-name>
```

如果存在本地缓存，需要先删除旧 dashboard 二进制或缓存包，再重启。

## 排坑记录

### 1. 修改 workflow 需要 token 有 workflow 权限

如果 GitHub token 没有 workflow 权限，推送 `.github/workflows/*.yml` 会失败：

```text
refusing to allow an OAuth App to create or update workflow without workflow scope
```

fine-grained token 至少需要：

```text
Contents: Read and write
Workflows: Read and write
Actions: Read and write
Metadata: Read-only
```

### 2. 不要把 token 发到聊天或日志

如果 token 已暴露，立即 revoke，重新生成。

### 3. `goreleaser/goreleaser-cross:v1.23` 容器没有 node

第一次用 JS 脚本执行：

```bash
node patch-files/scripts/patch-cdn.js resource/template
```

失败：

```text
node: not found
```

所以最终改为 Python 脚本：

```bash
python3 patch-files/scripts/patch-cdn.py resource/template
```

### 4. `IPINFO_TOKEN` 为空不应阻塞 CDN 修复

GeoIP 数据库只影响 IP 国家/地区识别、国旗等显示，不影响 CPU、内存、流量、在线状态等核心监控。

因此 token 为空时应跳过更新，而不是让整个 release 失败。

### 5. 只改 master 模板不够

因为 workflow 固定 checkout：

```text
v0.20.13
```

所以 master 上直接改 `resource/template` 不会进入 release 构建。必须在 workflow checkout 固定 tag 后执行 patch。

## 后续维护建议

1. 保持 `v0.20.13` release tag 不变，但允许 Actions 刷新同名 release 附件。
2. 所有针对旧面板的前端 CDN 修复，都放进 `scripts/patch-cdn.py`。
3. `Argo-Nezha-Service-Container` 只负责下载 `lgpay/nezha-dashboard` 已修补 release，不再依赖运行时替换二进制内模板。
4. 如果以后新增主题或资源，再扩展 `scripts/patch-cdn.py` 的替换表，并让脚本的残留检测兜底。
5. 每次发布后，用 `bytecdntp 0` 的方式验证 release 包。
