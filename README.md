Dirtycow

Original code found at: https://www.exploit-db.com/exploits/40616/

This really just has customized payloads.

Custom payloads put the original binary used for the exploit back (since, if this isn't done and 
the original binary is called by a system process, the system usually hangs...).

Kernel ver: 3.0.x to 4.9.x

Custom payloads (32 and 64 bit x86):

- Basic interactive shell.
- Run /tmp/pl.sh as root (you make this file).
