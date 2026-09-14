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

## 二、部署

当前已通过 **GitHub Pages** 部署，直接访问：

**https://licongxiao19.github.io/vocab-site/**

仓库：`licongxiao19/vocab-site`（master 分支根目录自动发布，推送即更新）。
Supabase 配置已预置在 `index.html` 中，任何设备打开即用，无需手动填写。

## 三、开始使用

1. 手机/电脑浏览器打开上面的网址（手机可用浏览器菜单「添加到主屏幕」，像 App 一样使用）。
2. 到「记录」页开始记单词；「单词」页搜索 / 编辑 / 删除；「复习」页刷卡。
3. 如需更换 Supabase 项目，在「设置」页填入新的 URL 和 anon key 即可覆盖预置配置。

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

