# PR Response Doc — CineLog Watchlist Feature

## AI Usage
**Comment 5 (devil's advocate):** Asked AI to attack my date-added draft. It flagged two gaps — newest-first buries long-standing intent, and I was dismissing alphabetical's findability. Both were real, so I named those costs explicitly and justified newest-first deliberately. Core argument was my own.

## Comment 1 — Rename
**What I did:** I renamed all instances of save_to_watchlist() into add_to_watchlist()
**Where I looked:** Ran a repo-wide grep for both save_to_watchlist and add_to_watchlist across all .py files. Call sites were the definition in services/watchlist_service.py, the import + call in routes/watchlist/watchlist.py, and the import + usage in tests/test_watchlist.py.
**How I verified:** I global searched for all instanced of save_to_watchlist() to ensure none remained.

## Comment 2 — Deduplication
**What I did:** Added deduplication logic into add_to_watchlist() using a similar formatting from add_to_collection()
**How I verified:** Ran the app live. first add returned 201, duplicate returned 409, and the DB held only one row.

## Comment 3 — Missing test
**What I did:** Added tests/test_watchlist.py with a test that adding a nonexistent film_id raises FilmNotFoundError.
**Test I used as a model:** tests/test_collection.py — its docstring points you there as the pattern for the codebase. I mirrored its app and sample_user fixtures (in-memory SQLite, create_all/drop_all teardown) and its pytest.raises structure.
**How I verified:** Ran pytest tests/test_watchlist.py -v — 1 passed.

## Comment 4 — Default visibility
**My position:** Keeping `public=True`. CineLog is a social film app, so adding to a watchlist is an implicitly social act — the same reason Letterboxd defaults watchlists to public.
**Reasoning:** I'm optimizing for the common case. Users add films they're excited to share, and public-by-default is what feeds discovery and the follow graph. Defaults are sticky, so a private default would leave sharing as a feature almost nobody opts into.
**Tradeoff acknowledged:** The cost is the less privacy-conservative default. A user who'd rather keep an entry private is exposed until they notice the flag. I accept that here because the data is low-sensitivity (films you want to watch), but only if `public` is editable at add-time; for anything more sensitive I'd flip to private-by-default.

## Comment 5 — Sort order
**My position:** Agreed — switched `get_watchlist()` from alphabetical to `WatchlistEntry.date_added.desc()` (newest first), matching `get_collection()`.

**Reasoning:** A watchlist isn't a reference table you look titles up in — it's a record of intent captured over time. On CineLog that intent is discovery-driven: users add films reacting to the feed. Date-added preserves that story; alphabetical erases it and floats "Amélie" above a film you added tonight and actually want to watch.

**Engagement with reviewer's point:** The reviewer's reason is consistency with `get_collection()`. I'd flag that consistency alone doesn't carry it — collection is a *watched* log, watchlist is a *to-watch* backlog, so "match the other list" is a false friend; the watchlist reasoning above is what actually settles it. It also doesn't decide *direction*, and there I'm accepting two real costs of newest-first: it buries long-standing intent, and it loses alphabetical's findability ("did I already add this?"). I still chose it because on CineLog top-of-list tracks current excitement. If those costs bite, the clean fix is a `?sort=` param defaulting to date-desc — flagging it as the next step, not shipping it here.

## Comment 6 — Rebase
**What conflicted:** A trivial `.gitignore` add/add (took the union). The real issue was silent: `main`'s migration deleted the `WatchlistEntry` model, and since no commit of mine touched `models.py`, git kept the deletion without flagging a conflict — dropping the model my feature needs.
**How I resolved it:** Kept main's UUID versions of `Film` and `CollectionEntry`, and re-added `WatchlistEntry` with `film_id` as a `String(36)` UUID instead of `Integer`, plus fixed the leftover integer references in the watchlist docstrings.
**How I verified no conflict remains:** `git log --merges origin/main..HEAD` is empty (linear history, no merge commits), all 5 tests pass, and I ran the add/dedup/not-found/view flow end-to-end with real UUIDs.

## PR Description
<!-- Written at the end — feature overview, design decisions, manual testing steps -->