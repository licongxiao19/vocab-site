# 单词本（移动端单词记录网页）

一个单文件 HTML 应用（`index.html`，无构建、无依赖），手机优先设计，数据存 Supabase 实现跨设备同步，可部署到 Netlify 当 App 用。

## 功能

- **记录**：输入单词 + 注释（必填）；词性 / 中文意思 / 例句按需展开填写，不需要可完全不填
- **自动合并**：同一个单词（忽略大小写）重复记录时，新注释自动归到该单词下，按时间区分
- **检索**：按单词、注释内容、中文意思、例句即时搜索
- **复习**：翻面单词卡（正面单词 → 点击翻面看全部注释），可标记「认识 / 不认识」、随机乱序、只刷不认识
- **备份**：一键导出 / 导入 JSON

## 一、创建 Supabase 数据库（一次性，约 5 分钟）

1. 打开 [supabase.com](https://supabase.com) 注册并新建一个免费项目（区域任选，记住数据库密码）。
2. 进入项目 → 左侧 **SQL Editor** → 新建查询，粘贴下面整段 SQL 并运行：

```sql
-- 单词表
create table words (
  id uuid primary key default gen_random_uuid(),
  word text not null,
  unknown_count int not null default 0,
  created_at timestamptz not null default now()
);

-- 忽略大小写的唯一约束：Apple / apple 视为同一词
create unique index words_word_lower_key on words (lower(word));

-- 注释表（一条注释对应一次记录，可带可选字段）
create table notes (
  id uuid primary key default gen_random_uuid(),
  word_id uuid not null references words(id) on delete cascade,
  note text not null,
  pos text,
  meaning text,
  example text,
  created_at timestamptz not null default now()
);

create index notes_word_id_idx on notes(word_id);

-- 开启行级安全，并允许匿名读写（个人单用户词库）
alter table words enable row level security;
alter table notes enable row level security;

create policy "anon all words" on words
  for all to anon using (true) with check (true);
create policy "anon all notes" on notes
  for all to anon using (true) with check (true);
```

> ⚠️ 说明：以上策略意味着**任何拿到 anon key 的人都能读写这份数据**。个人背单词用途一般可接受（anon key 本就公开在网页里，且数据不敏感）。如果介意，可定期在设置页导出 JSON 备份，或后续为项目接入 Supabase Auth 登录。

3. 进入项目 → **Settings → API**，复制：
   - **Project URL**（形如 `https://xxxx.supabase.co`）
   - **anon public key**（`eyJ...` 开头的长字符串）

## 二、部署到 Netlify

两种方式任选：

- **拖拽部署（最快）**：打开 [app.netlify.com/drop](https://app.netlify.com/drop)，把包含 `index.html` 的整个文件夹拖进去即可。
- **Git 部署**：把本目录推到 GitHub 仓库，在 Netlify「Add new site → Import an existing project」选择该仓库。无需构建命令，发布目录留空（即根目录）。

部署后得到一个固定网址，手机浏览器打开 → 菜单「添加到主屏幕」，即可像 App 一样使用。

## 三、开始使用

1. 打开网址 → 右下角「设置」页 → 填入 Project URL 和 anon key → 「保存配置并连接」。
2. 每台设备（手机 / 电脑）都填一次同样的配置，数据自动共用。
3. 到「记录」页开始记单词；「单词」页搜索 / 编辑 / 删除；「复习」页刷卡。

## 本地调试

直接双击打开 `index.html` 即可，或在目录下运行任意静态服务器，例如：

```
npx serve .
```

## 数据说明

- 单词合并按忽略大小写匹配，显示保留首次录入的拼写。
- 「认识 / 不认识」按单词维度计数（`unknown_count`），点「认识」清零。
- 每次成功联网拉取后会缓存到浏览器，断网时只读查看，写入需联网。
- 建议定期在设置页「导出全部数据」做备份。
