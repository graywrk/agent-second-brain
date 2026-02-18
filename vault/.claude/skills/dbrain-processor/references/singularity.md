# Singularity Integration

## Available MCP Tools

### Projects
- `mcp__singularity__add-project` — create project
- `mcp__singularity__update-project` — modify project
- `mcp__singularity__find-projects` — search projects
- `mcp__singularity__get-project` — get project details

### Tasks
- `mcp__singularity__add-task` — create task
- `mcp__singularity__update-task` — modify task
- `mcp__singularity__find-tasks` — search tasks
- `mcp__singularity__find-completed-tasks` — completed tasks
- `mcp__singularity__complete-task` — mark as done

### Task Groups
- `mcp__singularity__add-task-group` — create task group
- `mcp__singularity__find-task-groups` — search task groups

### Notes
- `mcp__singularity__add-note` — create note (isNote: true)
- `mcp__singularity__update-note` — modify note

### Habits
- `mcp__singularity__add-habit` — create habit
- `mcp__singularity__update-habit` — modify habit
- `mcp__singularity__get-habit-progress` — habit progress

### User Info
- `mcp__singularity__user-info` — verify connection

---

## Pre-Creation Checklist

### 1. Check Workload (REQUIRED)

```
find-tasks:
  startDate: "YYYY-MM-DD"
  endDate: "YYYY-MM-DD"
  limit: 50
```

Build workload map before adding tasks.

### 2. Check Duplicates (REQUIRED)

```
find-tasks:
  searchText: "key words from new task"
```

If similar exists → mark as duplicate, don't create.

---

## Priority System

| Priority | Value | Use Case |
|----------|-------|----------|
| Default | 1 | Most tasks |
| High | 2 | Important/urgent |
| Low | 0 | Optional tasks |

### Priority Keywords

| Keywords | Priority |
|----------|----------|
| срочно, критично, дедлайн клиента | 2 |
| важно, приоритет, до конца недели | 2 |
| нужно, надо | 1 |
| (strategic, R&D, long-term) | 0-1 |

---

## Date & Time Format

**CRITICAL: Use GMT+3 timezone for all dates.**

### useTime Parameter

| Setting | Meaning |
|---------|---------|
| `useTime: false` | Date only (no specific time) |
| `useTime: true` | Exact time in GMT+3 |

### Date Examples

```javascript
// Date only
{
  "start": "2026-02-20",
  "useTime": false
}

// Date and time (GMT+3)
{
  "start": "2026-02-20T14:30:00",
  "useTime": true
}
```

### Russian → Date Mapping

| Russian | Start Format | useTime |
|---------|--------------|---------|
| сегодня | today's date | false |
| завтра | tomorrow | false |
| в понедельник | next Monday | false |
| в 14:00 | date + "T14:00:00" | true |

---

## Notifications Format

```javascript
{
  "notifies": [60, 15],  // minutes from larger to smaller
  "notify": 1,           // main notification enabled
  "alarmNotify": false   // alarm (default: false, only if explicitly requested)
}
```

Example: notify 1 hour before and 15 minutes before.

---

## Project Emoji Format

**CRITICAL: Use hex Unicode code WITHOUT prefix.**

| Emoji | Hex Code | Wrong |
|-------|----------|-------|
| 💞 | "1f49e" | "U+1F49E", "💞" |
| 🎯 | "1f3af" | "U+1F3AF", "🎯" |
| 🚀 | "1f680" | "U+1F680", "🚀" |
| ⭐️ | "2b50" | "U+2B50", "⭐️" |

```javascript
{
  "icon": "1f49e"  // correct
}
```

---

## Note Content Format (Delta)

**CRITICAL: Pass array directly, NOT object with ops.**

```javascript
{
  "content": [
    {"insert": "Heading\n"},
    {"insert": "Text content"}
  ]
}
```

**RULE: Last insert MUST end with newline.**

---

## Projects & Task Groups

### Adding Tasks to Projects

**CRITICAL: Always place task in base task group unless specified otherwise.**

1. Find project's base task group
2. Add task to that group
3. If no groups → use project root

### Notebook Projects

- "Create notebook" → create project with `isNotebook: true`
- "Add to notebook" → create task with `isNote: true`
- Notes default to `isNote: true` in notebooks

---

## Habits

### Habit Colors (String Values)

Use ONLY these values (NOT hex codes):

```
"red" | "pink" | "purple" | "deepPurple" | "indigo" |
"lightBlue" | "cyan" | "teal" | "green" | "lightGreen" |
"lime" | "yellow" | "amber" | "orange" | "deepOrange" |
"brown" | "grey" | "blueGrey"
```

### Habit Status

- Active habit: `status: 0`
- Always create habits with `status: 0`

---

## Task Creation

```javascript
{
  "title": "Task name",
  "start": "2026-02-20",
  "useTime": false,
  "priority": 1,
  "projectId": "...",
  "taskGroupId": "...",  // optional: base group if not specified
  "notifies": [60, 15],
  "notify": 1
}
```

### Task Title Style

User prefers: прямота, ясность, конкретика

✅ Good:
- "Отправить презентацию клиенту"
- "Созвон с командой по проекту"
- "Написать пост про [тема]"

❌ Bad:
- "Подумать о презентации"
- "Что-то с клиентом"
- "Разобраться с AI"

---

## Important Fields

### showInBasket

**CRITICAL: Never set showInBasket unless explicitly requested.**

### modificatedDate vs createdDate

- Large difference = task was rescheduled
- `modificatedDate` = last change (not completion time)
- Technical field, don't set manually

---

## Anti-Patterns (НЕ СОЗДАВАТЬ)

Based on user preferences:

- ❌ "Подумать о..." → конкретизируй действие
- ❌ "Разобраться с..." → что именно сделать?
- ❌ Абстрактные задачи без Next Action
- ❌ Дубликаты существующих задач
- ❌ Задачи без дат
- ❌ Задачи с showInBasket: true (без запроса)
- ❌ Emoji项目中使用表情符号而非hex代码

---

## Error Handling

CRITICAL: Никогда не предлагай "добавить вручную".

If MCP tool fails:
1. Include EXACT error message in report
2. Continue with next entry
3. Don't mark as processed
4. User will see error and can debug

WRONG output:
  "Не удалось добавить (MCP недоступен). Добавь вручную: Task title"

CORRECT output:
  "Ошибка создания задачи: [exact error from MCP tool]"

---

## Timezone Summary

**For ALL date/time operations:**
- User timezone: **GMT+3**
- When user says "14:00" → convert to GMT+3
- When user says "today" → use today's date in GMT+3
- Store in ISO format with timezone offset if needed

---

## Quick Reference

### Pre-Creation Check
1. ✅ Check workload (find-tasks)
2. ✅ Check duplicates (find-tasks with search)
3. ✅ Determine priority (based on domain)
4. ✅ Set date/time (GMT+3, useTime correctly)
5. ✅ Configure notifies (if needed)
6. ✅ Find base task group (if project-based)

### Task Structure
```javascript
{
  "title": "Clear action verb + object",
  "start": "YYYY-MM-DD" or "YYYY-MM-DDTHH:MM:SS",
  "useTime": false or true,
  "priority": 0, 1, or 2,
  "notifies": [60, 15],  // optional
  "notify": 1            // optional
}
```

### Note Structure
```javascript
{
  "title": "Note title",
  "isNote": true,
  "content": [
    {"insert": "Note content\n"}
  ]
}
```

### Project Structure
```javascript
{
  "name": "Project name",
  "icon": "1f49e",  // hex without prefix
  "color": "purple",  // for habits only
  "isNotebook": false  // true for notebooks
}
```
