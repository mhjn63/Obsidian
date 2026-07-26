Before you touch a single command in this room, there's one mental model you need to lock in: Dirty Frag is not a new idea. It's the third vulnerability in a bug class that's been quietly recurring across the Linux kernel for years. Dirty Pipe abused pipe buffers. Copy Fail abused AF_ALG. Dirty Frag abuses network socket buffer fragments. Same underlying trick, different plumbing. If you've done the Copy Fail room, the moment you read the word "page cache write" you already know what's coming.

**The setup**

The room drops you into a terminal logged in as `karen` — a completely unprivileged user on Ubuntu 24.04. No sudo. No special groups. Nothing. Confirm this yourself first:

bash

```bash
id
# uid=1001(karen) gid=1001(karen) groups=1001(karen)
```

That's your starting point. Root is the destination. The exploit gets you there without touching a single file on disk.

**What you're actually exploiting**

The attack chain starts with one elegant abuse of how `splice()` works. When you splice data from a file into a socket, the kernel doesn't copy the file's contents — it hands the socket a direct reference to the file's page cache page. That's the whole point of zero-copy I/O. Performance win for legitimate use cases, catastrophic for security when downstream code forgets the page it's about to mutate isn't its own private memory.

Both Dirty Frag variants weaponize this. They plant a reference to a sensitive file's cached page into a network socket buffer's fragment slot, then trigger an in-place cryptographic operation that writes back into that same page. The file on disk? Completely untouched. The version in memory that the kernel loads for every `execve()`? Silently rewritten. Your file integrity monitoring tool checks disk. The kernel executes from memory. The gap between those two things is the entire exploit.

**Variant one — xfrm-ESP: 48 writes, one shellcode, nine years undetected**

The first path runs through the IPsec ESP input path (`esp_input()`) in the kernel's IPv4 and IPv6 networking code. The bug sat there since January 2017 — nine years — because the UDP datagram path never set the `SKBFL_SHARED_FRAG` flag that would have forced downstream code to copy splice'd pages into private memory before touching them.

The exploitation sequence is methodical. The exploit enters a user namespace to gain `CAP_NET_ADMIN`, registers an IPsec security association with a fully attacker-controlled `seq_hi` value, then fires 48 crafted UDP packets through a loopback ESPINUDP socket — each one splice'd from `/usr/bin/su`'s page cache at a different 4-byte offset. Each packet triggers the ESP in-place AEAD decrypt, which writes those controlled 4 bytes into the page cache at the target offset. After 48 passes, 192 bytes of a tiny static ELF have been assembled over the start of the cached `su` binary — an ELF whose entry point calls `setgid(0); setuid(0); execve("/bin/sh")`. The setuid bit on `/usr/bin/su` is still intact. Execute it and the kernel grants you UID 0.

On Ubuntu though — this path hits a wall. Ubuntu's AppArmor policy blocks unprivileged user namespace creation. `unshare(CLONE_NEWUSER)` returns `-EPERM` and the whole chain collapses before it starts. That's exactly why variant two exists.

**Variant two — RxRPC: three writes, an empty password field, five milliseconds**

The fallback lives in the RxRPC kernel module — the kernel implementation of the AFS RPC protocol. Same primitive, different crypto: an 8-byte in-place single-block decrypt using `pcbc(fcrypt)`. No namespace required. No special privileges. Just `add_key("rxrpc")` calls and an `AF_RXRPC` socket, both accessible to any unprivileged user.

The constraint here is that the 8 bytes written are `fcrypt_decrypt(C, K)` — where `C` is whatever ciphertext already lives at that file offset, and `K` is the session key you register. You control `K`, not the output directly. To land a specific byte pattern you brute-force `K` in user space until the decryption produces what you want.

Writing 192 bytes of shellcode via brute-force is infeasible. So the RxRPC variant targets `/etc/passwd` instead. The first line is `root:x:0:0:root:/root:/bin/bash`. Three overlapping 8-byte writes at offsets 4, 6, and 8 overwrite bytes 4 through 15, clearing the password field. Result: `root::0:0:GGGGGG:/root:/bin/bash`. PAM's `nullok` option sees an empty password field and returns success without prompting. Run `su -` and you drop straight to UID 0 with no authentication challenge. The three brute-force operations complete in 4.7ms, 5.1ms, and around one second. Start to root in under two seconds total.

**Building and running the exploit**

The PoC is already on the machine at `/home/karen/dirtyfrag/exp.c`. Build it:

bash

```bash
cd /home/karen/dirtyfrag
gcc -O0 -Wall -o exp exp.c -lutil
./exp
```

On Ubuntu 24.04 with standard AppArmor policy, the output walks you through exactly what happens:

bash

```bash
[+] xfrm-ESP variant: unshare(CLONE_NEWUSER) -> EPERM (AppArmor)
[+] falling back to RxRPC variant
[+] brute forcing K_A for offset 4..11   "::"      ...done in 4.7ms
[+] brute forcing K_B for offset 6..13   "0:"      ...done in 5.1ms
[+] brute forcing K_C for offset 8..15   "0:GGG..."...done in 1.04s
[+] splice trigger A: ok
[+] splice trigger B: ok
[+] splice trigger C: ok
[+] /etc/passwd line 1 modified, calling su
# id
uid=0(root) gid=0(root) groups=0(root)
```

**The flag and the forensics check**

bash

```bash
cat /root/flag.txt
```

Before you exit the root shell, run the integrity check the room asks for:

bash

```bash
sha256sum /etc/passwd /usr/bin/su
```

Both hashes match a freshly installed system. The page cache was rewritten. The disk was never touched. AIDE, Tripwire, and IMA would all report both files as clean. This is the whole point — detection has to happen at the syscall level in real time, not from disk-based integrity tooling after the fact.

Once you exit the root shell, the exploit calls `posix_fadvise(POSIX_FADV_DONTNEED)` on the affected pages, the kernel evicts them, and the next read of either file reloads the clean copy from disk. No forensic trace. If you didn't catch it happening, it's already gone.

**Applying the mitigation**

The room walks through the interim fix — blocking both attack surfaces by denylisting the modules that deliver them:

bash

```bash
sudo sh -c 'printf "install esp4 /bin/false\ninstall esp6 /bin/false\ninstall rxrpc /bin/false\n" > /etc/modprobe.d/dirtyfrag.conf'
sudo rmmod esp4 esp6 rxrpc 2>/dev/null
sudo sh -c 'echo 3 > /proc/sys/vm/drop_caches'
```

Rerun the exploit. Both paths now fail at the `socket()` call with `EAFNOSUPPORT` — the modules that handle those socket families no longer load. The xfrm-ESP patch is already upstream as of May 8, 2026. The RxRPC patch has no merged fix in any tree at time of disclosure — the module denylist is your only interim control.

**What this room teaches that other rooms don't**

Most privilege escalation rooms hand you a misconfigured sudo entry or a SUID binary with a known CVE. Dirty Frag is different. There's no misconfiguration. There's no obvious attack surface. You're exploiting a fundamental architectural assumption — that a kernel subsystem performing in-place crypto on a network buffer would never be handed a page it doesn't own. That assumption was wrong in 2017, it was still wrong in 2023, and the reason it keeps reappearing is that the performance motivation behind zero-copy I/O keeps creating new instances of the same pattern across different subsystems.

File integrity monitoring isn't sufficient on its own. Real-time syscall visibility — watching for `splice()` from setuid binaries into network sockets, watching for `AF_RXRPC` socket creation outside AFS client processes — is the only detection layer that actually works here.