# 桥 · Ponte

> © 2026 申屠余夏 (Shentu Yuxia). All rights reserved. Third-party components keep their own licences (`vendor/supabase-LICENSE`, fonts: SIL OFL 1.1).

一个只属于两个语伴的在线语言学习空间：视频通话、按语言分段的计时、共同的词汇表、纠错记录、笔记和课后总结。
A private language-exchange room for two: video call, a timer split by language, shared words, corrections, notes and summaries.

纯网页应用，部署在 GitHub Pages，免费。

## 怎么用

1. 打开网站，填空间名称和密码（至少 8 个字符），点「创建」，得到一个专属链接（`…/#r=xxxx`）。
2. 把链接发给语伴，密码最好用另一种方式告诉对方。
3. 两个人都打开链接，填名字、选「我在学」，输入密码。勾选「记住我」后，这台设备下次直接进入。
4. 点「开始视频通话」。两个人都点了以后就会自动接通。
5. 计时器：默认中文 30 分钟、葡语 30 分钟，可以改先说哪种、每段多长。换语言和结束时会响一下。
6. 「新的一次」开始新一次学习；以前的都在「历次学习」里；「导出」下载这一次的 Markdown 文件（词汇、纠错、笔记、总结）。

## 技术和隐私

- **视频**：WebRTC，两台设备直接连接（点对点），视频不经过任何服务器。用 Cloudflare 和 Google 的免费 STUN 服务器找到对方。
  如果两边网络都很严格、一直连不上，可以在「历次学习 → 高级设置」里填一个 TURN 中继服务器（例如 Cloudflare Realtime TURN，每月有免费额度）。
- **同步和保存**：用 Supabase（和 Entrelinhas 同一个项目，表 `tandem_rooms`）。
  - 密码只在浏览器里用：PBKDF2（210 000 次）从「链接里的房间号 + 密码」推导出两样东西：数据库里的行号（SHA-256）和 AES-GCM 密钥。
  - 数据库里存的只是加密后的内容；实时消息（同步、视频连接信息）也是加密的。服务器和维护者都读不到。
  - 数据库只通过两个函数读写：`tandem_get(id)` 和 `tandem_put(id, data, rev)`（带版本号，两个人同时改也不会互相覆盖：会先合并再保存）。
- **忘了密码就打不开**，没有找回办法。
- Supabase 免费项目如果一周没人用会暂停，到 Supabase 控制台点一下恢复即可，数据不会丢。

## 文件

```
index.html            整个应用
vendor/supabase.js    supabase-js（MIT）
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
