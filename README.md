# OverTheWire — Bandit Writeup

My walkthrough of all 34 levels (0 → 33) of [OverTheWire: Bandit](https://overthewire.org/wargames/bandit), the wargame that teaches the Linux command-line fundamentals every security practitioner needs.

**A note on passwords:** Following OverTheWire's own guidance, I don't publish level passwords — that spoils the challenge for others. This writeup documents the method and commands for each level. Run them yourself to recover the password and advance.

## How to connect

Every level is reached over SSH on port 2220. Once you have the password for `banditN`, log in with:

```bash
ssh bandit<N>@bandit.labs.overthewire.org -p 2220
```

The starting password for `bandit0` is `bandit0`. Each level reveals the password for the next.

## Levels

### Level 0 → 1 — Read a file

The password sits in a file called `readme` in the home directory.

```bash
cat readme
```

### Level 1 → 2 — A file named `-`

A leading dash is read as an option flag, so reference it by path.

```bash
cat ./-
```

### Level 2 → 3 — Spaces in the filename

Escape the spaces or quote the name.

```bash
cat "spaces in this filename"
```

### Level 3 → 4 — Hidden file

The file is dot-hidden inside `inhere/`.

```bash
ls -la inhere/
cat inhere/...Hiding-From-You
```

### Level 4 → 5 — The only human-readable file

Of many files in `inhere/`, only one is ASCII text.

```bash
file ./inhere/*
cat ./inhere/-file07   # whichever one is "ASCII text"
```

### Level 5 → 6 — Find by file properties

The target is 1033 bytes, not executable, human-readable.

```bash
find inhere/ -size 1033c ! -executable
```

### Level 6 → 7 — Find across the whole server

Owned by user `bandit7`, group `bandit6`, 33 bytes.

```bash
find / -user bandit7 -group bandit6 -size 33c 2>/dev/null
```

### Level 7 → 8 — grep for a keyword

The password is on the line next to the word `millionth`.

```bash
grep millionth data.txt
```

### Level 8 → 9 — The line that occurs only once

```bash
sort data.txt | uniq -u
```

### Level 9 → 10 — Readable strings near `=`

```bash
strings data.txt | grep '==='
```

### Level 10 → 11 — Base64 decode

```bash
base64 -d data.txt
```

### Level 11 → 12 — ROT13

Rotate letters 13 places with `tr`.

```bash
cat data.txt | tr 'A-Za-z' 'N-ZA-Mn-za-m'
```

### Level 12 → 13 — Repeatedly compressed hexdump

Reverse the hexdump, then peel back each layer of compression (gzip, bzip2, tar…).

```bash
mkdir /tmp/work && cp data.txt /tmp/work && cd /tmp/work
xxd -r data.txt > data.bin
# Inspect with `file`, then decompress accordingly, repeating until plaintext:
file data.bin   # e.g. gzip -> `zcat`, bzip2 -> `bzcat`, tar -> `tar xf`
```

This one teaches you to identify file types with `file` and chain decompression tools until you hit the password.

### Level 13 → 14 — Log in with an SSH key

No password this time — a private key is provided.

```bash
ssh -i sshkey.private bandit14@localhost -p 2220
```

### Level 14 → 15 — Submit a password over a raw socket

Send the current level's password to a local service on port 30000.

```bash
echo "<bandit14 password>" | nc localhost 30000
```

### Level 15 → 16 — Submit over SSL

Same idea, but the service on port 30001 speaks TLS.

```bash
echo "<bandit15 password>" | openssl s_client -connect localhost:30001 -quiet
# or: ncat --ssl localhost 30001
```

### Level 16 → 17 — Port scan, then talk SSL to the right port

One port in 31000–32000 returns a private key over SSL.

```bash
nmap -p 31000-32000 localhost          # find open ports
nmap -sV -p 31000-32000 localhost      # identify which speak SSL
echo "<bandit16 password>" | openssl s_client -connect localhost:<port> -quiet
```

Save the returned key to a file, `chmod 600` it, and SSH in as bandit17.

### Level 17 → 18 — Diff two files

The new password is the changed line between `passwords.old` and `passwords.new`.

```bash
diff passwords.old passwords.new
```

### Level 18 → 19 — `.bashrc` logs you out on login

Run a command non-interactively so the logout shell never gets the chance.

```bash
ssh bandit18@bandit.labs.overthewire.org -p 2220 -t "cat readme"
```

### Level 19 → 20 — setuid binary

`bandit20-do` runs commands as bandit20.

```bash
./bandit20-do cat /etc/bandit_pass/bandit20
```

### Level 20 → 21 — setuid binary + your own listener

`suconnect` connects to a port you choose; feed it the current password and it returns the next.

```bash
# terminal 1 — listen and send bandit20's password:
echo "<bandit20 password>" | nc -lvp 4444
# terminal 2 — tell suconnect to connect to that port:
./suconnect 4444
```

### Level 21 → 22 — A cron job

A scheduled job writes the password to a known file.

```bash
cat /etc/cron.d/cronjob_bandit22
cat /usr/bin/cronjob_bandit22.sh   # see where it writes
cat /tmp/<file-it-writes>
```

### Level 22 → 23 — Cron job with a computed filename

The job derives the filename from the username via md5.

```bash
cat /usr/bin/cronjob_bandit23.sh                 # understand the logic
echo I am user bandit23 | md5sum | cut -d ' ' -f 1
cat /tmp/<that-hash>
```

### Level 23 → 24 — Cron runs scripts from a writable directory

Drop a script that copies bandit24's password somewhere world-readable, then wait for cron to execute it as bandit24.

```bash
mkdir -p /tmp/mywork && cd /tmp/mywork
cat > exploit.sh <<'EOF'
#!/bin/bash
cat /etc/bandit_pass/bandit24 > /tmp/mywork/pw.txt
EOF
chmod 777 exploit.sh pw.txt 2>/dev/null; touch pw.txt; chmod 666 pw.txt
cp exploit.sh /var/spool/bandit24/foo/
# wait ~1 min, then:
cat /tmp/mywork/pw.txt
```

### Level 24 → 25 — Brute-force a 4-digit PIN

A daemon on port 30002 wants `<password> <PIN>`; try all 10,000 PINs.

```bash
for pin in {0000..9999}; do
  echo "<bandit24 password> $pin"
done | nc localhost 30002 | grep -v "Wrong"
```

The one line that isn't "Wrong! Please enter the correct current password and pin code." contains the next password.

### Level 25 → 26 — Escape a restricted pager

bandit26's shell runs `more` on a text file and exits. Shrink your terminal window so `more` is forced into interactive mode, then:

- press `v` to open `vi`
- in vi: `:set shell=/bin/bash` then `:shell`

You now have a real shell as bandit26.

```bash
ssh -i bandit26.sshkey bandit26@localhost -p 2220
# (shrink the window first so `more` paginates)
```

### Level 26 → 27 — setuid binary from the vi shell

From the shell you just spawned:

```bash
./bandit27-do cat /etc/bandit_pass/bandit27
```

### Level 27 → 28 — Clone a git repo

```bash
git clone ssh://bandit27-git@localhost:2220/home/bandit27-git/repo /tmp/repo27
cat /tmp/repo27/README.md
```

### Level 28 → 29 — Password in git history

It was committed, then redacted — check earlier commits.

```bash
git clone ssh://bandit28-git@localhost:2220/home/bandit28-git/repo /tmp/repo28
cd /tmp/repo28 && git log -p   # read the pre-redaction commit
```

### Level 29 → 30 — Password on another branch

```bash
cd /tmp/repo29 && git branch -a
git checkout dev && cat README.md
```

### Level 30 → 31 — Password in a git tag

```bash
cd /tmp/repo30 && git tag
git show secret
```

### Level 31 → 32 — Push to the repo

The server reveals the password when you push a file it expects, even past `.gitignore`.

```bash
cd /tmp/repo31
echo 'May I come in?' > key.txt
git add -f key.txt
git commit -m "add key"
git push origin master   # password is returned in the push output
```

### Level 32 → 33 — The "uppercase shell"

The shell uppercases everything you type, which breaks commands. `$0` expands to the shell program name and isn't affected, so it drops you into a normal shell.

```bash
$0
cat /etc/bandit_pass/bandit33
```

### Level 33 → 34 — The end

`cat README.txt` — there's no level 34 yet. You're done. 🎉

## Skills this wargame builds

- **File hunting & inspection:** `find` with predicates, `file`, `strings`, `grep`, `sort`/`uniq`
- **Encodings & compression:** base64, ROT13, hexdump reversal, layered decompression
- **Networking:** `nc`, SSL clients (`openssl s_client`, `ncat`), port scanning with `nmap`
- **Access control:** SSH keys, setuid binaries, privilege boundaries
- **Automation:** bash loops and brute-forcing, understanding cron jobs
- **Version control forensics:** recovering secrets from git history, branches, tags, and pushes
- **Shell escapes:** breaking out of restricted pagers and sanitised shells

---

Part of my [OverTheWire](https://overthewire.org/wargames/) series. Writeups for Natas, Leviathan, Krypton, and Narnia to follow.
