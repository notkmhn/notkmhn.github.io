---
author: kmhn
date: 2021-08-21
description: While writing an exploit for a Java EL injection vulnerability on a particularly
  weird setup using Java 7, I needed to find a way to encode Java class bytecode...
lastmod: 2021-08-21
layout: post
tags:
- java
- short
title: Java 7+ - using a BigInteger for (limited) text to binary decoding
---

While writing an exploit for a [Java EL injection](https://owasp.org/www-community/vulnerabilities/Expression_Language_Injection) vulnerability on a particularly weird setup using Java 7, I needed to find a way to encode Java class bytecode as text and decode it on the vulnerable machine for an exploit chain. I couldn't find any of the usual suspect classes to do base64 encoding or other options available on the classpath of the target.

Instead, I've found that in this particular scenario, it was much easier to use `java.math.BigInteger`. Granted, while it doesn't work for the conventional text to binary decoding scenario, it works quite well when you have control over both ends of the line. That being said, I would definitely use a more standard solution for general development. There are some limitations including the lack of handling for leading zero bytes, which I didn't have to deal with since Java classes always start with the magic sequence `0xCAFEBABE`.

Here's a Python implementation for the sending side. Note that it's necessary to make sure that the data is interpreted as a signed integer, because `java.math.BigInteger#toByteArray` will [return the sign-extended 2's complement representation](https://docs.oracle.com/javase/7/docs/api/java/math/BigInteger.html#toByteArray()).
```python
# From https://stackoverflow.com/a/1181922
def b36encode(number, alphabet='0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ'):
    """Converts an integer to a base36 string."""
    if not isinstance(number, int):
        raise TypeError('number must be an integer')

    base36 = ''
    sign = ''

    if number < 0:
        sign = '-'
        number = -number

    if 0 <= number < len(alphabet):
        return sign + alphabet[number]

    while number != 0:
        number, i = divmod(number, len(alphabet))
        base36 = alphabet[i] + base36

    return sign + base36

def encode(data: bytes) -> str:
    datalong = int.from_bytes(data, byteorder='big', signed=True)
    return b36encode(datalong).lower()
```
Here are a couple of examples, first we encode some data:
```python
>>> encode([1, 2, 3, 4])
'a2f44'
>>> encode([240, 253, 4, 16])
'-45y3n4'
```

And we decode them by using the appropriate constructor, then invoking `java.math.BigInteger#toByteArray`:
```java
public static void main(String[] args) throws IOException {
  System.out.println((Arrays.toString(new BigInteger("a2f44", 36).toByteArray())));
  // [1, 2, 3, 4]
  System.out.println((Arrays.toString(new BigInteger("-45y3n4", 36).toByteArray())));
  // [-16, -3, 4, 16]
}
```