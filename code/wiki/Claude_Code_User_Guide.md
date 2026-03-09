# Claude Code User Guide - Making Programming Simple

## Table of Contents
1. [Introduction](#introduction)
2. [Why Choose Claude Code](#why-choose-claude-code)
3. [Installation Guide](#installation-guide)
4. [Getting Started](#getting-started)
5. [Core Features Demo](#core-features-demo)
6. [Slash Commands Reference](#slash-commands-reference)
7. [Keyboard Shortcuts](#keyboard-shortcuts)
8. [Best Practices](#best-practices)
9. [FAQ](#faq)

## Introduction

### 🎓 What is Claude Code and Why It Exists

- **Claude** is Anthropic’s powerful AI model, designed to understand language, logic, math, and code more deeply than typical chatbots. Its latest version, **Claude Opus 4**, released on **May 22 2025**, is especially strong at multi‑step reasoning and large‑codebase tasks (e.g. coding), scoring high on software‑engineering benchmarks.
- **Claude Code** is a terminal‑based assistant that uses Claude Opus 4 or Sonnet 4 to help developers *write*, *debug*, *navigate*, and *refactor* code directly from the command line. It comes fully integrated with Git, test systems, and other tools you already use [Anthropic](https://docs.anthropic.com/en/docs/claude-code/overview).

---

### 🚀 High‑Level Features & Powers

| Capability                    | Description                                                                 |
|------------------------------|-----------------------------------------------------------------------------|
| **Native Terminal Integration** | Works directly in the terminal—no external server or complex setup required. |
| **Understands Entire Codebase** | After `cd` into a folder, Claude Code analyzes project structure and logic automatically. |
| **Natural Language Interaction** | Supports English and other languages. Simply describe your intent—Claude answers, writes, or modifies code accordingly. |
| **Privacy-Conscious Design** | Connects directly to Anthropic’s API with no intermediaries. Always asks for permission before modifying files. |
| **Real Execution Power**      | Goes beyond suggestions—can run code, debug, edit files, handle Git operations, and automate developer tasks. |

Think of it as your programming partner that understands your needs and helps you complete programming tasks!

## Why Choose Claude Code?

### Key Advantages
1. **Intelligent Understanding** - Understands natural language descriptions, no complex commands needed
2. **Full-Stack Capabilities** - From frontend to backend, Python to JavaScript, Claude Code can help
3. **Real-time Operations** - Create, edit, and run code directly
4. **Error-Friendly** - Automatically finds and fixes common errors
5. **Learning Assistant** - Explains code in detail to help you learn programming

### Who Should Use It?
- 👶 Programming Beginners: Learn programming through natural language
- 👨‍💻 Professional Developers: Boost development efficiency
- 🎓 Students: Complete programming assignments and projects
- 🔬 Researchers: Quickly implement algorithms and data processing

## Installation Guide

### Prerequisites: Install Node.js (Version 18 or Higher)

#### Windows Users (Using WSL)
```bash
# 1. First install WSL (run in PowerShell as Administrator)
wsl --install

# 2. After restarting, open WSL terminal
# 3. Install Node.js in WSL
1) install nvm(Node Version Manager): 
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
2) reboot terminal or run the following: 
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"

3)install Node.js 18: nvm install 18

# 4. Verify installation
node --version  # Should show v18.x.x or higher
```

#### Mac Users
```bash
# Open terminal
1) install nvm(Node Version Manager): 
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
2) reboot terminal or run the following: 
export NVM_DIR="$HOME/.nvm"      
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
3)install Node.js 18: nvm install 18

# Verify installation
node --version
```

#### Linux Users
```bash
# Ubuntu/Debian
#1) install nvm(Node Version Manager): 
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
#2) reboot terminal or run the following: 
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
#3)install Node.js 18:
nvm install 18

# Verify installation
node --version
```

### Installing Claude Code

```bash
# Install globally using npm
npm install -g @anthropic-ai/claude-code

# Verify installation
claude --version
```

### Setting Up API Key (OR just use Claude account with subscription：recommend)
```bash
# Get API key from: https://console.anthropic.com/
# Set environment variable

# Linux/Mac/WSL
export ANTHROPIC_API_KEY="your-api-key"

# Add to config file for permanent setup
echo 'export ANTHROPIC_API_KEY="your-api-key"' >> ~/.bashrc
source ~/.bashrc
```

## [Getting Started](https://docs.anthropic.com/en/docs/claude-code/quickstart)

### Basic Environment Setup

1. **Cd to your Working Directory**
```bash
cd my-first-project
```

2. **Start Claude Code**
```bash
claude
```

3. **Basic Interaction Example**
```
You: Hello! Please create a simple Python program that prints "Hello World"

Claude: I'll help you create a simple Python program.

[Creates hello.py]
print("Hello World")

You: Run this program

Claude: [Runs python hello.py]
Hello World
```

## Core Features Demo

### 1. Code Explanation
```bash
# Test command
Please explain how the code in mwe/calculator.py works
```
OR One-time non-interactive execution: Execute a prompt directly, print the result, and exit: 
```bash
# In the terminal outside claude
claude -p "Please explain how the code in mwe/calculator.py works" 
```
### 2. Debugging
```bash
# Test command
My mwe/buggy_code.py has errors, please help me debug
```

### 3. Writing New Code
```bash
# Test command
Help me write a simple todo list manager in Python, save it to mwe/todo_app.py
```

### 4. Editing Existing Code
```bash
# Test command
Add multiplication and division functions to mwe/calculator.py
```

### 5. Git Integration
```bash
# Test command
Help me commit all changes with message 'Add new features'
```

### 6. Chart Generation
```bash
# Test command
Use matplotlib to create a bar chart showing monthly sales data, save to test/outputs/sales_chart.png
```

### 7. Testing and Debugging
```bash
# Test command
Write unit tests for mwe/calculator.py
```

### 8. Documentation Generation
```bash
# Test command
Generate API documentation for all Python files in mwe directory
```

### 9. Code Review
```bash
# Test command
Review the code quality of mwe/calculator.py and suggest improvements
```

## Slash Commands Reference

| Command | Function | Example |
|---------|----------|---------|
| `/help` | Show help information | `/help` |
| `/new` | Create new file | `/new python script.py` |
| `/edit` | Edit file | `/edit calculator.py` |
| `/run` | Run code | `/run python script.py` |
| `/test` | Run tests | `/test` |
| `/git` | Git operations | `/git status` |
| `/search` | Search code | `/search "function calculate"` |
| `/explain` | Explain code | `/explain calculator.py` |
| `/debug` | Debug code | `/debug error.py` |
| `/docs` | Generate documentation | `/docs generate` |

## Keyboard Shortcuts

| Shortcut | Function |
|----------|----------|
| `Ctrl+C` | Interrupt current operation |
| `Ctrl+D` | Exit Claude Code |
| `Ctrl+L` | Clear screen |
| `Tab` | Auto-complete |
| `↑/↓` | Browse command history |

## Best Practices

### 1. Clear Instruction Format
```
❌ Bad example:
"write code"

✅ Good example:
"Please create a Python function that takes two number parameters, returns their sum, and includes error handling"
```

### 2. Plan Before Execute
```
Recommended workflow:
1. "Help me plan the implementation steps for a user login system"
2. "Now let's implement step 1: create user model"
3. "Continue with step 2: create login interface"
```

### 3. Provide Context
```
✅ "Add a new API endpoint /users to the existing Flask application"
✅ "Write tests for calculator.py using pytest framework"
```

### 4. Iterative Improvement
```
1. "Create a simple calculator"
2. "Add square root functionality"
3. "Add history feature"
4. "Optimize the UI design"
```

### 5. Use Code Review
```
Regular use:
"Review the code I just wrote, check for potential issues"
"How's the performance of this code? Any optimization suggestions?"
```

## FAQ

### Q: What programming languages does Claude Code support?
A: Supports almost all mainstream languages including Python, JavaScript, Java, C++, Go, Rust, etc.

### Q: How large of a project can it handle?
A: Can handle projects of all sizes, from single files to large codebases.

### Q: How is my code security protected?
A: Claude Code runs locally, your code is not uploaded to the cloud (unless you explicitly request it).

### Q: What to do when encountering errors?
A: Simply tell Claude Code about the error you're facing, and it will help analyze and solve it.

### Q: What is the cost & rate‑limits?
A: Claude Code usage consumes tokens. Some high‑usage users have hit weekly caps even under the $200/month Max plan, and Anthropic is enforcing new limits beginning August 28, 2025

---

## Video Tutorial

[Introducing Claude Code](https://youtu.be/AJpK3YTTKZ4)

## Additional Resources

- 📚 [Official Documentation](https://docs.anthropic.com/claude-code)
- 💬 [Community Forum](https://community.anthropic.com)
- 🐛 [Report Issues](https://github.com/anthropics/claude-code/issues)
- 📧 [Contact Support](support@anthropic.com)

---

Remember: Claude Code is your programming partner, don't be afraid to try and ask questions! 🚀
