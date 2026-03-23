Run the full test suite and report results.

Steps:
1. Run: `python -m unittest discover -s test -t . -v 2>&1 | tail -20`
2. Report: number of tests run, pass/fail count, any failures with details
3. If any tests fail, read the failing test file and the source file it tests, then diagnose the root cause
