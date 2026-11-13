# 🤖 3D 机器人个人导航站 · 搭建教程

> ## 🌐 我的个人网站：[https://l.lrbilool.kdns.fr/](https://l.lrbilool.kdns.fr/)
>
> **这篇教程搭的就是上面这个网站** —— 左边一只会动的 3D 机器人，右边一块毛玻璃金色面板，把项目、社区和联系方式全收在一页里。先打开成品看效果，喜欢再往下搭。

![在线演示](https://img.shields.io/badge/在线演示-l.lrbilool.kdns.fr-e8c268?style=for-the-badge)

## ✨ 特点

- 🆓 **全免费**：跑在 Cloudflare Workers 免费版上，无服务器、无数据库
- 📄 **单文件**：整个网站就是一个网页文件，部署 = 粘贴一次代码
- 🤖 **3D 机器人**：WebGL 实时渲染，能拖动旋转、滚轮缩放，Idle / Run / Dance / Wave 四种动作可切换
- 🔗 **链接面板**：右侧毛玻璃面板收纳全部链接，图标 + 标题 + 副标题，一目了然
- 📱 **自适应**：电脑上左右并排，手机上自动切换成上下布局

## 🧠 一、整体思路

```
3D 角色网页（底版） + 外壳样式 + 导航面板
        = 一个完整的 index.html
        → 整页打包进一个 Cloudflare Worker
        → 绑定自定义域名，上线
```

三层各管一件事，互不干扰：

| 层 | 管什么 |
|---|---|
| 底版 | 机器人本体：模型、动画、渲染 |
| 外壳 | 页面长相：背景、标题、动作按钮 |
| 面板 | 你的全部链接：分区 + 卡片 |

部署时把整个页面塞进一个 Worker 脚本里（推荐 base64 内嵌，理由见第四节），Worker 对任何路径都返回这一页，另外留一个 `/health` 路径做健康检查。就这么简单，没有后端。

## 🛠 二、准备工作

1. 一个 [Cloudflare](https://dash.cloudflare.com/) 账号（免费版够用）
2. 一个 3D 角色网页当底版。常见做法：用 [Three.js](https://threejs.org/) 加载 [Mixamo](https://www.mixamo.com/) 的免费角色和动画（Idle、Running、Dancing、Waving 正好对应四种动作），再用 OrbitControls 实现拖动旋转和滚轮缩放。这一步网上教程很多，本文不展开；**任何现成的 WebGL 单页都能当底版**
3. （可选）一个域名，免费域名也行，能把 DNS 托管到 Cloudflare 即可

## 🎨 三、加外壳和导航面板

底版之外，你只需要往页面里加三样东西：左上角站名标题、左下角动作按钮、右侧面板。结构骨架：

```html
<canvas id="c"></canvas>                    <!-- 机器人画布，全屏固定 -->
<div id="hud">站名<small>PERSONAL NAVIGATION</small></div>
<div id="controls">动作按钮（Idle / Run / Dance / Wave）</div>

<aside id="nav">                            <!-- 右侧链接面板 -->
  <div class="profile">头像 · 名字 · 一句话介绍</div>
  <h2>核心入口</h2>
  <a class="card" href="https://example.com">
    <span class="ic">🤖</span>
    <span class="tx"><span class="tt">卡片标题</span><div class="ss">副标题说明</div></span>
    <span class="ar">›</span>
  </a>
  <!-- 更多卡片照此复制… -->
</aside>
```

面板好看的关键就几条 CSS：

```css
#c   { position:fixed; inset:0; width:100vw; height:100vh; }              /* 画布铺满 */
#nav { position:fixed; top:22px; right:22px; bottom:22px; width:402px;    /* 右侧固定栏 */
       overflow-y:auto; border-radius:22px; padding:26px 22px;
       background:linear-gradient(165deg, rgba(22,28,42,.88), rgba(13,17,23,.82));
       backdrop-filter:blur(18px);                                        /* 毛玻璃 */
       border:1px solid rgba(212,175,55,.30); }                           /* 金色细边 */
.card{ display:flex; align-items:center; gap:12px; padding:11px 13px;
       border-radius:14px; background:rgba(255,255,255,.035); }           /* 卡片 */
a.card:hover { background:rgba(232,194,104,.09); transform:translateX(-3px); }
@media (max-width:900px){ #nav{ top:auto; left:12px; right:12px; width:auto; max-height:54vh; } }
```

链接全部写在面板这一段里，一行一张卡，增删改都在这里 —— 这就是以后日常维护唯一要动的地方。

## 📦 四、打包成 Worker

为什么用 base64 内嵌：成品页面有 1 MB 以上，里面全是引号和模板字符串，直接塞进 JS 字符串极易转义出错；转成 base64 后就是纯文本，怎么粘贴都不会坏。

生成 base64（一行 Python 就够）：

```python
import base64, pathlib
print(base64.b64encode(pathlib.Path("index.html").read_bytes()).decode())
```

Worker 本体非常短：

```js
const HTML_B64 = "……把上一步生成的 base64 粘贴到这里……";

export default {
  async fetch(request) {
    if (new URL(request.url).pathname === "/health") return new Response("ok");
    const bin = atob(HTML_B64);
    const bytes = Uint8Array.from(bin, ch => ch.charCodeAt(0));
    return new Response(bytes, {
      headers: { "content-type": "text/html; charset=utf-8", "cache-control": "public, max-age=300" }
    });
  }
};
```

## 🚀 五、部署（两种方式选一个）

**方式 A：后台粘贴（最简单）**

1. Cloudflare 后台 → **Workers & Pages** → **Create Worker**，起个名字
2. **Edit code**：删掉默认代码，把 Worker 全文粘贴进去 → **Deploy**
3. 打开分配给你的 `https://<名字>.<你的子域名>.workers.dev`，看到机器人和面板就是成功

**方式 B：Wrangler 命令行**

```toml
# wrangler.toml
name = "personal-nav"
main = "worker.js"
compatibility_date = "2025-01-01"
```

```bash
npm install -g wrangler
wrangler login
wrangler deploy
```

## 🌍 六、绑定自定义域名

1. 域名先托管到 Cloudflare（后台 **Websites** 添加站点，按提示改 DNS）
2. 进入你的 Worker → **Settings → Domains & Routes → Add → Custom Domain**
3. 填完整子域名（例如 `l.example.com`），等几分钟证书自动签发完成

绑好后如果想只留正式入口，可以在同一页把 workers.dev 入口关掉 —— 本站就是这样，只保留一个正式域名。

## 🔧 七、日常维护

改链接只需要三步，约 2 分钟：

1. 改面板里的卡片行（地址 / 图标 / 标题 / 副标题）
2. 重新生成 base64、更新 Worker 代码
3. 重新部署

页面设了 5 分钟缓存，改完想立刻验证就开无痕窗口看。

## ❓ 八、常见问题

- **机器人不显示、只有面板？** 机器人是 WebGL 渲染的，浏览器太旧或禁用了 WebGL 就看不到，换 Chrome / Edge。另外打印或导出 PDF 时机器人也不会出现，那是正常现象，不代表网站坏了。
- **粘贴代码时编辑器很卡？** 打包后的文件 1~2 MB 是正常体积（整页 base64），等它粘完再点部署；嫌卡就用 Wrangler。
- **部署报大小超限？** 免费版的限制看的是压缩后大小（1 MB），base64 文本压缩率很高，一般不会触线。
- **部署后还是旧内容？** 缓存还没过期，等 5 分钟或用无痕窗口。

## ⭐ 九、成品与作者

> 🌐 **个人网站：[https://l.lrbilool.kdns.fr/](https://l.lrbilool.kdns.fr/)** —— 本教程的成品，机器人和全部链接都在里面，欢迎来逛。
>
> 👨‍💻 作者 GitHub：[https://github.com/Armorys](https://github.com/Armorys)
>
> 觉得有用的话，给个 **⭐ Star** 支持一下，也欢迎照着教程搭一个你自己的。

---

## English Summary

A single-page personal navigation site with a live 3D robot (Three.js / WebGL, four switchable animations) and a frosted-glass link panel. The whole page is embedded into one Cloudflare Worker script (base64) and served on a custom domain — free tier, no backend, no database. **Live demo (my personal website): [https://l.lrbilool.kdns.fr/](https://l.lrbilool.kdns.fr/)**
