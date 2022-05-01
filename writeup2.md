# Change init script at booting

We can make the ISO boot using `"/bin/sh"` using the command `live /boot/ init=/bin/sh`. This can be done when we start the VM,
and we type the combination `control` + `alt` + `F1`
```bash
boot: live /boot/ init=/bin/sh
error: unexpectedly disconnected from boot status daemon
[...]
/bin/sh: 0: can't access tty: job control turned off
# whoami
root
# cd /root
README
# cat README
CONGRATULATIONS !!!!
To be continued...
```
