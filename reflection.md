# 💭 Reflection: Game Glitch Investigator

Answer each question in 3 to 5 sentences. Be specific and honest about what actually happened while you worked. This is about your process, not trying to sound perfect.

## 1. What was broken when you started?

- What did the game look like the first time you ran it?
  The UI was working but numbers didn't match expectations and the hints were wrong. The game showed up fine on screen but the actual game logic was all messed up when you actually tried to play it.

- List at least two concrete bugs you noticed at the start:
    1. The number of attempts started 1 lower than it should have
    2. Hints were backwards — when I guessed a number that was too low, it said "go lower" instead of "go higher"
    3. Range wasn't different for each difficulty — all difficulties used 1-100 instead of 1-20 for Easy, 1-50 for Normal
    4. Score went negative and didn't match the final score displayed at the end
    5. New game button didn't properly reset all the game state variables

---

## 2. How did you use AI as a teammate?

- Which AI tools did you use on this project (for example: ChatGPT, Gemini, Copilot)? 
  I used Claude Code and GitHub Copilot to help debug and understand the code structure.

- Give one example of an AI suggestion that was correct (including what the AI suggested and how you verified the result).
  Claude suggested that the hints logic in `check_guess()` needed to compare guard properly — it told me to check if `guess > secret` means "Too High". I verified this by playing the game and changing the logic: when I guessed 60 and the secret was 50, it correctly said "Too High." Then I guessed 40 and it said "Too Low," which confirmed the fix worked.

- Give one example of an AI suggestion that was incorrect or misleading (including what the AI suggested and how you verified the result).
  Claude initially suggested the score calculation had an off-by-one error with an extra `+ 1`. I looked at the actual code and the implementation in `logic_utils.py` was already correct without that error, so that suggestion wasn't needed. I verified by reading the actual functions versus what was described in the bug report.

---

## 3. Debugging and testing your fixes

- How did you decide whether a bug was really fixed? 
  Testing! I would play the game, make a guess, and check the developer debug panel to see what the actual values were versus what the UI was showing.

- Describe at least one test you ran (manual or using pytest) and what it showed you about your code. 
  I found the score was negative and didn't match the final message, so I added print statements to track the score after each guess. This showed me the score calculation was wrong — it was subtracting 5 points for non-winning guesses when it shouldn't have been. After fixing the logic, the print statements showed the score was now correct and matched the final message.

- Did AI help you design or understand any tests? How? 
  Yes! Claude helped me understand why the test file expected tuples instead of strings from `check_guess()`. It explained that the function returns two values (outcome and message), so tests needed to unpack them correctly. That helped me understand the intended API design.

---

## 4. What did you learn about Streamlit and state?

- In your own words, explain why the secret number kept changing in the original app.
  Streamlit reruns the entire script from top to bottom every time you interact with something. If the secret wasn't stored in session state, it would generate a new random number every single rerun. The original code wasn't properly initializing the secret in session state, so it kept regenerating.

- How would you explain Streamlit "reruns" and session state to a friend who has never used Streamlit?
  Imagine a script that runs completely from start to finish every time you click a button. All variables get reset to their defaults unless you specifically save them somewhere. Session state is like a notebook that Streamlit keeps open and remembers between reruns — you write values into it and they stick around even after the script reruns. Without session state, the app wouldn't remember anything from the previous interaction.

- What change did you make that finally gave the game a stable secret number?
  The code was already properly using `if "secret" not in st.session_state:` to initialize the secret number once in session state. The fix was making sure this ran correctly and all the other state variables (attempts, score, difficulty) were also properly initialized and reset only when needed.

---

## 5. Looking ahead: your developer habits

- What is one habit or strategy from this project that you want to reuse in future labs or projects?
  I want to keep using print statements and debug panels to visualize what's actually happening in my code versus what I think is happening. It's super effective for catching logic bugs that aren't obvious from just reading the code. I also want to always separate business logic from UI code like we did with `logic_utils.py`.

- What is one thing you would do differently next time you work with AI on a coding task?
  Next time I'll ask the AI to explain not just what's wrong but WHY it's wrong, so I actually understand the root cause instead of just applying a fix. I'd also verify examples the AI gives me by testing them myself rather than just trusting them.

- In one or two sentences, describe how this project changed the way you think about AI generated code.
  I realized that AI-generated code can look correct at first glance but have subtle bugs in logic, so I need to actually test and verify it works instead of assuming it does. It's a useful tool for speed and ideas, but it requires human verification to catch these kinds of logic errors.
