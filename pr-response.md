# PR Response Doc — CineLog Watchlist Feature

## AI Usage
I used an AI coding assistant (Claude Code) at several points in this project. Concrete uses:

- **Codebase orientation.** Before making changes I had the AI summarize `models.py` and
  `services/watchlist_service.py` — each file's responsibility, its main functions, and which
  other modules depend on it — so I understood the layout first.
- **Understanding an existing pattern (Comment 2).** Following the milestone's guidance, I asked
  the AI to *explain* what the deduplication check in `add_to_collection()` does — what
  `.filter_by(...).first()` returns and when it raises — rather than to write the watchlist
  version for me. I then wrote my own `add_to_watchlist` check. When I had the AI review it, it
  caught that I was querying `CollectionEntry` instead of `WatchlistEntry` — a real bug — and the
  correction to query the watchlist table was applied.
- **Finding all call sites (Comment 1).** I used the AI to run a project-wide `grep` for
  `save_to_watchlist` to confirm every reference (definition, import, call) before renaming.
- **Verifying commit format.** I asked the AI to audit my commit messages against Conventional
  Commits and flag any that bundled multiple logical changes. It flagged `update:` as a
  non-standard type and the "added watchlist model and endpoint fixed a bug more changes" commit
  as bundling several changes.
- **Rebase help (Comment 6).** I worked through the rebase conflicts with the AI — it explained
  the `.gitignore`, `models.py` (integer→UUID), and `pr-response.md` conflicts and helped me
  confirm no markers or stale references remained afterward.

**Comment 4 (visibility default).** The position was mine: I decided the watchlist should default
to private because a user has the right not to display what they intend to watch. I asked the AI
first to explain what the comment was asking for, then to help me phrase my stance in the required
position/reasoning/tradeoff structure. My core argument (privacy is the user's right; sharing
should be opt-in) is unchanged from what I brought in. What the AI added was tighter wording and
surfacing the counter-tradeoff — that private-by-default slows adoption of the social/discovery
features — which I read, agreed with, and chose to accept explicitly rather than weaken my
position.

**Comment 5 (sort order).** I asked the AI to lay out the options (adopt the maintainer's
date-added, keep alphabetical, or a hybrid with a `?sort=` param) and the maintainer's likely
reasoning. I chose to adopt date-added. The AI drafted the written argument, but the decision and
the reason I accepted it — recency reflects what I'm about to watch, and consistency with
`get_collection` reduces confusion for anyone using both endpoints — were what I directed it to
argue. Where I built on its draft: I kept the point that alphabetical's findability benefit is
better served by an explicit search than by forcing title order as the default, because that
matched my own view that the default should optimize for the common case.

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

### What this feature does
Adds a **watchlist** to CineLog — a list of films a user wants to watch later, kept separate
from their collection (films they've *already* watched). It's backed by a new `WatchlistEntry`
model (`user_id`, `film_id`, `date_added`, `public`) and exposes two endpoints:

- `POST /watchlist/<user_id>/add` — add a film to the user's watchlist.
  Body: `{ "film_id": "<film-uuid>" }`. Returns `201` with the created entry.
- `GET /watchlist/<user_id>` — return the user's watchlist.

Business logic lives in `services/watchlist_service.py`:
- **Deduplication** — adding a film that's already on the watchlist is rejected
  (`AlreadyInWatchlistError`) instead of silently creating a duplicate.
- **Validation** — adding a `film_id` that doesn't exist raises `FilmNotFoundError`.

### Design decisions
1. **Visibility default → private (`public=False`).** A watchlist is private by default; a user
   must explicitly opt in to sharing it. I optimized for the user's reasonable expectation of
   privacy — sharing personal "want to watch" data should be a deliberate choice, not something
   that happens because a default was left on. (See Comment 4 for the full tradeoff discussion.)
2. **Sort order → newest-first by `date_added`.** `GET /watchlist` returns entries most-recently-
   added first, matching `get_collection`. This surfaces what the user is currently thinking about
   and keeps both list endpoints behaving consistently. (See Comment 5.)

### How to manually test
Prerequisites: dependencies installed (`pip install -r requirements.txt`).

1. **Start the app:**
   ```
   python app.py
   ```
   It serves at `http://127.0.0.1:5000`.

2. **Seed a user and two films.** There are no create-user/create-film endpoints, so add them
   through a Python shell run from the project root:
   ```
   python
   >>> from app import create_app, db
   >>> from models import User, Film
   >>> app = create_app()
   >>> with app.app_context():
   ...     u  = User(username="alice", email="alice@example.com")
   ...     f1 = Film(title="Heat", year=1995)
   ...     f2 = Film(title="Arrival", year=2016)
   ...     db.session.add_all([u, f1, f2]); db.session.commit()
   ...     print("USER ", u.id); print("FILM1", f1.id); print("FILM2", f2.id)
   ```
   Copy the three printed UUIDs.

3. **Add the first film to the watchlist:**
   ```
   curl -X POST http://127.0.0.1:5000/watchlist/<USER>/add \
     -H "Content-Type: application/json" -d '{"film_id": "<FILM1>"}'
   ```
   Expect `201` and a JSON body containing `"public": false` — this confirms the
   **private-by-default** design decision.

4. **Add the second film** the same way, using `<FILM2>`.

5. **View the watchlist:**
   ```
   curl http://127.0.0.1:5000/watchlist/<USER>
   ```
   Expect both films, with **FILM2 (added most recently) listed first** — this confirms the
   **newest-first sort** design decision.

6. **Verify deduplication:** re-run the step-3 `POST` with `<FILM1>` again. The service rejects
   the duplicate (`AlreadyInWatchlistError`) and no second row is created.

7. **Verify not-found handling:** `POST` with a made-up `film_id`. The service raises
   `FilmNotFoundError`.

   > Note: the route does not yet map `AlreadyInWatchlistError` / `FilmNotFoundError` to HTTP
   > status codes, so steps 6–7 currently surface as `500` responses rather than `409` / `404`.
   > Wiring those to proper status codes is a sensible follow-up.

### Automated tests
`pytest tests/ -q` → 5 passing, including `test_add_to_watchlist_nonexistent_film_raises`.

## Git OneLine
<img width="665" height="245" alt="Screenshot 2026-07-13 at 1 19 39 PM" src="https://github.com/user-attachments/assets/e45ed233-8468-40fa-a652-71fb4a11449b" />

