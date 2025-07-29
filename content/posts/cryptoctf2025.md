+++
title = "Crypto CTF 2025"
date = '2025-07-15T04:58:38+05:30'
cover = ""
coverCaption = ""
tags = ["crypto", "ctf"]
keywords = ["cryptography", "ctf","lattices"]
description = "Writeups for some of the CryptoCTF 2025 challenges I've solved"
showFullContent = false
readingTime = true
hideComments = false
color = "green" #color from the theme settings
+++

# Vinad

We know that one of the primes used for RSA encryption is `vinad(r,R)` where r is a random 512 bit number and R is a 512 length array of 512 bit random numbers.

The `vinad` function generates a bitstring by concatenating the output of `parinad(x^r)` for each element x of R and it converts this to an integer. The `parinad` function outputs 0 if a number contains even number of 1 bits or 1 if its odd. An important observation is `parinad(a^b) = parinad(a) ^ parinad(b)`

Since `parinad(x)` can only be either 0 or 1, the prime p has only two possible values `vinad(0,R)` or `vinad(1,R)`. Even the value of e has the same two possible values.

```python
from Crypto.Util.number import *

def parinad(n):
    return bin(n)[2:].count('1') % 2

def vinad(x, R):
    return int(''.join(str(parinad(x ^ r)) for r in R), 2)

f = open('output.txt','r')
R = eval(f.readline().strip().split('=')[1])
n = eval(f.readline().strip().split('=')[1])
c = eval(f.readline().strip().split('=')[1])

#assert isPrime(vinad(0,R))
assert isPrime(vinad(1,R))
p = vinad(1,R)
assert n%p == 0
q = n//p
d = inverse(p,(p-1)*(q-1))
m = long_to_bytes((pow(c,d,n)-sum(R))%n)

if b'CCTF{' in m:
    print(m)
else:
    d = inverse(vinad(0,R),(p-1)*(q-1))
    m = long_to_bytes((pow(c,d,n)-sum(R))%n)
```

flag: `CCTF{s0lV1n9_4_Syst3m_0f_L1n3Ar_3qUaTi0n5_0vEr_7H3_F!3lD_F(2)!}`

# Interpol

For each character in the flag, the source scripts generates a point (x,y) of the form `[(-(1 + (19*n - 14) % len(flag)), ord(flag[(63 * n - 40) % len(flag)]))]`. It also randomly creates noise points. Then it forms a lagrange polynomial with all these points and dumps it. 

We do not know the length of the flag, so we brute the flag length, try to generate flag characters using the polynomial, if every character is valid(for a given x < flag length y = p(x) is an integer), then our flag length is right and the flag we generated is correct.

```python
from sage.misc.persist import loads

p = loads(open('output.raw', 'rb').read())

for length in range(10, 100):
    flag_cand = [None] * length
    valid = True
    for n in range(length):
        x = -(1 + (19*n - 14) % length)
        y = p(x)
        if not y.is_integer():
            valid = False
            break
        original_index = (63 * n - 40) % length
        flag_cand[original_index] = chr(int(y))
    if valid:
        if None not in flag_cand:
            candidate_flag = "".join(flag_cand)
            if candidate_flag.startswith("CCTF{") and candidate_flag.endswith("}"):
                print(candidate_flag)
```

flag: `CCTF{7h3_!nTeRn4t10naL_Cr!Min41_pOlIc3_0r9An!Zati0n!}`

# Toffee

The flag is stored as the secret key used during ECDSA signing. The nonce during the signing process is generated using an LCG where the seed is the nonce generated for the previous signature generated, this is something we can exploit. We can also enter our own seeds to see the output of the LCG function `toffee` that outputs `(u*k+v)%_n` where k is the seed. If we give a seed of 0, we can recover v and a seed of 1 gives of u+v from which we get u. 

On a closer look at the signing process each time we sign a message, we see that we currently have two unknowns skey and k.
`s = inverse(k, _n) * (h + r * skey) % _n`

Since two consecutive nonces are related like mentioned above, if we sign a message(or different messages) twice, we have two equations and two unknowns which is easily solvable.

```
(s1*k1 -h)*r1_inv - (s2*(u*k1+v)-h)*r2_inv = 0 
s1*k1*r1_inv - h*r1_inv - s2*u*k1*r2_inv -s2*v*r2_inv + h*r2_inv = 0
k1*(s1*r1_inv -s2*u*r2_inv) = h*(r1_inv-r2_inv) + s2*v*r2_inv 
```
From this we can recover k1, and therefore skey.

```python
from pwn import *
from hashlib import sha512
from Crypto.Util.number import *

p = 0xaeaf714c13bfbff63dd6c4f07dd366674ebe93f6ec6ea51ac8584d9982c41882ebea6f6e7b0e959d2c36ba5e27705daffacd9a49b39d5beedc74976b30a260c9
a, b = -7, 0xd3f1356a42265cb4aec98a80b713fb724f44e747fe73d907bdc598557e0d96c5
_n = 0xaeaf714c13bfbff63dd6c4f07dd366674ebe93f6ec6ea51ac8584d9982c41881d942f0dddae61b0641e2a2cf144534c42bf8a9c3cb7bdc2a4392fcb2cc01ef87
x = 0xa0e29c8968e02582d98219ce07dd043270b27e06568cb309131701b3b61c5c374d0dda5ad341baa9d533c17c8a8227df3f7e613447f01e17abbc2645fe5465b0
y = 0x5ee57d33874773dd18f22f9a81b615976a9687222c392801ed9ad96aa6ed364e973edda16c6a3b64760ca74390bb44088bf7156595f5b39bfee3c5cef31c45e1

r = remote('91.107.188.9',31111)

r.sendline(b'g')
r.sendline(b'0')
r.recvuntil(b'toffee(u, v, _k) = ')
v = int(r.recvline().strip())

r.sendline(b'g')
r.sendline(b'1')
r.recvuntil(b'toffee(u, v, _k) = ')
u = (int(r.recvline().strip()) - v)%_n

r.sendline(b's')
r.sendline(b'e')
r.recvuntil(b' = ')
r1 = int(r.recvline().strip())
r.recvuntil(b' = ')
s1 = int(r.recvline().strip())

r.sendline(b's')
r.sendline(b'e')
r.recvuntil(b' = ')
r2 = int(r.recvline().strip())
r.recvuntil(b' = ')
s2 = int(r.recvline().strip())

h = bytes_to_long(sha512(b'e').digest())

r1_inv = inverse(r1,_n)
r2_inv = inverse(r2,_n)

k1 = (((h*(r1_inv-r2_inv) + s2*v*r2_inv)%_n)*inverse((s1*r1_inv-s2*u*r2_inv)%_n,_n))%_n
skey = ((s1*k1 - h)*r1_inv)%_n
print(long_to_bytes(skey))
```

flag: `CCTF{4fFin3Ly_r3lA7eD_n0nCE5_aR3_!n5eCuR3!}`

# Silky

Essentially two different arrays are generated: key which is a 19 element array of random integers ranging from `-5 to 5` and R an array of many 19 element arrays containing integers ranging from `550 to -550`. One key point is that for each element of an array in R, `R[i]-key[i]` is within the bounds -550 to 550. To solve this I found the minimum and maximum values at a particular index in each array, and found their average which gave the key element at that index. This is because for the condition to hold true with R containing a lot of array samples, the mean of min and max should be centered around the actual key value.

```python
from pwn import *
import math
import re

r = remote("91.107.252.0", 31131)
min_vals = [math.inf] * 19
max_vals = [-math.inf] * 19

r.recvuntil(b"[Q]uit")

for i in range(10):
    r.sendline(b'm')
    response = r.recvuntil(b"[Q]uit", timeout=10).decode(errors = 'ignore')
    for line in response.splitlines():
        if line.strip().startswith('\u2503'):
            line = line.strip().lstrip('\u2503').strip()
            numbers = [int(n) for n in re.findall(r'-?\d+', line)]
            for j in range(0, len(numbers), 19):
                vector = numbers[j:j + 19]
                if len(vector) == 19:
                    for k in range(19):
                        if vector[k] < min_vals[k]:
                            min_vals[k] = vector[k]
                        if vector[k] > max_vals[k]:
                            max_vals[k] = vector[k]

key = []
for i in range(19):
    key.append(round((min_vals[i] + max_vals[i]) / 2.0))

key = ",".join([str(k) for k in key])

r.sendline(b'g')
r.recvuntil(b'Please submit the secret key:')
r.sendline(key.encode())
r.recvline()
print(r.recvline())
```

flag: `CCTF{k3Y_R3c0vEry_4TtaCk_On_A_l3AkY_5e4Si9n!}`