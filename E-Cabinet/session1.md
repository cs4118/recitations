Cabinet Inspector Session 1
===========================

The following shell session inspects the virtual addresses of a shared library
in two daemon processes.

Note that this shell session was run as root (hence the `#` terminal prompt).

--------

Look for PIDs for `NetworkManager` and `sshd`:

```
# ps aux | grep '/usr/sbin/' | grep -E 'NetworkManager|sshd'
root       497  0.0  0.4 406400 15788 ?        Ssl  15:57   0:00 /usr/sbin/NetworkManager --no-daemon
root       556  0.0  0.1  15852  6952 ?        Ss   15:57   0:00 /usr/sbin/sshd -D
```


First look at virtual memory mapped into `NetworkManager` (497) with arbitrary offset:

```
# cat /proc/497/maps | grep 'libc-'
7f622ab5c000-7f622ab7e000 r--p 00000000 08:01 3146365                    /usr/lib/x86_64-linux-gnu/libc-2.28.so
7f622ab7e000-7f622acc6000 r-xp 00022000 08:01 3146365                    /usr/lib/x86_64-linux-gnu/libc-2.28.so
7f622acc6000-7f622ad12000 r--p 0016a000 08:01 3146365                    /usr/lib/x86_64-linux-gnu/libc-2.28.so
7f622ad12000-7f622ad13000 ---p 001b6000 08:01 3146365                    /usr/lib/x86_64-linux-gnu/libc-2.28.so
7f622ad13000-7f622ad17000 r--p 001b6000 08:01 3146365                    /usr/lib/x86_64-linux-gnu/libc-2.28.so
7f622ad17000-7f622ad19000 rw-p 001ba000 08:01 3146365                    /usr/lib/x86_64-linux-gnu/libc-2.28.so
# ./cabinet_inspector 7f622ab5c069 497
inspecting cabinet for 0x7f622ab5c069 of pid=497

[405] inspect_cabinet():
paddr: 0x115a3c069
pf_paddr: 0x115a3c000
pte_paddr: 0x10d2c7000
pmd_paddr: 0x10d623000
pud_paddr: 0x10c215000
p4d_paddr: 0x10c215000
pgd_paddr: 0x10c90a000
dirty: no
refcount: 57

[proc] pagemap:
paddr: 0x115a3c069
pf_paddr: 0x115a3c000
exclusively mapped: no
file-page: yes
swapped: no
present: yes
```

Then look at virtual memory mapped into `sshd` (556) with a different arbitrary
offset:
```
# cat /proc/556/maps | grep 'libc-'
7f0e57aa1000-7f0e57ac3000 r--p 00000000 08:01 3146365                    /usr/lib/x86_64-linux-gnu/libc-2.28.so
7f0e57ac3000-7f0e57c0b000 r-xp 00022000 08:01 3146365                    /usr/lib/x86_64-linux-gnu/libc-2.28.so
7f0e57c0b000-7f0e57c57000 r--p 0016a000 08:01 3146365                    /usr/lib/x86_64-linux-gnu/libc-2.28.so
7f0e57c57000-7f0e57c58000 ---p 001b6000 08:01 3146365                    /usr/lib/x86_64-linux-gnu/libc-2.28.so
7f0e57c58000-7f0e57c5c000 r--p 001b6000 08:01 3146365                    /usr/lib/x86_64-linux-gnu/libc-2.28.so
7f0e57c5c000-7f0e57c5e000 rw-p 001ba000 08:01 3146365                    /usr/lib/x86_64-linux-gnu/libc-2.28.so
# ./cabinet_inspector 7f0e57aa1420 556
inspecting cabinet for 0x7f0e57aa1420 of pid=556

[405] inspect_cabinet():
paddr: 0x115a3c420
pf_paddr: 0x115a3c000
pte_paddr: 0x10df82000
pmd_paddr: 0x10df95000
pud_paddr: 0x10dc25000
p4d_paddr: 0x10dc25000
pgd_paddr: 0x10dc32000
dirty: no
refcount: 57

[proc] pagemap:
paddr: 0x115a3c420
pf_paddr: 0x115a3c000
exclusively mapped: no
file-page: yes
swapped: no
present: yes
```

Note that although we used two different virtual addresses for `libc` in both
`NetworkManager` and `sshd`, they are mapped to the same physical page frame
(check `pf_paddr`)!
