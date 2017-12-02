---
layout: post
title:  "Why you can't recover your KeePass password"
date:   2015-05-02 00:00:00
categories: keepass security csharp
image-base: /assets/images/posts/2015-05-02-why-you-cant-recover-your-keepass-password
---

It was a pretty vexing problem I faced; not only had I forgotten my password for the [KeePass](http://keepass.info/) password manager, but the developer of this tool had made a concerted effort to drastically slow down my efforts when repeatedly guessing what my password could have been via a technique called [brute-forcing](http://en.wikipedia.org/wiki/Brute-force_attack).

What peeked my curiosity though, is why it wasn't feasible to recover the password via brute-forcing? Or by a [dictionary attack](https://en.wikipedia.org/wiki/Dictionary_attack) for that matter. The following shows how KeePass intentionally slows down the process that is required to run in order to decrypt the stored credentials, and thus determine if the password is correct.

You can explore these steps further with a [small tool I wrote](https://github.com/FlyingTopHat/KeePasswd).

![Process of authenticating a password]({{ page.image-base }}/process.png)


## 1. Reading the Database's Header

Before asking the user for their password, KeePass first reads all the relevant data from the database's unencrypted header. This data is then used during the following steps to authenticate the key and decrypt the database's body.

```
Signature 1                 Signature 2         File Version
0x00000000                  0x00000000          0x00000000

Fields
Type ID                      Data Size           Data
0x04 = Master Seed           0x00000000          0x00...
0x05 = Transform Seed        0x00000000          0x00...
0x06 = Transform Rounds      0x0000000000000000  0x00...
0x07 = Encryption IV         0x0000              0x00...
0x09 = Expected Start Bytes  0x00000000          0x00...
0x00 = End of Header 		
```

Using the [KeePasswd tool](https://github.com/FlyingTopHat/KeePasswd) you can view the header of a KeePass database by calling the following:

```bash
$ keepasswd -file "C:\ExampleDatabase.kdbx" -header

Master Seed: 681576DF73AC6AE38A21494AF04723E1968168EE08CC4D1893F72D3F309E42BF
Encryption IV: 5B61F08E5BDD401F72C0C43DA15E2700
Transform Rounds: 6000
Transform Seed: C723515A30197B562906A32BBC371FC8C9433DA589A499FF59587A62EC8F3055
Expected Start-Bytes: A45D4E050BF15F3F66B357E1BDC86343C75D6A72CE976665C6E341D2EE015ADC
```

## 2. Create the Composite Key

The process starts by creating a master key from all of the keys provided by the user (generally a password, key-file and/or Windows User Account). This is achieved by appending the bytes of the [SHA256](https://en.wikipedia.org/wiki/SHA256) hash for each of the keys into a single composite, which is then hashed with with SHA256.

```csharp
byte[][] keys = {
    Encoding.UTF8.GetBytes("Example"),
    ...
};

byte[] compositeKey;
using(var stream = new MemoryStream())
{
    foreach(byte[] key in keys)
    {
        byte[] hashedKey = (new SHA256Managed()).ComputeHash(key);
        stream.Write(hashedKey, 0, hashedKey.Length);
    }

    compositeKey = (new SHA256Managed()).ComputeHash(stream.ToArray());
}
```
The resulting composite key is then passed to the next process.

## 3. Transformation of the Key

The key transformation stage is rather clever in that it transforms the key many times to slow down the whole authentication process; thus reducing the feasibility of guessing the password via dictionary or brute-force attacks.

This process works by using the [AES algorithm](https://en.wikipedia.org/wiki/Advanced_Encryption_Standard), with the Transform Seed (taken from the header), to transform each 16 bit half of the 32 bit key by the amount of times specified in the Transform Rounds header field.</p>

```csharp
var rijndael = new RijndaelManaged
{
    Key = Header.TransformSeed,
    ...
};

ICryptoTransform crypto = rijndael.CreateEncryptor();
for (ulong i = 0; i < Header.TransformRounds; ++i)
{
   crypto.TransformBlock(compositeKey, 0, 16, compositeKey, 0);
   crypto.TransformBlock(compositeKey, 16, 16, compositeKey, 16);
}

transformedKey = (new SHA256Managed()).ComputeHash(compositeKey);
```

As with the last step, the result of this transform is hashed using SHA256, which is then passed to the next part of the process.

## 4. Seed the Key

The resulting transformed key is now joined to the Master Seed (taken from the header) and then hashed with SHA256 once again to produce the seeded master key that is ready to be used for decrypting the database's body.</p>

```csharp
byte[] masterKey;
using(var stream = new MemoryStream())
{
   stream.Write(Header.MasterSeed, 0, Header.MasterSeed.Length);
   stream.Write(transformedKey, 0, transformedKey.Length);

   masterKey = (new SHA256Managed()).ComputeHash(stream.ToArray());
}
```

## 5. Compare Decrypted Stream with Expected Bytes

The last part of the process is to decrypt part of the encrypted stream (situated after the KeePass file's header) and then compare the first few bytes with the ExpectedStartBytes header field.

If the decrypted bytes match the bytes stored in the header field then the password is correct and the rest of the stream, containing the user's data, can be decrypted.

```csharp
var rijndael = new RijndaelManaged
{
    Key = masterKey,
    IV = Header.EncryptionIV,
    ...
};

ICryptoTransform decryptor = r.CreateDecryptor();

int startBytesLength = Header.ExpectedStartBytes.Length;
byte[] decryptedBytes = new byte[startBytesLength];

var cryptoStream = new CryptoStream(databaseFileBodyStream, decryptor, CryptoStreamMode.Read);
cryptoStream.Read(decryptedBytes, 0, decryptedBytes.Length);

bool isPasswordCorrect = areBytesEqual(decryptedBytes, Header.ExpectedStartBytes);
Console.WriteLine("Password is " + (isPasswordCorrect ? "Correct" : "Wrong"));
```

## Finally

The whole process at the time of creating the database is configured to take approximately 1 second on the host's machine, which adds up to a considerable amount of time when you're trying to guess thousands of passwords.

To see the whole process in action with the [KeePasswd tool](https://github.com/FlyingTopHat/KeePasswd) you can call the following:

```bash
$ keepasswd -file "C:\\ExampleDatabase.kdbx" -passwords test1,test2,test3

The password is 'test2'
```
