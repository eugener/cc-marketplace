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
    delete item 1 of matches
    return "Deleted: TODO_NAME"
  else
    return "No to-do found with that name"
  end if
end tell'
```

Always ask for confirmation before deleting.

## Step 9: Search To-Dos by Name

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

## Step 10: List Projects

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

## Step 11: List Areas

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

## Step 12: List Tags

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

## Step 13: Move a To-Do to a Project

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

## Step 14: Show a List in Things UI

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

## Guidelines

- Always format list output as markdown tables
- Confirm before destructive actions (delete, cancel, complete multiple)
- When a name match is ambiguous (multiple results), list them and ask the user to pick
- Percent-encode all URL scheme parameter values
- If an AppleScript command fails, report the error clearly
- For bulk operations, show a preview and ask before executing
