
---
description: Manage Things 3 to-dos, projects, and lists via AppleScript and URL scheme (macOS only)
user-invocable: true
allowed-tools:
  - Bash
  - AskUserQuestion
---

Manage the Things 3 task manager on macOS. Use AppleScript (`osascript`) for reads and modifications. Use the `things:///` URL scheme (no auth token) for creating to-dos and projects with scheduling parameters.

## Step 1: Verify Things 3 Is Installed

```bash
osascript -e 'tell application "System Events" to get name of every application process' 2>/dev/null | grep -q "Things3" || open -a "Things3" 2>/dev/null
osascript -e 'tell application "Things3" to get name' 2>/dev/null
```

If Things 3 is not installed, tell the user and stop.

## Step 2: List To-Dos from Built-in Lists

Built-in list names: `Inbox`, `Today`, `Anytime`, `Upcoming`, `Someday`, `Logbook`.

```bash
osascript -e '
tell application "Things3"
  set output to ""
  repeat with t in to dos of list "LIST_NAME"
    set taskName to name of t
    set taskStatus to status of t as string
    set taskNotes to notes of t
    set dueVal to ""
    try
      set dueVal to due date of t as string
    end try
    set tagVal to tag names of t
    set output to output & taskName & "\t" & taskStatus & "\t" & dueVal & "\t" & tagVal & "\t" & taskNotes & linefeed
  end repeat
  return output
end tell'
```

Replace `LIST_NAME` with the requested list. Format output as a markdown table with columns: Name, Status, Due Date, Tags, Notes (truncate notes to first 60 chars).

## Step 3: List To-Dos in a Project

```bash
osascript -e '
tell application "Things3"
  set proj to first project whose name is "PROJECT_NAME"
  set output to ""
  repeat with t in to dos of proj
    set taskName to name of t
    set taskStatus to status of t as string
    set dueVal to ""
    try
      set dueVal to due date of t as string
    end try
    set tagVal to tag names of t
    set output to output & taskName & "\t" & taskStatus & "\t" & dueVal & "\t" & tagVal & linefeed
  end repeat
  return output
end tell'
```

Replace `PROJECT_NAME` with the actual project name.

## Step 4: Create a To-Do (URL Scheme)

Use the `things:///add` URL scheme. No auth token required.

```bash
open "things:///add?title=TITLE&when=WHEN&deadline=DEADLINE&tags=TAGS&notes=NOTES&checklist-items=ITEM1%0aITEM2&list=PROJECT_NAME&show-quick-entry=false&reveal=true"
```

Parameters (all optional except title):
- `title` -- to-do name (percent-encode spaces as `%20`)
- `when` -- `today`, `tomorrow`, `evening`, `anytime`, `someday`, or `YYYY-MM-DD`
- `deadline` -- `YYYY-MM-DD`
- `tags` -- comma-separated tag names
- `notes` -- text (percent-encode)
- `checklist-items` -- newline-separated (`%0a`) sub-tasks
- `list` -- project or area name to file into
- `heading` -- section within project

If no `when` or `list` is specified, the to-do goes to Inbox.

Percent-encode all parameter values. Use `python3 -c "import urllib.parse; print(urllib.parse.quote('VALUE'))"` for encoding.

## Step 5: Create a Project (URL Scheme)

```bash
open "things:///add-project?title=TITLE&when=WHEN&deadline=DEADLINE&tags=TAGS&notes=NOTES&to-dos=TODO1%0aTODO2%0aTODO3&area=AREA_NAME&reveal=true"
```

Parameters (all optional except title):
- `title` -- project name
- `to-dos` -- newline-separated (`%0a`) initial tasks
- `area` -- area to file into
- Other params same as to-do creation

## Step 6: Complete a To-Do

```bash
osascript -e '
tell application "Things3"
  set matches to (every to do whose name is "TODO_NAME" and status is open)
  if (count of matches) > 0 then
    set status of item 1 of matches to completed
    return "Completed: " & name of item 1 of matches
  else
    return "No open to-do found with that name"
  end if
end tell'
```

If multiple matches, list them and ask the user which one to complete.

## Step 7: Cancel a To-Do

```bash
osascript -e '
tell application "Things3"
  set matches to (every to do whose name is "TODO_NAME" and status is open)
  if (count of matches) > 0 then
    set status of item 1 of matches to canceled
    return "Canceled: " & name of item 1 of matches
  else
    return "No open to-do found with that name"
  end if
end tell'
```

## Step 8: Delete a To-Do

```bash
osascript -e '
tell application "Things3"
  set matches to (every to do whose name is "TODO_NAME")
  if (count of matches) > 0 then
    set aToDo to item 1 of matches
    move aToDo to list "Trash"
    return "Deleted: TODO_NAME"
  else
    return "No to-do found with that name"
  end if
end tell'
```

Always ask for confirmation before deleting.

## Step 9: Update a To-Do

Find a to-do by name, then modify its properties via AppleScript assignment. If multiple matches, list them and ask the user which one to update.

```bash
osascript -e '
tell application "Things3"
  set matches to (every to do whose name is "TODO_NAME" and status is open)
  if (count of matches) = 0 then
    return "No open to-do found with that name"
  end if
  set aToDo to item 1 of matches
  -- Apply only the properties the user wants to change:
  set name of aToDo to "NEW_TITLE"
  set notes of aToDo to "NEW_NOTES"
  set due date of aToDo to date "YYYY-MM-DD"
  set tag names of aToDo to "tag1, tag2"
  return "Updated: " & name of aToDo
end tell'
```

Available property changes (include only the lines the user requests):
- Title: `set name of aToDo to "new title"`
- Notes: `set notes of aToDo to "new notes"`
- Due date: `set due date of aToDo to date "YYYY-MM-DD"`
- Tags: `set tag names of aToDo to "tag1, tag2"`
- Clear a property: `set due date of aToDo to missing value` (works for notes, due date, tags)

## Step 10: Reschedule a To-Do

Move a to-do to a different scheduling list or date.

```bash
osascript -e '
tell application "Things3"
  set matches to (every to do whose name is "TODO_NAME" and status is open)
  if (count of matches) = 0 then
    return "No open to-do found with that name"
  end if
  set aToDo to item 1 of matches
  -- Use ONE of the following depending on the target:
  -- Today:
  move aToDo to list "Today"
  -- Someday:
  move aToDo to list "Someday"
  -- Anytime:
  move aToDo to list "Anytime"
  -- Specific date or tomorrow:
  set activation date of aToDo to date "YYYY-MM-DD"
  -- Evening (set activation date to today, then move to Evening):
  move aToDo to list "Evening"
  return "Rescheduled: " & name of aToDo
end tell'
```

Pick only the relevant move/schedule line for the user's request. For "tomorrow", compute tomorrow's date and use `set activation date of aToDo to date "YYYY-MM-DD"`.

## Step 11: Search To-Dos by Name

```bash
osascript -e '
tell application "Things3"
  set output to ""
  set matches to (every to do whose name contains "SEARCH_TERM")
  repeat with t in matches
    set taskName to name of t
    set taskStatus to status of t as string
    set projName to ""
    try
      set projName to name of project of t
    end try
    set output to output & taskName & "\t" & taskStatus & "\t" & projName & linefeed
  end repeat
  return output
end tell'
```

Format results as a markdown table: Name, Status, Project.

## Step 12: List Projects

```bash
osascript -e '
tell application "Things3"
  set output to ""
  repeat with p in projects
    set projName to name of p
    set projStatus to status of p as string
    set todoCount to count of to dos of p
    set areaName to ""
    try
      set areaName to name of area of p
    end try
    set output to output & projName & "\t" & projStatus & "\t" & todoCount & "\t" & areaName & linefeed
  end repeat
  return output
end tell'
```

Format as markdown table: Project, Status, To-Do Count, Area.

## Step 13: List Areas

```bash
osascript -e '
tell application "Things3"
  set output to ""
  repeat with a in areas
    set output to output & name of a & linefeed
  end repeat
  return output
end tell'
```

## Step 14: List Tags

```bash
osascript -e '
tell application "Things3"
  set output to ""
  repeat with t in tags
    set output to output & name of t & linefeed
  end repeat
  return output
end tell'
```

## Step 15: Move a To-Do to a Project

```bash
osascript -e '
tell application "Things3"
  set matches to (every to do whose name is "TODO_NAME" and status is open)
  if (count of matches) > 0 then
    set targetProj to first project whose name is "PROJECT_NAME"
    move item 1 of matches to targetProj
    return "Moved to " & name of targetProj
  else
    return "No open to-do found with that name"
  end if
end tell'
```

## Step 16: Show a List in Things UI

```bash
osascript -e '
tell application "Things3"
  activate
  show list "LIST_NAME"
end tell'
```

Valid list names: `Inbox`, `Today`, `Anytime`, `Upcoming`, `Someday`, `Logbook`.

To show a project:

```bash
osascript -e '
tell application "Things3"
  activate
  show first project whose name is "PROJECT_NAME"
end tell'
```

## Step 17: Today Summary

Query the Today list and display a compact markdown summary.

```bash
osascript -e '
tell application "Things3"
  set allToDos to to dos of list "Today"
  set totalCount to count of allToDos
  set completedCount to 0
  set remainingItems to ""
  set overdueItems to ""
  set todayDate to current date
  repeat with t in allToDos
    if status of t is completed then
      set completedCount to completedCount + 1
    else
      set taskName to name of t
      set dueVal to ""
      set tagVal to tag names of t
      try
        set dueVal to due date of t as string
        if due date of t < todayDate then
          set overdueItems to overdueItems & "- " & taskName & " (due: " & dueVal & ")" & linefeed
        end if
      end try
      set remainingItems to remainingItems & "- " & taskName
      if dueVal is not "" then set remainingItems to remainingItems & " | due: " & dueVal
      if tagVal is not "" then set remainingItems to remainingItems & " | tags: " & tagVal
      set remainingItems to remainingItems & linefeed
    end if
  end repeat
  set remainingCount to totalCount - completedCount
  return "TOTAL:" & totalCount & linefeed & "DONE:" & completedCount & linefeed & "REMAINING:" & remainingCount & linefeed & "---REMAINING---" & linefeed & remainingItems & "---OVERDUE---" & linefeed & overdueItems
end tell'
```

Parse the output and format as:

```
### Today
- Total: X | Done: Y | Remaining: Z
#### Remaining
<remaining items list>
#### Overdue
<overdue items or "None">
```

## Guidelines

- Always format list output as markdown tables
- Confirm before destructive actions (delete, cancel, complete multiple)
- When a name match is ambiguous (multiple results), list them and ask the user to pick
- Percent-encode all URL scheme parameter values
- If an AppleScript command fails, report the error clearly
- For bulk operations, show a preview and ask before executing
