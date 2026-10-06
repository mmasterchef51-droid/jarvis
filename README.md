# JARVIS — Lean, Reliable, Flexible & Scalable Master Specification

**Document purpose:** Single-source technical blueprint for building a lightweight autonomous computer/business assistant with strong verification, minimal AI-token usage, modular tools, and cloud-first development/testing.

**Core principles:**
1. One conversational AI model: **Gemini 3.1 Flash Live** for real-time voice/audio-to-audio interaction, planning, function calling, and spoken responses. Google currently documents it as a low-latency audio-to-audio model with native audio, function calling, thinking controls, and Live API support. citeturn843254search0turn843254search9turn843254search10
2. Deterministic executors do the repetitive work; Gemini is not called for every mouse click, file operation, DOM lookup, or routine state check.
3. Tool adapters are capability-based, not provider-based. Jarvis can choose **Chrome DevTools Protocol/Chrome DevTools MCP** or **Playwright MCP** per browser task. Chrome DevTools MCP is actively evolving, and Playwright MCP exposes structured accessibility-based browser control plus optional capabilities. citeturn843254search3turn843254search1turn843254search6
4. Every action has an observable success condition and verification step.
5. Least privilege, explicit permission gates, audit logs, and safe defaults.
6. No giant framework dependency. Prefer Python stdlib + small focused packages; TypeScript only where browser tooling requires it.
7. No duplicated “manager” layers. One orchestrator, one tool registry, one event bus, one policy engine, one verifier.
8. Prefer event-driven async I/O and OS/browser APIs; keep CPU work off the hot path.
9. Everything is plugin-capable through a stable adapter contract.
10. Long-term scalability comes from replacing adapters, not rewriting the core.

---

## 1. TARGET CAPABILITY

Jarvis should be able to:

- Understand natural Urdu, Hindi, and English commands in speech.
- Respond with natural voice and selectable voice/persona options while using the Gemini Live audio model.
- Plan multi-step jobs.
- Choose tools automatically.
- Control browsers with either Playwright or Chrome DevTools tooling depending on the job.
- Control Windows desktop and applications.
- Manage files and folders.
- Work with WhatsApp Web/Desktop where technically and legally supported.
- Interact with YouTube, Facebook, and Instagram through official interfaces or user-controlled UI automation where permitted.
- Coordinate specialized agents.
- Ask for confirmation before risky/external actions.
- Verify that actions actually happened.
- Recover from transient failures.
- Maintain compact memory instead of repeatedly sending full history to the model.
- Run most routine automation without another model call.

---

## 2. LEAN PROJECT STRUCTURE

Use one repository. Do not create dozens of microservices or duplicate abstraction layers.

```text
jarvis/
├── app.py                  # application entry point + lifecycle
├── config.py               # validated configuration
├── core.py                 # orchestrator + task lifecycle
├── events.py               # internal event bus
├── policy.py               # permission/risk policy
├── memory.py               # compact working/long-term memory
├── verifier.py             # independent outcome verification
├── registry.py             # tool/skill registry
├── gemini_live.py          # ONLY LLM/voice adapter
├── tools.py                # common tool contracts + execution helpers
├── adapters.py             # browser/desktop/app adapter implementations
├── plugins.py              # optional plugin loader
├── ui/                     # minimal local UI
├── tests/                  # deterministic + behavioral tests
├── scenarios/              # human-readable end-to-end test cases
└── data/                   # local state, logs, caches (gitignored)
```

### Rules for file discipline
- No `utils2.py`, `manager_final.py`, `agent_new.py`, etc.
- A feature belongs in one obvious module.
- One source of truth for configuration.
- One registry for skills/actions.
- One verifier interface.
- One event schema.
- No circular imports.
- Keep adapters thin and replaceable.
- Use typed dataclasses/protocols for stable contracts.

---

## 3. CORE RUNTIME PIPELINE

```text
MIC / TEXT
   ↓
Gemini Live session
   ↓
Intent + task plan
   ↓
Policy check
   ↓
Tool/skill resolver
   ↓
Deterministic executor
   ↓
Observation
   ↓
Verifier
   ├─ PASS → continue
   ├─ RETRY → bounded recovery
   └─ FAIL → diagnose / ask user / stop
   ↓
Compact result → Gemini Live
   ↓
Natural response
```

### Token-saving rule
Gemini should receive **semantic state**, not raw low-level traces.

Bad:
```text
click x=483 y=221
move x=485 y=221
wait 0.3
screenshot 1920x1080...
```

Good:
```text
Result: clicked “Submit”; button changed to “Submitted”; URL unchanged.
```

### Fast-path execution
Routine commands such as “open Chrome”, “mute”, “create folder”, “copy file”, “go back”, “volume 30%”, “pause video” should resolve to deterministic skills without generating a new LLM turn.

---

# 4. MULTI-AGENT SYSTEM

Agents are **logical roles**, not separate AI models. All language reasoning uses the same Gemini 3.1 Flash Live model adapter where reasoning is actually needed.

### Agents
1. **Commander** — converts user request into goal + constraints.
2. **Planner** — builds a short step graph.
3. **Browser Agent** — browser actions.
4. **Desktop Agent** — OS/UI actions.
5. **File Agent** — filesystem actions.
6. **Messaging Agent** — WhatsApp/app messaging.
7. **Media Agent** — YouTube/social media workflows.
8. **Business Agent** — structured business workflows.
9. **Security Agent** — permissions, secrets, policy, suspicious actions.
10. **Verifier Agent** — judges evidence and task completion.
11. **Recovery Agent** — bounded recovery from known failure classes.
12. **Memory Agent** — compacts durable preferences/facts.

### Agent communication
Use a small typed task envelope:

```text
Task {
  id
  parent_id
  goal
  constraints
  required_evidence
  risk_level
  deadline
  allowed_tools
  state
}
```

Agents return:

```text
Outcome {
  status: success | partial | failed | blocked
  evidence
  state_changes
  next_action
  user_confirmation_required
}
```

No agent directly modifies another agent's private state.

---

# 5. GEMINI-ONLY VOICE / AI POLICY

### Required conversational model
**Gemini 3.1 Flash Live (`gemini-3.1-flash-live-preview`)**.

Google currently describes it as a low-latency audio-to-audio model for real-time dialogue and voice-first applications; it supports audio input/output, function calling, search grounding, and adjustable thinking. citeturn843254search9turn843254search10

### Voice requirements
- Urdu support.
- Hindi support.
- English support.
- Automatic language detection/continuation.
- Male voice options.
- Female voice options where the selected Gemini Live voice configuration supports them.
- Calm / professional / energetic personas implemented by system instructions, not extra models.
- Interruption handling.
- Barge-in: user can speak while Jarvis is talking.
- Low-latency streaming.
- Short confirmation responses for routine tasks.
- Full response only when requested.

### Important architecture constraint
Do **not** call a second STT model or TTS model in the normal live-voice path. The Gemini Live native-audio session handles the conversation. Google also documents native audio as improving naturalness and multilingual performance. citeturn843254search9

---

# 6. BROWSER ENGINE SELECTION

Jarvis exposes one logical browser API:

```text
browser.navigate()
browser.click()
browser.type()
browser.extract()
browser.wait()
browser.verify()
```

Underneath, Jarvis can route to:

### Engine A — Chrome DevTools / CDP
Best for:
- live running Chrome
- DevTools-level inspection
- network / performance debugging
- existing browser sessions where connection is supported

### Engine B — Playwright
Best for:
- deterministic locators
- robust web workflows
- accessibility snapshots
- cross-browser flows
- assertions, traces, and test automation

Playwright MCP currently exposes structured accessibility snapshots and supports browsers including Chrome, Firefox, WebKit and Edge; it also supports connecting to existing Chrome/Edge sessions through CDP/extension workflows. citeturn843254search1turn843254search4turn843254search7

### Engine selection heuristic
- Existing live Chrome / DevTools inspection → prefer CDP/DevTools.
- Stable element-based workflow → prefer Playwright.
- Need accessibility snapshot → prefer Playwright.
- Need network/performance/devtools evidence → prefer DevTools.
- Engine failure → fallback to the other engine when safe.

---

# 7. BROWSER AUTOMATION SKILLS — 120+

1. Open new browser tab
2. Close current tab
3. Switch tab
4. List tabs
5. Focus tab
6. Navigate to URL
7. Go back
8. Go forward
9. Reload page
10. Hard reload
11. Stop loading
12. Read page title
13. Read current URL
14. Read accessibility snapshot
15. Find text
16. Find button
17. Find link
18. Find input
19. Find checkbox
20. Find radio button
21. Find select
22. Find image
23. Find heading
24. Find table
25. Find dialog
26. Find iframe
27. Click element
28. Double click
29. Right click
30. Hover
31. Move pointer
32. Drag element
33. Drag to target
34. Scroll page
35. Scroll element
36. Scroll to text
37. Scroll to element
38. Page up
39. Page down
40. Home position
41. End position
42. Focus element
43. Type text
44. Fill input
45. Clear input
46. Append text
47. Press key
48. Press shortcut
49. Select option
50. Check box
51. Uncheck box
52. Toggle control
53. Upload file
54. Download file
55. Wait for selector
56. Wait for text
57. Wait for URL
58. Wait for network idle
59. Wait for load state
60. Wait for dialog
61. Read input value
62. Read visible text
63. Read attribute
64. Read computed state
65. Read element count
66. Check visibility
67. Check enabled state
68. Check checked state
69. Capture screenshot
70. Capture element screenshot
71. Record trace
72. Record video for test
73. Inspect console errors
74. Read console logs
75. Inspect network requests
76. Inspect response status
77. Inspect failed requests
78. Open DevTools inspection
79. Inspect DOM
80. Inspect accessibility tree
81. Inspect performance metrics
82. Inspect page errors
83. Detect broken links
84. Extract table
85. Extract list
86. Extract cards
87. Extract article text
88. Extract metadata
89. Extract links
90. Extract images
91. Extract form schema
92. Detect cookie banner
93. Accept cookies
94. Reject optional cookies
95. Open menu
96. Close menu
97. Open modal
98. Close modal
99. Handle alert
100. Handle confirm
101. Handle prompt
102. Switch iframe
103. Return from iframe
104. Open new window
105. Switch popup
106. Close popup
107. Set viewport
108. Set zoom
109. Emulate mobile viewport
110. Set geolocation when explicitly configured
111. Change locale when testing
112. Inspect local storage
113. Inspect session storage
114. Read cookies
115. Clear site data
116. Check authentication state
117. Preserve login state
118. Verify post-login landing page
119. Verify form submission result
120. Verify download completion
121. Verify upload completion
122. Verify navigation success
123. Verify text changed
124. Verify element appeared
125. Verify URL pattern
126. Verify page title
127. Verify expected screenshot region
128. Retry transient page action
129. Fallback from locator strategy
130. Fallback Playwright ↔ DevTools
131. Detect CAPTCHA and stop/ask user
132. Detect rate limiting
133. Detect offline state
134. Reconnect browser session
135. Restore previous tab state
136. Save browser state
137. Create isolated test context
138. Start headed browser
139. Start headless browser
140. Attach to existing Chrome
141. Attach to Edge
142. Capture network HAR-like evidence when supported
143. Inspect request headers when permitted
144. Inspect response headers
145. Inspect page resource timing
146. Detect page crash
147. Detect renderer hang
148. Detect unexpected redirect
149. Confirm logout
150. Confirm deletion result

---

# 8. COMPUTER CONTROL SKILLS — 120+

1. Open Start menu
2. Search Start menu
3. Launch application
4. Close application
5. Minimize window
6. Maximize window
7. Restore window
8. Move window
9. Resize window
10. Snap window left
11. Snap window right
12. Snap window top
13. Switch window
14. List windows
15. Focus window
16. Lock workstation
17. Sign out
18. Sleep
19. Hibernate
20. Restart with confirmation
21. Shut down with confirmation
22. Wake monitor where supported
23. Change volume
24. Mute volume
25. Unmute volume
26. Change microphone level
27. Mute microphone
28. Unmute microphone
29. Open task manager
30. Inspect CPU usage
31. Inspect RAM usage
32. Inspect disk usage
33. Inspect network state
34. Inspect battery state
35. Inspect temperature when sensors are available
36. Take desktop screenshot
37. Capture active window
38. Capture region
39. Start screen recording
40. Stop screen recording
41. Copy
42. Cut
43. Paste
44. Undo
45. Redo
46. Select all
47. Open context menu
48. Press key
49. Hold key
50. Release key
51. Type text
52. Move mouse
53. Click
54. Double click
55. Right click
56. Middle click
57. Drag
58. Scroll up
59. Scroll down
60. Horizontal scroll
61. Open notification center
62. Dismiss notification
63. Read visible notification text
64. Toggle Wi-Fi
65. Toggle Bluetooth
66. Open network settings
67. Connect known Wi-Fi
68. Disconnect Wi-Fi
69. Open sound settings
70. Open display settings
71. Change display arrangement
72. Open accessibility settings
73. Open privacy settings
74. Open Windows settings
75. Open control panel components
76. Open device manager
77. Open services
78. Open event viewer
79. Open clipboard history
80. Open emoji panel
81. Change keyboard layout
82. Switch language input
83. Open File Explorer
84. Navigate Explorer path
85. Create folder
86. Create file
87. Rename item
88. Copy item
89. Move item
90. Delete item to recycle bin
91. Restore item from recycle bin
92. Empty recycle bin with confirmation
93. Search files
94. Preview file
95. Open file
96. Choose default app
97. Compress folder
98. Extract archive
99. Mount ISO where supported
100. Unmount drive where supported
101. Eject removable drive
102. Inspect drive free space
103. Open terminal
104. Run safe shell command
105. Stop process
106. Start process
107. Inspect process list
108. Inspect startup apps
109. Restart application
110. Check application health
111. Read clipboard
112. Write clipboard
113. Compare clipboard before/after
114. Detect active screen
115. Determine current foreground app
116. Detect modal dialog
117. Wait for visual state
118. Verify text on screen
119. Verify window title
120. Verify application started
121. Verify application closed
122. Recover focus after pop-up
123. Retry transient desktop action
124. Detect blocked action
125. Ask for confirmation before destructive action
126. Create local snapshot before risky file operation
127. Restore snapshot where implemented
128. Switch virtual desktop
129. Create virtual desktop
130. Move app to desktop
131. Open Run dialog
132. Open PowerShell
133. Open Command Prompt
134. Open WSL terminal
135. Copy terminal output
136. Save terminal output to evidence
137. Open browser profile
138. Open app URL directly
139. Verify network connectivity
140. Verify DNS resolution
141. Detect system idle state
142. Detect user activity
143. Temporarily pause automation
144. Resume automation
145. Emergency stop all active jobs
146. List active Jarvis jobs
147. Cancel current job
148. Pause current job
149. Resume paused job
150. Open Jarvis dashboard

---

# 9. FILE MANAGEMENT — 30+

1. Create file
2. Create folder
3. Rename file
4. Rename folder
5. Move file
6. Move folder
7. Copy file
8. Copy folder
9. Delete file
10. Delete folder
11. Restore from recycle bin
12. Search by filename
13. Search by extension
14. Search by content where supported
15. Sort by name
16. Sort by size
17. Sort by date
18. Get file metadata
19. Get file size
20. Get folder size
21. List directory
22. Compare two files
23. Compare two folders
24. Detect duplicate files
25. Archive files
26. Extract archive
27. Calculate checksum
28. Create backup copy
29. Restore backup copy
30. Securely redact application logs
31. Watch folder for changes
32. Detect newly created file
33. Detect modified file
34. Detect deleted file
35. Validate file type
36. Validate expected output file
37. Wait for file creation
38. Wait for download completion
39. Open file with chosen application
40. Generate evidence manifest

---

# 10. WHATSAPP WEB/DESKTOP — 120+

**Important safety/product rule:** Jarvis must not impersonate a human or conceal that an AI is participating in a live conversation/call. For voice calls or real-time conversation, require an explicit user-enabled disclosure/identity policy and only operate where the platform and applicable rules permit it.

Skills:
1. Open WhatsApp Web
2. Open WhatsApp Desktop
3. Detect logged-in state
4. Detect QR login screen
5. Ask user to complete login
6. Search contact
7. Search chat
8. Search group
9. Open contact
10. Open group
11. Read visible chat
12. Read recent messages
13. Search chat message
14. Search media in chat
15. Search links in chat
16. Read message timestamps
17. Read sender names
18. Read message status
19. Read reactions
20. Read pinned-message state
21. Send text message
22. Reply to message
23. Forward message
24. React to message
25. Edit sent message where supported
26. Delete message for self where supported
27. Delete message for everyone where supported
28. Quote message
29. Copy message text
30. Star message
31. Unstar message
32. Pin chat
33. Unpin chat
34. Archive chat
35. Unarchive chat
36. Mute chat
37. Unmute chat
38. Mark unread
39. Mark read
40. Create group where supported
41. Add participant
42. Remove participant
43. Promote admin where permitted
44. Demote admin where permitted
45. Leave group
46. Rename group where permitted
47. Read group description
48. Change group description where permitted
49. Change group icon where permitted
50. Find group participants
51. Open contact info
52. Open group info
53. Read shared media
54. Read shared documents
55. Download document
56. Upload document
57. Send document
58. Download image
59. Send image
60. Download video
61. Send video
62. Send audio file
63. Download audio file
64. Send voice note where supported by UI
65. Play voice message where supported
66. Pause voice message
67. Resume voice message
68. Read voice-message duration
69. Attach file
70. Attach photo
71. Attach video
72. Attach contact
73. Attach location
74. Share current location only with explicit confirmation
75. Open camera control where supported
76. Capture photo where supported
77. Capture video where supported
78. Search chats by date where supported
79. Detect unread count
80. Read notification preview
81. Open from notification
82. Detect new incoming message
83. Trigger workflow on incoming message
84. Save message for task context
85. Summarize visible chat locally via compact extraction before model use
86. Draft reply
87. Ask user approval before sensitive reply
88. Send approved reply
89. Schedule reminder for reply
90. Detect repeated message
91. Detect likely spam and stop automation
92. Rate-limit outgoing messages
93. Enforce recipient allowlist mode
94. Enforce group allowlist mode
95. Confirm before mass messaging
96. Export permitted chat data
97. Generate chat audit record
98. Verify sent message appears in thread
99. Verify attachment appears
100. Verify reply is attached to correct message
101. Detect send failure
102. Retry safe send once
103. Stop after repeated failure
104. Reconnect session
105. Reopen target chat
106. Restore previous chat after pop-up
107. Detect WhatsApp logout
108. Detect network offline
109. Detect UI version change
110. Detect unsupported control
111. Pause on verification screen
112. Ask user to intervene on QR/OTP
113. Make call where officially available and user-authorized
114. Detect incoming call
115. Accept call where supported and explicitly authorized
116. Decline call
117. End call
118. Detect call connection state
119. Capture call status metadata
120. Route microphone/audio only with explicit user permission
121. Use Gemini Live for AI speech in an allowed live interaction
122. Announce that the participant is speaking with an AI before interaction
123. Stop AI speech immediately on user interrupt
124. End call on user command
125. Log call start/end times
126. Verify call ended
127. Fail closed if audio route is unavailable
128. Disable autonomous call answering by default

---

# 11. YOUTUBE — 110+

1. Open YouTube
2. Search video
3. Search channel
4. Open video
5. Open channel
6. Read title
7. Read channel name
8. Read visible description
9. Read visible transcript/captions where available
10. Find video duration
11. Play video
12. Pause video
13. Resume video
14. Skip forward
15. Skip backward
16. Set playback speed
17. Set quality
18. Toggle captions
19. Change captions language where available
20. Mute video
21. Unmute video
22. Change volume
23. Fullscreen
24. Exit fullscreen
25. Picture-in-picture where supported
26. Add to queue
27. Remove from queue
28. Save to Watch Later
29. Remove from Watch Later
30. Like video
31. Unlike video
32. Open comments
33. Read visible comments
34. Search comments when supported
35. Post approved comment
36. Edit comment where supported
37. Delete own comment where supported
38. Reply to comment
39. Subscribe to channel
40. Unsubscribe
41. Open notifications
42. Open subscriptions feed
43. Open history
44. Open playlists
45. Create playlist where supported
46. Add to playlist
47. Remove from playlist
48. Rename playlist where permitted
49. Open Shorts
50. Search Shorts
51. Play next Short
52. Scroll feed
53. Open video in new tab
54. Copy video URL
55. Copy channel URL
56. Extract visible metadata
57. Extract visible chapter labels
58. Detect live stream
59. Open live chat where available
60. Read visible live chat
61. Pause live chat reading
62. Search channel videos
63. Sort channel videos where available
64. Open community posts where visible
65. Read visible post
66. Like community post where available
67. Reply to community post where available
68. Open Studio when user is authenticated
69. Read dashboard metrics where available
70. Open content list
71. Open video analytics
72. Read views
73. Read watch time
74. Read engagement metrics
75. Read visibility status
76. Detect processing status
77. Upload video through approved UI
78. Set title
79. Set description
80. Set thumbnail
81. Set visibility
82. Set playlist
83. Set audience controls
84. Add tags where still available
85. Schedule upload where available
86. Start upload verification
87. Verify upload completed
88. Verify processing started
89. Detect upload failure
90. Retry upload safely
91. Stop upload
92. Open copyright/claim notices where shown
93. Read visible restriction notices
94. Open channel customization where available
95. Read subscriber count where public
96. Detect age/region restrictions
97. Detect login requirement
98. Detect CAPTCHA
99. Detect rate limit
100. Detect unavailable video
101. Save evidence screenshot
102. Save extracted metadata
103. Generate task result
104. Stop before irreversible creator actions without confirmation
105. Enforce rate limits
106. Avoid spam comments
107. Avoid repetitive engagement loops
108. Respect platform access restrictions
110. Verify final state after every creator action

---

# 12. FACEBOOK — 110+

1. Open Facebook
2. Read visible feed
3. Search people
4. Search pages
5. Search groups
6. Search posts
7. Open profile
8. Open page
9. Open group
10. Open post
11. Read visible post text
12. Read visible comments
13. Read notifications
14. Open Messenger
15. Search conversation
16. Open conversation
17. Read visible messages
18. Send approved text message
19. Reply to message
20. Attach image
21. Attach file where supported
22. Send approved attachment
23. Like post
24. Remove like
25. React to post
26. Comment on post
27. Reply to comment
28. Edit own comment where supported
29. Delete own comment
30. Share post
31. Copy post link
32. Save post
33. Remove saved post
34. Follow person/page
35. Unfollow
36. Like page
37. Unlike page
38. Join group when permitted
39. Leave group
40. Read group rules where visible
41. Read group posts
42. Search group posts
43. Create approved post in allowed group/page
44. Edit own post where supported
45. Delete own post
46. Add photo to post
47. Add video to post
48. Add link to post
49. Create story where supported
50. Read story
51. Navigate story
52. React to story
53. Reply to story
54. Open marketplace where available
55. Search marketplace
56. Open listing
57. Read listing details
58. Save listing
59. Message seller only with user approval
60. Open events
61. Search events
62. Open event
63. Read event info
64. Mark interested/going where supported with confirmation
65. Open page insights where authenticated
66. Read page reach metrics
67. Read engagement metrics
68. Read follower metrics
69. Open creator dashboard where available
70. Read notifications count
71. Detect account restriction notice
72. Detect security challenge
73. Detect CAPTCHA
74. Detect login state
75. Detect checkpoint
76. Pause for human intervention
77. Detect message send failure
78. Retry safe send once
79. Verify message appearance
80. Verify comment appearance
81. Verify reaction state
82. Verify post publication
83. Verify share result
84. Save screenshot evidence
85. Save URL evidence
86. Search by exact text
87. Search by date where available
88. Open media viewer
89. Download permitted media
90. Upload permitted media
91. Open notifications item
92. Mark notifications read where available
93. Mute conversation
94. Unmute conversation
95. Archive conversation
96. Unarchive conversation
97. Pin conversation where available
98. Unpin conversation
99. Flag suspicious UI state
100. Enforce recipient/page/group allowlist
101. Rate-limit messaging
102. Block mass unsolicited outreach
103. Prevent repetitive engagement
104. Ask confirmation before public posting
105. Ask confirmation before page/group moderation changes
106. Confirm irreversible deletion
107. Verify logout if requested
108. Reconnect session
109. Recover from modal/popup
110. Produce final action receipt

---

# 13. INSTAGRAM — 110+

1. Open Instagram
2. Search username
3. Search hashtags
4. Search posts
5. Open profile
6. Read visible bio
7. Read visible post captions
8. Read visible comments
9. Read notifications
10. Open direct messages
11. Search conversation
12. Open conversation
13. Read visible messages
14. Send approved DM
15. Reply to DM
16. Attach image
17. Attach video
18. Send approved attachment
19. Like post
20. Unlike post
21. Comment on post
22. Reply to comment
23. Edit own comment where supported
24. Delete own comment
25. Save post
26. Unsave post
27. Follow account
28. Unfollow account
29. Mute account
30. Unmute account
31. Open followers list
32. Open following list
33. Search followers
34. Open story
35. Next story
36. Previous story
37. React to story
38. Reply to story
39. Create story where supported
40. Add text to story where supported
41. Add approved media to story
42. Publish approved story
43. Verify story publication
44. Open reels
45. Search reels
46. Play reel
47. Pause reel
48. Like reel
49. Comment on reel
50. Share reel
51. Copy reel link
52. Save reel
53. Open post composer
54. Upload approved photo
55. Upload approved video
56. Add caption
57. Add approved hashtags
58. Add location where user specifies
59. Tag approved account
60. Choose audience where available
61. Publish approved post
62. Save draft where supported
63. Open drafts
64. Edit draft
65. Delete draft
66. Open professional dashboard where available
67. Read reach
68. Read impressions
69. Read engagement
70. Read follower growth
71. Read content performance
72. Read insights by post
73. Open notifications item
74. Mark notifications read where available
75. Archive post where available
76. Unarchive post
77. Delete post
78. Restore deleted post where supported
79. Edit profile where permitted
80. Edit bio where permitted
81. Read account status
82. Detect restricted action
83. Detect login challenge
84. Detect CAPTCHA
85. Detect suspicious-login warning
86. Pause for human intervention
87. Verify DM sent
88. Verify comment posted
89. Verify like state
90. Verify follow state
91. Verify post published
92. Verify story published
93. Verify deletion result
94. Verify archive result
95. Capture evidence screenshot
96. Capture URL evidence
97. Search content by exact phrase
98. Search visible comments
99. Filter notifications by type where available
100. Enforce messaging allowlist
101. Rate-limit outgoing DMs
102. Block unsolicited bulk messaging
103. Avoid repetitive likes/comments/follows
104. Ask confirmation before public post
105. Ask confirmation before deleting content
106. Ask confirmation before account-setting changes
107. Recover from modal
108. Recover from expired session
109. Reopen target screen
110. Produce final action receipt

---

# 14. BUSINESS / PRODUCTIVITY SKILLS — 60+

1. Create task
2. Read task list
3. Update task
4. Complete task
5. Reopen task
6. Set priority
7. Set due date
8. Create reminder
9. Cancel reminder
10. Create recurring reminder
11. Build checklist
12. Run checklist
13. Draft email text
14. Read permitted email
15. Search email
16. Summarize email
17. Extract action items
18. Draft calendar event
19. Create calendar event after confirmation
20. Move event
21. Cancel event with confirmation
22. Read schedule
23. Prepare meeting agenda
24. Prepare meeting notes
25. Convert notes to tasks
26. Generate report from structured data
27. Read CSV
28. Write CSV
29. Read JSON
30. Write JSON
31. Validate configuration
32. Check service health
33. Run diagnostic
34. Generate audit report
35. Generate task receipt
36. Track job status
37. Retry failed job
38. Cancel job
39. Pause job
40. Resume job
41. Queue batch task
42. Rate-limit batch task
43. Generate business summary
44. Compare two reports
45. Extract KPIs
46. Detect anomalies in structured metrics
47. Create standard operating procedure draft
48. Convert SOP into executable checklist
49. Route request to human
50. Request approval
51. Record approval
52. Record rejection
53. Escalate urgent issue
54. Create incident record
55. Close incident
56. Maintain audit trail
57. Generate daily dashboard summary
58. Generate weekly dashboard summary
59. Maintain user preference profile
60. Compact old conversation memory

---

# 15. SECURITY / RELIABILITY

### Permission levels
- **L0:** read-only/local observation
- **L1:** reversible local action
- **L2:** external communication
- **L3:** public posting / account changes
- **L4:** destructive / financial / security-sensitive actions

Require explicit confirmation for L2+ by default unless user has created a narrow allowlist.

### Fail-safe rules
- Never claim success without evidence.
- Never delete outside the target path.
- Never send a message when recipient identity is uncertain.
- Never post publicly without required approval.
- Never bypass CAPTCHA, 2FA, security challenges, or access controls.
- Never silently continue after repeated failures.
- Emergency stop cancels all active jobs.
- Every external action gets a compact evidence record.

### Evidence types
- URL
- window title
- DOM/accessibility state
- text confirmation
- file existence
- checksum
- process state
- UI state
- API response status where officially available
- screenshot/trace only when needed

---

# 16. INDEPENDENT VERIFIER

The verifier must not simply trust the executor's “done” message.

Example:

```text
USER: “Create a folder named invoices.”

Executor:
  mkdir invoices

Verifier:
  path exists? yes
  is directory? yes
  correct parent? yes

Outcome: PASS
```

Browser example:

```text
USER: “Open the BBC website.”
Executor: navigate
Verifier:
  URL host matches expected
  page title exists
  no navigation error
Outcome: PASS
```

WhatsApp example:

```text
Executor: send message
Verifier:
  correct chat open
  exact text appears as sent message
Outcome: PASS
```

### Verification strategy
1. Prefer semantic state.
2. Use screenshots only when semantic state is insufficient.
3. Use second-source evidence when possible.
4. Require an expected postcondition for every skill.
5. Keep retries bounded (default 1–2).

---

# 17. ZERO-CPU / LOW-CPU DESIGN

“Zero CPU” is not physically possible while software is running. The target is **very low idle CPU** and event-driven execution.

### Rules
- No constant screen polling loop.
- No continuous screenshots.
- No 10ms mouse-position polling.
- No repeated full DOM dumps.
- Event-driven OS listeners where available.
- Browser accessibility snapshot only when needed.
- Cache stable state.
- Debounce rapid events.
- Use async I/O.
- Sleep when idle.
- Keep GUI lightweight.
- Run heavy processing remotely where feasible.
- Compress logs before sending them to Gemini.
- Store long-term memory locally in compact structured form.
- Never send the entire history on every turn.

### AI-call budget strategy
- 1 Gemini session for active conversation.
- Function calls for tools.
- No new model call for deterministic follow-up clicks.
- Tool-result compression.
- Reuse short-lived task context.
- Cache stable observations.
- Batch related observations.
- Ask Gemini only when ambiguity/decision-making exists.

---

# 18. MEMORY DESIGN

Three layers only:

### Working memory
Current task, active page/app, last action, expected state.

### Preferences
Language preference, voice preference, confirmation defaults, favorite applications.

### Durable facts
User-approved stable facts needed for future work.

No giant conversation transcript is loaded for every action.

---

# 19. PLUGIN SYSTEM

Every plugin implements:

```text
Plugin {
  id
  version
  capabilities()
  health_check()
  execute(action, args)
  verify(action, expected)
  shutdown()
}
```

Plugins are isolated and optional.

Suggested plugins:
- Browser
- Desktop
- Files
- WhatsApp
- YouTube
- Facebook
- Instagram
- Email
- Calendar
- Business tools

---

# 20. AUTOMATION RECIPE FORMAT

Each skill should be declarative where possible:

```yaml
id: browser.search
risk: L0
inputs:
  query: string
preconditions:
  - browser.ready
steps:
  - navigate_search_url
  - fill_search_box
  - submit
postconditions:
  - results_visible
verification:
  evidence: page_state
retry: 1
fallback:
  - browser.search_alt_locator
```

This keeps skills understandable, testable, and replaceable.

---

# 21. TEST LAB — CLOUD-FIRST

The development/test environment should live in the cloud whenever possible, while the user's laptop is only the eventual endpoint.

### Test layers

**Unit tests**
- policy
- registry
- state machine
- parsers
- file operations
- verifier

**Integration tests**
- Gemini Live tool calls
- browser adapters
- Windows adapter
- plugin loading

**Behavioral tests**
- give Jarvis a real command
- allow it to use a test browser/desktop
- observe result
- independent verifier determines PASS/FAIL

### Golden scenario
```text
Prompt:
  “Open Chrome, go to example.com, tell me the page title.”

Expected:
  1. Chrome opens or existing session is reused.
  2. URL becomes example.com.
  3. Page title is observed.
  4. Gemini speaks the verified title.

Failure examples:
  - wrong URL
  - browser did not open
  - title was guessed without evidence
  - Jarvis reported success after timeout
```

### Adversarial scenarios
- browser closed unexpectedly
- network disconnected
- page changed structure
- login expired
- popup blocks action
- tool returns stale element
- command is ambiguous
- user interrupts mid-task
- two jobs run simultaneously
- filesystem target disappears
- external application freezes

---

# 22. ACCEPTANCE CRITERIA

Jarvis is ready for a stable release only when:

1. Core starts cleanly.
2. No circular imports.
3. Every registered skill has a verifier.
4. Every destructive/external skill has policy metadata.
5. Browser can select Playwright or DevTools per task.
6. Voice loop runs through Gemini Live.
7. Urdu/Hindi/English command tests pass.
8. Low-CPU idle test passes.
9. Recovery tests pass.
10. Emergency stop works.
11. Audit receipt exists for external actions.
12. Behavioral tests prove actions actually happened.
13. Failed actions are never reported as completed.
14. Plugin failure does not crash the entire core.
15. New skills can be added without editing the orchestrator.

---

# 23. IMPLEMENTATION ORDER

Phase 1 — Core
- lifecycle
- config
- Gemini Live
- registry
- policy
- verifier
- events

Phase 2 — Browser
- common browser API
- Playwright adapter
- CDP/DevTools adapter
- engine selector
- browser tests

Phase 3 — Desktop
- application launch
- window control
- keyboard/mouse
- process/state verification

Phase 4 — Files
- safe filesystem skills
- evidence
- backups

Phase 5 — Multi-agent routing
- commander
- planner
- specialized logical agents
- task graph

Phase 6 — WhatsApp
- Web/Desktop adapter
- messaging
- media
- calls only with explicit disclosure/permission policy

Phase 7 — Social platforms
- YouTube
- Facebook
- Instagram

Phase 8 — Cloud Test Lab
- cloud Windows/browser environment
- scenario generator
- verifier
- failure corpus
- regression suite

Phase 9 — Optimization
- token compression
- caching
- event-driven execution
- latency metrics
- CPU/RAM budget

---

# 24. PERFORMANCE TARGETS

These are engineering targets, not guarantees:

- Idle CPU: ideally near-background/service level on a modern machine.
- No permanent high-frequency polling loops.
- First voice response: optimized for low-latency streaming through Gemini Live.
- Deterministic actions: sub-second where the OS/app responds immediately.
- Browser actions: dominated by page/network latency rather than AI latency.
- Memory growth: bounded by explicit cache policies.
- Logs: compact and rotated.

---

# 25. FINAL DESIGN SUMMARY

**Jarvis should be a small core with many replaceable skills, not a huge pile of agents.**

```text
                   GEMINI LIVE
                 (one AI brain)
                       │
                Commander/Planner
                       │
                 Policy + Registry
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
     Browser         Desktop         Files
   Playwright/       Windows        Local FS
      CDP
        │              │              │
        ├──────────────┼──────────────┤
        ▼              ▼              ▼
    WhatsApp       YouTube       FB / Instagram
        │              │              │
        └──────────────┼──────────────┘
                       ▼
                    VERIFY
                       │
              PASS / RETRY / FAIL
                       │
                       ▼
                Gemini Live reply
```

### Non-negotiable principles

**Gemini decides.**

**Deterministic tools execute.**

**Verifier proves.**

**Policy controls.**

**Events coordinate.**

**Plugins extend.**

**Caching keeps AI/token load low.**

**Cloud test labs prove real behavior.**

---

## CURRENT TECHNOLOGY NOTES

- Google lists Gemini 3.1 Flash Live as a current audio-to-audio/live model and documents its native audio, function calling, thinking, and Live API capabilities. citeturn843254search0turn843254search9turn843254search10
- Playwright MCP provides browser automation via structured accessibility snapshots, with browser control, testing, tracing, and additional capabilities. citeturn843254search1turn843254search4turn843254search6
- Playwright can connect to an existing Chrome/Edge session through supported CDP/extension approaches, which supports the “live browser control” requirement. citeturn843254search7
- Chrome DevTools MCP continues to receive updates; Chrome 152 documentation notes new DevTools MCP improvements in August 2026. citeturn843254search3

**Important:** The action catalogs above are a product specification/inventory. Some social-platform actions depend on the current UI, account permissions, platform policies, available official APIs, and the exact desktop/web client. The implementation should dynamically probe capability availability rather than assuming every action is always possible.

---

# 26. DYNAMIC MULTI-AGENT EXPANSION — REQUIRED AMENDMENT

**This section is additive. Nothing previously specified in this document is deleted or replaced.**

The original agent architecture remains intact. This amendment adds dynamic task delegation, explicit model routing, and ten additional platform systems so Jarvis can scale long-running jobs without turning every small action into an expensive AI request.

## 26.1 Dynamic Agent Tasking

Jarvis must be able to create, run, pause, resume, retry, cancel, and merge agent tasks dynamically.

### Core rule
- A normal short command should stay inside the main Jarvis process.
- A long, parallel, ambiguous, or failure-prone task may be decomposed into child tasks.
- Child tasks are created only when their expected value exceeds orchestration overhead.
- Jarvis may spawn multiple specialist tasks in parallel.
- Jarvis must enforce maximum depth, maximum fan-out, deadlines, budgets, and permissions.
- Every child task receives only the context it needs.
- Child agents return structured evidence, not long natural-language transcripts.
- The parent task owns the final user-facing result.
- Agent execution must be cancellable.
- A failed child task must not corrupt unrelated tasks.

### Example

```text
User: "Research these competitors, compare them, open the results, make a report, and save it."

Jarvis Commander
   ↓
Task graph
   ├── Search/Research task
   ├── Browser extraction task
   ├── Comparison task
   ├── Report-generation task
   └── File-save + verification task
              ↓
        parent verifier
              ↓
        Gemini Live response
```

## 26.2 Model Routing Policy

### A. Conversational / voice model — ONLY
**Gemini 3.1 Flash Live** is the only conversational model in the live voice path.

Use it for:
- spoken conversation
- user intent
- dialogue state
- spoken responses
- tool/function calling from the live session
- high-level planning when necessary

Do not add another STT/TTS/conversational model to the normal voice path.

### B. Computer Control Agent model pool
The Computer Control Agent may dynamically choose between:

1. **Gemini 3.5 Flash-Lite** — preferred when computer-use/tool execution is supported and a task benefits from stronger action selection or visual/computer reasoning.
2. **Gemini 3.1 Flash-Lite** — preferred for lightweight, high-frequency classification, extraction, state interpretation, simple planning, and low-cost control decisions.

The choice is Jarvis's, based on:
- task complexity
- visual ambiguity
- number of steps
- latency target
- estimated token cost
- failure risk
- whether computer-use capability is available for the selected model/version

Important verified capability note: Google currently lists **Gemini 3.5 Flash-Lite** with Computer Use support (Preview), while **Gemini 3.1 Flash-Lite** is optimized for high-frequency lightweight tasks and currently does not list Computer Use. Therefore the router must not attempt an unsupported Computer Use call on 3.1 Flash-Lite; it should use 3.1 for analysis/routing/state tasks and 3.5 Flash-Lite for supported direct computer-use execution. citeturn243282search3turn243282search4

### C. Browser Automation Agent model pool
The Browser Automation Agent should use the Gemma family for browser-side reasoning, element/state interpretation, search-query assistance, and extraction/reranking where appropriate.

Requested target pool:
- **Gemma 4 31B**
- **Gemma 4 21B** (requested configuration slot)

**Model availability guard:** Google's current official Gemma 4 page lists **31B, 26B, and 12B** plus E4B/E2B variants; it does **not** currently list a 21B Gemma 4 size. Therefore the architecture must keep the 21B slot configurable but fall back to the officially available 26B variant unless a valid 21B checkpoint/model identifier becomes available. Never silently pretend that an unavailable 21B model exists. citeturn243282search5

### D. Search
Search is a capability, not a separate chat model.

Jarvis should use:
- deterministic search APIs/connectors where available
- browser search through the Browser Agent when needed
- Gemma 4 for query expansion, result classification, extraction, deduplication, and relevance scoring when an AI step is required
- Gemini Live only when the search result needs to be discussed naturally with the user

Do not send every search result page back through Gemini. Keep retrieval and filtering structured.

---

# 27. AGENT ORCHESTRATOR ATTRIBUTES

Each dynamic agent must expose the following attributes:

```text
AgentProfile {
    id
    role
    model_pool
    capabilities
    required_permissions
    max_parallel_tasks
    max_depth
    timeout_seconds
    token_budget
    cpu_budget
    memory_budget
    retry_policy
    evidence_requirements
    risk_level
    health_state
    cancellation_support
    fallback_agent
    version
}
```

### Mandatory agent states

```text
IDLE → QUEUED → RUNNING → WAITING → VERIFYING → SUCCESS
                                  ↘ RETRY
                                  ↘ FAILED
                                  ↘ BLOCKED
                                  ↘ CANCELLED
```

### Agent selection score

The router should select an agent/model using a deterministic score rather than asking an LLM which model to use for every action:

```text
score = capability_fit
      + evidence_fit
      + reliability_score
      + latency_fit
      + cost_fit
      - unsupported_feature_penalty
      - risk_penalty
```

---

# 28. FIFTEEN CORE JARVIS SYSTEMS

The existing systems in this document remain. The following map consolidates the complete runtime into **15 major systems**. **Systems 1–5 correspond to the original core architecture; Systems 6–15 are the ten newly added systems in this amendment.**

## System 1 — Voice & Conversation System
**Purpose:** natural live interaction.

**Attributes:**
- Model: Gemini 3.1 Flash Live only
- Input: voice + text
- Languages: Urdu, Hindi, English
- Output: native streamed audio
- Voice options: selectable supported male/female voices
- Barge-in: required
- Latency target: low
- Context: compact session state
- Function calling: required
- Failure mode: fall back to text UI without introducing another voice model

## System 2 — Command / Intent System
**Purpose:** convert user requests into structured commands.

**Attributes:**
- normalized intent
- entities
- constraints
- urgency
- risk level
- expected outcome
- required evidence
- direct-vs-agent decision
- confirmation requirement

## System 3 — Tool & Skill System
**Purpose:** expose all capabilities through one registry.

**Attributes:**
- stable skill IDs
- typed inputs/outputs
- permission metadata
- executor binding
- verifier binding
- timeout
- retry policy
- version
- health state
- deprecation state

## System 4 — Browser / Computer Execution System
**Purpose:** deterministic interaction with browser and desktop environments.

**Attributes:**
- browser backend: CDP/Chrome DevTools or Playwright
- desktop backend: Windows APIs / supported UI automation
- screenshots only when necessary
- DOM/accessibility/state extraction
- action batching
- bounded retries
- post-action verification
- session reuse

## System 5 — Verification System
**Purpose:** independently determine whether a requested outcome happened.

**Attributes:**
- expected state
- observed state
- evidence set
- confidence
- pass/fail/partial
- retry recommendation
- contradiction detection
- stale-state detection
- audit record

---

## System 6 — Dynamic Agent Orchestrator NEW
**Purpose:** create and coordinate child agents only when useful.

**Attributes:**
- task graph
- parent/child IDs
- dependency graph
- parallelism limit
- deadline
- budget
- cancellation
- priority
- failure propagation policy
- result aggregation

## System 7 — Model Router NEW
**Purpose:** choose the smallest suitable model for each subtask.

**Attributes:**
- model capability matrix
- model health
- latency estimate
- token estimate
- task complexity
- computer-use capability check
- browser-model availability
- fallback order
- per-task budget
- routing telemetry

## System 8 — Task Queue & Scheduler NEW
**Purpose:** run tasks immediately, later, or in parallel.

**Attributes:**
- FIFO + priority queues
- scheduled time
- dependencies
- concurrency limit
- retry windows
- deadline handling
- cancellation
- persistence
- deduplication
- starvation prevention

## System 9 — World-State / Observation System NEW
**Purpose:** maintain a compact structured view of what the computer/browser currently looks like.

**Attributes:**
- active application
- active window
- URL
- focused element
- selected file
- recent state changes
- known task facts
- freshness timestamp
- source/evidence
- invalidation rules

## System 10 — Recovery & Self-Healing System NEW
**Purpose:** recover from routine failures without restarting the entire system.

**Attributes:**
- failure taxonomy
- bounded retries
- alternate executor
- stale-state refresh
- session reconnect
- tool fallback
- rollback
- escalation threshold
- recovery evidence
- circuit breaker

## System 11 — Resource Governor NEW
**Purpose:** keep CPU/RAM/network use low and predictable.

**Attributes:**
- CPU budget
- RAM budget
- token budget
- screenshot frequency limit
- browser-session reuse
- process reuse
- idle sleep
- batching
- backpressure
- thermal/resource safety hooks

## System 12 — Security & Permission System NEW
**Purpose:** control what Jarvis is allowed to do.

**Attributes:**
- capability permissions
- per-tool scopes
- read/write separation
- confirmation gates
- secret isolation
- sensitive-data handling
- audit trail
- emergency stop
- session lock
- policy version

## System 13 — Memory & Context Compression System NEW
**Purpose:** preserve useful history while minimizing token load.

**Attributes:**
- short-term state
- task memory
- user preferences
- durable facts
- summaries
- embeddings/indexes only when useful
- TTL
- deduplication
- relevance score
- privacy controls

## System 14 — Plugin & Integration System NEW
**Purpose:** add WhatsApp, YouTube, Facebook, Instagram and future integrations without modifying the core.

**Attributes:**
- plugin manifest
- version
- permissions
- capabilities
- lifecycle hooks
- health check
- install/uninstall
- configuration schema
- compatibility version
- isolated failure boundary

## System 15 — Observability & Test-Lab System NEW
**Purpose:** make Jarvis measurable, debuggable, and continuously testable.

**Attributes:**
- structured logs
- action traces
- screenshots/evidence when allowed
- test scenarios
- expected outcomes
- actual outcomes
- replay support
- regression detection
- performance metrics
- cloud test execution
- pass/fail dashboards

---

# 29. DYNAMIC LONG-TASK EXECUTION

Jarvis must decide dynamically whether a request is:

### Level 0 — Direct fast path
One deterministic skill, zero additional agent.

### Level 1 — Sequential task
A short chain of deterministic skills.

### Level 2 — Single specialist agent
One Browser, Computer, File, Messaging, Media, or other specialist agent.

### Level 3 — Parallel agent task graph
Multiple specialists work simultaneously with a parent orchestrator.

### Level 4 — Long-running autonomous workflow
Persistent task graph with checkpoints, recovery, verification, and progress reporting.

### Dynamic delegation example

```text
User: "Find the best three competitors, visit their sites, collect pricing,
compare them, make a report, save it, and send it to me."

Commander
  ↓
Orchestrator
  ├── Browser Agent / Gemma 4
  │      ├── Search
  │      ├── Open sites
  │      ├── Extract pricing
  │      └── Verify pages
  ├── Computer Agent / Gemini 3.5 Flash-Lite or 3.1 Flash-Lite
  │      └── File/report interaction if needed
  ├── File Agent
  │      └── Save report
  └── Verifier
         └── Confirm all requested outputs exist
              ↓
         Gemini 3.1 Flash Live
              ↓
         spoken answer
```

The user should not need to know which agents were used unless they ask.

---

# 30. COST / TOKEN / CPU MINIMIZATION FOR MULTI-AGENTS

Dynamic agents must **reduce unnecessary AI work**, not multiply it.

### Do not do this

```text
click → AI call
check → AI call
move → AI call
wait → AI call
read button → AI call
click → AI call
```

### Do this

```text
Gemini plans:
  "Open settings and enable X."

Deterministic executor:
  locate known control
  click
  wait for known state
  verify

Gemini receives:
  "X enabled successfully."
```

### Required optimization mechanisms

1. **Skill macros** — combine common deterministic steps into one executable skill.
2. **State caching** — avoid re-reading unchanged state.
3. **Action batching** — execute compatible actions together.
4. **Event triggers** — react to state changes instead of polling.
5. **Model escalation** — use the larger/stronger model only when needed.
6. **Model demotion** — return to lightweight reasoning after ambiguity is resolved.
7. **Task memoization** — reuse safe repeated results.
8. **Prompt compression** — send facts, not transcripts.
9. **Evidence compaction** — store structured evidence rather than full screenshots unless necessary.
10. **Agent reuse** — reuse warm agent sessions where safe instead of recreating them.

---

# 31. COMPUTER CONTROL AGENT — REQUIRED DESIGN

The Computer Control Agent is responsible for desktop navigation and computer-level actions.

### Model selection

```text
Task complexity LOW
    → Gemini 3.1 Flash-Lite for state interpretation / routing

Task complexity HIGH or supported Computer Use required
    → Gemini 3.5 Flash-Lite

Task complete / evidence available
    → return structured result; do not call another model
```

The executor itself should remain deterministic wherever possible.

### Computer agent evidence
Every important action should be able to produce one or more of:
- process/window state
- application state
- UI accessibility state
- file existence/state
- title/text state
- screenshot evidence when required
- OS-level confirmation

---

# 32. BROWSER AUTOMATION AGENT — REQUIRED DESIGN

The Browser Automation Agent is responsible for web navigation, extraction, search, interaction, and verification.

### Model selection

```text
Browser task
   ↓
Gemma 4 31B / configured smaller Gemma 4 variant
   ↓
reason about page/query/state
   ↓
CDP or Playwright executor
   ↓
structured observation
   ↓
verify
```

### Browser backend selection remains dynamic

**Chrome DevTools / CDP:** preferred for live attached Chrome, browser debugging, network/performance inspection, deep Chrome state, and direct DevTools control.

**Playwright:** preferred when isolated, repeatable, structured browser automation is safer or faster.

Jarvis must choose the backend per task through the capability router; it must not be hard-coded to one browser engine.

---

# 33. SEARCH AGENT DESIGN

Search should be treated as a pipeline:

```text
User question
 ↓
Query normalizer
 ↓
Search provider(s)
 ↓
Deduplicate
 ↓
Fetch/extract
 ↓
Gemma relevance filter / reranker when needed
 ↓
Evidence verifier
 ↓
Compact facts
 ↓
Gemini Live response
```

The search agent must avoid sending complete search pages to the conversational model when only a few facts are needed.

---

# 34. LONG-TASK CHECKPOINTS

For tasks longer than a configured threshold, Jarvis must checkpoint:

```text
Checkpoint {
    task_id
    completed_steps
    current_step
    pending_steps
    evidence_refs
    state_snapshot
    agent_states
    model_choices
    retries_used
    budget_used
    next_resume_action
}
```

A crash or restart should resume from the latest valid checkpoint when safe.

---

# 35. FAILURE ISOLATION RULES

- A browser failure must not crash the voice system.
- A WhatsApp plugin failure must not crash the core orchestrator.
- One child agent failure must not cancel unrelated successful children unless the parent dependency graph requires it.
- Model/API failure must trigger the configured compatible fallback, not an uncontrolled model switch.
- A verification failure must not be reported as success.
- Repeated identical failures must open a circuit breaker.
- Dangerous or high-impact external actions must stop at the permission layer when confirmation is required.

---

# 36. IMPLEMENTATION CONTRACT FOR THE BUILDER AI

Any AI coding agent used to build Jarvis must preserve this document and its earlier sections.

### The builder must
- never delete existing required capabilities without explicit authorization
- prefer modifying the smallest necessary module
- run focused tests before broad tests
- run regression tests after architecture changes
- update only the affected plugin/adapter when possible
- keep APIs backwards-compatible where practical
- produce evidence for completed work
- report unsupported requested model IDs instead of fabricating them
- avoid adding dependencies without justification
- avoid duplicate files and duplicate abstractions
- keep the repository lean

### Definition of done for a feature

```text
SPECIFIED
   ↓
IMPLEMENTED
   ↓
UNIT TESTED
   ↓
INTEGRATION TESTED
   ↓
BEHAVIORALLY TESTED
   ↓
FAILURES FIXED
   ↓
REGRESSION TEST PASSED
   ↓
VERIFICATION EVIDENCE STORED
   ↓
READY
```

---

# 37. MODEL AVAILABILITY NOTES — VERIFIED AUGUST 2026

- **Gemini 3.5 Flash-Lite** is a current model and Google lists Computer Use support (Preview). citeturn243282search4turn243282search1
- **Gemini 3.1 Flash-Lite** is current and optimized for low-latency/high-volume lightweight tasks; its official capability table currently does not list Computer Use. citeturn243282search3
- **Gemini 3.1 Flash-Lite Preview** is shut down; use the GA `gemini-3.1-flash-lite` identifier instead. citeturn243282search2turn243282search0
- **Gemma 4** currently documents 31B, 26B, and 12B sizes plus E4B/E2B variants. The requested 21B size is therefore kept as a configurable slot with a compatibility check/fallback, not as an assumed official model. citeturn243282search5

---

# 38. FINAL ADDITIVE REQUIREMENT SUMMARY

This amendment adds:

- Dynamic child-agent spawning for long tasks.
- Parent/child task graphs.
- Agent budgets and cancellation.
- Dynamic Computer Agent routing between Gemini 3.5 Flash-Lite and Gemini 3.1 Flash-Lite according to capability and task needs.
- Browser Agent routing around Gemma 4 31B plus a configurable smaller Gemma 4 slot; current official 26B is the documented fallback for the requested 21B slot.
- Search as a structured capability using deterministic retrieval plus Gemma-based filtering/reranking where useful.
- Ten additional systems with explicit attributes:
  1. Dynamic Agent Orchestrator
  2. Model Router
  3. Task Queue & Scheduler
  4. World-State / Observation
  5. Recovery & Self-Healing
  6. Resource Governor
  7. Security & Permission
  8. Memory & Context Compression
  9. Plugin & Integration
  10. Observability & Test-Lab
- A consolidated 15-system architecture.
- Long-task checkpoints and failure isolation.
- Stronger CPU/token minimization rules.
- Builder-AI rules that preserve the existing specification rather than deleting or duplicating it.

**Nothing earlier in this master specification is intentionally removed by this amendment.**

# 39. ADDITIVE ARCHITECTURE AMENDMENT — CLOUD GEMMA, SELF-EXTENSION, MULTI-TERMINAL, PREMIUM UI, SAAS READINESS

**Status:** Additive amendment. Existing sections and all prior capability inventories remain intact. This section adds requirements; it does not remove earlier content.

## 39.1 Non-negotiable preservation rule

The implementation builder must treat the whole file as cumulative. No prior action, skill, system, architecture rule, test scenario, or requirement may be deleted merely because a newer section introduces a newer design. When a new requirement conflicts with an old implementation detail, the builder must preserve the user-facing capability and upgrade the internal implementation through a compatibility layer or migration.

---

# 40. MODEL LOCATION AND RUNTIME POLICY

## 40.1 No local Gemma model files

All Gemma models used by Jarvis must be accessed through the **Gemini API using the user's configured Gemini API key**. Gemma must NOT be downloaded, loaded into local RAM, or executed by local CPU/GPU unless a future explicit configuration changes this policy.

```text
Jarvis Browser/Search Agent
        |
        v
Gemini API Gateway
        |
        +--> Gemma 4 31B / supported model endpoint
        +--> configured smaller Gemma 4 endpoint
```

This keeps the local application lightweight and moves model compute to Google's cloud service. The actual model ID must always be validated against the currently available Gemini API catalog at runtime/build time; unsupported IDs must never be silently fabricated.

## 40.2 Computer-agent models

The Computer Control Agent has two configurable preferred model slots:

- **Gemini 3.5 Flash-Lite** — preferred when computer-use capability, multimodal interpretation, or richer action reasoning is needed.
- **Gemini 3.1 Flash-Lite** — preferred for lightweight, high-frequency reasoning where computer-use capability is not required and deterministic tools can perform the actual action.

Jarvis chooses between them dynamically using a **Model Selection Policy**, based on task complexity, latency target, required capability, context size, current quota, and estimated token cost.

Google currently lists `gemini-3.5-flash-lite` as a current model, and `gemini-3.1-flash-lite` as the GA high-volume workhorse. Google also lists `gemini-3.1-flash-lite` with a May 7, 2027 shutdown date at the time of this amendment, so the model registry must support future replacement without architecture rewrites. citeturn876503search2turn876503search3

## 40.3 Voice model is exclusive

The **conversational voice system must use the Gemini Live native-audio model selected in the voice configuration**. No second conversational LLM may silently replace it.

Voice responsibilities:

- speech input
- turn detection
- conversational state
- natural spoken output
- interruption handling
- barge-in handling
- tool/function calling
- Urdu support
- Hindi support
- English support
- language mixing where the user naturally mixes languages
- male voice options
- additional available voice options
- configurable speaking style, speed, and expressiveness within the model/API's supported controls

The system must separate **voice conversation** from **tool execution**. The voice model speaks/plans/calls tools; deterministic executors perform the work.

---

# 41. DYNAMIC MULTI-AGENT TASK SYSTEM

Jarvis must not start all agents for every request. It uses **dynamic agent spawning** only when task complexity justifies it.

## 41.1 Parent/child task model

```text
User request
    |
    v
Main Orchestrator
    |
    +--> task analysis
    |
    +--> simple task ------------------> direct executor
    |
    +--> complex task -----------------> Agent Task Graph
                                      |
                +---------------------+---------------------+
                |                     |                     |
                v                     v                     v
        Computer Agent         Browser Agent         Search/Research Agent
                |                     |                     |
                +---------- evidence/results -----------+
                                      |
                                      v
                              Result Merger
                                      |
                                      v
                                Main Jarvis
```

## 41.2 Agent lifecycle

Every dynamic agent has:

- `agent_id`
- `parent_task_id`
- `task_type`
- `required_capabilities`
- `model_policy`
- `tool_allowlist`
- `data_scope`
- `max_steps`
- `max_runtime_ms`
- `token_budget`
- `cpu_budget`
- `memory_budget`
- `retry_policy`
- `checkpoint_policy`
- `verification_policy`
- `cancel_token`
- `priority`
- `status`
- `result`
- `evidence`
- `error`

## 41.3 Spawn policy

Jarvis should spawn an agent only when at least one of these is true:

1. The task contains independent parallel subtasks.
2. A specialist capability is required.
3. A long workflow can be decomposed into independently verifiable phases.
4. Continuing in one conversational context would produce excessive token usage.
5. A separate execution environment improves reliability.

For short tasks, **do not spawn agents**.

## 41.4 Parallelism limits

Use bounded concurrency rather than unrestricted agent creation. The scheduler must apply:

- global maximum agents
- per-capability maximum agents
- per-task maximum agents
- API concurrency limits
- CPU utilization threshold
- RAM threshold
- network budget
- user priority

---

# 42. COMPUTER CONTROL AGENT — ADVANCED DESIGN

The Computer Control Agent is a specialized executor/orchestrator for the operating system and desktop applications.

### Model policy

- Primary capable model: **Gemini 3.5 Flash-Lite**
- Lightweight reasoning model: **Gemini 3.1 Flash-Lite**
- Jarvis chooses automatically.

### Execution policy

The model should NOT be asked to reason over every low-level interaction. It should produce a compact action plan such as:

```text
open_app -> wait_for_ready -> focus_window -> perform_action -> verify_state
```

The local executor then performs these deterministic steps.

### Multiple terminal system

Computer Control must support **multiple independent terminal sessions** simultaneously.

Each terminal session has:

- unique terminal ID
- shell type
- working directory
- environment snapshot
- process group
- stdin/stdout/stderr streams
- timeout
- cancellation token
- exit code
- resource counters
- audit record

Example:

```text
Terminal Manager
├── terminal-01 : PowerShell
├── terminal-02 : CMD
├── terminal-03 : WSL Ubuntu
├── terminal-04 : project test shell
└── terminal-05 : background service shell
```

Terminals must be isolated from one another unless Jarvis explicitly passes a result or artifact between them.

The application must never freeze because one terminal process hangs. Every terminal operation is asynchronous or externally monitored with a hard timeout.

---

# 43. SELF-EXTENSION / PLUGIN CREATOR SYSTEM

This is a core Jarvis capability.

## 43.1 Purpose

When Jarvis determines that no registered capability can satisfy a request, it may invoke the **Plugin Creator Agent**.

The flow is:

```text
User request
   |
   v
Capability Registry lookup
   |
   +--> capability exists --> execute + verify
   |
   +--> capability missing
             |
             v
       Plugin Creator
             |
             v
      inspect plugin template
             |
             v
      generate implementation
             |
             v
       write ACTUAL files
             |
             v
       syntax/type checks
             |
             v
       isolated plugin tests
             |
             v
       security/policy review
             |
             v
       hot-load plugin
             |
             v
       run requested action
             |
             v
          verify
```

## 43.2 Actual filesystem requirement

Plugin creation must write to the **real configured Jarvis installation/plugin directory**. No fake virtual filesystem, pretend write, simulation-only file, or temporary in-memory-only implementation may be presented as the finished plugin.

Example:

```text
jarvis/
└── plugins/
    ├── manifest.json
    ├── loader.py
    ├── builtin/
    └── generated/
        └── <plugin_name>/
            ├── plugin.json
            ├── __init__.py
            ├── actions.py
            ├── tests/
            └── README.md
```

The exact final directory layout may be optimized during implementation, but there must remain one clear plugin root and one stable manifest contract.

## 43.3 Plugin template-first workflow

Before creating a plugin, Jarvis loads the canonical plugin template/schema and validates:

- plugin ID
- display name
- version
- entry point
- permissions
- required dependencies
- actions
- events
- settings schema
- configuration defaults
- health check
- uninstall metadata
- rollback metadata
- tests

## 43.4 Hot loading

A newly created and validated plugin must be **loaded without restarting the whole Jarvis process**.

Preferred architecture:

```text
Plugin Registry
     |
     +--> discover manifest
     +--> validate
     +--> sandbox/import
     +--> register actions
     +--> register permissions
     +--> run health check
     +--> activate atomically
```

The loader should support:

- atomic activation
- rollback on failure
- dependency resolution
- plugin versioning
- plugin disable/enable
- plugin unload where safe
- plugin health status
- stale-plugin detection
- cache invalidation only for the affected plugin

No full Jarvis restart should be required for ordinary plugin creation/update.

## 43.5 Plugin safety and reliability

A generated plugin must not automatically receive unrestricted machine access. It receives only declared permissions that pass the policy engine.

The plugin creator must create tests before activation whenever the new plugin introduces executable behavior.

---

# 44. MEMORY ARCHITECTURE — SIX DISTINCT MEMORY TYPES

Jarvis must not place all memory in one giant prompt. Memory is stored, indexed, compressed, retrieved, and summarized by type.

## Memory 1 — Working Memory

Very short-lived current-task state:

- current objective
- current step
- active tool
- current UI state
- pending confirmation
- current errors

TTL: task/session dependent.

## Memory 2 — Episodic Memory

Important past events:

- what happened
- what action was taken
- outcome
- date/time
- relevant evidence

Used for continuity without replaying complete logs.

## Memory 3 — Semantic Long-Term Memory

Stable knowledge Jarvis has learned about the user/system/project:

- preferences
- stable project facts
- recurring workflows
- tool capabilities
- important decisions

Must be editable and forgettable.

## Memory 4 — Procedural Memory

“How to do things” knowledge:

- successful workflows
- action sequences
- recovery strategies
- plugin usage patterns
- verified recipes

Procedural memory should preferably reference deterministic tools rather than raw prose.

## Memory 5 — Tool/Capability Memory

Machine-readable knowledge of:

- available tools
- actions
- schemas
- permissions
- health
- latency
- reliability score
- last successful use
- last failure
- version

This is a registry/cache, not a giant model prompt.

## Memory 6 — Preference/Personalization Memory

User interaction preferences:

- preferred language
- preferred voice
- preferred confirmation style
- UI preferences
- preferred applications
- verbosity preference
- automation defaults

Sensitive information must be isolated and access-controlled.

### Memory efficiency rule

Only retrieve the smallest memory slices needed for the current task. Never attach all six memory stores to every model call.

---

# 45. LOW-TOKEN / LOW-CPU / LOW-LATENCY ENGINE

This is a primary architectural goal.

## 45.1 Token minimization

Jarvis must use models only for tasks that require model intelligence.

Do NOT use an AI model for:

- clicking a known button when a selector/action is already verified
- moving a file to a known path
- reading an exit code
- checking whether a process exists
- polling a known UI state
- comparing a known string
- opening a configured application
- repeating a deterministic workflow
- formatting a known JSON object
- routing a registered tool by exact capability ID

Instead use deterministic executors.

### Tool discovery optimization

The model must not receive the full 200+/100+ action catalog every time.

Use hierarchical retrieval:

```text
User intent
   ↓
Capability category
   ↓
Small candidate set
   ↓
Exact action schema
   ↓
Execute
   ↓
Verify
```

The model sees perhaps a small set of relevant action definitions rather than the entire registry.

## 45.2 Local CPU minimization

Use:

- async event loops
- subprocesses for heavy tasks
- non-blocking I/O
- OS APIs instead of screenshot polling
- event-driven window/process notifications where available
- cached UI state
- incremental DOM/accessibility observations
- bounded worker pools
- debounced events
- adaptive polling intervals
- backoff on repeated failures
- lazy initialization
- lazy imports
- memory-mapped or streaming file operations for large data
- compressed logs
- bounded caches

Avoid:

- continuous full-screen screenshots
- continuous OCR
- busy loops
- 10/20/50 agent processes running idle
- duplicate browser instances
- re-reading entire pages when a targeted state query is sufficient
- re-sending entire conversation history
- repeated model calls for deterministic state checks

## 45.3 Instant-response architecture

The UI must remain responsive even while Jarvis executes a long task.

Never execute a long automation loop on the GUI/event-loop thread.

```text
Node.js UI thread
       |
       +--> IPC/API request
                |
                v
          async Jarvis Core
                |
     +----------+----------+
     |                     |
 deterministic tools      AI calls
     |                     |
     +----------+----------+
                |
                v
         event/result stream
                |
                v
            Node.js UI
```

If a task takes longer than the UI responsiveness threshold, the UI immediately transitions to a progress state instead of blocking.

The application must not display a generic **“Not Responding”** condition because a long operation is running.

---

# 46. RESPONSIVE WINDOW / APPLICATION STABILITY

The following are mandatory engineering controls:

- GUI thread must never run blocking I/O.
- Terminal processes must run outside the GUI event loop.
- Browser communication must be asynchronous.
- Network requests require timeout and cancellation.
- Model calls require timeout and retry policy.
- Tool execution requires watchdogs.
- Every long operation must report progress.
- Every task must be cancellable.
- Deadlocks must be detectable.
- Worker exhaustion must be detectable.
- Unhandled exceptions must be captured at the task boundary.
- Crash recovery must preserve the last durable checkpoint.
- On restart, incomplete safe tasks may resume from checkpoint where supported.

---

# 47. BROWSER AUTOMATION — ENHANCED HUMAN-LIKE INTERACTION MODEL

Jarvis must provide high-quality, **natural user-style browser interaction** for reliability and usability. This means it should interact through legitimate browser mechanisms and produce realistic sequences of actions rather than using brittle fixed-coordinate scripts.

It must NOT be designed to bypass CAPTCHA, anti-bot protections, access controls, platform safeguards, or detection systems. When a site presents a challenge that requires user action, Jarvis should pause and request/allow appropriate user input.

## 47.1 Dual browser execution modes

Jarvis may choose between:

- **Chrome DevTools Protocol / Chrome DevTools MCP** for an already-running/live Chrome instance, deep browser inspection, debugging, performance, tabs, network/console state, and direct DevTools workflows.
- **Playwright** for structured browser automation, isolated sessions, robust selectors, navigation, waiting, screenshots, downloads, and cross-browser test environments.

The choice is per task, not global.

## 47.2 Natural interaction requirements

Browser Agent should prefer:

- semantic selectors
- accessibility tree data
- visible labels
- element roles
- DOM state
- network completion signals
- application state
- deliberate action pacing where user experience benefits
- focus management
- correct scrolling
- context-aware typing
- checking for disabled/loading/error states
- confirmation before irreversible external actions

It should not randomly inject delays merely to imitate a bot avoidance strategy.

## 47.3 Browser observation hierarchy

Use the cheapest reliable observation first:

1. known application state
2. DOM/accessibility state
3. browser event
4. network state
5. targeted screenshot
6. visual model reasoning only when needed

---

# 48. SEARCH SYSTEM

Search must be a first-class capability but should not create unnecessary model calls.

```text
Intent
 ↓
Query normalization
 ↓
Search provider
 ↓
result filtering
 ↓
relevance ranking
 ↓
optional Gemma reranking/summarization
 ↓
compact evidence package
```

Use deterministic search retrieval first. Use the Gemma model through the Gemini API only when semantic ranking, extraction, clustering, or summarization adds measurable value.

The complete search results page must never be stuffed into the main conversation context by default.

---

# 49. PREMIUM NODE.JS GUI SYSTEM

The desktop graphical interface must be implemented with **Node.js/TypeScript** and a modern desktop UI layer suitable for a standalone application.

The implementation may use a lightweight Node desktop shell such as Electron or another suitable Node-compatible desktop runtime if justified during implementation, but the UI architecture must remain separate from Jarvis core.

## 49.1 GUI design goals

- premium
- modern
- attractive
- fast
- responsive
- minimal visual clutter
- strong hierarchy
- keyboard accessible
- voice-first friendly
- live task visualization
- agent/task visibility without overwhelming users
- plugin management
- model settings
- browser/desktop status
- memory controls
- logs/evidence
- permissions
- health dashboard

## 49.2 Exact color system

Use a controlled design token system; do not scatter arbitrary hex colors through components.

### Core palette

- **App background:** `#080B12`
- **Primary surface:** `#0F1420`
- **Secondary surface:** `#151C2A`
- **Elevated surface:** `#1B2434`
- **Border:** `#273246`
- **Primary text:** `#F5F7FB`
- **Secondary text:** `#A8B1C2`
- **Muted text:** `#707B8F`
- **Primary accent:** `#7C5CFF`
- **Primary accent hover:** `#947AFF`
- **Secondary accent / cyan:** `#22D3EE`
- **Success:** `#32D583`
- **Warning:** `#F5B544`
- **Error:** `#FF5C6C`
- **Info:** `#5EA7FF`
- **Agent-active indicator:** `#A78BFA`
- **Voice-active glow:** `#22D3EE`
- **Border focus:** `#9B8AFB`

### Color usage

- Never use pure `#000000` as the main background.
- Never use pure white for all text.
- Use accent colors sparingly.
- Success/warning/error colors are semantic, not decorative.
- Keep contrast accessible.
- Use a maximum of one dominant accent per screen.

## 49.3 Typography

Preferred UI font stack:

```text
Inter, ui-sans-serif, system-ui, -apple-system, "Segoe UI", sans-serif
```

Urdu/Hindi fallback:

```text
"Noto Sans Arabic", "Noto Sans Devanagari", Inter, sans-serif
```

Exact font assets must not be bundled without verifying licensing. The design system should rely on installed/system or correctly licensed fonts.

## 49.4 UI screens

Required primary screens:

1. Home / conversation
2. Live voice
3. Active task
4. Agent activity
5. Browser control
6. Computer control
7. Files
8. Plugins
9. Memory
10. Search
11. Settings
12. Security/permissions
13. Logs/evidence
14. Health/performance
15. Developer/debug mode

## 49.5 Main screen layout

```text
┌─────────────────────────────────────────────────────────┐
│ JARVIS                       status • voice • settings  │
├─────────────────────────────────────────────────────────┤
│                                                         │
│              conversation / task workspace             │
│                                                         │
│                                                         │
├──────────────────────┬──────────────────────────────────┤
│ Tasks / Agents       │ Live actions / evidence         │
│                      │                                  │
├──────────────────────┴──────────────────────────────────┤
│  🎙 Voice input            Type a command...        ➤   │
└─────────────────────────────────────────────────────────┘
```

## 49.6 UI performance

The GUI must render state changes from an event stream. It must not repeatedly poll the whole backend state.

Use incremental updates:

```text
backend event -> small state patch -> UI store -> affected components only
```

---

# 50. INDUSTRY-STANDARD OBSERVABILITY

Every task has structured telemetry:

- task ID
- parent task ID
- timestamp
- action ID
- tool ID
- agent ID
- model ID
- latency
- token estimate/usage where available
- retry count
- result status
- verification status
- error code
- evidence references

Logs must be structured and bounded. Large raw screenshots/video should not be kept by default.

## 50.1 Reliability score

Every tool/action may maintain:

```text
success_rate
average_latency
recent_failure_rate
verification_rate
availability
last_success
last_failure
version
```

Jarvis may prefer a more reliable deterministic tool even when two tools can theoretically accomplish the same action.

---

# 51. VERIFICATION-FIRST EXECUTION

Jarvis must distinguish these states:

```text
PLANNED
DISPATCHED
RUNNING
OBSERVED
VERIFIED
FAILED
CANCELLED
UNKNOWN
```

The assistant must never claim **VERIFIED** merely because a tool call returned without an exception.

### Example

```text
Instruction:
"Open Chrome and search X."

Action result:
process started

Verification 1:
Chrome window exists -> PASS

Verification 2:
page URL changed -> PASS

Verification 3:
search page state observed -> PASS

Final:
VERIFIED
```

If evidence is insufficient:

```text
UNKNOWN
```

Jarvis must report uncertainty rather than invent success.

---

# 52. SELF-HEALING SYSTEM

Recovery hierarchy:

1. repeat deterministic step once if idempotent
2. refresh/re-query state
3. use alternate compatible adapter
4. restart affected plugin/worker only
5. rollback last plugin update if relevant
6. spawn specialist recovery agent for complex cases
7. request user assistance when external interaction is genuinely required

Never restart the entire Jarvis application as the default recovery action.

---

# 53. CONFIGURATION AND FLEXIBILITY

All non-secret system policies must be configuration-driven:

- model selection
- model priorities
- API endpoints
- browser mode
- plugin directory
- memory limits
- concurrency
- CPU thresholds
- timeout values
- retry policies
- UI theme
- voice selection
- language priorities
- permissions
- logging level
- evidence retention

Secrets/API keys must be stored outside source code in a secure configuration mechanism appropriate to the operating system.

---

# 54. SCALABILITY ARCHITECTURE

The initial build remains a single installable application, but internal interfaces must allow later separation into services without rewriting business logic.

Stable boundaries:

```text
UI
  ↕
Application API / IPC
  ↕
Core Orchestrator
  ↕
Capability Registry
  ↕
Executors / Agents / Adapters
  ↕
External systems
```

No business logic should be locked inside GUI components.

No browser-specific logic should be hard-coded into the core orchestrator.

No provider-specific model logic should be scattered throughout tools.

---

# 55. INDEPENDENT APP + INSTALLER + SAAS PRODUCT END STATE

The final product must evolve from a development repository into an **independent installable application** and later a **SaaS product**.

## 55.1 Standalone desktop application

Required final deliverables:

- independent application executable/package
- own application identity/name
- own icons/assets
- own configuration directory
- own data directory
- own updater/migration mechanism
- own installer
- clean uninstall
- versioned releases
- crash recovery
- diagnostic export

## 55.2 Installer

The installer must:

- detect supported OS/runtime requirements
- install required bundled components
- create required directories
- initialize configuration
- register file associations only when explicitly desired
- create shortcuts only when selected
- validate permissions
- perform post-install health check
- support upgrade without destroying user data

## 55.3 SaaS readiness

The architecture must support future cloud services such as:

- account management
- subscription/billing integration
- cloud sync
- team workspaces
- plugin distribution
- remote task execution
- usage accounting
- audit logs
- organization policies
- model/API management
- secure secrets storage

Local-first functionality should remain possible where privacy, latency, and platform constraints permit.

The SaaS layer must not require rewriting Jarvis core. It should sit above stable APIs.

---

# 56. DEVELOPER EXPERIENCE / ZERO-MESS STANDARD

The codebase must remain:

- understandable
- searchable
- typed
- documented where decisions matter
- consistently named
- testable
- modular
- minimally dependent

### File creation rule

Before creating a new file, the builder must answer internally:

1. Does an existing module already own this responsibility?
2. Can the feature be added without a new file?
3. Would a new file create a duplicate abstraction?

Create a file only when it represents a real stable boundary.

### No temporary junk

Do not leave:

- generated debug scripts
- unused screenshots
- temporary patch files
- copied source trees
- backup duplicates
- experimental dead code
- downloaded model weights
- unnecessary caches in the product repository

Use external temporary directories or CI workspaces for build artifacts.

---

# 57. TEST-LAB REQUIREMENTS FOR THE NEW SYSTEMS

The cloud-first test lab must be capable of testing not only code correctness but behavioral correctness.

## 57.1 Behavioral test format

```text
SCENARIO
INPUT COMMAND
EXPECTED ACTIONS
OBSERVABLE CHECKPOINTS
EXPECTED FINAL STATE
FAILURE CONDITIONS
RECOVERY EXPECTATION
PASS/FAIL
```

## 57.2 Dynamic-agent tests

Examples:

- spawn no agent for simple action
- spawn browser agent for browser-heavy task
- spawn computer agent for desktop task
- spawn both when a task crosses browser and desktop domains
- merge results correctly
- cancel child task
- recover child task
- enforce child token budget

## 57.3 Plugin creator tests

Examples:

- detect missing capability
- call plugin creator
- generate real files
- validate manifest
- run plugin tests
- reject invalid plugin
- activate valid plugin without restart
- rollback failed hot-load
- persist plugin across restart

## 57.4 Responsiveness tests

The test lab must verify:

- UI remains interactive during long tasks
- terminal hang does not freeze GUI
- network timeout does not freeze GUI
- model timeout does not freeze GUI
- browser hang does not freeze GUI
- plugin failure does not freeze core
- high event volume is throttled/coalesced

---

# 58. NATURAL INTERACTION QUALITY — NOT “BOT-LIKE” IN USER EXPERIENCE

The browser and computer systems should feel natural and reliable to the user by:

- choosing meaningful targets instead of brittle coordinates
- maintaining focus correctly
- waiting for actual state transitions instead of fixed arbitrary sleeps
- scrolling only as needed
- typing into the correct active field
- preserving normal user context
- handling popups and loading states
- using appropriate keyboard shortcuts when faster and reliable
- minimizing needless movement and repeated actions
- confirming important actions
- verifying the resulting state

This requirement is about **natural, robust user interaction**, not evading anti-bot/security systems. Jarvis must respect site policies, access controls, authentication, and user consent.

---

# 59. ADDITIONAL SYSTEMS TO COMPLETE THE INDUSTRY-STANDARD DESIGN

The previously defined 15 systems remain active. The following supporting systems are now explicitly required as well:

## 59.1 Lifecycle Manager
Attributes:
- startup stages
- readiness checks
- dependency ordering
- graceful shutdown
- restart scope
- crash recovery

## 59.2 Capability Registry
Attributes:
- action ID
- schema
- category
- permissions
- adapter
- health
- reliability
- version
- aliases

## 59.3 Event Bus
Attributes:
- event type
- event ID
- timestamp
- source
- target
- correlation ID
- payload schema
- replay policy

## 59.4 Policy Engine
Attributes:
- risk level
- permissions
- confirmation requirement
- allowed applications
- allowed domains
- allowed file roots
- audit requirements

## 59.5 Resource Governor
Attributes:
- CPU ceiling
- RAM ceiling
- concurrent jobs
- network budget
- model concurrency
- terminal count
- browser count
- degradation policy

## 59.6 Update/Migration Manager
Attributes:
- version
- migration ID
- backup policy
- rollback
- compatibility
- health check

## 59.7 Secrets Manager
Attributes:
- encrypted storage
- secret ID
- scope
- rotation
- expiry
- redaction
- access audit

## 59.8 Artifact Manager
Attributes:
- artifact ID
- source task
- file type
- checksum
- location
- retention
- cleanup policy

## 59.9 Notification Manager
Attributes:
- channel
- priority
- user preference
- deduplication
- quiet hours
- escalation

## 59.10 Localization Manager
Attributes:
- language
- locale
- script
- date format
- number format
- voice language
- translation resources

These systems are intentionally lightweight and should not become separate microservices in the initial desktop product.

---

# 60. MASTER PERFORMANCE TARGETS

These are engineering targets, not guarantees:

- near-instant UI acknowledgement for commands
- no GUI blocking from long-running work
- no continuous idle model calls
- no local Gemma inference
- no unnecessary screenshot loop
- no full action catalog in every prompt
- deterministic execution for repeated tasks
- bounded concurrency
- bounded memory/cache growth
- graceful degradation under API latency
- task cancellation available at every long-running boundary
- targeted recovery rather than full-app restart

For performance validation, measure rather than assume:

- idle CPU
- active CPU
- RAM
- startup time
- command acknowledgement latency
- first model token latency
- tool execution latency
- verification latency
- UI frame responsiveness
- API token usage
- number of model calls per task

---

# 61. BUILD ORDER FOR THE FINAL PRODUCT

The builder should implement in dependency order:

```text
1. Repository + contracts
2. Lifecycle Manager
3. Config + Secrets
4. Event Bus
5. Capability Registry
6. Policy Engine
7. Core Orchestrator
8. Deterministic execution layer
9. Verification Engine
10. Memory layer
11. Gemini Live voice
12. Computer Control Agent
13. Multi-terminal manager
14. Browser Agent + CDP/Playwright adapters
15. Search
16. Plugin runtime
17. Plugin Creator / self-extension
18. Node.js GUI
19. Behavioral Test Lab
20. Packaging / Installer
21. SaaS API boundary
22. Performance hardening
23. Security review
24. Regression suite
25. Release candidate
```

The builder must continuously run focused tests after each phase rather than waiting until the entire system is written.

---

# 62. FINAL ADDITIVE CHECKLIST

The finished Jarvis must satisfy all of the following:

- [ ] Existing master specification preserved
- [ ] All previous 898 capability entries preserved
- [ ] Dynamic multi-agent orchestration
- [ ] Dynamic child-agent spawning only when justified
- [ ] Computer Agent with Gemini 3.5 Flash-Lite / 3.1 Flash-Lite selection
- [ ] Browser Agent with cloud-accessed Gemma model policy
- [ ] Gemma accessed through Gemini API key, not local model weights
- [ ] Gemini-native conversational voice
- [ ] Natural Urdu/Hindi/English voice interaction
- [ ] Male voice and other configurable voice options where supported
- [ ] Multiple independent terminal sessions
- [ ] Actual filesystem writes for generated plugins
- [ ] Canonical plugin template
- [ ] Plugin validation and tests
- [ ] Plugin hot-load without full Jarvis restart
- [ ] Plugin rollback
- [ ] Six distinct memory types
- [ ] Low-token tool routing
- [ ] Low-CPU deterministic execution
- [ ] Responsive GUI with no blocking long tasks
- [ ] Node.js/TypeScript interactive premium GUI
- [ ] Exact centralized color tokens
- [ ] Browser CDP/Chrome DevTools + Playwright selectable per task
- [ ] Natural robust browser interaction, without bypassing site safeguards
- [ ] Search system with compact evidence retrieval
- [ ] Verification-first completion claims
- [ ] Self-healing/recovery
- [ ] Industry-standard observability
- [ ] Independent desktop application
- [ ] Dedicated installer
- [ ] SaaS-ready API boundaries
- [ ] Scalable architecture
- [ ] Flexible adapters
- [ ] Cloud-first build/test strategy
- [ ] Full regression and behavioral test lab

**This section is additive. It must be implemented without deleting earlier sections, capabilities, or action inventories.**
