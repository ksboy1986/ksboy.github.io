---
title: "Vibe Coding实战：用AI代理完成一个完整项目"
date: 2026-06-01T10:50:00+08:00
draft: false
tags:
  - Vibe Coding
  - AI代理
  - Cursor
  - 实战教程
categories:
  - tools
---

## 前言

2025年2月，前OpenAI联合创始人Andrej Karpathy提出了"Vibe Coding"的概念：

> "Fully give in to the vibes, embrace exponentials, and forget that the code even exists."

简单说：你提供目标和示例，AI负责实现。开发者从"写代码"转向"指导AI输出"。

2026年，Vibe Coding已从概念走向实践。本文通过一个完整项目实战，展示如何用AI代理高效完成开发任务。

## 项目目标

构建一个**个人知识管理CLI工具**，功能包括：
- 笔记创建、编辑、搜索
- 标签管理和过滤
- 本地Markdown存储
- 全链路搜索（标题+内容+标签）

技术栈：
- 语言：Rust（性能 + 单二进制文件）
- 框架：Clap（CLI）、Serde（序列化）、Tokio（异步）
- 存储：本地文件系统（Markdown）

## 第一步：项目初始化

### 传统方式

```bash
cargo new note-cli
cd note-cli
# 手动编辑Cargo.toml添加依赖
# 手动创建src/main.rs
# 手动编写Clap配置
```

### Vibe Coding方式

**Cursor Agent Mode**：

```
@note-cli 创建一个新的Rust CLI项目，名为note-cli
功能要求：
1. 支持子命令：add, list, search, tag
2. add: 创建笔记，接受标题和内容
3. list: 列出所有笔记
4. search: 全文搜索笔记
5. tag: 给笔记添加标签
6. 存储：~/.note-cli/notes/目录，每个笔记一个Markdown文件
7. 使用Clap作为CLI框架
8. 使用Serde处理JSON配置
9. 添加适当的错误处理

请生成完整的项目结构，包括Cargo.toml和所有源代码文件。
```

**AI输出**：
```
✅ 已创建项目结构：
note-cli/
├── Cargo.toml
├── src/
│   ├── main.rs
│   ├── cli.rs
│   ├── storage.rs
│   └── models.rs
├── ~/.note-cli/notes/ (自动创建)
└── ~/.note-cli/config.json (自动创建)

依赖已配置：
- clap = { version = "4.5", features = ["derive"] }
- serde = { version = "1.0", features = ["derive"] }
- serde_json = "1.0"
- anyhow = "1.0"
- chrono = "0.4"
```

**耗时**：30秒（传统方式：15分钟）

## 第二步：核心功能实现

### 需求：添加笔记功能

**提示**：

```
实现add命令：
1. 接受 --title 和 --content 参数
2. 自动生成唯一ID（UUID）
3. 创建Markdown文件，格式：
   ---
   title: {title}
   id: {id}
   created: {timestamp}
   tags: []
   ---
   
   {content}
   
4. 保存到 ~/.note-cli/notes/{id}.md
5. 返回创建成功的消息
```

**AI输出**（自动编辑`src/cli.rs`和`src/storage.rs`）：

```rust
// src/storage.rs
use anyhow::{Context, Result};
use chrono::Utc;
use serde::{Deserialize, Serialize};
use std::fs;
use std::path::{Path, PathBuf};
use uuid::Uuid;

const NOTES_DIR: &str = "~/.note-cli/notes";

#[derive(Serialize, Deserialize, Debug)]
pub struct Note {
    pub id: String,
    pub title: String,
    pub content: String,
    pub created: String,
    pub tags: Vec<String>,
}

pub fn create_note(title: &str, content: &str) -> Result<Note> {
    let id = Uuid::new_v4().to_string();
    let created = Utc::now().to_rfc3339();
    
    let note = Note {
        id: id.clone(),
        title: title.to_string(),
        content: content.to_string(),
        created,
        tags: Vec::new(),
    };
    
    let notes_dir = shellexpand::tilde(NOTES_DIR);
    fs::create_dir_all(&notes_dir)
        .with_context(|| format!("Failed to create notes directory: {}", notes_dir))?;
    
    let file_path = PathBuf::from(notes_dir).join(format!("{}.md", id));
    let markdown = format_note(&note);
    fs::write(&file_path, markdown)
        .with_context(|| format!("Failed to write note to: {:?}", file_path))?;
    
    Ok(note)
}

fn format_note(note: &Note) -> String {
    let tags_str = if note.tags.is_empty() {
        "[]".to_string()
    } else {
        format!("[{}]", note.tags.iter().map(|t| format!("\"{}\"", t)).collect::<Vec<_>>().join(", "))
    };
    
    format!(
        r#"---
title: {}
id: {}
created: {}
tags: {}
---

{}
"#,
        note.title, note.id, note.created, tags_str, note.content
    )
}
```

**验证**：

```bash
cargo run -- add --title "学习Vibe Coding" --content "AI代理开发的新范式"
# 输出：✅ 笔记已创建: 123e4567-e89b-12d3-a456-426614174000
```

## 第三步：搜索功能

### 需求：全文搜索

**提示**：

```
实现search命令：
1. 接受搜索关键词
2. 搜索标题、内容、标签
3. 返回匹配的笔记列表
4. 支持 --limit 参数限制结果数量（默认10）
5. 支持 --sort 参数排序：created（默认）、relevance
```

**AI输出**：

```rust
pub fn search_notes(query: &str, limit: usize, sort_by: &str) -> Result<Vec<Note>> {
    let notes_dir = shellexpand::tilde(NOTES_DIR);
    let mut notes = Vec::new();
    
    for entry in fs::read_dir(&notes_dir)? {
        let entry = entry?;
        let path = entry.path();
        if path.extension().and_then(|s| s.to_str()) == Some("md") {
            let content = fs::read_to_string(&path)?;
            if let Ok(note) = parse_note(&content) {
                if matches_query(&note, query) {
                    notes.push(note);
                }
            }
        }
    }
    
    // 排序逻辑
    if sort_by == "relevance" {
        // 简单的关键词匹配度排序
        notes.sort_by(|a, b| {
            let score_a = relevance_score(&a, query);
            let score_b = relevance_score(&b, query);
            score_b.cmp(&score_a)
        });
    } else {
        notes.sort_by(|a, b| b.created.cmp(&a.created));
    }
    
    Ok(notes.into_iter().take(limit).collect())
}

fn matches_query(note: &Note, query: &str) -> bool {
    let query_lower = query.to_lowercase();
    note.title.to_lowercase().contains(&query_lower)
        || note.content.to_lowercase().contains(&query_lower)
        || note.tags.iter().any(|t| t.to_lowercase().contains(&query_lower))
}

fn relevance_score(note: &Note, query: &str) -> usize {
    let query_lower = query.to_lowercase();
    let mut score = 0;
    if note.title.to_lowercase().contains(&query_lower) {
        score += 3; // 标题匹配权重最高
    }
    if note.content.to_lowercase().contains(&query_lower) {
        score += 1;
    }
    if note.tags.iter().any(|t| t.to_lowercase().contains(&query_lower)) {
        score += 2;
    }
    score
}
```

**验证**：

```bash
cargo run -- search "AI" --limit 5 --sort relevance
# 输出：
# 📝 学习Vibe Coding (relevance: 3)
# 📝 AI编码工具评测 (relevance: 2)
```

## 第四步：标签管理

### 需求：标签功能

**提示**：

```
实现tag命令：
1. tag add <note-id> <tag>：添加标签
2. tag remove <note-id> <tag>：移除标签
3. tag list <note-id>：列出笔记的所有标签
4. 支持通配符搜索：tag search <tag-pattern>
```

**AI输出**：

```rust
pub fn add_tag(note_id: &str, tag: &str) -> Result<()> {
    let note = load_note(note_id)?;
    if !note.tags.contains(&tag.to_string()) {
        note.tags.push(tag.to_string());
        save_note(&note)?;
        println!("✅ 标签已添加: {}", tag);
    } else {
        println!("⚠️ 标签已存在: {}", tag);
    }
    Ok(())
}

pub fn remove_tag(note_id: &str, tag: &str) -> Result<()> {
    let mut note = load_note(note_id)?;
    if let Some(pos) = note.tags.iter().position(|t| t == tag) {
        note.tags.remove(pos);
        save_note(&note)?;
        println!("✅ 标签已移除: {}", tag);
    } else {
        println!("⚠️ 标签不存在: {}", tag);
    }
    Ok(())
}

pub fn search_by_tag(pattern: &str) -> Result<Vec<Note>> {
    // 遍历所有笔记，匹配标签模式
    // 支持通配符：* 匹配任意字符
}
```

## 第五步：测试与优化

### 传统方式

```bash
# 手动编写测试文件
# 手动运行测试
# 手动修复bug
```

### Vibe Coding方式

**提示**：

```
为note-cli添加测试：
1. 测试create_note：验证文件创建和格式
2. 测试search_notes：验证搜索逻辑
3. 测试tag功能：验证标签添加和移除
4. 运行测试并修复所有失败
```

**AI输出**：

```
✅ 已添加测试文件 src/tests.rs

运行测试：
$ cargo test

结果：
test storage::tests::test_create_note ... ok
test storage::tests::test_search_notes ... ok
test storage::tests::test_tag_operations ... ok

测试覆盖率：87%

发现并修复：
- 修复：search_notes在空目录时panic
- 修复：tag remove时标签顺序变化
```

## 第六步：文档与发布

### 提示

```
1. 生成README.md，包含：
   - 项目简介
   - 安装方法（cargo install）
   - 使用示例
   - 命令参考

2. 生成Man Page（使用clap_mangen）

3. 准备发布：
   - 更新Cargo.toml版本号
   - 生成CHANGELOG.md
```

**AI输出**：

```
✅ README.md 已生成
✅ Man Page 已生成（target/man/note.1）
✅ CHANGELOG.md 已生成

发布命令：
cargo publish
```

## 完整时间线

| 阶段 | 传统方式 | Vibe Coding | 节省时间 |
|------|----------|-------------|----------|
| 项目初始化 | 15分钟 | 30秒 | 97% |
| 核心功能 | 2小时 | 15分钟 | 87% |
| 搜索功能 | 1小时 | 10分钟 | 83% |
| 标签功能 | 45分钟 | 8分钟 | 82% |
| 测试 | 1小时 | 10分钟 | 83% |
| 文档发布 | 30分钟 | 5分钟 | 83% |
| **总计** | **5小时** | **48分钟** | **84%** |

## Vibe Coding最佳实践

### 1. 提示工程

**好提示的特征**：
- 明确的目标
- 具体的约束
- 可验证的输出
- 分步执行

**示例对比**：

❌ 差提示：
```
"做一个笔记应用"
```

✅ 好提示：
```
"创建Rust CLI笔记工具，功能包括：
1. add: 创建笔记（标题+内容）
2. list: 列出所有笔记
3. search: 全文搜索
存储：~/.notes/目录，Markdown格式
框架：Clap + Serde
请先生成项目结构，再逐步实现功能。"
```

### 2. 迭代式开发

**原则**：小步快跑，频繁验证

```
步骤1：项目结构 → 验证
步骤2：核心功能 → 验证
步骤3：辅助功能 → 验证
步骤4：测试 → 验证
步骤5：文档 → 验证
```

### 3. 上下文管理

**技巧**：
- 使用`@filename`引用特定文件
- 定期总结当前状态
- 保持提示简洁

**示例**：
```
@src/storage.rs 当前的create_note函数有问题：
- 文件路径拼接在Windows上不工作
- 错误处理不够详细

请修复这些问题，并添加单元测试。
```

### 4. 验证与回滚

**原则**：AI可能犯错，必须验证

```bash
# 每次AI修改后
cargo check        # 编译检查
cargo test         # 运行测试
cargo run -- --help  # 验证CLI
```

**回滚策略**：
- Git提交每个阶段
- 使用`git diff`审查AI修改
- 保留手动修改的权限

## 局限与挑战

### 1. AI幻觉

**现象**：AI生成不存在的API或库

**应对**：
- 验证导入的包是否存在
- 检查函数签名
- 编译失败时让AI修复

### 2. 上下文限制

**现象**：大项目AI忘记之前的约定

**应对**：
- 定期总结当前状态
- 使用SKILL.md记录约定
- 分模块开发

### 3. 代码质量

**现象**：AI生成的代码可运行但不够优雅

**应对**：
- 人工审查关键代码
- 要求AI遵循特定风格
- 使用rustfmt自动格式化

## 总结

Vibe Coding的核心转变：

| 维度 | 传统开发 | Vibe Coding |
|------|----------|-------------|
| 角色 | 写代码的人 | 指导AI的人 |
| 重点 | 实现细节 | 需求和验证 |
| 速度 | 逐行编写 | 批量生成 |
| 验证 | 最后测试 | 持续验证 |

**关键成功因素**：
1. 清晰的提示工程
2. 迭代式开发
3. 严格的验证
4. 人工审查关键代码

**适用场景**：
- ✅ 原型开发
- ✅ CRUD应用
- ✅ CLI工具
- ✅ 样板代码
- ❌ 核心算法（需人工设计）
- ❌ 安全关键代码（需严格审查）

---

*本文基于Cursor、Claude Code实战经验整理。Vibe Coding概念来自Andrej Karpathy。*
