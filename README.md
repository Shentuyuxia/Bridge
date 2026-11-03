# 桥 · Ponte

> © 2026 申屠余夏 (Shentu Yuxia). All rights reserved. Third-party components keep their own licences (`vendor/supabase-LICENSE`, fonts: SIL OFL 1.1).

一个只属于两个语伴的在线语言学习空间：视频通话、按语言分段的计时、共同的词汇表、纠错记录、笔记和课后总结。
A private language-exchange room for two: video call, a timer split by language, shared words, corrections, notes and summaries.

纯网页应用，部署在 GitHub Pages，免费。

## 怎么用

1. 打开网站，填空间名称、你的名字、你在学的语言，点「创建」。
2. 马上会弹出一个**一次性邀请链接**，发给语伴或老师。这个链接只能用一次，7 天内有效：对方点「接受邀请」后，他的设备就记住了钥匙，链接随即失效，别人再打开也进不来。
3. 以后每台设备直接打开网站就进入。换新设备（比如你的手机）时，在空间里点「邀请」再生成一个新链接给自己用。
4. 没有钥匙的设备打开空间链接，只会看到「这台设备还不能进入」。
5. 点「开始视频通话」，两个人都点了以后自动接通。
6. 计时器：默认中文 30 分钟、葡语 30 分钟，可以改先说哪种、每段多长。换语言和结束时会响一下。
7. **说话时长**：通话时，每台设备只听自己的麦克风，算出这个人说了多久，并按计时器的语言时段分开（中文 / 葡语 / 计时开始前算「其他」）。正在说话的人名字前会亮绿点。每一段说满 20 秒以后，会显示「葡语时段里，Emo 说了 62%」这样的比例。「历次学习」里有每次学习你说了多久的图（深色 = 你在学的语言），可以看进步。导出的文件里也有。
8. 「课文」：把课文贴进「编辑」，两个人可以同时修改（Yjs 协同编辑，互不覆盖）。在「阅读 · 标注」里选中一段文字，可以标记、写批注，或者直接加到词汇。「放大课文」让课文区变宽、视频变小。
9. 「笔记」同样是两个人同时编辑。
10. 「新的一次」开始新一次学习；以前的都在「历次学习」里；「导出」可以下载 Word 文件、打印 / 存为 PDF，或者 Markdown 文本。

屏幕共享只能看、不能一起改（所有视频工具都是这样）；要一起改的内容放在「课文」里。

## 技术和隐私

- **视频**：WebRTC，两台设备直接连接（点对点），视频不经过任何服务器。用 Cloudflare 和 Google 的免费 STUN 服务器找到对方。
  如果两边网络都很严格、一直连不上，可以在「历次学习 → 高级设置」里填一个 TURN 中继服务器（例如 Cloudflare Realtime TURN，每月有免费额度）。
- **同步和保存**：用 Supabase（和 Entrelinhas 同一个项目，表 `tandem_rooms`）。
  - 每个空间有一把随机的 256 位钥匙，只存在成员的设备上。从它推导出数据库里的行号（SHA-256）和 AES-GCM 密钥。
  - 一次性邀请：链接里只有一个随机邀请码（不是钥匙）。服务器存的是邀请码的 SHA-256 和用邀请码加密过的钥匙；「接受邀请」时服务器把它交出一次并立即删除（表 `tandem_invites`，函数 `tandem_invite_create` / `tandem_invite_redeem`）。邀请 7 天后过期。
  - 聊天软件生成链接预览时不会用掉邀请：邀请码在 `#` 后面，不会发给任何服务器，而且必须点「接受邀请」才会兑换。
  - 数据库里存的只是加密后的内容；实时消息（同步、视频连接信息）也是加密的。服务器和维护者都读不到。
  - 数据库只通过两个函数读写：`tandem_get(id)` 和 `tandem_put(id, data, rev)`（带版本号，两个人同时改也不会互相覆盖：会先合并再保存）。
- **钥匙只在设备上**：清除浏览器数据或换设备后，需要空间里的另一个人（或你自己的另一台设备）再发一个邀请链接。如果所有设备都丢了钥匙，空间就打不开了。
- Supabase 免费项目如果一周没人用会暂停，到 Supabase 控制台点一下恢复即可，数据不会丢。

## 文件

```
index.html            整个应用
vendor/supabase.js    supabase-js（MIT）
vendor/yjs.min.js     Yjs 协同编辑（MIT，见 vendor/yjs-LICENSE）
vendor/printkit.js    Word / PDF 导出（和 Entrelinhas、字里、栞 共用）
fonts/                Source Sans 3、Source Serif 4（SIL OFL 1.1）
icon.svg, icon-*.png, manifest.webmanifest
```

本地测试：`python3 -m http.server`，打开 `http://localhost:8000/?local=1`（`?local=1` 不连服务器，同一个浏览器的两个标签页互相通话，只用来测试）。

## 数据库（已经建好；新项目才需要运行）

```sql
create table public.tandem_rooms (
  id text primary key check (id ~ '^[0-9a-f]{64}$'),
  data text not null check (octet_length(data) < 4000000),
  rev bigint not null default 1,
  updated_at timestamptz not null default now()
);
alter table public.tandem_rooms enable row level security;
revoke all on public.tandem_rooms from anon, authenticated;
-- 函数 tandem_get / tandem_put：见 Supabase 迁移 "tandem_rooms"
```
