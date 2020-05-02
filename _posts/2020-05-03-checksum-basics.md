---
title: Checksum basics
tags: checksum
categories: io
---

Checksum is a small-sized datum derived from a block of data to validate data integrity:
1. during network transition, eg. download files and check its md5
2. for on disk files, eg. storage failure or file damaged



~~~java

                <-----------   Code Word  --------->
                +----------------+-----------------+
                |   Data Word    |  Code Sequence  |
                +----------------+-----------------+
~~~

* *Data Word*: the data you want to protect
* *Check Sequence*: the result of checksum calculation
* *Code Word*: Data Word with Check Sequence Appended

*Data word* is considered error-free if checksum calculated based on *data word* matches the *code sequence* from *code word*.


# Checksum algorithm

To evaluate a good checksum algorithm, we need to know how many bit erros it can detect with fixed *data word* size.

## Bit Parity


~~~java
        int data = 100;
        int parity = 0;

        while (data != 0)
        {
            parity ^= (data & 1);
            data >>>= 1;
        }

/*
        data=100 -> binary=1100100                 -> parity=1
        data=101 -> binary=1100101 (one bit flip)  -> parity=0  detected
        data=97  -> binary=1100001 (two bit flips) -> parity=1  undetected
*/
~~~

Simply do a XOR on all bits from the *data word*, the result is either 0 or 1.

This allows to catch all *odd* bit flip, but fails to catch all *even* bit flips, as the result of XOR will be the same when *even* number of bits are flipped.

## LRC

~~~java
        byte[] data = "jasonstack".getBytes();
        byte lrc = 0;

        for (byte b : data)
            lrc ^= b;

/*
        data=stack  -> lrc=1101110

                1110011
              ^ 1110100
              ^ 1100001
              ^ 1100011
              ^ 1101011 
            -----------
              = 1101110

        data=stock  -> lrc=1100000
*/
~~~

LRC, aka *Longitudinal Redundancy Check*, works by computing XOR on all bytes in the *data word* to create a one byte checksum. It can be considered at computing parity at each bit position in the byte.

* catches all odd numbers of bit erros in the same bit positions, but similar to parity, it fails to catch all even numbers of bit erros in the same bit positions.
* catches 1 bit error and all erros in one byte

## Integer Addition Checksum

~~~java
        byte[] data = "stack".getBytes();

        long checksum = 0;
        for (byte b : data)
            checksum += b;

        // truncate to byte
        byte result = (byte) (checksum & 0xFF);

/*
        data=stack  -> checksum=10110

                  1110011
                + 1110100
                + 1100001
                + 1100011
                + 1101011
               -----------
            =  1000010110

        data=stock  -> checksum=100100
*/
~~~

*Integer addition checksum* is similar to LRC, except it uses addition instead of XOR.

It can detect two bit flips if they don't compensate each other (aka. 0->1 and 1->0 in the same bit position is compensated) in the same bit position and they are not on the top most bit position, as carry-over bits are truncated.

## One’s Complement Addition Checksum

~~~java
        byte[] data = "stock".getBytes();

        long temp = 0;
        for (byte b : data)
            temp += b;

        byte checksum = 0;
        while (temp != 0)
        {
            checksum += (temp & 0xFF);
            temp >>>= 8;
        }

/*
        data=stack  -> checksum=11000

                  1110011
                + 1110100
                + 1100001
                + 1100011
                + 1101011
               -----------
            =  1000010110
                        
                 00010110
                     + 10
               -----------
               =    11000


        data=stock  -> checksum=100110
*/
~~~

On top of *integer addition checksum*, *one’s complement addition checksum* adds carry-out bits back, but it doesn't handle compensating bit errors.

## Adler checksum

~~~java
/*
        251 is largest prime under 8 bits
        let A = 1; B = 0;

        for every byte in data word:
        A = (A + byte) % 251
        B = (B + A) % 251

        checksum = B << 8 | A
*/   

        Adler32 checksum = new Adler32();

        byte[] data = "jasonstack".getBytes();
        checksum.update(data);

        long checksumValue = checksum.getValue();
~~~

Adlter checksum works by maintaining two running one's complement checksums and concatenate both as checksum value.

Switched bits can be detected, as checksum result is order dependent.

## CRC

~~~java

/*
11010011101100 000 <--- input right padded by 3 bits
1011               <--- 4-bit divisor(aka. 11) from polynomial x³ + x + 1
01100011101100 000 <--- XOR with the divisor, the rest are unchanged
 1011               <--- divisor ...
00111011101100 000
  1011
00010111101100 000
   1011
00000001101100 000 <--- divisor move to aligh with next 1
       1011       
00000000110100 000
        1011
00000000011000 000
         1011
00000000001110 000
          1011
00000000000101 000
           101 1
-----------------
00000000000000 100 <--- remainder 3-bit CRC
*/

    CRC32 checksum = new CRC32();

    byte[] data = "jasonstack".getBytes();
    checksum.update(data);

    long checksumValue = checksum.getValue();
~~~

CRC stands for *Cyclic redundancy check*, which is popular checksum implementation based on the remainder of a *polynomial division* of their contents.

It uses division instead of addition, so CRC is better at mixing bits from *data word* than adler checksum.

The coefficients of polynomial *1x^3 + 0x^2 + 1x + 1* will be used as divisor, eg. *1011*


# What should be the block size

All depends on how many bit error you want to catch.

With larger block size, the number of bit errors that can be detected will reduce.
For example, 32-bit CRC should detect all 2 bit errors when *data word* is less than 8mb, or detect all 3 bit errors when *data word*
is less than 11kb.

Cassandra uses 32-bit CRC with 32kb block size for inter-node transition.

# References:
[Checksum][checksum-wiki]

[Adler-32][adler-32]

[CRC][crc-wiki]

[Checksum and CRC Data Integrity
Techniques for Aviation][crc-ref]


[checksum-wiki]:https://en.wikipedia.org/wiki/Checksum
[adler-32]:https://en.wikipedia.org/wiki/Adler-32
[crc-wiki]:https://en.wikipedia.org/wiki/Cyclic_redundancy_check
[crc-ref]:http://users.ece.cmu.edu/~koopman/pubs/koopman14_crc_faa_conference_presentation.pdf