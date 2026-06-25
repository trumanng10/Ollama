# Lab: Build a Local Agentic AI Python Code Quality Reviewer on Ubuntu

## 1. What You Are Building

You will build a local AI system that can review Python code quality on your Ubuntu machine.

The system will:

1. Read your Python file.
2. Run automated code-quality tools.
3. Send the code and tool results to a local LLM.
4. Generate a structured AI review report.
5. Suggest bugs, risks, improvements, and test cases.

The important point is this: **your source code stays inside your own machine**. Ollama runs models locally and exposes a local API at `localhost:11434` when it is running. ([Ollama Documentation][1])

---

# 2. Recommended Architecture

```text
Developer
   |
   v
Python Code File
   |
   v
Static Analysis Layer
   |-- Ruff: fast linting
   |-- Pylint: deeper code quality review
   |-- Mypy: type checking
   |-- Bandit: security scanning
   |
   v
Local LLM Layer
   |-- Ollama
   |-- qwen2.5-coder:7b
   |
   v
Agentic Review Script
   |
   v
Markdown Code Quality Report
```

My opinion: **do not use LLM alone for code review**. The best design is: traditional tools find objective issues, then the LLM explains and prioritises them.

---

# 3. Recommended Model

For beginners, use:

```bash
qwen2.5-coder:7b
```

Qwen2.5-Coder is available in multiple model sizes including **0.5B, 1.5B, 3B, 7B, 14B, and 32B**, and is designed for code generation, code reasoning, and code fixing. ([Ollama][2])

## Model Selection Guide

| Your Machine               | Recommended Model   |
| -------------------------- | ------------------- |
| CPU only, 8GB RAM          | `qwen2.5-coder:3b`  |
| Normal laptop, 16GB RAM    | `qwen2.5-coder:7b`  |
| Good workstation, 32GB RAM | `qwen2.5-coder:14b` |
| High-end GPU machine       | `qwen2.5-coder:32b` |

For your first lab, I recommend:

```bash
ollama run qwen2.5-coder:7b
```

---

# 4. Part A — Prepare Ubuntu

## Step 1: Open Terminal

Press:

```text
Ctrl + Alt + T
```

Then check your Ubuntu version:

```bash
lsb_release -a
```

Check system memory:

```bash
free -h
```

Check CPU:

```bash
lscpu
```

Check disk space:

```bash
df -h
```

You should ideally have at least:

```text
RAM: 16GB recommended
Disk: 20GB free minimum
OS: Ubuntu 22.04 or 24.04
```

---

# 5. Part B — Install System Packages

Run:

```bash
sudo apt update
```

Then install basic tools:

```bash
sudo apt install curl git nano python3 python3-pip python3-venv -y
```

Check Python version:

```bash
python3 --version
```

Check pip version:

```bash
pip3 --version
```

Expected example:

```text
Python 3.10.x or newer
pip 22.x or newer
```

---

# 6. Part C — Install Ollama

Run this command:

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

This is the standard Linux install command shown in Ollama installation guidance. Ollama also provides a local REST API for generating model responses. ([Ollama Documentation][1])

Check installation:

```bash
ollama --version
```

Start Ollama service manually if needed:

```bash
ollama serve
```

If it says the address is already in use, that is usually fine because Ollama is already running.

Open a second terminal and test the local API:

```bash
curl http://localhost:11434/api/generate -d '{
  "model": "qwen2.5-coder:7b",
  "prompt": "Say hello",
  "stream": false
}'
```

At this stage, it may fail if the model is not downloaded yet. That is normal.

---

# 7. Part D — Download the Local Coding Model

Run:

```bash
ollama pull qwen2.5-coder:7b
```

Or run directly:

```bash
ollama run qwen2.5-coder:7b
```

The first time you run it, Ollama will download the model.

Test it:

```bash
ollama run qwen2.5-coder:7b
```

Then type:

```text
Review this Python code: def add(a,b): return a+b
```

Exit the Ollama chat by typing:

```text
/bye
```

---

# 8. Part E — Create Your Project Folder

Create a new folder:

```bash
mkdir ~/ai-code-reviewer
cd ~/ai-code-reviewer
```

Create a Python virtual environment:

```bash
python3 -m venv .venv
```

Activate it:

```bash
source .venv/bin/activate
```

Your terminal should now show something like:

```text
(.venv) user@ubuntu:~/ai-code-reviewer$
```

Upgrade pip:

```bash
pip install --upgrade pip
```

---

# 9. Part F — Install Python Code Quality Tools

Install the tools:

```bash
pip install ruff pylint mypy bandit pytest requests
```

What each tool does:

| Tool       | Purpose                                   |
| ---------- | ----------------------------------------- |
| `ruff`     | Fast Python linting and formatting checks |
| `pylint`   | Deeper code-quality and style analysis    |
| `mypy`     | Static type checking                      |
| `bandit`   | Security issue scanning                   |
| `pytest`   | Unit testing                              |
| `requests` | Allows Python to call Ollama API          |

Ruff’s own FAQ recommends using Ruff together with a type checker such as Mypy because Ruff gives fast lint feedback while the type checker finds type-related issues. ([Astral Docs][3])

Check installation:

```bash
ruff --version
pylint --version
mypy --version
bandit --version
pytest --version
```

---

# 10. Part G — Create a Bad Python File for Testing

Create a sample file:

```bash
nano sample_bad_code.py
```

Paste this code:

```python
import os

def get_user(id):
    query = "SELECT * FROM users WHERE id = " + str(id)
    print(query)
    return os.system(query)

def calculate(x,y):
    return x+y
```

Save in nano:

```text
Ctrl + O
Enter
Ctrl + X
```

This code is intentionally weak. It contains poor naming, no type hints, poor spacing, possible unsafe command execution, and poor design.

---

# 11. Part H — Run Manual Code Quality Checks

Run Ruff:

```bash
ruff check sample_bad_code.py
```

Run Pylint:

```bash
pylint sample_bad_code.py
```

Run Bandit:

```bash
bandit sample_bad_code.py
```

Run Mypy:

```bash
mypy sample_bad_code.py
```

At this point, you are doing the review manually.

Next, we automate it and ask the local LLM to explain the results.

---

# 12. Part I — Create the AI Reviewer Script

Create a file:

```bash
nano review_code.py
```

Paste the full script below:

````python
import argparse
import subprocess
import sys
from datetime import datetime
from pathlib import Path

import requests

MODEL = "qwen2.5-coder:7b"
OLLAMA_URL = "http://localhost:11434/api/generate"


def run_command(command: list[str]) -> str:
    """
    Run a terminal command and return its output.

    This is used to run:
    - ruff
    - pylint
    - bandit
    - mypy
    """
    try:
        result = subprocess.run(
            command,
            capture_output=True,
            text=True,
            timeout=60,
            check=False,
        )

        output = ""

        if result.stdout:
            output += result.stdout

        if result.stderr:
            output += "\n" + result.stderr

        if not output.strip():
            output = "No issues reported."

        return output.strip()

    except FileNotFoundError:
        return (
            f"Command not found: {command[0]}\n"
            f"Please install it first. Example:\n"
            f"pip install ruff pylint mypy bandit pytest requests"
        )

    except subprocess.TimeoutExpired:
        return f"Command timed out: {' '.join(command)}"

    except Exception as error:
        return f"Error running command {' '.join(command)}: {error}"


def ask_ollama(prompt: str) -> str:
    """
    Send a prompt to local Ollama model and return the response.
    """
    try:
        response = requests.post(
            OLLAMA_URL,
            json={
                "model": MODEL,
                "prompt": prompt,
                "stream": False,
            },
            timeout=180,
        )

        response.raise_for_status()
        return response.json()["response"]

    except requests.exceptions.ConnectionError as error:
        raise RuntimeError(
            "Cannot connect to Ollama.\n"
            "Please make sure Ollama is running.\n"
            "Try this command in another terminal:\n"
            "ollama serve"
        ) from error

    except requests.exceptions.Timeout as error:
        raise RuntimeError(
            "Ollama request timed out.\n"
            "Try a smaller model, for example: qwen2.5-coder:3b"
        ) from error

    except requests.exceptions.HTTPError as error:
        raise RuntimeError(
            f"Ollama returned an HTTP error: {error}\n"
            f"Please check whether the model exists:\n"
            f"ollama list"
        ) from error


def build_review_prompt(
    file_path: str,
    code: str,
    ruff_output: str,
    pylint_output: str,
    bandit_output: str,
    mypy_output: str,
) -> str:
    """
    Build the prompt that will be sent to the local LLM.
    """
    return f"""
You are a senior Python code quality reviewer.

Your job is to review the Python code below using both:
1. Your own code reasoning
2. The static analysis tool outputs

Please produce a professional code review report.

Use this format:

# AI Code Quality Review Report

## 1. Executive Summary
Summarize the overall code quality in beginner-friendly language.

## 2. Critical Bugs
List bugs that may cause wrong output, crashes, or dangerous behavior.

## 3. Security Risks
Explain any security risks clearly.

## 4. Code Quality Issues
Comment on readability, naming, structure, duplication, and maintainability.

## 5. Type Safety Issues
Explain missing type hints or possible type problems.

## 6. Refactoring Suggestions
Suggest practical improvements.

## 7. Improved Code
Provide a safer and cleaner version of the code.

## 8. Suggested Pytest Unit Tests
Provide simple pytest test cases.

## 9. Priority Ranking
Create a table with:
- Issue
- Severity: High / Medium / Low
- Recommended action

Important rules:
- Do not exaggerate.
- Do not rewrite the whole application unnecessarily.
- Explain in a way a beginner can understand.
- Be practical.
- Keep security recommendations safe and defensive.
- Do not suggest destructive terminal commands.

Python file name:
{file_path}

Python code:
```python
{code}
```

Ruff output:
```text
{ruff_output}
```

Pylint output:
```text
{pylint_output}
```

Bandit output:
```text
{bandit_output}
```

Mypy output:
```text
{mypy_output}
```
"""


def review_python_file(file_path: str) -> tuple[str, str]:
    """
    Review one Python file using static analysis tools and local LLM.

    Returns:
        tuple[str, str]: report file name and AI report content
    """
    path = Path(file_path)

    if not path.exists():
        raise FileNotFoundError(f"File not found: {file_path}")

    if path.suffix != ".py":
        raise ValueError("This reviewer only supports .py files.")

    code = path.read_text(encoding="utf-8")

    print("Running Ruff...")
    ruff_output = run_command(["ruff", "check", file_path])

    print("Running Pylint...")
    pylint_output = run_command(["pylint", file_path])

    print("Running Bandit...")
    bandit_output = run_command(["bandit", file_path])

    print("Running Mypy...")
    mypy_output = run_command(["mypy", file_path])

    prompt = build_review_prompt(
        file_path=file_path,
        code=code,
        ruff_output=ruff_output,
        pylint_output=pylint_output,
        bandit_output=bandit_output,
        mypy_output=mypy_output,
    )

    print("Sending results to local Ollama model...")
    ai_report = ask_ollama(prompt)

    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    report_file = f"code_review_report_{timestamp}.md"

    Path(report_file).write_text(ai_report, encoding="utf-8")

    return report_file, ai_report


def parse_arguments() -> argparse.Namespace:
    """
    Parse command-line arguments.

    Example:
        python review_code.py sample_bad_code.py
        python review_code.py app.py
    """
    parser = argparse.ArgumentParser(
        description="Review Python code quality using local Ollama and static analysis tools."
    )

    parser.add_argument(
        "file",
        nargs="?",
        default="sample_bad_code.py",
        help="Python file to review. Default: sample_bad_code.py",
    )

    return parser.parse_args()


def main() -> int:
    """
    Main program entry point.
    """
    args = parse_arguments()

    try:
        report_file, report = review_python_file(args.file)

        print("\n" + "=" * 80)
        print("AI CODE REVIEW COMPLETED")
        print("=" * 80)
        print(f"Report saved to: {report_file}")
        print("=" * 80)
        print(report)

        return 0

    except Exception as error:
        print("\n" + "=" * 80)
        print("AI CODE REVIEW FAILED")
        print("=" * 80)
        print(error)
        return 1


if __name__ == "__main__":
    sys.exit(main())


````

Save:

```text
Ctrl + O
Enter
Ctrl + X
````

---

# 13. Part J — Run the AI Reviewer

Make sure your virtual environment is active:

```bash
source .venv/bin/activate
```

Run the script:

```bash
python review_code.py
```

Expected process:

```text
Running Ruff...
Running Pylint...
Running Bandit...
Running Mypy...
Sending results to local Ollama model...
AI CODE REVIEW COMPLETED
Report saved to: code_review_report_YYYYMMDD_HHMMSS.md
```

List generated reports:

```bash
ls -lh
```

Open the report:

```bash
cat code_review_report_*.md
```

Or use nano:

```bash
nano code_review_report_*.md
```

---

# 14. Part K — Expected Findings

The AI should explain issues such as:

| Issue                       | Why It Matters                                |
| --------------------------- | --------------------------------------------- |
| `id` used as parameter name | Shadows Python built-in `id()`                |
| SQL string concatenation    | Bad practice; can become injection risk       |
| `os.system(query)`          | Unsafe and logically wrong for database query |
| Missing type hints          | Harder to maintain                            |
| Poor spacing                | Violates style expectations                   |
| No docstrings               | Harder for team members to understand         |
| No tests                    | Cannot verify correctness                     |

A better version of the simple `calculate` function should look like this:

```python
def calculate(x: int, y: int) -> int:
    return x + y
```

---

# 15. Part L — Make It More Agentic

Now we improve from a simple script into an **agentic workflow**.

## Agent Design

```text
Agent 1: Scanner Agent
- Reads code file
- Runs Ruff, Pylint, Bandit, Mypy

Agent 2: Reviewer Agent
- Sends all findings to local LLM
- Produces explanation

Agent 3: Refactor Agent
- Suggests safer code

Agent 4: Test Agent
- Suggests pytest unit tests

Agent 5: Report Agent
- Saves the final report into Markdown
```

In this beginner version, all agents are inside one Python script as functions. Later, you can separate them into different files.

---

# 16. Part M — Improved Agentic Version

Create another file:

```bash
nano agentic_review.py
```

Paste this:

````python
import subprocess
import requests
from pathlib import Path
from datetime import datetime

MODEL = "qwen2.5-coder:7b"
OLLAMA_URL = "http://localhost:11434/api/generate"


class ScannerAgent:
    """
    Agent 1:
    Runs static analysis tools against the Python file.
    """

    def run_command(self, command: list[str]) -> str:
        try:
            result = subprocess.run(
                command,
                capture_output=True,
                text=True,
                timeout=60
            )

            output = result.stdout + "\n" + result.stderr

            if not output.strip():
                return "No issues reported."

            return output.strip()

        except Exception as error:
            return f"Error running {' '.join(command)}: {error}"

    def scan(self, file_path: str) -> dict:
        return {
            "ruff": self.run_command(["ruff", "check", file_path]),
            "pylint": self.run_command(["pylint", file_path]),
            "bandit": self.run_command(["bandit", file_path]),
            "mypy": self.run_command(["mypy", file_path]),
        }


class LocalLLMAgent:
    """
    Base agent:
    Sends prompts to local Ollama model.
    """

    def ask(self, prompt: str) -> str:
        response = requests.post(
            OLLAMA_URL,
            json={
                "model": MODEL,
                "prompt": prompt,
                "stream": False
            },
            timeout=180
        )

        response.raise_for_status()
        return response.json()["response"]


class ReviewerAgent(LocalLLMAgent):
    """
    Agent 2:
    Reviews code quality and explains findings.
    """

    def review(self, file_path: str, code: str, scan_results: dict) -> str:
        prompt = f"""
You are Reviewer Agent.

Review this Python code for:
- bugs
- readability
- maintainability
- security
- performance
- type safety

Python file:
{file_path}

Code:
```python
{code}
````

Static analysis results:
{scan_results}

Return a clear beginner-friendly review.
"""
return self.ask(prompt)

class RefactorAgent(LocalLLMAgent):
"""
Agent 3:
Suggests improved code.
"""

```
def suggest_refactor(self, code: str, review: str) -> str:
    prompt = f"""
```

You are Refactor Agent.

Based on the review below, suggest a safer and cleaner version of the Python code.

Rules:

* Keep the code simple.
* Do not add unnecessary frameworks.
* Add type hints.
* Add comments only where useful.
* Explain what changed.

Original code:

```python
{code}
```

Review:

```text
{review}
```

"""
return self.ask(prompt)

class TestAgent(LocalLLMAgent):
"""
Agent 4:
Suggests pytest unit tests.
"""

```
def suggest_tests(self, code: str, review: str) -> str:
    prompt = f"""
```

You are Test Agent.

Create beginner-friendly pytest unit tests for this Python code.

Original code:

```python
{code}
```

Review:

```text
{review}
```

Return only useful tests and explain how to run them.
"""
return self.ask(prompt)

class ReportAgent:
"""
Agent 5:
Combines all outputs and saves a Markdown report.
"""

```
def create_report(
    self,
    file_path: str,
    scan_results: dict,
    review: str,
    refactor: str,
    tests: str
) -> str:
    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    report_file = f"agentic_code_review_{timestamp}.md"

    report = f"""
```

# Agentic AI Code Review Report

## File Reviewed

`{file_path}`

## 1. Static Analysis Results

### Ruff

```text
{scan_results["ruff"]}
```

### Pylint

```text
{scan_results["pylint"]}
```

### Bandit

```text
{scan_results["bandit"]}
```

### Mypy

```text
{scan_results["mypy"]}
```

## 2. Reviewer Agent Output

{review}

## 3. Refactor Agent Output

{refactor}

## 4. Test Agent Output

{tests}
"""

```
    Path(report_file).write_text(report, encoding="utf-8")
    return report_file
```

def main():
file_path = "sample_bad_code.py"
path = Path(file_path)

```
if not path.exists():
    raise FileNotFoundError(f"Cannot find {file_path}")

code = path.read_text(encoding="utf-8")

print("Agent 1: Scanner Agent running...")
scanner = ScannerAgent()
scan_results = scanner.scan(file_path)

print("Agent 2: Reviewer Agent running...")
reviewer = ReviewerAgent()
review = reviewer.review(file_path, code, scan_results)

print("Agent 3: Refactor Agent running...")
refactor_agent = RefactorAgent()
refactor = refactor_agent.suggest_refactor(code, review)

print("Agent 4: Test Agent running...")
test_agent = TestAgent()
tests = test_agent.suggest_tests(code, review)

print("Agent 5: Report Agent running...")
report_agent = ReportAgent()
report_file = report_agent.create_report(
    file_path=file_path,
    scan_results=scan_results,
    review=review,
    refactor=refactor,
    tests=tests
)

print("\nAgentic review completed.")
print(f"Report saved to: {report_file}")
```

if **name** == "**main**":
main()

````

Save and run:

```bash
python agentic_review.py
````

Open the generated report:

```bash
ls -lh
cat agentic_code_review_*.md
```

---

# 17. Part N — Review Your Own Python File

Put your real Python file into the folder:

```bash
cp /path/to/your_file.py ~/ai-code-reviewer/
```

Example:

```bash
cp ~/Downloads/app.py ~/ai-code-reviewer/
```

Edit the script:

```bash
nano agentic_review.py
```

Find:

```python
file_path = "sample_bad_code.py"
```

Change to:

```python
file_path = "app.py"
```

Run:

```bash
python agentic_review.py
```

---

# 18. Part O — Common Beginner Errors and Fixes

## Error 1: `ollama: command not found`

Fix:

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

Then restart terminal:

```bash
exec bash
```

---

## Error 2: Cannot connect to Ollama

You may see:

```text
Connection refused
```

Fix:

```bash
ollama serve
```

Then open another terminal and run:

```bash
python agentic_review.py
```

---

## Error 3: Model not found

You may see:

```text
model qwen2.5-coder:7b not found
```

Fix:

```bash
ollama pull qwen2.5-coder:7b
```

---

## Error 4: Python package not found

Example:

```text
ModuleNotFoundError: No module named 'requests'
```

Fix:

```bash
source .venv/bin/activate
pip install requests
```

Or reinstall all tools:

```bash
pip install ruff pylint mypy bandit pytest requests
```

---

## Error 5: Model is too slow

Use a smaller model:

```bash
ollama pull qwen2.5-coder:3b
```

Then edit your Python script:

```python
MODEL = "qwen2.5-coder:3b"
```

Run again:

```bash
python agentic_review.py
```

---

# 19. Part P — Recommended Folder Structure

Your final folder should look like this:

```text
ai-code-reviewer/
│
├── .venv/
├── sample_bad_code.py
├── review_code.py
├── agentic_review.py
├── code_review_report_20260626_120000.md
└── agentic_code_review_20260626_120500.md
```

---

# 20. Part Q — Optional: Add a Simple Run Command

Create a shell script:

```bash
nano run_review.sh
```

Paste:

```bash
#!/bin/bash

source .venv/bin/activate
python agentic_review.py
```

Save and exit.

Make it executable:

```bash
chmod +x run_review.sh
```

Run:

```bash
./run_review.sh
```

---

# 21. Part R — Optional: Review All Python Files in a Folder

Later, you can upgrade the script to review all `.py` files.

Example concept:

```python
from pathlib import Path

for file in Path(".").glob("*.py"):
    print(file)
```

This allows your agent to review a whole project instead of one file.

---

# 22. Business Use Case for Nexplore

## Use Case Name

**Private Local AI Code Quality Reviewer for Enterprise Python Projects**

## Target Users

* Internal developers
* Interns
* Software trainers
* Enterprise IT teams
* Banking and insurance teams
* Government project teams
* Data analytics teams

## Business Value

This solution helps organisations:

* Improve code quality
* Reduce junior developer mistakes
* Review code before deployment
* Keep source code private
* Train developers using AI feedback
* Generate documentation and test cases faster

## Why This Is a Strong Agentic AI Lab

This is not just “chat with AI.” It shows a real enterprise pattern:

```text
Tool Execution + LLM Reasoning + Report Generation + Human Approval
```

That is exactly how practical Agentic AI should be taught.

---

# 23. Final Recommended Stack

For your first implementation, use this:

```text
Ubuntu 22.04 / 24.04
Python 3.10+
Ollama
qwen2.5-coder:7b
Ruff
Pylint
Mypy
Bandit
Pytest
Markdown report
```

**Qwen2.5-Coder 7B + Ruff + Bandit + Mypy is the best beginner-to-enterprise combination**. 
It is simple enough for a training lab, but serious enough to demonstrate real private AI code review.

[1]: https://docs.ollama.com/api/introduction?utm_source=chatgpt.com "Introduction"
[2]: https://ollama.com/library/qwen2.5-coder?utm_source=chatgpt.com "qwen2.5-coder"
[3]: https://docs.astral.sh/ruff/faq/?utm_source=chatgpt.com "FAQ | Ruff"
