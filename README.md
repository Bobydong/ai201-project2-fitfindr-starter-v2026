# FitFindr

> ### 👋 Start here
>
> **New to this repo? Read [RUNNING.md](RUNNING.md) first** — setup, every
> command, and what to do when something breaks.
>
> Once `python test.py` passes:
>
> ```bash
> python app.py listings --full -n 6      # read the data (Milestone 1)
> python app.py fields                    # what you can filter on
> python app.py ask 'vintage graphic tee under $30'
> ```
>
> All three tools are stubs, so that last command will do nothing useful yet.
> That's the starting position.
>
> **The rest of this file is your submission.** Fill it in as you go.

---

<!-- ─────────────────────────────────────────────────────────────────────────
     HOW TO USE THIS FILE

     This is your submission. Fill each section in as you finish the milestone
     it belongs to — don't leave it all to the end.

     Unit 3 asks for the first five sections. Unit 4 adds the five below them.
     Leave the unit 4 sections alone until then; they're here so you know
     what's coming.

     Everything is pasted as TEXT. No screenshots, no images, no video links.
     A typed block of output gets full credit; a picture of the same output
     gets none.
     ───────────────────────────────────────────────────────────────────────── -->

<!-- ═══════════════════════ UNIT 3 — THE BUILD ═══════════════════════ -->

## What This Does

<!-- Three or four sentences: what a user asks for, and what they get back. -->



---

## Tool Inventory

<!-- Four lines per tool. This is worth 2 points and it's the single most
     common place students lose them.

     "Returns a list" earns NOTHING. The description has to say what is IN
     the list.

     The empty case isn't optional either — it's the thing your loop branches
     on, and if you don't decide it here you'll discover it as a crash in
     Milestone 5. -->

### `search_listings`

- **What it does:** Narrows the 40 mock listings in `data/listings.json` down to the ones matching a keyword description, and optionally a size and a price ceiling, ranked best match first.
- **Inputs:** `description` (str, required) — free-text keywords, e.g. `"vintage graphic tee"`; `size` (str or None, default `None`) — one size token such as `"M"`, `"W30"`, `"US 9"`, where `None` skips size filtering entirely; `max_price` (float or None, default `None`) — inclusive dollar ceiling, where `None` skips price filtering entirely.
- **Returns:** A `list[dict]` of at most `config.SEARCH_RESULT_LIMIT` (10) listing dicts, sorted by keyword-overlap score descending, ties broken by `price` ascending then `id` ascending so the order is identical on repeated runs. Each dict is a listing copied out of the data file unmodified, with all eleven fields: `id` (str), `title` (str), `description` (str), `category` (str — one of `tops`, `bottoms`, `outerwear`, `shoes`, `accessories`), `style_tags` (list[str]), `size` (str), `condition` (str — `excellent`, `good`, or `fair`), `price` (float), `colors` (list[str]), `brand` (str or `None` — **`None` for 32 of the 40 listings**), `platform` (str — `depop`, `poshmark`, or `thredUp`).
- **When it has nothing:** `[]`. An empty list — never `None`, never an exception, never a "no results" string. This is the single value the planning loop branches on.

**Size match rule** (part of the spec, because `"s" in "us 9"` and `"l" in "xl"` are both `True` and a plain substring test would return shoes to someone asking for a small top): uppercase both sides, split each on `/`, whitespace, and parentheses into tokens, and count it a match when **every** token of the requested size appears in the listing's token set. So `"M"` matches `"S/M"` and `"M/L"`; `"L"` matches `"L/XL"` but not `"XL"`; `"XL"` matches `"XL (oversized)"`; `"W30"` matches `"W30 L30"`; `"US 9"` and `"9"` both match `"US 9"`; and `"S"` does **not** match `"US 9"`. One exception, deliberate: any listing whose size contains `ONE SIZE` matches every request, because a one-size beanie does fit someone who asked for an M.

**Scoring rule:** lowercase `description`, split on non-alphanumerics, drop the stopwords `a an the for in of my and to with looking want need size under`, and score each listing by how many distinct remaining query words appear anywhere in its searchable text — `title`, `description`, `style_tags`, `colors`, `category`, and `brand` when it isn't `None`. Anything scoring 0 is dropped.

### `suggest_outfit`

- **What it does:** Asks the model, through `generate()`, for one or two outfits built around a thrifted item, naming pieces the user already owns whenever it can.
- **Inputs:** `new_item` (dict, required) — one listing dict in the exact eleven-field shape `search_listings` returns; `wardrobe` (dict, required) — `{"items": list[dict]}`, where each item has `id` (str), `name` (str), `category` (str), `colors` (list[str]), `style_tags` (list[str]), `notes` (str or `None`). **`items` may be empty.**
- **Returns:** A non-empty `str` of plain prose, no fixed format. With a non-empty wardrobe it names at least one wardrobe item by its `name` so the suggestion is checkably about clothes the user owns. With an empty wardrobe it returns general styling advice for the item on its own, and names no wardrobe pieces because there are none to name.
- **When it has nothing:** There is no empty return. An empty wardrobe is a *different prompt*, not a failure — it still returns a non-empty string. The function never returns `""`, never returns `None`, and never raises on an empty wardrobe, so the loop does not branch here. (A model outage raises `ModelUnavailable`, which `run_agent` catches in unit 4; that is the transport failing, not this tool having nothing to say.)

### `create_fit_card`

- **What it does:** Asks the model, through `generate()`, for a short caption someone would actually post about the find.
- **Inputs:** `outfit` (str, required) — the string `suggest_outfit` returned; `new_item` (dict, required) — the same eleven-field listing dict.
- **Returns:** A non-empty `str` of two to four sentences that reads like a post rather than a product description, mentioning the item, its `price`, and its `platform` once each. It must differ across different items, and across repeated runs on the same item — `TEMPERATURE = 0.9` and `CACHE_ENABLED` in `config.py` are the two reasons it might not.
- **When it has nothing:** When `outfit` is `""` or whitespace only, return the literal string `"No outfit to write a card about — suggest_outfit returned nothing."` and make no model call. A descriptive string, not an exception, and not an empty string.

---

## Planning Loop

<!-- Your branch rule, stated as a rule — the condition AND both paths — plus
     the file and function that holds it.

     Like this:
       "If search_listings returns an empty list, put a message in the session
        and stop. Otherwise take the first result and go to suggest_outfit."
        — agent.py::run_agent

     The grader checks your code against what you claim here, so the file and
     function have to be real. -->

**Branch rule:** If `search_listings` returns an empty list, write into `session["error"]` a message naming what the user could change — raise the price ceiling, drop or widen the size, use fewer keywords — and return the session immediately, leaving `selected_item`, `outfit_suggestion`, and `fit_card` all `None`. Otherwise take `search_results[0]` as `session["selected_item"]` and carry on to `suggest_outfit`, then `create_fit_card`.

**Where it lives:** `agent.py::run_agent`

**How the query is parsed:** regex plus string splitting — no model call, so parsing costs nothing and is identical on every run. Price: `(?:under|below|less than|max|<)\s*\$?\s*(\d+(?:\.\d{1,2})?)`, falling back to a bare `\$\s*(\d+(?:\.\d{1,2})?)`, then `float()`. Size: `\bsize\s+([A-Za-z0-9/. ]{1,8}?)\b` first; only if that finds nothing, a bare-token pass for `\b(XS|S|M|L|XL|XXL|W\d{2}|US\s?\d(?:\.5)?)\b`. Description: the original query with the matched price and size spans cut out, then the same stopword list `search_listings` uses. Anything not found stays `None`, which means that filter is skipped rather than applied with a guess.

**What moves through the session:** `query` (set by `new_session`) → `parsed` (`{"description": str, "size": str | None, "max_price": float | None}`) → `search_results` (`list[dict]`) → **branch**: either `error` (str) and stop, or `selected_item` (dict) → `outfit_suggestion` (str) → `fit_card` (str). `wardrobe` goes in at `new_session` and is only read, never written. Each step reads its input back out of the session rather than from a local variable, so a trace can print the whole state at any point.

---

## Sample Run

<!-- Two things go here.

     1. One FULL query and its output, pasted as text.
     2. Your three per-tool terminal tests — the command and what it printed. -->

**One full query**

```
$ python app.py ask '...'

```

**The three tools, tested one at a time**

```
$ python -c "from tools import search_listings; print(search_listings('graphic tee', max_price=30))"

```

```
$ python -c "from tools import suggest_outfit; ..."

```

```
$ python -c "from tools import create_fit_card; ..."

```

---

## How I Used AI

<!-- Two specific moments. What you asked, what came back, what you changed.

     "I used Claude to help me code" is not enough.

     "I gave Claude my search_listings spec. It returned None on no match
     instead of an empty list, so I changed it" is the level we want. -->

**Moment 1**

- *What I asked for:*
- *What came back:*
- *What I changed:*

**Moment 2**

- *What I asked for:*
- *What came back:*
- *What I changed:*

<!-- ═══════════════════════ UNIT 4 — THE TEST ═══════════════════════

     Don't fill these in during unit 3.
     ═══════════════════════════════════════════════════════════════════ -->

---

## Run Log — Before

<!-- Five criteria, five tries each, in this exact format.

     Five, because your criteria are written out of five. Mark each try PASS
     or FAIL, count the passes, and read that count against your target — a
     row targeting 4 of 5 with three PASS cells is MISSED (3/5).

     `python run_eval.py --label before` runs everything and writes the table
     into results/. Paste it here and fill in the verdicts. -->

| Criterion | Target | Try 1 | Try 2 | Try 3 | Try 4 | Try 5 | Verdict |
|---|---|---|---|---|---|---|---|
| 1.  |  |  |  |  |  |  |  |
| 2.  |  |  |  |  |  |  |  |
| 3.  |  |  |  |  |  |  |  |
| 4.  |  |  |  |  |  |  |  |
| 5.  |  |  |  |  |  |  |  |

**Real output from one try**, pasted as text, naming the file and function
that produced it:

```

```

---

## Verdicts and Diagnoses

<!-- MET or MISSED per criterion against LAST UNIT's target, plus a sentence on
     how you decided.

     Then, for every miss: which of the four places it happened — a tool, the
     loop's branch, the session, or the model's output — AND the mechanism.

     Not a diagnosis:  "The fit card was bad."
     A diagnosis:      "The fit card criterion missed on 2 of 5 items. Both had
                        an empty brand field. My prompt puts the brand in the
                        first sentence, so the card opened with a blank and read
                        like a fragment. The tool worked; the prompt assumed a
                        field that isn't always there."

     Look for a pattern. Three misses on the same tool is one problem, not
     three. -->

| # | Criterion | Target | Verdict | How I decided |
|---|---|---|---|---|
| 1 |  |  |  |  |
| 2 |  |  |  |  |
| 3 |  |  |  |  |
| 4 |  |  |  |  |
| 5 |  |  |  |  |

**Diagnoses**



---

## Loop Trace

<!-- One full run, printed step by step, with the MCP call visible in it.

     `python app.py ask '...' --trace` once you've added the trace.step()
     calls in Milestone 2.

     Worth pasting BOTH the happy path and the empty-search path. The empty
     one should be visibly shorter, because it stops. If your two traces are
     the same length, your branch isn't working — and this is the fastest way
     anyone will ever find that out. -->

**Happy path**

```

```

**Empty search**

```

```

**On the MCP move:** <!-- what changed in your code, and whether anything
behaved differently afterwards. If the rewire didn't work, say exactly where it
broke — the error text and the last thing that worked. That earns the point in
full. -->



---

## The Improvement

<!-- What you changed, why your diagnosis pointed at it, and the after-run in
     the same table format. One change, measured properly.

     `python run_eval.py --label after` -->

**What I changed:**

**Which failure it was meant to fix:**

### Run Log — After

| Criterion | Target | Try 1 | Try 2 | Try 3 | Try 4 | Try 5 | Verdict |
|---|---|---|---|---|---|---|---|
| 1.  |  |  |  |  |  |  |  |
| 2.  |  |  |  |  |  |  |  |
| 3.  |  |  |  |  |  |  |  |
| 4.  |  |  |  |  |  |  |  |
| 5.  |  |  |  |  |  |  |  |

**Did it help, and how do I know:**

<!-- If it made things worse, say that. Honestly reported, that earns full
     credit and is more interesting than one that worked. -->



---

## What's Still Broken

<!-- For each criterion still missed: what you'd do, and why you stopped where
     you did. "I ran out of time" is fine if it's true. Pretending nothing is
     left is not. -->



<!-- ═════════════════════════════════════════════════════════════════════

     SUBMISSION CHECKLIST — unit 3

       [ ] criteria.md has five numbered criteria, each with a target
       [ ] Each criterion has a reason underneath it
       [ ] All five unit 3 sections above have real content
       [ ] Tool Inventory: all three tools, inputs WITH TYPES, a specific
           return value, and the empty case
       [ ] Planning Loop names the branch rule and agent.py::run_agent
       [ ] Sample Run: one full query plus the three per-tool tests, as text
       [ ] At least four new commits
       [ ] Repository URL submitted — WRITE IT DOWN, you submit the same one
           next unit

     SUBMISSION CHECKLIST — unit 4

       [ ] mcp_server.py exists with one tool registered
           (or a written record of exactly where the rewire broke)
       [ ] Run Log — Before, five criteria, five tries each
       [ ] Real output pasted underneath, naming file and function
       [ ] A verdict on every criterion
       [ ] A diagnosis for every miss, naming a place AND a mechanism
       [ ] Loop Trace, with the MCP call visible in it
       [ ] All three failure modes triggered and handled
       [ ] One improvement, with Run Log — After in the same format
       [ ] What's Still Broken
       [ ] At least four new commits
       [ ] The SAME repository URL as last unit

     Do not delete and recreate this repository. Your commit history is what
     shows your criteria existed before your results did.
     ═════════════════════════════════════════════════════════════════════ -->

---

📖 **How to run this project: [RUNNING.md](RUNNING.md)**
