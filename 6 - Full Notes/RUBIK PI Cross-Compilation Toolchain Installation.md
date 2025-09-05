

## error - During a Yocto build for Rubik Pi on modern systems (e.g., Ubuntu 24.04 with Linux 6.6+), you may encounter:

text

CopyEdit

`PermissionError: [Errno 1] Operation not permitted  Traceback (most recent call last):   File ".../bitbake-worker", line 268, in child     bb.utils.disable_network(uid, gid)   File ".../bb/utils.py", line 1653, in disable_network     with open("/proc/self/uid_map", "w") as f: PermissionError: [Errno 1] Operation not permitted`

The build then fails during tasks like `do_rm_work`.

## Cause - Increased Security restrictions in Ubuntu
Modern Linux distributions (e.g., Ubuntu 24.04, Kernel 6.6+) have increased security restrictions:

- By default, **unprivileged user namespaces are disabled**, preventing processes from writing to `/proc/self/uid_map`.
    
- BitBake uses `disable_network()` to sandbox tasks, which fails if user namespaces aren't allowed.