# PR Response Doc — CineLog Watchlist Feature

## AI Usage
<!-- Fill in at the end — how you used AI tools during this project -->

## Comment 1 — Rename
**What I did:**
Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py`
(both the `def` and its docstring) to match the `verb_to_noun` convention used by
`add_to_collection()`.

**How I found all call sites:**
I did a project-wide search rather than trusting the one call site the reviewer named.
I ran `grep -rn "save_to_watchlist" --include="*.py" .` which returned three hits:
- `services/watchlist_service.py:12` — the function definition
- `routes/watchlist/watchlist.py:8` — the `from ... import` statement
- `routes/watchlist/watchlist.py:32` — the actual call
The reviewer mentioned the call site; the search also caught the **import line**, which is
easy to miss. I updated all three.

**How I verified:**
Re-ran `grep -rn "save_to_watchlist" --include="*.py" .` → zero results (no stale references
left). Then `grep -rn "add_to_watchlist"` confirmed the definition, import, and call all use
the new name consistently, and the full test suite still imports/runs, so the import resolves.

## Comment 2 — Deduplication
**What I did:**
Added a duplicate guard to `add_to_watchlist()`, patterned directly on `add_to_collection()`
in `services/collection_service.py`. I added an `AlreadyInWatchlistError` exception (mirroring
`AlreadyInCollectionError`) and, after the film-exists check and before creating the entry,
query for an existing row:
```python
existing = WatchlistEntry.query.filter_by(user_id=user_id, film_id=film_id).first()
if existing:
    raise AlreadyInWatchlistError(...)
```

**How I understood the model logic:**
I studied `add_to_collection()`'s check: `.filter_by(user_id=..., film_id=...).first()` returns
the matching row or `None`; if a row comes back (truthy) it raises `AlreadyInCollectionError`
and creates nothing. The key adaptation was querying `WatchlistEntry` — an initial copy of the
pattern queried `CollectionEntry`, which would have checked the wrong table (letting watchlist
duplicates through while wrongly blocking already-watched films). I corrected it to query
`WatchlistEntry`.

**How I verified the deduplication works:**
I wrote a small ad-hoc script against an in-memory SQLite DB: create a user + film, call
`add_to_watchlist` twice, and check the outcome. Result:
- 1st add → succeeds
- 2nd add → raises `AlreadyInWatchlistError` ("Film '1' is already on this user's watchlist")
- row count for `(user_id, film_id)` = 1 (no duplicate persisted)

## Comment 3 — Missing test
**What I did:**
Created a new self-contained `tests/test_watchlist.py` following the same fixture and assertion
structure as `tests/test_collection.py` (inline `app`, `sample_user`, `sample_film` fixtures),
and added `test_add_to_watchlist_nonexistent_film_raises`.

**Which test I used as my model:**
`test_add_to_collection_nonexistent_film_raises` in `tests/test_collection.py`. It uses
`pytest.raises(FilmNotFoundError)` around an `add_*` call with a non-existent film id. I mirrored
that structure. One adaptation: the collection test uses a UUID string for the fake id, but on
this branch `Film.id` is still an **integer**, so I used a non-existent integer id (`999999`).

**How I verified:**
Ran `pytest tests/test_watchlist.py -v` → `1 passed`. (Before the fix it errored with
`fixture 'app' not found`; adding the fixtures inline, matching the collection file, resolved it.)

## Comment 4 — Default visibility
**My position:**
The `public` field on `WatchlistEntry` should default to `False`, not `True`. A watchlist
should be private by default, and a user should have to explicitly opt in to sharing it. I changed `models.py` to reflect this
(`public = db.Column(db.Boolean, default=False)`).

**Reasoning:**
I'm optimizing for the user's reasonable expectation of privacy. A watchlist is a record of
what someone *intends* to watch, which can be personal — a film tied to a sensitive topic, a
mood, or something they simply don't want associated with them publicly. If a user wants to
save a movie without others seeing it, that's well within their rights, and the system should
respect that by default. Privacy-by-default also follows the safer principle that sharing
personal data should be a deliberate choice the user makes, not something that happens
automatically because they didn't find and flip a setting. With `default=True`, a user can
expose their list without ever consciously deciding to — that's a decision being made *for*
them, which I don't think is acceptable for this kind of data.

**Tradeoff acknowledged:**
The cost of defaulting to private is that the social / discovery side of the feature is weaker
out of the box. If watchlists are meant to help friends see what each other wants to watch and
get recommendations, a private-by-default list makes that value invisible until users
explicitly opt in — and many never will, so the shared-discovery experience will feel emptier
and adoption of the social features will be slower. I accept that tradeoff: I'd rather the
sharing feature grow more slowly from users who actively choose it than turn every user's list
public by default and risk exposing something they wanted kept private. The engagement upside
of `public=True` isn't worth overriding a reasonable privacy expectation without consent.

## Comment 5 — Sort order
**My position:**
I'm adopting the maintainer's preference: `get_watchlist` should return entries sorted by
date added, newest first, rather than alphabetically by title. I changed
`services/watchlist_service.py` from `.order_by(Film.title.asc())` to
`.order_by(WatchlistEntry.date_added.desc())` (and dropped the now-unneeded `.join(Film)`,
matching how `get_collection` is written).

**Reasoning:**
The behavior I'm optimizing for is "show me what I'm currently thinking about watching." When
a user adds a film to their watchlist, that recent addition is the one most top-of-mind, so
surfacing it at the top matches intent — the list reads like a queue of recent interest rather
than a static catalog. It also makes the two list endpoints behave identically: `get_collection`
already returns newest-first, so a user (and any client code) can rely on one predictable
ordering rule across the app instead of learning that "watched" sorts by recency but "want to
watch" sorts by title. Consistency here isn't just tidiness — it removes a genuine source of
confusion for anyone consuming both endpoints.

**Engagement with reviewer's point:**
The maintainer's argument rests on two things — consistency with `get_collection` and recency
as a proxy for relevance — and I think both hold up under scrutiny, which is why I'm agreeing
rather than deferring. I did seriously weigh the counter-argument for alphabetical: a watchlist
is a "pick something to watch" list, and A–Z ordering is stable and makes a specific title easy
to locate. But that advantage mostly matters for long lists, and even then it's better served by
an explicit search/filter than by forcing every user into title order by default. The cost of
alphabetical is that the list silently reshuffles the mental model — the film you just added
could land anywhere in the middle — whereas newest-first keeps recent actions visible where the
user expects them. So I'm not just accepting the maintainer's preference; I'm agreeing that for
the default view, recency + cross-endpoint consistency beat findability, and findability is the
thing better solved separately if it becomes a real need.

## Comment 6 — Rebase
**What conflicted:**
Rebasing `feature/watchlist` onto `origin/main` produced three conflicts, plus one
side effect I had to clean up afterward:
1. **`.gitignore` (add/add):** both `main` and my branch had independently added a
   `.gitignore`. The entries mostly overlapped; `main`'s version additionally listed
   `.pytest_cache/`.
2. **`models.py` (the real one — the integer→UUID migration):** `main` had migrated
   `Film.id` (and `CollectionEntry.film_id`) from an integer to a UUID
   (`db.String(36)`). My branch added a new `WatchlistEntry` class whose
   `film_id` was still declared `db.Integer`. `main` had no `WatchlistEntry` at all, so
   the conflict was "keep my new class, but its foreign key type disagrees with the
   migrated `Film.id` it points at."
3. **`pr-response.md` (modify/delete):** `main` never had this file; my branch added it.
4. **Side effect — a dropped commit:** when the rebase first stopped on the `.gitignore`
   conflict, my Comment 1 rename commit got skipped, so `routes/watchlist/watchlist.py`
   reverted to importing the old `save_to_watchlist` name while the service defined
   `add_to_watchlist`. This surfaced as an `ImportError` only after the rebase finished.

**How I resolved it:**
1. **`.gitignore`:** took the union of both sides (kept every entry, including `main`'s
   `.pytest_cache/`) and removed the conflict markers.
2. **`models.py`:** kept my `WatchlistEntry` class and changed its foreign key from
   `db.Column(db.Integer, ...)` to `db.Column(db.String(36), db.ForeignKey("film.id"))`
   so it matches the UUID `Film.id` on `main`. An integer FK pointing at a UUID primary
   key would break the relationship, so this type change is the actual point of the
   rebase — it brings the watchlist feature in line with the migration. My Comment 4
   change (`public` defaulting to `False`) was preserved in the same class.
3. **`pr-response.md`:** kept my version (the filled-in response doc), since `main` simply
   didn't have the file.
4. **Dropped rename:** re-applied the rename in `routes/watchlist/watchlist.py`
   (`save_to_watchlist` → `add_to_watchlist`) and committed it as a follow-up
   (`fix: restore add_to_watchlist route`).

**How I verified no conflict remains:**
- `git status` is clean and `git log --oneline` shows a linear history on top of
  `main`'s merge commit (`bbe206c`), with no rebase in progress.
- Searched the tree for leftover conflict markers
  (`grep -rn "<<<<<<<|=======|>>>>>>>"`) → none.
- Searched for stale references (`grep -rn "save_to_watchlist"`) → none; the route and
  service now agree on `add_to_watchlist`.
- Confirmed both `film_id` foreign keys (`CollectionEntry` and `WatchlistEntry`) are now
  `db.String(36)`, matching `Film.id`.
- Ran the full test suite (`pytest tests/ -q`) → **5 passed**. This is what caught the
  dropped-rename `ImportError`; after re-applying the rename, the suite went green.

## PR Description
<!-- Written at the end — feature overview, design decisions, manual testing steps -->
