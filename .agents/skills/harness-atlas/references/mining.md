# Mining commands

Prompt files:
```bash
git ls-files | grep -E '\.(txt|md|prompt)$' | grep -iE 'prompt|tool|agent|system|instruction'
git ls-files | xargs grep -lE 'You are |IMPORTANT:|NEVER ' 2>/dev/null   # inline prompts in code
```

Most-edited prompt files:
```bash
git log --format= --name-only -- <files> | sort | uniq -c | sort -rn | head -20
```

History per file (follow renames; skip whitespace):
```bash
git log -p --follow -w --format='=== %h %ad %s' --date=short -- <file>
```

Fix commits:
```bash
git log --oneline -i --grep=compaction --grep=cache --grep=truncat --grep=overflow \
  --grep=retry --grep=abort --grep=interrupt --grep=doom --grep=loop --grep=reasoning \
  --grep='tool call' --grep=permission | head -300
```

Removed tools / features:
```bash
git log --diff-filter=D --name-only --format='=== %h %s' -- '*tool*' | head -100
```

Constants:
```bash
git grep -nE '\b[A-Z][A-Z0-9_]{3,}\s*[:=]\s*[0-9][0-9_.*]*' -- '*.ts' '*.py' '*.rs' '*.go' | grep -v test
```

Absences:
```bash
git log --oneline -i --grep=embedding --grep=rag --grep=vector --grep='repo map' --grep=sandbox --grep=planner
```
Then check README, specs/, docs/, CONTRIBUTING, issue discussions for rationale.
