# 1. Mark the final 0.1 state
```
git tag -a v0.1-final -m "Final state of logger 0.1"
git push origin v0.1-final
```

# 2. Export the tracked files (submodule contents are left out)
```
mkdir ../logger_0.2
git archive HEAD | tar -x -C ../logger_0.2
cd ../logger_0.2
```
# 3. Start the new repo and re-add PAL at the same commit
```
git init -b main
rm .gitmodules && rmdir stm/stm_logger_0.1/PAL
git submodule add https://github.com/Rhygen-Tech/pal.git stm/stm_logger_0.1/PAL
git -C stm/stm_logger_0.1/PAL checkout 2252ca3
```

# 4. Drop the tracked log data
```
rm -r laptop/DG_Python_Scripts/decode/4B010EA4_logs
```

# 5. First commit, then publish
```
git add -A
git commit -m "Initial commit: logger 0.2, based on logger_0.1 @ v0.1-final"
gh repo create Rhygen-Tech/logger_0.2 --private --source=. --remote=origin --push
```

# 6. Turn the hook on only after the first push, or it will block that push to main
```
git config core.hooksPath .githooks
```
