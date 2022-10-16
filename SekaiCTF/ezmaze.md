# EZmaze

## The Challenge
We are given the source code of the challege:
```python
#!/usr/bin/env python3
import os
import random
from Crypto.Util.number import *

from flag import flag

directions = "LRUD"
SOLUTION_LEN = 64

def toPath(x: int):
	s = bin(x)[2:]
	if len(s) % 2 == 1:
		s = "0" + s

	path = ""
	for i in range(0, len(s), 2):
		path += directions[int(s[i:i+2], 2)]
	return path

def toInt(p: str):
	ret = 0
	for d in p:
		ret = ret * 4 + directions.index(d)
	return ret

def final_position(path: str):
	return (path.count("R") - path.count("L"), path.count("U") - path.count("D"))

class RSA():
	def __init__(self):
		self.p = getPrime(512)
		self.q = getPrime(512)
		self.n = self.p * self.q
		self.phi = (self.p - 1) * (self.q - 1)
		self.e = 65537
		self.d = pow(self.e, -1, self.phi)

	def encrypt(self, m: int):
		return pow(m, self.e, self.n)

	def decrypt(self, c: int):
		return pow(c, self.d, self.n)

def main():
	solution = "".join([random.choice(directions) for _ in range(SOLUTION_LEN)])
	sol_int = toInt(solution)
	print("I have the solution of the maze, but I'm not gonna tell you OwO.")

	rsa = RSA()
	print(f"n = {rsa.n}")
	print(f"e = {rsa.e}")
	print(hex(rsa.encrypt(sol_int)))

	while True:
		try:
			opt = int(input("Options: 1. Decrypt 2. Check answer.\n"))
			if opt == 1:
				c = int(input("Ciphertext in hex: "), 16)
				path = toPath(rsa.decrypt(c))
				print(final_position(path))
			elif opt == 2:
				path = input("Solution: ").strip()
				if path == solution:
					print(flag)
				else:
					print("Wrong solution.")
					exit(0)
			else:
				print("Invalid option.")
				exit(0)
		except:
			exit(0)

if __name__ == "__main__":
	main()
```
To summarize, a random two-dimensional walk of length 64 is generated.
The goal of the challenge is to guess the walk.
However, what you are given is an RSA-encrypted version of the walk, interpreted as a base 4 string.
That is left is 0, right is 1, up is 2 and down is 3.

Whatever we send to the challenge will be RSA-decrypted, and interpreted as a walk.
You are returned the coordinates of whatever position the walk ended up on.
The coordinates are calculated as follows: x is the number of right steps minus the number of left steps, likewise with y for up and down.


## The Solution

We can trivially send the encrypted walk we are given, and in return we get the x and y coordinates that result from it.
In other words: the number of ones minus the number of zeroes and the number of twos minus the number of threes in the base 4 number.
Moreover, similarly to a blinding attack, we can multiply the encrypted number by any constant.

$$ c = m^e $$

$$ 2^e \times c = 2^e \times m^e = (2m)^e $$

...and after decryption:

$$ c^d = m^{ed} = m $$

$$ (2m)^{ed} = 2m $$

And that's all the RSA knowledge you need!
The main insight we need to solve this challenge is that we can figure out how many digits are in `n * solution`.
How can we do that?
To start, we know the solution is 64 digits long.
In the case where it starts with `L` (0), we just start over. We don't take L's.
Now if we sum the x and y coordinates, we will always find that the sum is even.
This is because the number of digits is even, and the coordinates are simply adding and subtracting groups of digits.
Hence the parity of the result will be preserved.
Try playing around with it in your head or on paper, and you'll see how it works.

Now if we multiply the solution by 4, that's the same as adding a 0 to the end (like multiplying by 10 in base 10).
So `4 * solution` is guaranteed to be 65 digits, and we can verify that the x and y coordinates always sum to an odd number.
If we were to multiply by 16, we would be adding two zeros, making the number of digits even again.
So we know there is a number between 4 and 16 which is the smallest number such that the number of digits is 66.
Say it's 7, then we know:

$$ 7 \times solution > 4^{65} > 6 \times solution $$

This gives us an upper and lower bound for the solution by simple division.
We can use the same process but staring at say `4^10 * solution` to attain a much more accurate bound.
To find this number n, we could simply try multiplying with every number.
While this works for the case of 7 (we only need to test 5, 6, 7), with bigger numbers we should use binary search.

If we multiply the solution by a large enough number, the upper or lower bound will be so precise that the bound will be the solution.


## Full Code
```python
from pwn import remote

r = remote("challs.ctf.sekai.team", 3005)

response = r.recv()
n = int(response.decode().split()[16])
p = int(response.decode().split()[20], 16)


def to_path(a):
    s = bin(a)[2:]
    if len(s) % 2 == 1:
        s = "0" + s
    return "".join("LRUD"[int(s[i : i + 2], 2)] for i in range(0, len(s), 2))


def parity(a):
    r.sendline(b"1")
    r.recv()
    r.sendline(hex(p * pow(a, 65537, n))[2:].encode())
    return sum(eval(r.recv().decode().split("\n")[0])) % 2


x = 2**140
assert parity(2 * x) == 1
# 50% of the time, it works every time

num = x
while num > 1:  # binary search
    num //= 2
    if not parity(x + num):
        x += num
        print(x)
ans = to_path(4**134 // x)
print(ans)

r.sendline(b"2")
r.recv()
r.sendline(ans.encode())
print(r.recv().decode())
r.close()
```
