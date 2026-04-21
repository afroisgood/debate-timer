# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

辯論計時器 — a two-page web app for a live debate event "生活之間：從住民到公民". No build step, no framework, no package manager. Pure HTML/CSS/JS deployed to GitHub Pages.

**Live URL**: https://afroisgood.github.io/debate-timer/

## Deploy

```bash
git add index.html vote.html
git commit -m "..."
git push origin main   # GitHub Pages auto-deploys from main branch
```

Changes are live within ~1 minute of push. No build or CI step.

## Architecture

### Two pages

| File | Role |
|---|---|
| `index.html` | Admin / main display — shown on the presenter's screen. Controls timer, manages stages, displays live votes and comments. |
| `vote.html` | Participant page — opened on phones via QR code. Submits votes, comments, likes, and topic ballot votes. |

### Firebase Realtime Database (compat SDK 10.12.0)

All state is shared via Firebase. No backend code exists — everything is client-side JS talking directly to the DB.

**Data paths:**
```
debate/
  config/
    currentStage     String  — stage name synced from index.html on every loadStage()
    votingOpen       Boolean — controls whether 觀眾投票 is accepting votes
    topicVotingOpen  Boolean — controls whether 下期題目票選 is accepting votes
  votes/
    {deviceId}/      { nickname, side, updatedAt }
  comments/
    {pushId}/        { nickname, text, side, createdAt, deleted }
  likes/
    {commentId}/
      {deviceId}     true
  topicVotes/
    {deviceId}/
      {topicIndex}   true
```

**Firebase rules** must explicitly grant `.read`/`.write` per sub-node. Reading a parent node (`debate/`) fails silently if the parent has no `.read` rule — always read individual sub-nodes.

### Stage system (index.html)

`STAGES` array drives everything: each entry is `[name, durationSeconds]`. Duration `0` = no timer (end states). Special stage names trigger different UI modes:

- `"自由辯論"` → dual-circle timer (正方/反方 independent countdowns)
- `"下期題目票選"` → full-width topic ballot display, 8-minute timer, auto-closes voting at 0
- `"活動結束"` → terminal state

`loadStage()` is the central function: stops timer, resets state, syncs `currentStage` to Firebase, shows/hides UI panels.

### Admin gate (index.html)

All timer/nav controls are `disabled` on load. `setControlsEnabled(bool)` enables/disables: prev/next stage, toggle, reset, ±time buttons, stage-select dropdown, dot bar navigation, switch-side button, and the reset-topic-votes button. Password is hardcoded as `ADMIN_PASSWORD`.

Keyboard shortcuts (space/arrows/R/F11) are also blocked when not admin.

### vote.html stage switching

`vote.html` listens to `debate/config/currentStage` and switches between three section states:
- Normal: shows `#vote-section`, `#comment-section`, `#feed-section`
- Topic ballot (`"下期題目票選"`): shows `#topic-vote-section` only
- Voting closed: `votingAllowed` flag blocks `castVote()`; `topicVotingAllowed` flag blocks `handleTopicVote()`

### Performance patterns

- **Like updates**: patch-only via `data-id` attribute — no full re-render of comment list
- **Like debounce**: `likingInFlight` Set prevents double-tap race conditions
- **Firebase cleanup**: `beforeunload` calls `.off()` on all refs in both files

### Topics constant

Both files contain an identical `TOPICS` array (6 strings). If topics change, update both files.

## Firebase Rules Reference

Current rules structure (update in Firebase Console if adding new paths):

```json
{
  "rules": {
    "debate": {
      "config":     { ".read": true, ".write": true },
      "votes":      { ".read": true, "$deviceId": { ".write": true } },
      "comments":   { ".read": true, "$commentId": { ".write": "..." } },
      "likes":      { ".read": true, "$commentId": { "$deviceId": { ".write": true } } },
      "topicVotes": { ".read": true, "$deviceId": { ".write": true } }
    }
  }
}
```
