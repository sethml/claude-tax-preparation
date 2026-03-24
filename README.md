# Tax Preparation Skill for Claude Code

Just tell Claude:

```text
Install and use https://github.com/alqz/claude-tax-preparation
```

This Skill helps Claude prepare your tax return: read the source docs, compute the return, fill the official PDF forms, and hand back a clear summary of what to review before you file.

## What It Does

- Reads W-2s, 1099s, brokerage statements, prior-year returns, and other source documents
- Researches current-year tax law, brackets, and thresholds from government sources
- Asks follow-up questions based on the forms and your documents — not a generic checklist
- Computes federal and state taxes in a structured workbook (all math in Python, never LLM reasoning)
- Downloads official blank PDF forms and fills them programmatically
- Validates results against form instructions and cross-checks internal consistency
- Checks for other obligations (FBAR, FATCA, gift tax, estimated payments, etc.)
- Presents a summary with filled PDFs, carryforwards, and a checklist of what to sign and file

## What It Looks Like

Start with a simple prompt:

![Starting prompt in Claude](docs/images/initial-prompt.png)

Claude asks the follow-up questions needed to finish the return:

![Filing questions UI](docs/images/ask-questions.png)

It works through the preparation steps and keeps track of progress:

![Workflow progress](docs/images/todo.png)
