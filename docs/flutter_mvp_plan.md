# Flutter 智能推题与笔记应用 MVP 实施方案

## 1. 目标与范围
- **目标设备**：Android 平板（API 26+），横屏体验优先。
- **MVP 范围**：完成题目推送→草稿撰写→总结归档→标签关联→历史回显的闭环；支持基础的日程提醒与本地数据持久化。
- **不在本阶段实现**：云同步、多端协同、AI 摘要建议，仅预留扩展接口。

## 2. 推荐技术栈
| 层级 | 技术/库 | 说明 |
| --- | --- | --- |
| 状态管理 | [Riverpod](https://riverpod.dev) + `StateNotifier` | 解耦依赖、便于在背景任务和 UI 间共享状态。 |
| 路由 | [GoRouter](https://pub.dev/packages/go_router) | 支持声明式导航与深度链接，便于根据通知直接打开刷题页。 |
| 本地数据库 | [Drift](https://drift.simonbinder.eu/)（基于 `sqlite3`） | 类型安全、支持复杂查询、内置迁移；如需极简可换用 `sqflite`。 |
| 定时任务 | [`workmanager`](https://pub.dev/packages/workmanager) + [`android_alarm_manager_plus`](https://pub.dev/packages/android_alarm_manager_plus`) | WorkManager 负责周期性计划；AlarmManager 负责精确唤起与前台通知。 |
| 加密与存储 | `flutter_secure_storage` 保存密钥，数据库使用 `sqlcipher` 或在表层做字段加密。 |
| UI 构建 | Flutter 3.x + 自定义响应式布局组件 | 适配横屏的左右分栏、可拖拽分隔。 |
| 依赖注入 | Riverpod Provider 容器 | 统一初始化仓库、服务与 UseCase。 |

## 3. 架构概览
采用 **分层架构**：
1. **Data 层**：Drift `Table` + DAO + Repository。处理 SQLite 读写、序列化、缓存。
2. **Domain 层**：用例（UseCase）封装业务流程（例如 `PushNextQuestionUseCase`、`SaveDraftUseCase`）。
3. **Presentation 层**：Riverpod Provider（ViewModel）驱动页面 Widget。

数据流向：
`Widget` → `ViewModel(StateNotifier)` → `UseCase/Repository` → `DAO` → `SQLite`，并通过 `Stream`/`Notifier` 反向驱动 UI 更新。

## 4. 数据库模型与访问
### 4.1 数据表
沿用需求中的核心表，推荐使用 Drift 定义：
```dart
class Questions extends Table {
  IntColumn get id => integer().autoIncrement()();
  TextColumn get questionId => text().unique()();
  TextColumn get material => text()();
  TextColumn get options => text()(); // JSON
  TextColumn get answer => text()();
  TextColumn get explanation => text().nullable()();
  TextColumn get tags => text().map(const StringListConverter()).nullable()();
  IntColumn get completedCount => integer().withDefault(const Constant(0))();
  DateTimeColumn get createdAt => dateTime().withDefault(currentDateAndTime)();
  DateTimeColumn get updatedAt => dateTime().nullable()();
}
```
`Drafts`、`Summaries`、`Tags`、`SummaryTags` 表可直接按需求 SQL 转换成 Drift 定义，新增索引：
- `CREATE INDEX idx_drafts_question ON drafts(question_id);`
- `CREATE INDEX idx_summaries_question ON summaries(question_id);`
- `CREATE INDEX idx_summary_tags_tag ON summary_tags(tag_id);`

### 4.2 数据访问层
- 为每张表生成 `DAO`：`QuestionDao`, `DraftDao`, `SummaryDao`, `TagDao`, `SummaryTagDao`。
- 建立 Repository 组合查询：
  - `QuestionRepository`：题目分页、随机抽取、更新完成次数、查询历史总结。
  - `DraftRepository`：保存草稿版本、获取最新草稿、统计完成次数。
  - `SummaryRepository`：保存总结、加载与标签、学科的关联。
  - `TagRepository`：维护树结构，提供递归查询与路径构造。
- 所有写操作包裹在 `transaction` 中，保证草稿-总结-计数一次提交。

### 4.3 数据迁移策略
- Drift `MigrationStrategy`：在 `onUpgrade` 中按版本增量执行 `ALTER TABLE` 或新建表。
- 通过 `AppDatabase.schemaVersion` 控制版本号（MVP 初始设为 1）。
- 写单元测试验证迁移脚本。

## 5. 推题与提醒机制
1. 用户在日程页创建计划（时间、科目、题数）。
2. 保存到 `schedules` 表，并通过 `workmanager` 注册周期任务。
3. WorkManager 在后台触发 → 根据计划调用 `PushNextQuestionUseCase`：
   - 查询满足条件的题目（优先未完成或需要复习的题）。
   - 生成通知 payload（题目 ID、科目）。
4. AlarmManager 将通知转为本地通知（使用 `flutter_local_notifications`），用户点击后通过 `GoRouter` 深链打开刷题页。
5. 若用户正在应用内，`Riverpod` Provider 直接触发推题状态更新。

## 6. 草稿与总结流程
1. **草稿阶段**：
   - 页面右侧三列 `TextField`（或 `Quill` 富文本）。
   - 点击“保存草稿”触发 `SaveDraftUseCase`：新增 `drafts` 记录 + `questions.completedCount++`。
2. **答案展示阶段**：
   - 左侧展示答案 & 解析。
   - 草稿区清空，同时将最新草稿缓存到 `ViewModel` 以便撤销。
3. **总结阶段**：
   - 用户填写总结并选择学科、知识点、标签。
   - `SaveSummaryUseCase`：保存 `summaries`、更新 `summary_tags`、若新标签自动插入。
4. **历史回显**：
   - 每次推题前加载 `SummaryRepository.watchSummaries(questionId)`，以时间逆序显示最近总结、标签卡片。

## 7. 标签系统实现
- `tags` 表通过 `parent_id` 实现多层级；使用 `path` 虚拟字段（如 `语文/阅读/人物形象`）方便展示。
- 提供 `TagTreeProvider` 构建树结构缓存在内存，支持：
  - 按编号/名称搜索（保留 `shortcode` 字段）。
  - 拖拽排序（更新 `position` 列，可选）。
- 标签选择器组件：
  - 支持键盘输入过滤、最近使用标签列表。
  - 草稿区上方显示选中标签卡片，允许滑块调整高度。

## 8. 日程管理界面
- 采用 `TableCalendar` + 自定义 `TimeSlotEditor`：
  - 日历上显示计划 badge。
  - 右侧侧栏展示当天计划列表，可拖拽调整时间。
- 数据模型：`Schedule`（subject, startTime, repeatRule, targetCount, isActive）。
- 日程启用/停用联动 WorkManager 的注册与取消。

## 9. UI 页面与导航
| 页面 | 关键组件 | 状态提供者 |
| --- | --- | --- |
| `HomeShell` | 底部/侧边导航，负责三大模块切换 | `AppShellController` |
| `QuestionPushPage` | 左侧 `QuestionCard`、右侧 `DraftEditor` & `SummaryForm`、`HistorySummaryPanel` | `QuestionSessionController` |
| `SchedulePage` | `CalendarView` + `ScheduleList` | `ScheduleController` |
| `TagManagerPage` | `TagTreeView`、`TagDetailPanel` | `TagController` |
| `SettingsPage` (可选) | 备份/加密/调试开关 | `SettingsController` |

布局建议：
- 使用 `LayoutBuilder` + `Flex` 实现左右 6:4 或 7:3 分栏。
- 引入自定义 `SplitView` 组件，允许拖动调整草稿与历史总结面板高度。
- Tablet 端支持快捷键：数字键匹配标签、`Ctrl+S` 保存草稿、`Ctrl+Enter` 保存总结。

## 10. 安全与离线策略
- 数据库文件存放于 `getDatabasesPath()`，首次启动生成加密密钥并保存在 `flutter_secure_storage`。
- 提供手动导出/导入功能（JSON/CSV），确保离线备份。
- 所有写操作在本地完成，网络权限可选；为未来云同步预留 `SyncService` 接口。

## 11. 测试与质量保障
- **单元测试**：仓库、UseCase、标签树构建、推题算法。
- **Widget 测试**：题目卡片、草稿→总结流程、标签选择器交互。
- **集成测试**：模拟日程触发、通知点击跳转流程（使用 `integration_test` + `workmanager` 测试 API）。
- **性能监控**：使用 `flutter_driver`/`devtools` 分析渲染时间，确保题目加载 < 2 秒。

## 12. 实施里程碑（建议 6 周）
1. **第 1 周**：项目脚手架、路由与主题、数据库模型定义、基础仓库。
2. **第 2 周**：题目推送页面（左侧题卡 + 草稿编辑器）+ 数据写入。
3. **第 3 周**：总结归档与标签关联流程，历史总结回显。
4. **第 4 周**：日程管理界面、WorkManager 后台任务。
5. **第 5 周**：标签管理界面完善、快捷标签、性能调优。
6. **第 6 周**：测试、加密/备份、错误处理与发布准备。

## 13. 扩展与预研建议
- 预留 `SyncRepository` 接口，未来可接入云端 REST/GraphQL。
- 设计 `AIService` 抽象，接入本地/云端 AI 生成摘要或推荐标签。
- 研究多设备协同方案（如使用 Supabase、Appwrite）。
- 建立数据字典文档与 API 文档，持续维护架构一致性。

## 14. 附录：核心序列图（文字描述）
1. **推题触发**：`WorkManager` → `PushNextQuestionUseCase` → `QuestionRepository.fetchNext()` → `NotificationService.show()`。
2. **草稿保存**：`DraftEditor` → `QuestionSessionController.saveDraft()` → `SaveDraftUseCase` → `DraftRepository.insert()` → UI 更新草稿历史。
3. **总结归档**：`SummaryForm` → `QuestionSessionController.saveSummary()` → `SaveSummaryUseCase`（含标签关联事务） → `SummaryRepository.insertWithTags()` → 历史总结面板刷新。

