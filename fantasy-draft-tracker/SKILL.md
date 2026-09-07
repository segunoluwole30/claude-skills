---
name: fantasy-draft-tracker
description: Build a live, interactive fantasy sports draft board (NFL, NBA, MLB, etc.) that the user clicks through during their actual draft to mark players as drafted, instead of typing names into chat one at a time. Use this whenever the user says they have an upcoming fantasy draft, wants a "draft cheat sheet," "draft tracker," "draft assistant," or "mock draft helper," wants help prepping for a snake/auction/linear draft, or asks to redo/rebuild the kind of live draft-pick assistance previously done by hand in chat. Always trigger this proactively before a live draft rather than falling back to manually tracking picks turn-by-turn in the conversation — manual tracking in chat is error-prone (stale player lists, missed picks) and is exactly the failure mode this skill exists to avoid.
---

# Fantasy Draft Tracker

Builds a self-contained interactive HTML artifact that acts as a live, filterable draft board for a fantasy sports draft. The user marks players off as they're drafted (by anyone in the league) directly in the artifact — fast clicks instead of typing full player lists into chat — and the board auto-highlights the best remaining players so Claude (or the user) can make a fast, accurate call when it's their turn.

This exists because live drafts move fast (sometimes 30-90 seconds per pick) and a user cannot type out the whole remaining player pool each turn. A static markdown cheat sheet is a good fallback but requires Claude to manually track who's gone from chat messages, which is error-prone over a long draft (stale state, missed picks, recommending already-taken players). The interactive artifact removes that failure mode: the checked-off state lives in the artifact itself, not in Claude's memory of the conversation.

## Step 1: Gather inputs

Before doing anything else, ask the user for (use ask_user_input_v0 tool for quick taps where possible):

1. **Sport** (NFL, NBA, MLB, etc.) — determines positions, stat categories, and where to research rankings.
2. **League size** (number of teams) — affects how deep the player pool needs to go and how fast positions dry up.
3. **Draft type** (snake, auction, linear/straight) — affects strategy notes, though the board itself works the same way for all types.
4. **Scoring format** (PPR/half-PPR/standard for NFL; category vs. points league for NBA; etc.)
5. **Roster/lineup settings** — starting slots and bench size (e.g. QB/RB/RB/WR/WR/TE/FLEX/K + 6 bench). If the user doesn't know exact bench size, a reasonable default is fine — note the assumption.
6. **Player pool depth** — default to top ~200 overall for a redraft league of typical size (10-14 teams); go deeper (250+) for very deep benches or leagues with 16+ teams; a smaller 6-8 team league can often get away with top 120-150.

Don't proceed to research until you have sport + league size + scoring format at minimum — draft type and roster settings can default sensibly if the user doesn't have them handy, but say what you assumed.

## Step 2: Research current rankings

Web search for current-season consensus rankings/ADP for the specified sport and scoring format. Look for:
- Overall top-N rankings with ADP (use multiple queries — "top 200 overall," then per-position if the overall list runs thin on depth)
- Position-specific rankings for full depth at every relevant position
- A recent injury/availability report to flag risk (aim for searches within the last few days of the actual draft date — this data goes stale fast)

For each player, capture: **rank, name, position, team, ADP (or auction value if it's an auction draft), a status flag (healthy/monitor/risk), and a one-line note** (role, injury detail, or context that would change a draft decision — e.g. "committee back," "recovering from ACL," "clear WR1 after teammate traded").

Aim for the same depth and rigor as a written cheat sheet, but you're structuring it as data for the artifact rather than prose for chat. Use enough searches to cover the full requested depth across every relevant position — don't stop at a shallow top-50 if the user needs 200.

## Step 3: Build the interactive artifact

Build a single-file HTML artifact (see `/mnt/skills/public/frontend-design/SKILL.md` for styling constraints if visual polish matters to the user — otherwise a clean, functional table-based UI is the priority here, not visual flourish).

Required features:
- **Full player table**: rank, name, position, team, ADP, status (color-coded dot or badge), note. Baked in as a JS data array in the code — no external fetch calls (this must work fully offline/standalone once built).
- **Mark-as-drafted control**: a click target (checkbox or button) per row that toggles a player to "drafted" — visually greys out / strikes through / moves to a collapsed "drafted" section, and removes them from the "best available" calculations.
- **Filter/sort**: filter by position (ALL/QB/RB/WR/TE/etc. or sport equivalent), sort by rank or ADP, and a text search box for jumping straight to a name during a fast-moving draft.
- **Best-available callout**: a prominent, always-visible section showing the top 3-5 remaining players overall (and ideally top remaining per position) — this is what the user glances at when it's their pick.
- **My team tracker**: a second click state (not just "drafted," but "drafted BY ME") so the user can distinguish their own picks from opponents' picks. Default to including this — it's needed for the copy-state feature below to give position-aware recommendations, not just "who's left."
- **Copy-state button ("Copy for Claude")**: Claude cannot see the artifact's live state — it renders and runs entirely in the user's browser, disconnected from the chat session. There is no live channel back to Claude as the user clicks through picks. So the artifact must include a button that generates a short, plain-text summary the user can paste into chat on demand, e.g.:

  ```
  DRAFTED (all): Gibbs, Bijan Robinson, Chase, McCaffrey, Taylor, ...
  MY TEAM: Nacua, Cook, Walker, McBride
  ```

  Implement this with a button that builds the string from current state and copies it to the clipboard (`navigator.clipboard.writeText`), plus a visible text box showing the same output as a fallback in case clipboard access is blocked in the artifact sandbox. Keep both lists as plain comma-separated names — no extra formatting — since that's fastest to paste and cheapest for Claude to parse. This is the mechanism that lets Claude give a real recommendation later: "who's already gone across the whole league" plus "what's already on my roster" is exactly what's needed to judge both availability and positional need.
- **Persist nothing to browser storage** — per artifact rules, no localStorage/sessionStorage. State lives in React/JS state for the session. Mention this to the user: if they refresh the page mid-draft, the checked-off state resets, so don't reload the tab during the live draft.

Keep the interaction fast: one click to mark a player drafted (and a second tap/toggle to flag it as the user's own pick), immediately visible re-sort of "best available." The user should almost never need to type a player name to Claude during the live draft — the only thing they paste in is the one-click copy-state summary, and only when they actually want a recommendation.

## Step 4: Deliver and set expectations

- Save to the outputs directory and present the file per standard artifact/file delivery.
- Tell the user plainly: rankings/ADP/injury data are a snapshot from research done today — if there's a gap of more than a day or two before the actual draft, especially close to a season opener when injury news moves fast, offer to re-run the research and rebuild the board right before draft day.
- Clarify the division of labor: the user does the clicking/tracking in the artifact themselves during the live draft — that's the source of truth for who's gone and what's on their roster, not Claude's running memory of the conversation.
- Explain the recommendation workflow explicitly: when the user wants a pick recommendation, they hit "Copy for Claude" in the artifact and paste the resulting DRAFTED/MY TEAM summary into chat. From that paste, cross-reference against the full researched player list (rank/ADP/status/notes) to name the best available player, and factor in the user's own roster (from MY TEAM) for positional need rather than pure best-player-available once their team is deep at a position. Do not attempt to track draft state from prior chat messages alone once this skill is in use — always work from the latest pasted summary, since it's the only state guaranteed to be accurate and current.
- If the user pastes a summary and a name in it doesn't match anything in the researched player list (a late-round/deep-league name that wasn't covered), say so plainly rather than guessing, and offer a quick web search to check that specific player if it matters for the recommendation.

## Step 5: Reuse across sports

This skill is sport-agnostic by design. When the user comes back for a different sport (e.g. NBA after football season), re-run Steps 1-4 fresh — league settings, positions, and current rankings all change, so don't reuse an old sport's artifact or data.
