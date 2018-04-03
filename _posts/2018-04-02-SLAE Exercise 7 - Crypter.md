---
layout: post
title: SLAE Exercise 7 - Crypter
subtitle: Python Custom Crypter
tags: [SLAE]
---

Overview
======

The final exercise for the SLAE course was to create a custom crypter. The requirements were lenient with the following stated:

  • Free to use any existing encryption schema
  • Can use any language
  
Requirements
------

The requirement of this exercise was to write a custom crypter using any language. For this exercise, Python was used along with the '[pycrypto]('https://www.dlitz.net/software/pycrypto/') module. First of all, it needs to be understood what is required.

*What is a crypter?*

This will be done by using the 'pycrypto' module alongside python to implement AES encryption to encrypt our chosen shellcode. The module 'pycrypto'  is a collection of cryptography related tooling that can be used to encrypt data using python. 

*How will this be done?*

This will be done by using the 'pycrypto' module alongside python to implement AES encryption to encrypt our chosen shellcode. The module 'pycrypto'  is a collection of cryptography related tooling that can be used to encrypt data using python. 

*What encryption method will be used?* 

For this exercise, AES (Advanced Encryption Standard) is utilised to create the crypter. AES is a symmetric-key algorithm which means that the same key is used to encrypt and decrypt the data.  

**Crypter**

The crypter is split into two parts, an 'encrypter' that will encrypt the given shellcode with a user-chosen key and the 'decrypter' that will decrypt the encrypted program with the same user-chosen key. For this exercise, the stack based execve shelcode that has been used before will be used again to demonstrate the program. This can be seen here:

	'\x31\xc0\x50\x68\x62\x61\x73\x68\x68\x62\x69\x6e\x2f\x68\x2f\x2f\x2f\x2f\x89\xe3\x50\x89\xe2\x53\x89\xe1\xb0\x0b\xcd\x80'
	
Firstly, the program to encrypt the shellcode was created. This can be found fully commented within my Github repository, and can also be seen below:

```
from Crypto.Cipher import AES
import base64

key = raw_input('Please enter a key to encrypt the shellcode with: ')
Shellcode = '\x31\xc0\x50\x68\x62\x61\x73\x68\x68\x62\x69\x6e\x2f\x68\x2f\x2f\x2f\x2f\x89\xe3\x50\x89\xe2\x53\x89\\xe1\xb0\x0b\xcd\x80'
blockSize = 16
padding = '?'
	
pad = lambda x: x + (blockSize - len(x) % blockSize) * padding
b64encodeAES = lambda y, x: base64.b64encode(y.encrypt(pad(x)))

cipher = AES.new(key)
encoded = b64encodeAES(cipher, Shellcode)


print 'Encrypted String: ' + encoded
```

As can be seen in this example, where we have the shellcode as listed above and the key 'aaaabbbbccccdddd', the encrypted payload is:

![string](https://raw.githubusercontent.com/14Deep/14deep.github.io/master/_posts/Images/EX%207/string.png)

This encrypted payload needs to be decrypted too, for this the decryption program was written. 
This can be found within my Github repository, and can also be seen below:

```
from Crypto.Cipher import AES
import base64
	
encryptedShellcode = raw_input('Please enter the encrypted payload: ')	
key = raw_input('Please enter the key to decrypt the payload: ')

padding = '?'
b64decodeAES = lambda c, e: c.decrypt(base64.b64decode(e)).rstrip(padding)

cipher = AES.new(key)
decoded = b64decodeAES(cipher, encryptedShellcode)
	
decodeFormatted = ' '
for x in bytearray(decoded):
	decodeFormatted += '\\x'
	convert = '%02x' % (x & 0xff)
	decodeFormatted += convert

print decodeFormatted 
```
	
As can be seen, when this is run with the encrypted payload and the same key the shellcode is decrypted and outputted:

![working](https://raw.githubusercontent.com/14Deep/14deep.github.io/master/_posts/Images/EX%207/working.png)


This blog post has been created for completing the requirements of the SecurityTube Linux Assembly Expert Certification:

http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert

Student ID: 1092

