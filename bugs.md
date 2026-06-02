# Bug Fixes

## BUG 1 — `classify_song`: Wrong priority order allows a Chill song to be labeled Hype

**Location:** `playlist_logic.py`, `classify_song` (~line 76)

### The problem

Two issues combined to misclassify songs:

1. **Priority order.** `genre == favorite_genre` was evaluated as part of the Hype
   condition *before* the Chill check ever ran. Because the default `favorite_genre`
   becomes `"ambient"` once a user selects it, a low-energy ambient song (energy 1–2)
   was classified as **Hype** regardless of its energy or title keywords.

2. **Casing mismatch.** `is_chill_keyword` checked the title against lowercase keywords,
   but `normalize_title` only `.strip()`s — it does **not** lowercase (unlike
   `normalize_genre`). So a title like `"Lofi Rain"` (capital L) never matched `"lofi"`.

### Expected behavior (per the Tinker spec)

- **Hype:** `energy >= hype_min_energy`, OR genre matches `favorite_genre`, OR genre contains hype keywords (`rock`, `punk`, `party`)
- **Chill:** `energy <= chill_max_energy`, OR title contains chill keywords (`lofi`, `ambient`, `sleep`)
- **Mixed:** everything else

A low-energy song must not be forced into Hype just because its genre happens to match
`favorite_genre`.

### Before

```python
is_hype_keyword = any(k in genre for k in hype_keywords)
is_chill_keyword = any(k in title for k in chill_keywords)

if genre == favorite_genre or energy >= hype_min_energy or is_hype_keyword:
    return "Hype"
if energy <= chill_max_energy or is_chill_keyword:
    return "Chill"
return "Mixed"
```

### After (chosen: Chill-first)

We use a **Chill-first** structure: an explicit chill signal (low energy or a chill
keyword) wins ties, so `favorite_genre` can never force a low-energy song into Hype.

```python
title_lower = title.lower()
is_hype_keyword = any(k in genre for k in hype_keywords)
is_chill_keyword = any(k in title_lower for k in chill_keywords)

# Chill-first: an explicit chill signal wins ties over hype signals.
if energy <= chill_max_energy or is_chill_keyword:
    return "Chill"
if genre == favorite_genre or energy >= hype_min_energy or is_hype_keyword:
    return "Hype"
return "Mixed"
```

The alternative **Hype-first** structure (Hype wins ties) is kept commented out in the
source for reference:

```python
# is_hype = energy >= hype_min_energy or is_hype_keyword
# is_chill = energy <= chill_max_energy or is_chill_keyword
# if is_hype:
#     return "Hype"
# if is_chill:
#     return "Chill"
# if genre == favorite_genre:   # tiebreaker only
#     return "Hype"
# return "Mixed"
```

### What the fix does

1. Lowercases the title before checking chill keywords, so `"Lofi Rain"` correctly matches `"lofi"`.
2. Checks the **Chill conditions first**, so a low-energy song is labeled Chill before
   `genre == favorite_genre` is ever considered — `favorite_genre` can no longer override
   a song that already qualifies as Chill.

### Trade-off vs. the Hype-first alternative

Energy thresholds can never conflict (`hype_min 7 > chill_max 3`), so the two approaches
only differ on **keyword cross-conflicts**. Chill-first makes Chill win those ties:

| Song | Hype-first | Chill-first (chosen) |
|------|-----------|----------------------|
| `"Sleep Storm"`, genre pop, energy 9 | Hype | **Chill** |
| genre `"rock"`, energy 2 | Hype | **Chill** |

### Verification

| Song | Before | After | Why |
|------|--------|-------|-----|
| ambient, energy 2 (favorite=ambient) | Hype ❌ | **Chill** ✅ | Chill checked first; favorite_genre no longer overrides it |
| "Lofi Rain", energy 5 | Mixed ❌ | **Chill** ✅ | Title lowercased before keyword check |
| "Sleep Storm", energy 9 | Hype | **Chill** ✅ | Chill keyword wins the tie |
| rock, energy 2 | Hype | **Chill** ✅ | Low energy wins over hype keyword |
| rock, energy 9 | Hype | **Hype** ✅ | High energy → Hype |
| ambient, energy 5 (favorite=ambient) | Hype | **Hype** ✅ | Not Chill, so favorite_genre applies |
| pop, energy 5 | Mixed | **Mixed** ✅ | Nothing matches → fallback |

---

## BUG 2 — `compute_playlist_stats`: `total` counts only Hype songs instead of all songs

**Location:** `playlist_logic.py`, `compute_playlist_stats` (~line 119)

### The problem

`total` was set to `len(hype)` instead of `len(all_songs)`. Two consequences:

1. The returned `total_songs` key reports `len(all_songs)` (the real count), but
   `hype_ratio` is computed as `len(hype) / len(hype)`, which is **always `1.0`** as long
   as there is at least one Hype song.
2. Adding songs to Chill and Mixed has no effect — the Hype Ratio stays pinned at `1.0`
   and never reflects the real proportion.

### Expected behavior (per the Tinker spec)

```
hype_ratio = len(hype) / total_all_songs
```

### Before

```python
total = len(hype)
hype_ratio = len(hype) / total if total > 0 else 0.0
```

### After

```python
total = len(all_songs)
hype_ratio = len(hype) / total if total > 0 else 0.0
```

### Verification

| Playlists | Before | After |
|-----------|--------|-------|
| 2 Hype, 5 Chill, 3 Mixed (10 total) | hype_ratio = 1.0 ❌ | hype_ratio = 0.2 ✅ |
| 5 Hype, 0 others | hype_ratio = 1.0 | hype_ratio = 1.0 ✅ |
| 0 songs | hype_ratio = 0.0 | hype_ratio = 0.0 ✅ |

---

## BUG 3 — `compute_playlist_stats`: `avg_energy` sums only Hype but divides by all songs

**Location:** `playlist_logic.py`, `compute_playlist_stats` (~line 143)

### The problem

`total_energy` summed energy from only the `hype` list, but `avg_energy` divided that sum
by `len(all_songs)`. The result is the total Hype energy spread across *all* songs — not a
real average of anything.

### Before

```python
total_energy = sum(song.get("energy", 0) for song in hype)
avg_energy = total_energy / len(all_songs)
```

### After

```python
total_energy = sum(song.get("energy", 0) for song in all_songs)
avg_energy = total_energy / len(all_songs)
```

### Verification

| Playlists | Before | After |
|-----------|--------|-------|
| 2 Hype (e8,e9), 5 Chill (e1), 3 Mixed (e5) | avg_energy = 1.7 ❌ | avg_energy = 3.7 ✅ |

---

## BUG 4 — `search_songs`: substring check is reversed

**Location:** `playlist_logic.py`, `search_songs` (~line 190)

### The problem

The match condition asked whether the song's field value is contained in the query
(`value in q`), instead of whether the field contains the query (`q in value`). As a
result, a song only matched when its **entire** field value was a substring of the query —
so normal partial searches (typing part of an artist or title) returned nothing.

### Expected behavior (per the Tinker spec)

A song matches when its `field` value **contains** the query string.

### Before

```python
value = str(song.get(field, "")).lower()
if value and value in q:
    filtered.append(song)
```

### After

```python
value = str(song.get(field, "")).lower()
if value and q in value:
    filtered.append(song)
```

### Verification

| Query (field) | Song artist | Before | After |
|---------------|-------------|--------|-------|
| `"beat"` (artist) | "The Beatles" | not found ❌ | **found** ✅ |
| `"beat"` (artist) | "Beat Crusaders" | not found ❌ | **found** ✅ |
| `"head"` (artist) | "Radiohead" | not found ❌ | **found** ✅ |
| `""` (empty) | any | all returned | all returned ✅ |

---

## BUG 5 — `random_choice_or_none`: crashes on an empty list

**Location:** `playlist_logic.py`, `random_choice_or_none` (~line 211)

### The problem

The function's name and `Optional[Song]` return type promise it can return `None`, but it
called `random.choice(songs)` unconditionally. `random.choice([])` raises `IndexError`, so
an empty list crashes instead of returning `None`.

### Expected behavior

Return `None` when there are no songs; otherwise return a random song.

### Before

```python
def random_choice_or_none(songs: List[Song]) -> Optional[Song]:
    """Return a random song or None."""
    import random

    return random.choice(songs)
```

### After

```python
def random_choice_or_none(songs: List[Song]) -> Optional[Song]:
    """Return a random song or None."""
    import random

    if not songs:
        return None
    return random.choice(songs)
```

### Verification

| Input | Before | After |
|-------|--------|-------|
| `[]` | `IndexError` ❌ | `None` ✅ |
| `[{...}]` | a song | a song ✅ |

---
---

# Refactors

These are code-quality improvements, not bug fixes — behavior is unchanged.

## REFACTOR 1 — Extract the duplicated genre list into one constant

**Location:** `app.py`, `profile_sidebar` and `add_song_sidebar`

### The problem

The same genre list was hard-coded in two different functions:

```python
options=["rock", "lofi", "pop", "jazz", "electronic", "ambient", "other"]
```

The "Favorite genre" dropdown and the "Song genre" dropdown are meant to offer the same
set of choices, but nothing kept them in sync. Adding a genre to one and forgetting the
other would let the lists drift apart — and since `classify_song` compares
`genre == favorite_genre`, a mismatch could quietly affect classification.

### Before

```python
# in profile_sidebar
profile["favorite_genre"] = st.sidebar.selectbox(
    "Favorite genre",
    options=["rock", "lofi", "pop", "jazz", "electronic", "ambient", "other"],
    index=0,
)

# in add_song_sidebar
genre = st.sidebar.selectbox(
    "Genre",
    options=["rock", "lofi", "pop", "jazz", "electronic", "ambient", "other"],
)
```

### After

```python
# module-level, near the top of app.py
GENRE_OPTIONS = ["rock", "lofi", "pop", "jazz", "electronic", "ambient", "other"]

# in profile_sidebar
profile["favorite_genre"] = st.sidebar.selectbox(
    "Favorite genre",
    options=GENRE_OPTIONS,
    index=0,
)

# in add_song_sidebar
genre = st.sidebar.selectbox(
    "Genre",
    options=GENRE_OPTIONS,
)
```

### Why it's better

- **Single source of truth** — both dropdowns now read from `GENRE_OPTIONS`, so they can
  never fall out of sync.
- **Easier to extend** — adding a genre is a one-line change in one place.
- **No behavior change** — the lists were identical, so the UI behaves exactly as before.
  `app.py` compiles cleanly and the literal list now appears only once.

---

## REFACTOR 2 — Extract a `format_song_label` helper for song display

**Location:** `app.py`, `render_playlist`, `lucky_section`, `history_section`

### The problem

The shared "Title by Artist" idea was rebuilt from scratch in three places, each reaching
into the song dict differently — some with `song['title']` (raises if missing), some with
`song.get('mood', '?')` (safe default). The shape of a song label lived in three spots, and
the inconsistent field access made it unclear whether the difference was intentional.

### Before

```python
# render_playlist
f"- **{song['title']}** by {song['artist']} "
f"(genre {song['genre']}, energy {song['energy']}, mood {mood}) [{tags}]"

# lucky_section
f"Lucky song: {pick['title']} by {pick['artist']} (mood {pick.get('mood', '?')})"

# history_section
f"{song.get('mood', '?')}: {song['title']} by {song['artist']}"
```

### After

```python
def format_song_label(song: Song) -> str:
    """Human-readable 'Title by Artist' label for a song."""
    title = song.get("title", "?")
    artist = song.get("artist", "?")
    return f"{title} by {artist}"

# lucky_section
f"Lucky song: {format_song_label(pick)} (mood {pick.get('mood', '?')})"

# history_section
f"{song.get('mood', '?')}: {format_song_label(song)}"

# render_playlist — left inline (see caveat below)
f"- **{song['title']}** by {song['artist']} "
f"(genre {song['genre']}, energy {song['energy']}, mood {mood}) [{tags}]"
```

### Caveat found during testing

`render_playlist` bolds **only the title** (`**Title** by Artist`). Using the helper there
would have bolded the whole label (`**Title by Artist**`) — a visible change in the
rendered markdown. Behavior-equivalence testing across all 22 default songs caught this, so
`render_playlist` was left formatting inline; the helper is used only in the two sites where
it is a true drop-in. The de-dup still covers the two identical cases, and the third has a
genuinely different presentation need.

### Why it's better

- **One home for the "Title by Artist" shape** in the two views that share it.
- **Consistent, safe field access** — `.get(..., "?")` in the helper, no `KeyError` risk.
- **Call sites read as intent** — the `Lucky song:` prefix and mood-first ordering stand out
  instead of being buried in string plumbing.
- **No visible behavior change** — verified: rendered text is byte-for-byte identical for all
  22 default songs across all three views.

---

## REFACTOR 3 — Replace `lucky_pick`'s if/elif chain with a lookup

**Location:** `playlist_logic.py`, `lucky_pick`

### The problem

Three branches each did the same kind of thing (pick which playlists feed the random
choice), with the `playlists.get("Hype", [])` lookups repeated across them.

### Before

```python
if mode == "hype":
    songs = playlists.get("Hype", [])
elif mode == "chill":
    songs = playlists.get("Chill", [])
else:
    songs = playlists.get("Hype", []) + playlists.get("Chill", [])

return random_choice_or_none(songs)
```

### After

```python
hype = playlists.get("Hype", [])
chill = playlists.get("Chill", [])

# Map each mode to its pool; "any" (the default) draws from Hype + Chill.
pools = {"hype": hype, "chill": chill}
songs = pools.get(mode, hype + chill)

return random_choice_or_none(songs)
```

### Why it's better

- **The mode → pool mapping is explicit and flat** instead of buried in branches.
- **Each playlist is fetched once** (`hype`, `chill`), removing the repeated `.get` calls.
- **No behavior change** — `hype`/`chill`/`any` and any unknown mode resolve to the same
  pools as before (verified: unknown modes fall back to `Hype + Chill`, same as `any`).
