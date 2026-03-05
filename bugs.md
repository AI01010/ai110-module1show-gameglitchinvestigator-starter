# Bug Report: Game Glitch Investigator

Full analysis of all errors found across `app.py`, `logic_utils.py`, `tests/test_game_logic.py`, and `requirements.txt`.

---

## Severity Legend

| Level | Meaning |
|---|---|
| **Critical** | Breaks core functionality or causes crashes |
| **High** | Wrong behaviour that is clearly visible to the player |
| **Medium** | Incorrect logic that produces subtly wrong results |
| **Low** | Code quality, dead code, or style issues |

---

## Bug 1 — `logic_utils.py` is completely unimplemented [Critical]

**File:** `logic_utils.py` lines 1–27
**File:** `tests/test_game_logic.py` line 1

Every function in `logic_utils.py` immediately raises `NotImplementedError`:

```python
def get_range_for_difficulty(difficulty: str):
    raise NotImplementedError("Refactor this function from app.py into logic_utils.py")
```

The test file imports `check_guess` from `logic_utils`:

```python
from logic_utils import check_guess
```

This means **every single test crashes with `NotImplementedError` before it can even assert anything**. The test suite gives false confidence — it runs without a syntax error but cannot test any real behaviour.

The intended architecture is clearly for logic to live in `logic_utils.py` and be imported into `app.py`, but this refactor was never done. `app.py` contains its own duplicate implementations of all four functions and never imports from `logic_utils` at all.

---

## Bug 2 — Tests assert against the wrong return type [Critical]

**File:** `tests/test_game_logic.py` lines 3–16

All three tests compare the return value of `check_guess` to a plain string:

```python
result = check_guess(50, 50)
assert result == "Win"          # FAILS: check_guess returns a tuple, not a string
```

But `check_guess` (in `app.py`, and as it would be in `logic_utils.py`) returns a **two-element tuple**:

```python
return "Win", "🎉 Correct!"
```

So `result` would be `("Win", "🎉 Correct!")`, and `("Win", "🎉 Correct!") == "Win"` is always `False`. Even if `logic_utils.py` were fully implemented, all three tests would fail because they test the wrong thing. The correct assertion would be:

```python
outcome, message = check_guess(50, 50)
assert outcome == "Win"
```

---

## Bug 3 — Invalid guesses consume an attempt [High]

**File:** `app.py` lines 148–155

The attempt counter is incremented **before** the guess is parsed:

```python
if submit:
    st.session_state.attempts += 1       # incremented here

    ok, guess_int, err = parse_guess(raw_guess)

    if not ok:
        st.error(err)                    # attempt wasted on bad input
```

If the player types `"abc"` or submits an empty field, they lose an attempt even though no real guess was made. The increment should only happen after a successful parse.

---

## Bug 4 — Off-by-one error in win score formula [Medium]

**File:** `app.py` lines 51–52

```python
points = 100 - 10 * (attempt_number + 1)
```

`attempt_number` is already the current attempt count (1 on the first guess, 2 on the second, etc.). The extra `+ 1` shifts all win scores down by 10 unnecessarily:

| Attempt | Points awarded | Points should be |
|---|---|---|
| 1st guess | 80 | 90 |
| 2nd guess | 70 | 80 |
| 9th+ guess | 10 (floored) | 10 |

The formula should be `100 - 10 * attempt_number`.

---

## Bug 5 — Switching difficulty mid-game does not reset state [Medium]

**File:** `app.py` lines 85–103

`low`, `high`, and `attempt_limit` are recomputed on every Streamlit rerun based on the current sidebar selection. But `st.session_state.secret` is only set once:

```python
if "secret" not in st.session_state:
    st.session_state.secret = random.randint(low, high)
```

If a player starts a Hard game (range 1–100, secret = 75), then switches to Easy (range 1–20), the secret remains 75. The player is now asked to guess a number between 1 and 20 when the answer is 75 — it is impossible to win. The attempts remaining count is also misleading because it recalculates against the new difficulty's limit while the attempt counter reflects guesses made under the old difficulty.

---

## Bug 6 — Score can go permanently negative with no floor [Medium]

**File:** `app.py` lines 57–61

Every wrong guess deducts 5 points:

```python
if outcome == "Too High":
    return current_score - 5

if outcome == "Too Low":
    return current_score - 5
```

The score starts at 0 and has no lower bound. On Easy with 8 attempts and no win, the final score is -35. On Hard with 4 attempts and no win, it is -15. There is no minimum score of 0 or any indicator to the player that a negative score is possible. The win path correctly floors at 10 points, but the wrong-guess path has no equivalent protection.

---

## Bug 7 — Dead code: `TypeError` branch in `check_guess` is unreachable [Low]

**File:** `app.py` lines 41–47

```python
    try:
        if guess > secret:
            return "Too High", "📉 Go LOWER!"
        else:
            return "Too Low", "📈 Go HIGHER!"
    except TypeError:
        g = str(guess)
        if g == secret:
            return "Win", "🎉 Correct!"
        if g > secret:
            return "Too High", "📉 Go LOWER!"
        return "Too Low", "📈 Go HIGHER!"
```

The `TypeError` path was the safety net for when `secret` was secretly cast to a string (the bug that caused reversed hints on even attempts). That string-coercion bug has been fixed — `guess` is always an `int` from `parse_guess` and `secret` is always an `int` from `session_state`. Comparing two integers never raises `TypeError`, so the entire `except` block is dead code that will never execute. Additionally, the string comparison logic inside it (`g > secret`) is itself incorrect — comparing a string to an integer in Python 3 raises a `TypeError` rather than silently doing the wrong thing.

---

## Bug 8 — Duplicate function definitions across `app.py` and `logic_utils.py` [Low]

**File:** `app.py` lines 4–65
**File:** `logic_utils.py` lines 1–26

The four game-logic functions (`get_range_for_difficulty`, `parse_guess`, `check_guess`, `update_score`) are fully implemented inside `app.py` and also stubbed in `logic_utils.py`. `app.py` never imports from `logic_utils`. This means:

- Any fix made to `app.py` does not affect `logic_utils.py` and vice versa.
- Tests that correctly import from `logic_utils` will never test the code that actually runs.
- There is no single source of truth for game logic.

The intended design is for the four functions to live only in `logic_utils.py`, be imported into `app.py`, and be tested via `tests/test_game_logic.py`.

---

## Bug 9 — `except Exception` is too broad in `parse_guess` [Low]

**File:** `app.py` lines 26–27

```python
    except Exception:
        return False, None, "That is not a number."
```

Catching `Exception` is overly wide. The only realistic failure from `int()` or `float()` on a bad string is `ValueError`. Using `except Exception` will also silently swallow unexpected errors like `MemoryError` or future refactoring mistakes, making them look like a user input error rather than a real bug. It should be:

```python
    except ValueError:
        return False, None, "That is not a number."
```

---

## Bug 10 — Outdated `altair<5` pin in `requirements.txt` [Low]

**File:** `requirements.txt` line 2

```
altair<5
```

Altair 5.x has been the stable, current release since 2023. Pinning below 5 forces installation of an outdated version and blocks any security or compatibility updates. Modern Streamlit works with Altair 5. This pin should be removed or updated to `altair>=5`.

---

## Summary Table

| # | File | Line(s) | Severity | Description |
|---|---|---|---|---|
| 1 | `logic_utils.py` | 1–27 | Critical | All functions raise `NotImplementedError`; tests always crash |
| 2 | `tests/test_game_logic.py` | 3–16 | Critical | Tests assert string equality against a tuple return value |
| 3 | `app.py` | 148–155 | High | Invalid guesses consume an attempt before parse check |
| 4 | `app.py` | 51–52 | Medium | Win score off by one due to `attempt_number + 1` |
| 5 | `app.py` | 85–103 | Medium | Switching difficulty does not reset secret or attempts |
| 6 | `app.py` | 57–61 | Medium | Score has no lower bound and can go deeply negative |
| 7 | `app.py` | 41–47 | Low | `TypeError` except branch is dead code, never reachable |
| 8 | `app.py` / `logic_utils.py` | all | Low | Logic duplicated in both files; no import relationship |
| 9 | `app.py` | 26–27 | Low | `except Exception` too broad; should be `except ValueError` |
| 10 | `requirements.txt` | 2 | Low | `altair<5` pin is outdated |
