---
name: siff26-ticket-planning
description: 用于根据任意影片清单与偏好规划 SIFF 2026 买票和观影行程；优先使用内置官方排片数据，并评估片长、散场缓冲、跨影院驾车时间与备选场次。
version: 1.1.0
author: Hermes Agent
metadata:
  hermes:
    tags: [siff, siff2026, film, cinema, movie, festival, tickets, itinerary, routing, schedule]
    category: leisure
---

# SIFF26 买票行程规划

## 适用范围

用于第 28 届上海国际电影节（SIFF 2026）的买票与观影行程规划。输入可以是任意影片清单、豆列、文本、截图、表格、本地文件、聊天记录中的片名，或用户直接给出的偏好；不要限定为豆瓣来源。

本 skill 内置 SIFF 2026 官方排片资料。除非用户明确要求刷新，否则不要重新抓官方排片页，优先读取本地参考文件：

- `references/siff2026/siff2026-official-cndata-20260603-001.json`
- `references/siff2026/第28届上海国际电影节排片表-官方.xlsx`
- `references/siff2026/siff2026-scrape-summary.json`
- `references/siff2026/manifest.json`
- `references/siff2026/siff2026-cinema-coordinates.csv`
- `references/siff2026/siff2026-cinema-driving-matrix-osrm.csv`
- `references/siff2026/siff2026-cinema-metro-matrix-estimated.csv`
- `references/siff2026/siff2026-cinema-nearest-metro-stations.csv`

不要把某次用户的片单、行程、抢票结果、评分快照写入 memory 或 skill；这里只保留可复用流程与官方排片源数据。

## 默认偏好

用户未指定时，按这些默认规则规划，并在输出中标注为默认假设：

- 优先工作日晚上、周五晚上、周末。
- 优先用户明确想看的影片；其次参考豆瓣/IMDb/Letterboxd 等评分与评价人数。
- 每场按 `开始时间 + 片长` 计算散场时间。
- 每场散场后默认加 30 分钟缓冲，再开始移动。
- 同影院连场优先；跨影院只在缓冲充足时安排。
- 高分片和稀缺场次优先，不让低优先级片破坏高优先级片的可行性。
- 若全覆盖会造成高风险转场，给出「高质量子集」和「全覆盖版本」两套方案。

只有当缺失信息会明显改变方案且无法自行查到时才问用户，例如：出发地、必看片/可放弃片、硬性日期、是否接受工作日下午、是否接受深夜返程。

## 标准流程

### 1. 提取与规范化影片

- 从用户输入中抽取片名、原名、英文名、年份、导演等。
- URL 用 `web_extract`、`browser` 或 `terminal` 取内容；截图先做视觉/OCR。
- 对中文名、外文名、别名做匹配，必要时用年份/导演消歧。
- 不确定是否同片时保留候选并标注歧义，不要强行合并。

### 2. 读取 SIFF 2026 排片

- 首选内置 JSON：`references/siff2026/siff2026-official-cndata-20260603-001.json`。
- 字段重点：`nameCn`、`nameEn`、`date`、`weekday`、`stime`、`length`、`cinema`、`hallsName`、`cinemaAddress`、`group`、`filmId`、`remarks`、`showType`、`liveActivity`。
- Excel 可作为人工核对或用户需要附件时的官方原始表。
- 若用户要求刷新，才重新抓官方数据，并说明刷新时间与来源 URL。

### 3. 获取优先级信号

优先级顺序：

1. 用户明确的「必看 / 想看 / 可放弃」。
2. 用户指定的平台评分。
3. 常见评分与评价人数：豆瓣、IMDb、Letterboxd 等。
4. 影展价值：4K 修复、导演回顾、少见格式、见面会、稀缺场次、特殊厅。
5. 时间与交通可行性。

不要编评分。查不到就写「未知」。

### 4. 建模每个场次

为每个候选场次计算：

- `start_time`
- `end_time = start_time + runtime`
- `leave_time = end_time + 30m`
- 时段标签：工作日白天 / 工作日晚 / 周五晚 / 周末白天 / 周末晚 / 深夜
- 是否冲突、是否近冲突、是否有见面会或备注

片长必须从排片字段或可信片源读取；缺失时标注估算，不要装作确定。

### 5. 评估交通与转场

不要再用「近距离 10–25 分钟 / 跨区 25–45 分钟」这类泛化规则替代查询结果。SIFF 2026 的影院间交通参考已经预先计算到参考文件：

驾车：

- `references/siff2026/siff2026-cinema-coordinates.csv`：44 家上海展映影院坐标与坐标来源。
- `references/siff2026/siff2026-cinema-driving-matrix-osrm.csv`：44 家影院两两之间 OSRM 驾车距离与自由流耗时。
- `references/siff2026/siff2026-cinema-routing-manifest.json`：驾车矩阵生成说明与校验哈希。

地铁：

- `references/siff2026/shanghai-metro-stationInfo-metroflow.csv`：上海地铁站点与邻接关系源数据。
- `references/siff2026/siff2026-cinema-nearest-metro-stations.csv`：每家影院最近地铁站、步行距离与步行时间估算。
- `references/siff2026/siff2026-cinema-metro-matrix-estimated.csv`：44 家影院两两之间的地铁移动时间估算。
- `references/siff2026/siff2026-cinema-metro-routing-manifest.json`：地铁矩阵生成说明与校验哈希。

使用方法：

1. 同影院：交通 0 分钟，只看厅间转场和休息缓冲。
2. 跨影院先同时查两张矩阵：
   - 驾车：从 `siff2026-cinema-driving-matrix-osrm.csv` 查 `from_cinema` → `to_cinema` 的 `distance_km` 与 `duration_min_freeflow_osrm`。
   - 地铁：从 `siff2026-cinema-metro-matrix-estimated.csv` 查 `from_cinema` → `to_cinema` 的 `total_metro_transfer_min_est`，并记录最近地铁站。
3. OSRM 驾车耗时是自由流基线，不是实时交通；按离场时段再加交通系数并标注为估算：
   - 工作日 07:30–09:30、17:00–19:30：建议 `max(OSRM×2.0, OSRM+20m)`。
   - 周末商圈高峰 13:00–21:00：建议 `max(OSRM×1.6, OSRM+15m)`。
   - 平峰：建议 `max(OSRM×1.3, OSRM+10m)`。
   - 远郊/停车困难/雨天：额外加 10–20m 风险缓冲。
4. 地铁矩阵是估算值：最近站步行（4.5km/h + 两端各 5m 进出站）+ MetroFlow 站点邻接图最短路；不是实时列车时刻、末班车或拥挤度。深夜场必须额外查末班车或改用驾车。
5. 计算可行性时分别评估驾车与地铁，默认选择风险更低且缓冲更大的方案。

驾车：

`leave_time + traffic_adjusted_drive_time + parking/walking_buffer <= next_start`

地铁：

`leave_time + total_metro_transfer_min_est + platform_wait_or_late_night_buffer <= next_start`

输出跨影院时必须给出：影院 A → 影院 B、驾车 OSRM 距离、驾车自由流耗时、按时段修正后的驾车耗时、地铁最近站、地铁总耗时估算、两种方式各自剩余缓冲、推荐方式、风险等级。

### 6. 生成方案

先满足硬约束，再做优化：

- 不安排时间重叠。
- 不安排缓冲不足的跨影院转场，除非用户明确接受风险。
- 抢票锚点优先：高分、稀缺、周末黄金档、连场结构核心。
- 对每个高优先级片准备备选场次。
- 输出「买票顺序」而不只是时间顺序。

推荐输出结构：

1. 数据来源与假设。
2. 买票优先级。
3. 主行程（日程按天列）。
4. 跨影院转场与风险。
5. 关键备选场次。
6. 可放弃/低优先级项。
7. 如有需要，附 CSV / ICS。

## 输出要求

每个推荐场次至少包含：

- 影片名
- 评分/优先级来源
- 日期、星期、开始时间
- 片长、散场时间、+30m 后可离场时间
- 影院、影厅、地址
- 交通与缓冲说明
- 风险等级：低 / 中 / 高

抢票顺序要说明原因，例如：高分、稀缺、连场关键、替代少、周末黄金档。

## 质量检查

最终答复前逐项检查：

- 推荐影片都来自用户给定清单，或明确标为备选扩展。
- 每个场次能在内置官方排片或刷新后的官方源中找到。
- 散场时间和 30 分钟缓冲已计算。
- 跨影院转场已给出车程估算、缓冲与风险。
- 工作日晚/周末偏好已体现。
- 高优先级片有备选方案。
- 评分、交通、票务规则等当前事实有来源；无法确认的明确标注为估算。

## 常见错误

- 只看开场时间，不算片长与散场后 30 分钟。
- 用直线距离代替驾车时间。
- 为了覆盖低分片，牺牲高分片或稀缺场次。
- 中文片名/外文片名未用年份、导演消歧。
- 把一次性行程结果写进 memory 或 skill。
- 明明已有内置 SIFF 2026 官方排片，却重复抓官方排片页。
