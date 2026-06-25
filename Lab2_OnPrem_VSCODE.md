# Lab: Agentic Coding Assistant with VS Code + Cline + Local Ollama

## Lab Objective

By the end of this lab, you will have:

```text
Ubuntu Linux
+ VS Code
+ Python development tools
+ Cline extension
+ Local Ollama model
+ Python code quality tools
+ Agentic coding workflow
```

This allows your local AI assistant to:

* Read your project files
* Understand your Python code
* Run terminal commands with your permission
* Execute `ruff`, `pylint`, `mypy`, `bandit`, and `pytest`
* Analyse errors
* Suggest fixes
* Edit code after approval
* Generate a code quality report

Cline is an AI coding agent that works inside the editor and terminal; it can read/write files, run commands, and assist with building software. ([Cline][1])

---

# 1. Target Architecture

```text
Developer
   |
   v
VS Code on Ubuntu
   |
   v
Cline Extension
   |
   v
Local Ollama API
   |
   v
Local Coding Model
   |
   v
Python Project Folder
   |
   |-- ruff
   |-- pylint
   |-- mypy
   |-- bandit
   |-- pytest
   |
   v
Agentic AI Code Review Report
```

 **Cline is better than a normal chatbot when you want “agentic coding”** because it can work with your project files and terminal, not just answer questions.

---

# 2. Prerequisites Checklist

Before starting, make sure these are ready:

| Item                      | Required? | Status                            |
| ------------------------- | --------: | --------------------------------- |
| Ubuntu Linux              |       Yes | Required                          |
| Internet access           |       Yes | For installing VS Code/extensions |
| Ollama installed          |       Yes | Already completed                 |
| Local model downloaded    |       Yes | Already completed                 |
| Python installed          |       Yes | We will check/install             |
| VS Code installed         |       Yes | We will install                   |
| Cline extension installed |       Yes | We will install                   |
| Python code quality tools |       Yes | We will install                   |

---

# 3. Part A — Verify Ollama Is Running

Even though Ollama is already installed, verify that the local service is working.

Open Terminal:

```bash
ollama list
```

You should see your downloaded model, for example:

```text
qwen2.5-coder:7b
```

Test Ollama:

```bash
ollama run qwen2.5-coder:7b
```

Type:

```text
Say hello and confirm you are ready for Python code review.
```

Exit:

```text
/bye
```

Check local API:

```bash
curl http://localhost:11434/api/tags
```

Expected result: you should see a JSON response containing your local model name.

If Ollama is not running, start it:

```bash
ollama serve
```

Ollama exposes a local API by default at `localhost:11434`, which is what Cline will connect to. ([Ollama][2])

---

# 4. Part B — Install Basic Ubuntu Developer Tools

Run:

```bash
sudo apt update
```

Install basic packages:

```bash
sudo apt install curl wget git nano build-essential python3 python3-pip python3-venv -y
```

Check versions:

```bash
python3 --version
pip3 --version
git --version
gcc --version
```

Expected:

```text
Python 3.10 or newer
pip installed
git installed
gcc installed
```

The VS Code Python extension still requires a Python interpreter installed separately, so installing `python3` is necessary. ([Visual Studio Code][3])

---

# 5. Part C — Install VS Code on Ubuntu

You have two common options. I recommend the **`.deb` method** for training labs because it is straightforward and closer to Microsoft’s official Linux setup flow. Microsoft’s Linux setup page provides `.deb` installation guidance for Debian/Ubuntu-based systems. ([Visual Studio Code][4])

## Option 1: Install VS Code Using `.deb`

Run:

```bash
cd ~/Downloads
```

Download VS Code:

```bash
wget -O code.deb "https://code.visualstudio.com/sha/download?build=stable&os=linux-deb-x64"
```

Install:

```bash
sudo apt install ./code.deb -y
```

Check:

```bash
code --version
```

Open VS Code:

```bash
code
```

---

## Option 2: Install VS Code Using Snap

```bash
sudo snap install code --classic
```

Open:

```bash
code
```

For Nexplore lab delivery, I prefer **Option 1 `.deb`**.

---

# 6. Part D — Install VS Code Extensions

Install the core extensions from terminal:

```bash
code --install-extension ms-python.python
code --install-extension ms-python.vscode-pylance
code --install-extension charliermarsh.ruff
code --install-extension saoudrizwan.claude-dev
```

What these extensions do:

| Extension          | Purpose                                     |
| ------------------ | ------------------------------------------- |
| Python             | Python support in VS Code                   |
| Pylance            | Better Python IntelliSense and type support |
| Ruff               | Fast Python linting support                 |
| Cline              | Agentic coding assistant                    |
| Ollama local model | AI brain used by Cline                      |

The Python extension helps VS Code work with Python projects, but the Python interpreter still needs to be installed separately. ([Visual Studio Code][5])

---

# 7. Part E — Create a Python Lab Project

Create project folder:

```bash
mkdir -p ~/agentic-python-review-lab
cd ~/agentic-python-review-lab
```

Create Python virtual environment:

```bash
python3 -m venv .venv
```

Activate it:

```bash
source .venv/bin/activate
```

Upgrade pip:

```bash
pip install --upgrade pip
```

Install Python code quality tools:

```bash
pip install ruff pylint mypy bandit pytest
```

Create requirements file:

```bash
pip freeze > requirements.txt
```

Open folder in VS Code:

```bash
code .
```

---

# 8. Part F — Select Python Interpreter in VS Code

Inside VS Code:

Press:

```text
Ctrl + Shift + P
```

Search:

```text
Python: Select Interpreter
```

Choose the interpreter inside your project:

```text
.venv/bin/python
```

This is important because VS Code must use your project’s virtual environment.

---

# 9. Part G — Create a Bad Python File for Testing

Inside VS Code, create a file:

```text
app.py
```

Paste this intentionally poor code:

```python
import os

def add(x,y):
    return x+y

def run_user_command(cmd):
    return os.system(cmd)

def calculate_discount(price, discount):
    final_price = price - price * discount
    return final_price

print(add(1,"2"))
```

This file has issues:

* Poor spacing
* Missing type hints
* Unsafe command execution
* Possible type bug: `1 + "2"`
* No tests
* No main guard
* Weak function documentation

---

# 10. Part H — Create a Basic Test File

Create a folder:

```bash
mkdir tests
```

Create test file:

```bash
nano tests/test_app.py
```

Paste:

```python
from app import add, calculate_discount


def test_add_numbers():
    assert add(1, 2) == 3


def test_calculate_discount():
    assert calculate_discount(100, 0.10) == 90
```

Save and exit.

Run tests:

```bash
pytest
```

You may see problems because the original `app.py` has this line:

```python
print(add(1,"2"))
```

That is good for the lab because Cline can help identify and fix it.

---

# 11. Part I — Run Code Quality Tools Manually First

Before using Cline, run tools yourself once.

```bash
ruff check app.py
```

```bash
pylint app.py
```

```bash
mypy app.py
```

```bash
bandit app.py
```

```bash
pytest
```

This gives you the baseline.

The agentic assistant should not replace these tools. It should help you understand and fix the findings.

---

# 12. Part J — Configure Cline to Use Local Ollama

Open VS Code.

Open the Cline panel from the sidebar.

Go to:

```text
Cline Settings
```

Set:

```text
API Provider: Ollama
Base URL: http://localhost:11434
Model: qwen2.5-coder:7b
```

Cline’s local model guidance says the basic pattern is: install a local runtime such as Ollama, start the local server, select the matching provider in Cline, and select the local model. ([Cline][6])

Ollama’s own Cline integration guide also says to open Cline settings, set API Provider to Ollama, choose or type a model, and update the context window. ([Ollama][7])

Recommended beginner settings:

```text
Use Compact Prompt: Enabled
Auto-approve file edits: Disabled
Auto-approve terminal commands: Disabled
Browser use: Disabled for this lab
Model: qwen2.5-coder:7b
Base URL: http://localhost:11434
```

My opinion: for a beginner lab, **do not enable full auto-approval**. Let Cline propose actions, then you approve them one by one.

---

# 13. Part K — First Cline Prompt: Project Inspection

In Cline chat, paste:

```text
You are my local on-prem Python code quality assistant.

Please inspect this project folder. Do not modify files yet.

Tasks:
1. Identify the Python files.
2. Explain the purpose of each file.
3. Check whether a virtual environment exists.
4. Check whether tests exist.
5. Recommend the first code quality commands I should run.

Important:
- Use only local tools.
- Do not call cloud APIs.
- Do not edit files until I approve.
```

Expected Cline behaviour:

```text
Cline may ask permission to list files or run terminal commands.
```

Approve safe commands such as:

```bash
ls
find . -name "*.py"
python3 --version
```

---

# 14. Part L — Second Cline Prompt: Run Code Quality Tools

Paste into Cline:

```text
Run a code quality review for this Python project.

Please run these commands one by one:
1. ruff check app.py
2. pylint app.py
3. mypy app.py
4. bandit app.py
5. pytest

After each command:
- Show me the result.
- Explain the meaning in beginner-friendly language.
- Do not modify files yet.
```

Approve each command when Cline asks.

Expected result:

| Tool   | Expected Finding                        |
| ------ | --------------------------------------- |
| Ruff   | Style/lint issues                       |
| Pylint | Missing docstrings, naming, type issues |
| Mypy   | Type mismatch may be detected           |
| Bandit | `os.system` security warning            |
| Pytest | May fail due to import-time error       |

---

# 15. Part M — Third Cline Prompt: Ask for Refactoring Plan

Paste:

```text
Based on the tool results, create a safe refactoring plan.

Do not edit the files yet.

Give me:
1. High priority issues
2. Medium priority issues
3. Low priority issues
4. Recommended fix order
5. Risks if we fix wrongly
6. Which files need to be edited
```

Expected recommendation:

```text
High:
- Remove unsafe os.system usage or restrict it.
- Fix print(add(1,"2")) type error.

Medium:
- Add type hints.
- Add main guard.
- Improve formatting.

Low:
- Add docstrings.
- Add more tests.
```

---

# 16. Part N — Fourth Cline Prompt: Approve Safe Edits

Now ask Cline to edit the file.

Paste:

```text
Please safely refactor app.py.

Approved changes:
1. Add type hints.
2. Fix spacing.
3. Add simple docstrings.
4. Move demo code under if __name__ == "__main__".
5. Replace print(add(1,"2")) with a valid example.
6. Keep run_user_command for now, but add a warning docstring that it is unsafe and should not be used with untrusted input.

Do not change the public function names.
After editing, show me exactly what changed.
```

Cline may ask permission to edit `app.py`.

Approve only after reviewing.

Expected improved `app.py`:

```python
import os


def add(x: int, y: int) -> int:
    """Return the sum of two integers."""
    return x + y


def run_user_command(cmd: str) -> int:
    """
    Run a shell command.

    Warning:
        This function is unsafe if cmd contains untrusted user input.
    """
    return os.system(cmd)


def calculate_discount(price: float, discount: float) -> float:
    """Calculate the final price after applying a discount."""
    final_price = price - price * discount
    return final_price


if __name__ == "__main__":
    print(add(1, 2))
```

---

# 17. Part O — Fifth Cline Prompt: Improve Security

Now guide Cline to improve the risky command function.

Paste:

```text
Now improve the security of app.py.

Please replace run_user_command with a safer function using subprocess.run.

Requirements:
1. Do not use shell=True.
2. Accept command as a list of strings, not one string.
3. Return stdout as text.
4. Handle errors safely.
5. Add type hints.
6. Update or add tests.

Before editing, explain the change.
```

Expected safer function:

```python
import subprocess


def run_command(command: list[str]) -> str:
    """Run a command safely without shell=True and return stdout."""
    result = subprocess.run(
        command,
        capture_output=True,
        text=True,
        check=True
    )
    return result.stdout
```

Bandit should no longer complain about `os.system`.

---

# 18. Part P — Sixth Cline Prompt: Generate Tests

Paste:

```text
Please generate or update pytest tests for app.py.

Test requirements:
1. Test add()
2. Test calculate_discount()
3. Test run_command() with a safe command
4. Avoid destructive or dangerous commands
5. Run pytest after creating the tests
6. Show me the final test result
```

Expected test file:

```python
from app import add, calculate_discount, run_command


def test_add_numbers():
    assert add(1, 2) == 3


def test_calculate_discount():
    assert calculate_discount(100, 0.10) == 90


def test_run_command_echo():
    output = run_command(["echo", "hello"])
    assert "hello" in output
```

Run:

```bash
pytest
```

Expected:

```text
3 passed
```

---

# 19. Part Q — Seventh Cline Prompt: Re-run Full Quality Check

Paste:

```text
Please run the final full quality check:

1. ruff check app.py tests/test_app.py
2. pylint app.py tests/test_app.py
3. mypy app.py tests/test_app.py
4. bandit app.py
5. pytest

Then create a final summary:
- What improved
- What risks remain
- Whether this code is ready for beginner lab submission
```

Expected result:

```text
Ruff: clean or minor issues
Pylint: improved score
Mypy: success or minor typing issue
Bandit: no os.system warning
Pytest: passed
```

---

# 20. Part R — Ask Cline to Generate a Markdown Report

Paste:

```text
Create a Markdown report named CODE_REVIEW_REPORT.md.

The report must include:
1. Project overview
2. Original issues found
3. Tools used
4. Refactoring changes made
5. Security improvements
6. Test results
7. Remaining recommendations
8. Beginner learning summary

Do not include secrets, API keys, or private credentials.
```

Expected file:

```text
CODE_REVIEW_REPORT.md
```

Open it:

```bash
cat CODE_REVIEW_REPORT.md
```

---

# 21. Part S — Final Folder Structure

Your project should look like this:

```text
agentic-python-review-lab/
│
├── .venv/
├── app.py
├── requirements.txt
├── CODE_REVIEW_REPORT.md
└── tests/
    └── test_app.py
```

---

# 22. Full Beginner Command Summary

Here is the compact command list.

```bash
sudo apt update
sudo apt install curl wget git nano build-essential python3 python3-pip python3-venv -y
```

```bash
cd ~/Downloads
wget -O code.deb "https://code.visualstudio.com/sha/download?build=stable&os=linux-deb-x64"
sudo apt install ./code.deb -y
code --version
```

```bash
code --install-extension ms-python.python
code --install-extension ms-python.vscode-pylance
code --install-extension charliermarsh.ruff
code --install-extension saoudrizwan.claude-dev
```

```bash
mkdir -p ~/agentic-python-review-lab
cd ~/agentic-python-review-lab
python3 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
pip install ruff pylint mypy bandit pytest
pip freeze > requirements.txt
code .
```

```bash
ruff check app.py
pylint app.py
mypy app.py
bandit app.py
pytest
```

---

# 23. How This Becomes “Agentic AI”

This lab is agentic because Cline performs a loop:

```text
Observe project
   ↓
Run tools
   ↓
Read errors
   ↓
Reason about fixes
   ↓
Propose plan
   ↓
Edit files with permission
   ↓
Run tests
   ↓
Improve again
   ↓
Generate report
```

This is much stronger than a normal chatbot because the AI assistant interacts with the local development environment.

---

# 24. Safety and Governance Rules for Enterprise Lab

For enterprise training, add these rules:

```text
1. Do not enable auto-approve for file edits during beginner labs.
2. Do not enable auto-approve for terminal commands.
3. Do not allow cloud provider fallback.
4. Use local Ollama provider only.
5. Review every file change before accepting.
6. Never paste production secrets into prompts.
7. Keep code inside local machine or approved on-prem server.
8. Use Git before allowing AI refactoring.
```

Create Git snapshot before Cline edits:

```bash
git init
git add .
git commit -m "Initial lab version before AI refactoring"
```

After Cline edits:

```bash
git diff
```

Commit approved changes:

```bash
git add .
git commit -m "Refactor Python code with local agentic AI assistant"
```

---

# 25. Final Recommendation

Truman, for this lab, my recommended teaching sequence is:

```text
Step 1: Install VS Code and Python tools
Step 2: Install Cline extension
Step 3: Connect Cline to local Ollama
Step 4: Create weak Python code
Step 5: Ask Cline to inspect only
Step 6: Ask Cline to run code quality tools
Step 7: Ask Cline for refactoring plan
Step 8: Approve safe edits
Step 9: Run tests
Step 10: Generate final Markdown report
```

**Cline + Ollama is the better “agentic” demo**, while **Continue + Ollama is the better beginner chat-style coding assistant**. 
This Cline lab is more impressive because learners can see the AI agent run terminal commands, inspect errors, fix code, and validate results locally.

[1]: https://docs.cline.bot/cline-overview?utm_source=chatgpt.com "Cline Overview - Cline"
[2]: https://docs.ollama.com/integrations/vscode?utm_source=chatgpt.com "VS Code"
[3]: https://code.visualstudio.com/docs/languages/python?utm_source=chatgpt.com "Python in Visual Studio Code"
[4]: https://code.visualstudio.com/docs/setup/linux?utm_source=chatgpt.com "Installing Visual Studio Code on Linux"
[5]: https://code.visualstudio.com/docs/python/python-tutorial?utm_source=chatgpt.com "Getting Started with Python in VS Code"
[6]: https://docs.cline.bot/running-models-locally/overview?utm_source=chatgpt.com "Local models"
[7]: https://docs.ollama.com/integrations/cline?utm_source=chatgpt.com "Cline"
