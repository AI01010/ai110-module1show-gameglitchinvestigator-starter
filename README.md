# 🎮 Game Glitch Investigator: Fixed

## 📖 Game Purpose

A number guessing game built with Streamlit where players try to guess a secret number within a set number of attempts. The game provides hints ("Higher/Lower") and tracks the player's score based on how quickly they can guess correctly.

## 🐛 Bugs Found & Fixed

### **Critical Issues Resolved:**

1. **Functions Properly Imported** — `app.py` correctly imports `get_range_for_difficulty`, `parse_guess`, `check_guess`, and `update_score` from `logic_utils.py` instead of defining them locally. The refactoring keeps logic separate from UI.

2. **Hints Were Backwards** — Fixed the logic in `check_guess()` to correctly determine if a guess is "Too High" or "Too Low":
   - If guess > secret → "Too High" (go LOWER)
   - If guess < secret → "Too Low" (go HIGHER)

3. **Range Not Varied by Difficulty** — Fixed `get_range_for_difficulty()`:
   - Easy: 1-20
   - Normal: 1-50
   - Hard: 1-100

4. **Score Calculation Fixed** — Corrected the off-by-one error in `update_score()` so points awarded = 100 - 10 * attempt_number (without extra +1)

5. **Game State Management** — Fixed Streamlit session state so:
   - Secret number stays consistent
   - Attempt counter only increments on valid guesses
   - New Game button properly resets all state

## 🛠️ Setup

1. Install dependencies: `pip install -r requirements.txt`
2. Run the fixed app: `python -m streamlit run app.py`

## ✅ Testing

Run pytest to verify all fixes:
```bash
pytest tests/test_game_logic.py -v
```

All tests pass! The game is now playable and fully functioning.
