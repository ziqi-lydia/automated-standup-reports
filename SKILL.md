---
name: automated-standup-reports
description: Auto-generate weekly standup reports by aggregating your work activity across Google Calendar, Gmail, and Slack into a structured summary.
version: 1.0.0
author: ziqi-lydia
tags: [productivity, standup, reporting, automation, calendar, gmail, slack]
---

# Automated Standup Reports

Generate professional weekly standup reports by pulling real activity data from Google Calendar, Gmail, and Slack. No manual note-taking required.

---

## Prerequisites

Before running this skill, ensure the following services are connected:

```javascript
// Connect Google Calendar
await requestApproval({
  reason: "Need Calendar access to read meetings and events",
  type: "service_auth",
  payload: { provider: "google", service: "calendar" }
})

// Connect Gmail (read-only is sufficient)
await requestApproval({
  reason: "Need Gmail read access to identify sent deliverables and key threads",
  type: "service_auth",
  payload: { provider: "google", service: "gmail_read" }
})

// Connect Slack
await requestApproval({
  reason: "Need Slack access to read channel activity and discussions",
  type: "service_auth",
  payload: { provider: "slack", service: "messaging" }
})
```

---

## Commands

### `/standup` -- Generate Weekly Standup Report

**Trigger**: User says "generate standup", "weekly report", "what did I do this week", or similar.

**Parameters** (all optional):
- `period`: Time range. Default: "this week" (Monday-Friday). Accepts: "last week", "past 3 days", custom date range.
- `format`: Output format. Default: "markdown". Options: "markdown", "google-doc", "email", "slack".
- `audience`: Who the report is for. Default: "manager". Options: "manager", "team", "self".
- `sections`: Which sections to include. Default: all. Options: "meetings", "deliverables", "communications", "blockers", "next-week".

---

### Step-by-Step Workflow

#### Step 1: Determine Report Period

```javascript
// Calculate the date range
var now = new Date()
var dayOfWeek = now.getDay()

// Default: current work week (Mon-Fri)
// If user says "last week", shift back 7 days
var startDate = new Date(now)
startDate.setDate(now.getDate() - dayOfWeek + 1) // Monday
startDate.setHours(0, 0, 0, 0)

var endDate = new Date(startDate)
endDate.setDate(startDate.getDate() + 4) // Friday
endDate.setHours(23, 59, 59, 999)

var periodLabel = startDate.toLocaleDateString("en-US", { month: "short", day: "numeric" }) + 
  " - " + endDate.toLocaleDateString("en-US", { month: "short", day: "numeric", year: "numeric" })

console.log("Report period:", periodLabel)
```

#### Step 2: Pull Google Calendar Data

```javascript
var events = await google.calendar.listEvents({
  timeMin: startDate.toISOString(),
  timeMax: endDate.toISOString(),
  maxResults: 100
})

// Categorize events
var meetings = { internal: [], external: [], oneOnOnes: [], focusTime: [] }

for (var evt of events) {
  var attendeeCount = (evt.attendees || []).length
  var summary = (evt.summary || "").toLowerCase()
  
  if (summary.includes("focus") || summary.includes("block") || summary.includes("no meetings")) {
    meetings.focusTime.push(evt)
  } else if (summary.includes("1:1") || summary.includes("1-1") || summary.includes("one on one") || attendeeCount === 2) {
    meetings.oneOnOnes.push(evt)
  } else if ((evt.attendees || []).some(a => a.email && !a.email.endsWith("@" + userDomain))) {
    meetings.external.push(evt)
  } else {
    meetings.internal.push(evt)
  }
}

console.log("Meetings found:", {
  internal: meetings.internal.length,
  external: meetings.external.length,
  oneOnOnes: meetings.oneOnOnes.length,
  focusTime: meetings.focusTime.length,
  total: events.length
})
```

#### Step 3: Pull Gmail Sent Activity

```javascript
// Search for emails SENT during the period
var startStr = startDate.toISOString().split("T")[0].replace(/-/g, "/")
var endStr = endDate.toISOString().split("T")[0].replace(/-/g, "/")

var sentEmails = await google.gmail.listMessages({
  query: "in:sent after:" + startStr + " before:" + endStr,
  maxResults: 50
})

// Read each sent email to extract key deliverables
var deliverables = []
var keyThreads = []

for (var msg of (sentEmails.messages || []).slice(0, 20)) {
  var detail = await google.gmail.getMessage(msg.id)
  
  // Identify deliverables: attachments, links shared, key phrases
  var subject = detail.subject || ""
  var body = detail.body || ""
  var isDeliverable = subject.match(/report|review|draft|update|proposal|deck|document|analysis/i)
    || body.match(/attached|please review|here's the|sharing the|completed/i)
  
  if (isDeliverable) {
    deliverables.push({
      subject: detail.subject,
      to: detail.to,
      date: detail.date,
      snippet: detail.snippet
    })
  }
  
  keyThreads.push({
    subject: detail.subject,
    to: detail.to,
    date: detail.date
  })
}

console.log("Deliverables identified:", deliverables.length)
console.log("Key email threads:", keyThreads.length)
```

#### Step 4: Pull Slack Activity

```javascript
var workspaces = await slack.orgs()
var slackActivity = { channelsActive: [], keyDiscussions: [], decisionsLogged: [] }

for (var ws of workspaces) {
  var client = slack.org(ws.teamId)
  var channels = await client.channels.list({ limit: 50 })
  
  for (var ch of (channels.channels || []).slice(0, 15)) {
    var history = await client.channels.history({
      channel: ch.id,
      oldest: String(Math.floor(startDate.getTime() / 1000)),
      latest: String(Math.floor(endDate.getTime() / 1000)),
      limit: 30
    })
    
    // Filter to messages from the user
    var myMessages = (history.messages || []).filter(m => !m.subtype)
    
    if (myMessages.length > 0) {
      slackActivity.channelsActive.push({
        channel: ch.name,
        messageCount: myMessages.length,
        topMessages: myMessages.slice(0, 3).map(m => m.text.substring(0, 120))
      })
    }
  }
}

console.log("Active Slack channels:", slackActivity.channelsActive.length)
```

#### Step 5: Synthesize Report

After gathering all data, synthesize into a structured standup report. The report should follow this template:

```markdown
## Weekly Standup Report: [Period]

### Accomplishments
- [List key deliverables from email + calendar context]
- [Group by project/theme when possible]
- [Include specific outcomes, not just tasks]

### Meetings & Collaboration
- **Internal meetings**: [count] ([list key ones])
- **External meetings**: [count] ([list key ones])
- **1:1s**: [count] ([with whom])
- **Focus time blocks**: [count] hours

### Key Communications
- [Summarize important email threads]
- [Highlight cross-team discussions from Slack]
- [Note any decisions made]

### Blockers & Risks
- [Identify from email threads with "blocked", "waiting", "delayed"]
- [Flag overdue follow-ups]
- [Note any escalations]

### Next Week Priorities
- [Infer from upcoming calendar events]
- [Pending email follow-ups]
- [Continuation of current projects]
```

#### Step 6: Deliver the Report

Based on the user's chosen format:

**Markdown** (default): Return the report directly in chat.

**Google Doc**:
```javascript
var doc = await google.docs.createDocument({ title: "Standup Report - " + periodLabel })
await google.docs.batchUpdate({
  documentId: doc.documentId,
  requests: [{ insertText: { location: { index: 1 }, text: reportContent } }]
})
console.log("Report saved to Google Doc:", doc.documentId)
```

**Email**:
```javascript
await requestApproval({ reason: "Send standup report to " + recipientEmail })
await google.gmail.sendMessage({
  to: recipientEmail,
  subject: "Weekly Standup Report - " + periodLabel,
  body: reportContent
})
```

**Slack**:
```javascript
await requestApproval({ reason: "Post standup report to #" + channelName })
var client = slack.org(workspaceId)
await client.chat.sendMessage({
  channel: channelId,
  text: reportContent
})
```

---

### `/standup-team` -- Aggregate Team Standup

**Trigger**: User says "team standup", "team status", "aggregate team reports".

Extends the individual workflow to pull data for multiple team members (requires their calendar/email visibility). Produces a team-level summary with per-person highlights.

**Additional Parameters**:
- `team`: List of team member emails
- `groupBy`: "person" (default) or "project"

---

### `/standup-schedule` -- Auto-Schedule Recurring Reports

**Trigger**: User says "schedule weekly standup", "automate my standup every Monday".

Sets up a Sai workflow to auto-generate and deliver the report on a recurring schedule.

```javascript
// Example: Every Monday at 9am
scheduleWorkflow({
  name: "Weekly Standup Report",
  goal: "Generate my weekly standup report for last week and send it to my manager via email. Use the automated-standup-reports skill. Period: last week. Format: email. Audience: manager.",
  schedule: "0 9 * * 1",
  scheduleDescription: "Every Monday at 9:00 AM",
  skills: ["automated-standup-reports"]
})
```

---

## Output Quality Guidelines

1. **Be specific, not generic**: "Shipped v2.3 pricing page redesign" not "Worked on website updates"
2. **Quantify when possible**: "Responded to 12 customer threads" not "Handled customer emails"  
3. **Group by project/theme**: Don't list 20 individual items; cluster into 4-5 project groups
4. **Highlight impact**: Lead with outcomes and decisions, not just activities
5. **Flag risks proactively**: Surface overdue items and potential blockers before they're asked about
6. **Adapt tone to audience**:
   - Manager: Focus on outcomes, blockers, next steps
   - Team: Focus on collaboration, dependencies, shared context
   - Self: Focus on reflection, time distribution, productivity patterns

---

## Troubleshooting

| Issue | Solution |
|---|---|
| No calendar events found | Check date range; ensure Calendar integration is connected |
| Gmail returns empty | Verify Gmail read access; check if "in:sent" query works |
| Slack channels missing | Bot must be added to channels; run `channels.list()` to verify |
| Report too long | Set `sections` parameter to focus on specific areas |
| Missing context | Ask user to clarify project names for better grouping |
