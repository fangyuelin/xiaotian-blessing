# H5 祝福弹窗页面 — 制作全流程

## 成品

`index.html` — 单个自包含文件（741KB），内含 17 张照片、1 段背景音乐、所有代码。

**使用方式**：部署到任意静态服务器，链接分享到微信/QQ 即可打开。用户点击「✨ 点击开启 ✨」后，背景音乐响起，照片随机满屏弹出。

---

## 一、技术架构

```
┌─────────────────────────────────────┐
│           index.html (741KB)        │
│  ┌───────────────────────────────┐  │
│  │  HTML 结构                    │  │
│  │  ├─ 开始页（标题 + 按钮）     │  │
│  │  └─ 弹窗容器                  │  │
│  ├───────────────────────────────┤  │
│  │  CSS（深色渐变背景 + 弹窗动画）│  │
│  ├───────────────────────────────┤  │
│  │  JavaScript                   │  │
│  │  ├─ 照片随机满屏生成          │  │
│  │  ├─ 弹入/弹出动画             │  │
│  │  ├─ 背景音乐控制（1.25x 速）  │  │
│  │  └─ 微信兼容处理              │  │
│  ├───────────────────────────────┤  │
│  │  17 张照片 (base64 内嵌)       │  │
│  │  背景音乐 (base64 内嵌)        │  │
│  └───────────────────────────────┘  │
└─────────────────────────────────────┘
```

**核心设计决策**：所有资源 base64 内嵌到 HTML，消除外部网络请求。原因——通过隧道/热点分享时，外部文件加载极慢，内嵌后页面加载即所有资源就绪。

---

## 二、工具链

| 工具 | 用途 | 安装方式 |
|------|------|----------|
| **Python 3 + Pillow** | 批量压缩/缩放图片 | `pip install Pillow` |
| **FFmpeg** | 截取、压缩、变速音频 | `winget install ffmpeg` 或官网下载 |
| **cloudflared** | 临时公网隧道（无需注册） | `winget install cloudflared` |
| **Python http.server** | 本地静态服务 | Python 自带 |

---

## 三、制作流程

### 第 1 步：准备原始素材

```
小田的祝福/
├── 弹窗素材/          ← 原始照片（任意尺寸）
│   ├── xxx_854_2.jpg
│   ├── xxx_855_2.jpg
│   └── ...（17张）
└── 背景音乐.mp3       ← 原始音频（任意时长）
```

### 第 2 步：压缩素材（关键！）

H5 分享场景下带宽极其有限（尤其手机热点），必须强力压缩。

**图片压缩** — Pillow 批量处理：

```python
from PIL import Image
import os, glob

for f in glob.glob('弹窗素材/*.jpg'):
    img = Image.open(f)
    w, h = img.size
    new_w = 400                    # 手机屏幕 30vw ≈ 120px，400px 足够
    new_h = int(h * new_w / w)
    img = img.resize((new_w, new_h), Image.LANCZOS)
    img.save(f'compressed/{os.path.basename(f)}', 'JPEG', quality=65, optimize=True)
```

- 原始：15KB ~ 5.6MB/张，合计 38MB
- 压缩后：12KB ~ 47KB/张，合计 ~410KB
- **缩小约 95%**

**音频压缩** — FFmpeg 截取前 33 秒 + 降码率：

```bash
ffmpeg -i 背景音乐.mp3 -ss 0 -t 33 -ac 1 -b:a 32k -ar 22050 compressed/背景音乐.mp3
```

| 参数 | 含义 |
|------|------|
| `-ss 0 -t 33` | 截取 0~33 秒 |
| `-ac 1` | 单声道 |
| `-b:a 32k` | 32kbps 码率 |
| `-ar 22050` | 22.05kHz 采样率 |

- 原始：5.7MB（4分06秒）
- 压缩后：130KB（33秒）
- **缩小约 98%**

### 第 3 步：编写 HTML 页面

**移动端适配要点：**

```html
<meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no, viewport-fit=cover">
```

- `user-scalable=no` 禁止双击缩放
- `viewport-fit=cover` 适配刘海屏安全区域

**卡片尺寸策略：**

```css
.photo-popup {
  width: 30vw;        /* 屏幕宽度的 30%，自适应 */
  max-width: 150px;   /* 大屏上限 */
  min-width: 85px;    /* 小屏下限 */
}
```

**弹窗动画：** `translate(-50%,-50%)` 居中定位 + CSS 变量 `--r` 控制随机旋转。

**音乐播放（关键）：**

```javascript
// 浏览器禁止无手势自动播放 → 必须通过按钮点击触发
document.getElementById('startBtn').addEventListener('click', () => {
    audio.playbackRate = 1.25;  // 1.25 倍速
    audio.play();
});
```

- `playbackRate` 必须在 `play()` 之前设置，否则部分浏览器不生效
- 微信环境需要监听 `WeixinJSBridgeReady` 事件

**照片随机满屏生成：**

```javascript
function randomPos() {
  const vw = window.innerWidth, vh = window.innerHeight;
  const halfW = vw * 0.15;       // 30vw 卡片半宽
  const halfH = halfW * (4/3);   // 3:4 比例半高
  return {
    x: halfW + 2 + Math.random() * (vw - halfW*2 - 4),  // 安全边距
    y: halfH + 2 + Math.random() * (vh - halfH*2 - 4)
  };
}
```

**z-index 递增**确保新卡片覆盖旧卡片：

```javascript
card.style.zIndex = spawnedCount;  // 每次 +1，不循环
```

### 第 4 步：Base64 内嵌（关键！）

将压缩后的图片和音频转换为 data URI，直接写入 HTML：

```python
import base64

# 嵌入图片
for photo in photos:
    with open(photo, 'rb') as f:
        b64 = base64.b64encode(f.read()).decode()
    data_uri = f'data:image/jpeg;base64,{b64}'
    html = html.replace(f'compressed/{name}', data_uri)

# 嵌入音频
with open('compressed/背景音乐.mp3', 'rb') as f:
    b64 = base64.b64encode(f.read()).decode()
data_uri = f'data:audio/mpeg;base64,{b64}'
html = html.replace("new Audio('compressed/背景音乐.mp3?v=' + Date.now())", f"new Audio('{data_uri}')")
```

最终 HTML：**741KB**（17 张照片 + 1 段音频 + 全部代码），完全自包含。

### 第 5 步：部署与分享

**方案 A — 临时公网隧道（推荐测试用）：**

```bash
# 启动本地服务
python -m http.server 8080

# 开启 Cloudflare 隧道（无密码页、无注册）
cloudflared tunnel --url http://localhost:8080
# → https://xxx.trycloudflare.com
```

**方案 B — 永久托管：**

| 平台 | 特点 |
|------|------|
| 腾讯云 EdgeOne Pages | 国内快，免费额度 |
| Cloudflare Pages | 全球 CDN，国内偶尔慢 |
| GitHub Pages | 免费，国内较慢 |

**方案 C — 直接传文件：**

由于只有一个 `index.html` 文件，也可以通过微信「文件传输助手」直接发送，对方用浏览器打开即可。

---

## 四、关键踩坑记录

| 问题 | 原因 | 解决 |
|------|------|------|
| 照片全挤在左上角 | 卡片 `translate(-50%,-50%)` 居中，但 safe area 没算卡片尺寸 | 根据卡片实际像素尺寸计算安全边距 |
| 后弹出的被旧的挡住 | z-index 循环 10~29，新旧颠倒 | z-index 严格递增（`spawnedCount`） |
| 弹窗不铺满整屏 | 卡片太大（40vw），安全范围窄 | 缩小到 30vw，扩大随机范围 |
| BGM 等 10+ 秒才响 | 音频文件 962KB，隧道下载慢 | 截取 33 秒 + 32kbps → 130KB |
| BGM 仍不出 | 隧道延迟高，浏览器放弃缓冲 | base64 内嵌到 HTML，零网络请求 |
| 照片隧道加载慢 | 17 张图逐个请求，带宽瓶颈 | 全部 base64 内嵌 |
| 微信自动播放失败 | 浏览器安全策略 | 开始按钮 → 用户手势触发 `play()` |
| localtunnel 有密码页 | localtunnel 默认防护 | 换 cloudflared，无密码页 |
| `playbackRate` 不生效 | 在 `play()` 之后设置 | 移动到 `play()` 之前 |

---

## 五、可调参数速查

在 `index.html` 中搜索对应变量即可修改：

| 变量 | 默认值 | 作用 |
|------|--------|------|
| `SPAWN_IVAL` | 220 (ms) | 照片生成间隔 |
| `MAX_VISIBLE` | 22 | 屏幕最大同时显示数 |
| `audio.volume` | 0.8 | 音量 (0~1) |
| `audio.playbackRate` | 1.25 | 播放速度 |
| `width: 30vw` | 30vw | 卡片尺寸 |
| `BLESSINGS` 数组 | 30 句 | 祝福语文案 |

---

## 六、文件清单

```
小田的祝福/
├── index.html          ← 成品（744KB，自包含，部署即用）
├── README.md           ← 本文档
├── .gitignore          ← Git 忽略规则
├── .nojekyll           ← GitHub Pages 静态文件标记
├── LICENSE             ← MIT 开源许可证
├── 弹窗素材/           ← 原始照片（源文件保留）
│   └── *.jpg × 17
└── 背景音乐.mp3        ← 原始音频（源文件保留）
```

---

## 七、永久部署方案 — GitHub Pages

### 7.1 隧道方案的三大痛点

最初使用 `cloudflared tunnel` 临时隧道，存在三个致命问题：

| 问题 | 根因 | 表现 |
|------|------|------|
| ① 终端关闭 → 链接失效 | 隧道进程运行在本地电脑，关闭终端即杀死进程 | 白屏，链接不可访问 |
| ② 断网 → 白屏 | 本地电脑是服务器，断网即下线 | 时好时坏，体验不可靠 |
| ③ 免费时效限制 | `trycloudflare.com` 隧道使用时间/带宽均有限制 | 链接不定时失效 |

**核心矛盾**：H5 分享需要 7×24 在线的服务器，但隧道方案把个人电脑当服务器——电脑关机、断网、隧道过期任何一个都会让链接报废。

### 7.2 解决思路

```
问题本质：把 HTML 从"自己电脑上临时托管" → "真正的云服务器上永久托管"
约束条件：免费、无需信用卡、国内能访问、部署简单
```

**方案筛选**：

| 平台 | 费用 | 国内速度 | 实名要求 | 评价 |
|------|------|---------|---------|------|
| Gitee Pages | 免费 | 🚀 快 | ⚠️ 需要 | ❌ 2024年5月已下线 |
| GitHub Pages | 免费 | 可接受 | ❌ 不需要 | ✅ 首选 |
| Cloudflare Pages | 免费 | 中等 | ❌ 不需要 | ✅ 备选 |
| 腾讯 EdgeOne Pages | 免费额度 | 🚀 快 | ⚠️ 需要 | ⚠️ 需实名 |

**最终选择**：GitHub Pages —— 免费、永久、无需实名认证、744KB 的页面在国内 2-3 秒可打开。

### 7.3 部署工具链

| 工具 | 用途 | 安装 |
|------|------|------|
| **Git** | 版本控制，推送代码 | 系统自带 |
| **GitHub CLI** (`gh`) | 命令行管理 GitHub 仓库 | `winget install GitHub.cli` |
| **GitHub Personal Access Token** | API 鉴权（需勾选 `repo` 权限） | GitHub → Settings → Developer settings |

### 7.4 部署步骤

**第 1 步：初始化本地 Git 仓库**

```bash
cd 小田的祝福/
git init
git config user.name "小田的祝福"
git add index.html README.md
git commit -m "初始版本"
```

**第 2 步：GitHub 认证**

```bash
# 安装 GitHub CLI
winget install GitHub.cli

# 创建 Token（需勾选 repo 权限）→ https://github.com/settings/tokens/new
# 用 Token 登录
gh auth login --with-token
```

**第 3 步：创建仓库并推送**

```bash
# 在 https://github.com/new 手动创建公开仓库（不勾任何初始化选项）

# 添加远程并推送
git remote add github https://github.com/fangyuelin/xiaotian-blessing.git
git push -u github master
```

**第 4 步：开启 GitHub Pages（API 方式）**

```bash
curl -X POST \
  -H "Authorization: Bearer <token>" \
  -H "Accept: application/vnd.github+json" \
  "https://api.github.com/repos/fangyuelin/xiaotian-blessing/pages" \
  -d '{"source":{"branch":"master","path":"/"}}'
```

也可以用 Web UI：仓库 → Settings → Pages → Source 选 `master` → Save。

**第 5 步：验证上线**

```bash
curl -s -o /dev/null -w "HTTP: %{http_code}" \
  "https://fangyuelin.github.io/xiaotian-blessing/"
# → HTTP: 200
```

### 7.5 踩坑记录

| 坑 | 原因 | 解决 |
|----|------|------|
| `gh auth login --web` 超时 | GitHub 国内访问慢，浏览器流程中断 | 改用 `--with-token` 方式登录 |
| `gh api` 返回 401 | Token 创建时未勾选任何权限 | 重新生成 Token，勾选 `repo` 权限 |
| `gh api /repos/...` 路径报错 | Git Bash 把 `/repos` 当作本地路径 | 去掉前导斜杠：`repos/...` |
| Web UI 点 Save 后 API 仍 404 | Pages 设置可能未真正保存 | 用 API `POST` 方式一键开启，结果确定性高 |
| 首次部署后 404 | GitHub Pages 第一次构建需要时间 | 等 30-60 秒后重试 |
| Gitee Pages 入口找不到 | Gitee Pages 于 2024年5月正式下线 | 改用 GitHub Pages |

### 7.6 日常更新流程

修改 `index.html` 后：

```bash
git add index.html
git commit -m "更新内容描述"
git push github master
# GitHub 自动重新部署，约 1 分钟后生效
```

### 7.7 效果对比

```
旧方案（cloudflared tunnel）:
  电脑开机 → 启动 http.server → 启动隧道 → 分享链接
  ⚠️ 关机/断网/隧道过期 → 链接失效

新方案（GitHub Pages）:
  推送代码 → GitHub 自动部署 → 分享链接
  ✅ 7×24 在线，永久有效，与你电脑状态完全无关
```
