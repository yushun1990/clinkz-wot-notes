# Writing Project State

Last updated: 2026-07-28

## Current Objective

初始化 `clinkz-wot-notes`，建立可供作者和新 AI 长期使用的文章仓库结构，
然后开始第一篇文章：

> WOT-001｜为什么物联网平台不应该从 MQTT Topic 开始设计

## Repository Status

- GitHub repository: `https://github.com/yushun1990/clinkz-wot-notes`
- Default branch: `main`
- At initialization time the repository is empty.
- Starter documentation has been generated locally for the Owner to review and commit.
- No article draft has been accepted yet.
- No Zhihu canonical URL has been registered yet.

## Stable Decisions

- The article repository is separate from the implementation repository.
- The column name is **从零设计一个 Rust WoT Runtime**.
- The subtitle is **ClinkZ-WoT 的架构、实现与 AI 协作开发实录**.
- The implementation repository remains the authority for code and architecture:
  `https://github.com/yushun1990/clinkz-wot`.
- Articles are explanatory and versioned against a specific main-project commit.
- The first season contains 12 priority articles.
- Writing should support both project communication and the Owner's real technical learning.
- AI must not hide understanding gaps by directly producing polished prose.

## Last Inspected Main-project Snapshot

The starter documents were prepared after inspecting the latest available main-project
entry files on 2026-07-28:

- `README.md`
- `AGENTS.md`
- `PROJECT_STATE.md`
- `PLAN.md`

At that inspection point, the main project described itself as:

- a protocol-neutral Rust W3C WoT runtime;
- Servient-orchestrated;
- based on immutable compiled plan sets;
- using Cargo-linked, application-registered Protocol Bindings for v1;
- under active v4.9 architecture closure;
- with M1 and M2 in progress;
- with the property-read handler slice awaiting independent review/admission.

This is only a continuation note. Future AI must re-fetch the main project and must not
treat this snapshot as permanently current.

## Current Writing Queue

1. `WOT-001` — research brief and owner understanding discussion;
2. `WOT-002` — outline only after WOT-001 direction is stable;
3. `WOT-003` — outline only after terminology in WOT-001/002 is consistent.

## Next Safe Actions

1. Commit the starter files to `clinkz-wot-notes`.
2. Confirm or adjust the public README wording and cover image.
3. Run the research-brief workflow for `WOT-001`.
4. Discuss the core argument with the Owner before drafting.
5. Create:
   `articles/01-wot-foundations/001-why-not-start-from-mqtt-topic.md`
6. Record the exact ClinkZ-WoT commit in the article Front Matter.
7. Update this file after the first research brief.

## Open Editorial Questions

- Whether the public article title prefix should always include
  `ClinkZ-WoT 设计笔记 NN`.
- Whether the author wants to publish the full GitHub article before or after the
  corresponding Zhihu post.
- Which content license should be selected for prose and diagrams.

These are editorial/product decisions for the Owner, not technical decisions for AI.

## Continuation Rule

A fresh AI should be able to continue by reading:

1. `AGENTS.md`;
2. this file;
3. `CONTENT_PLAN.md`;
4. `EDITORIAL_GUIDE.md`;
5. `SOURCE_POLICY.md`;
6. the latest ClinkZ-WoT main repository state.
