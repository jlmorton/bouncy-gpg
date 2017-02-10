[![Build Status](https://travis-ci.org/neuhalje/bouncy-gpg.svg?branch=master)](https://travis-ci.org/neuhalje/bouncy-gpg)
[![codecov](https://codecov.io/gh/neuhalje/bouncy-gpg/branch/master/graph/badge.svg)](https://codecov.io/gh/neuhalje/bouncy-gpg)
[![license](http://www.wtfpl.net/wp-content/uploads/2012/12/wtfpl-badge-4.png)](http://www.wtfpl.net/)

Mission Statement
=======================

  **Make using [Bouncy Castle](http://bouncycastle.org/) with [OpenPGP](https://tools.ietf.org/html/rfc4880) ~~great~~ fun again!**

This project gives you the following super-powers

- encrypt, decrypt, sign and verify GPG/PGP files with just a few lines of code
- protect all the data at rest by reading encrypted files with [transparent GPG decryption](src/main/java/name/neuhalfen/projects/crypto/bouncycastle/openpgp/decrypting/DecryptionStreamFactory.java)
- you can even [decrypt a gpg encrypted ZIP and re-encrypt each file in it again](examples/reencrypt/src/main/java/name/neuhalfen/projects/crypto/bouncycastle/openpgp/example/MainExplodedSinglethreaded.java) -- never again let plaintext hit your servers disk!

Examples
==========

_Bouncy GPG_ comes with several [examples](examples) build in.

Encrypting a file
-------------------

The following snippet encrypts `/tmp/plaintext.txt` to `recipient@example.com` and signs with `sender@example.com`.
The encrypted file is written to `/tmp/encrypted.gpg`.

```java
final KeyringConfig keyringConfig = KeyringConfigs
   .withKeyRingsFromFiles(
        "pubring.gpg",
        "secring.gpg", 
        KeyringConfigCallbacks.withPassword("s3cr3t"));

try (
        final FileOutputStream fileOutput = new FileOutputStream("/tmp/encrypted.gpg");
        final BufferedOutputStream bufferedOut = new BufferedOutputStream(fileOutput);

        final OutputStream outputStream = BouncyGPG
                .encryptToStream()
                .withConfig(keyringConfig)
                .withStrongAlgorithms()
                .toRecipient("recipient@example.com")
                .andSignWith("sender@example.com")
                .binaryOutput()
                .andWriteTo(bufferedOut);

        final FileInputStream is = new FileInputStream("/tmp/plaintext.txt")
) {
    Streams.pipeAll(is, outputStream);
}
```

Decrypting a file and validating the signature
-------------------------------------------------

The following snippet decrypts the file created in the snippet above.


```java
final KeyringConfig keyringConfig = KeyringConfigs
   .withKeyRingsFromFiles(
         "pubring.gpg",
         "secring.gpg", 
         KeyringConfigCallbacks.withPassword("s3cr3t"));

try (
        final FileInputStream cipherTextStream = new FileInputStream("/tmp/encrypted.gpg");

        final FileOutputStream fileOutput = new FileOutputStream(destFile);
        final BufferedOutputStream bufferedOut = new BufferedOutputStream("/tmp/plaintext.txt");

        final InputStream plaintextStream = BouncyGPG
               .decryptAndVerifyStream()
               .withConfig(keyringConfig)
               .andIgnoreSignatures()
               .fromEncryptedInputStream(cipherTextStream)

) {
    Streams.pipeAll(plaintextStream, bufferedOut);
}
```

Demos
=========

The directory [examples](examples) contains several examples that show how easy some common use cases are implemented.

[demo_decrypt.sh](examples/decrypt)
-----------------------------------------

Decrypt a file and verify the signature.

* `decrypt.sh  SOURCEFILE DESTFILE`

Uses the testing keys to decrypt a file. Useful for performance measurements and `gpg` interoperability.

[demo_encrypt.sh](examples/encrypt)
-----------------------------------------

Encrypt and sign a file.

* `encrypt.sh  SOURCEFILE DESTFILE` 

Uses the testing keys to encrypt a file. Useful for performance measurements and `gpg` interoperability.


[demo_reencrypt.sh](examples/reencrypt)
-----------------------------------------

A GPG encrypted ZIP file is decrypted on the fly. The structure of the ZIP is then written to disk. All files are re-encrypted before saving them.

* `demo_reencrypt.sh TARGET` -- decrypts an encrypted ZIP file containing  three files (total size: 1.2 GB) AND
   re-encrypts each of the files in the ZIP to the `TARGET` dir.

[The sample](examples/reencrypt/src/main/java/name/neuhalfen/projects/crypto/bouncycastle/openpgp/example/MainExplodedSinglethreaded.java)
shows how e.g. batch jobs can work with large files without leaving plaintext on disk (together with
[Transparent GPG decryption](src/main/java/name/neuhalfen/projects/crypto/bouncycastle/openpgp/decrypting/SignatureValidatingInputStream.java)).

This scheme has some very appealing benefits:
* Data in transit is _always_ encrypted with public key cryptography. Indispensable when you have to use `ftp`,
  comforting when you use `https` and the next Heartbleed pops up.
* Data at rest is _always_ encrypted with public key cryptography. When (not if) you get hacked, this can make all the
  difference between _"Move along folks, nothing to see here!"_ and _"I lost confidential customer data to the competition"_.
* You still need to protect the private keys, but this is considerable easier than the alternatives.

Consider the following batch job:

1. The customer sends a large (several GB) GPG encrypted ZIP archive containing a directory structure with several
   data files
2. Your `pre-processing` needs to split up the data for further processing
3. `pre-processing` stream-processes the GPG/ZIP archive
    1. The GPG stream is decrypted using the [BouncyGPG.decryptAndVerifyStream()](src/main/java/name/neuhalfen/projects/crypto/bouncycastle/openpgp/BouncyGPG.java) `InputStream`
    2. The ZIP file is processed with [ExplodeAndReencrypt](src/main/java/name/neuhalfen/projects/crypto/bouncycastle/openpgp/reencryption/ExplodeAndReencrypt.java)
        1. Each file from the archive is [processed](src/main/java/name/neuhalfen/projects/crypto/bouncycastle/openpgp/reencryption/ZipEntityStrategy.java)
        2. And transparently  encrypted with GPG and stored for further processing
4. The `processing` job  [transparently reads](src/main/java/name/neuhalfen/projects/crypto/bouncycastle/openpgp/decrypting/SignatureValidatingInputStream.java) the files without writing plaintext to the disk.


HOWTO
===========

Have a look at the example classes to see how easy it is to use Bouncy Castle PGP.

#1 Register Bouncy Castle Provider
-------------------------------

Add bouncy castle as a dependency and then install the provider before in your application.

### Add Build Dependency

```groovy

// in build.gradle add a dependency to bouncy castle and bouncy-gpg
//  ...
dependencies {
    compile 'org.bouncycastle:bcprov-jdk15on:1.56'
    compile 'org.bouncycastle:bcpg-jdk15on:1.56'
    //  ...
    compile 'name.neuhalfen.projects.crypto.bouncycastle.openpgp:bouncy-gpg:2.+'
   // ...
  }
```

### Install Provider

```java
    // in one of you classed install the BC provider
        if (Security.getProvider(BouncyCastleProvider.PROVIDER_NAME) == null) {
            Security.addProvider(new BouncyCastleProvider());
        }
```

#2 Important Classes
-------------------


| Class                         | Use when you want to                                                                |
|:------------------------------|:------------------------------------------------------------------------------------|
| [`BouncyGPG`](src/main/java/name/neuhalfen/projects/crypto/bouncycastle/openpgp/BouncyGPG.java) | Starting point for the convenient fluent en- and decryption API. |
| [`KeyringConfigs`](src/main/java/name/neuhalfen/projects/crypto/bouncycastle/openpgp/keys/keyrings/KeyringConfigs.java) | Create default implementations for GPG keyring access. You can also create your own implementations by implementing  [`KeyringConfig`](src/main/java/name/neuhalfen/projects/crypto/bouncycastle/openpgp/keys/keyrings/KeyringConfig.java). |
| [`KeyringConfigCallbacks`](src/main/java/name/neuhalfen/projects/crypto/bouncycastle/openpgp/keys/callbacks/KeyringConfigCallbacks.java) | Used by  [`KeyringConfigs`](src/main/java/name/neuhalfen/projects/crypto/bouncycastle/openpgp/keys/keyrings/KeyringConfigs.java). Create default implementations to provide secret-key passwords.  |
| [`DefaultPGPAlgorithmSuites`](src/main/java/name/neuhalfen/projects/crypto/bouncycastle/openpgp/algorithms/DefaultPGPAlgorithmSuites.java) |  Select from predefined algorithms suites or create your won with `PGPAlgorithmSuite`. |
| ['ReencryptExplodedZipSinglethread'](src/main/java/name/neuhalfen/projects/crypto/bouncycastle/openpgp/reencryption/ReencryptExplodedZipSinglethread.java) | [Work with encrypted ZIPs](examples/reencrypt/src/main/java/name/neuhalfen/projects/crypto/bouncycastle/openpgp/example/MainExplodedSinglethreaded.java) |

FAQ
=====

<dl>
  <dt>Why should I use this?</dt>
  <dd>For common use cases this project is easier than vanilla Bouncy Castle. It also has a pretty decent unit test
  coverage. It is free (speech & beer).</dd>

  <dt>Can I just grab a class or two for my project?</dt>
  <dd>Sure! Just grab it and hack away! The code is placed under the <a href="LICENSE">WTFPL</a>, you can't get much
   more permissive than this.</dd>

   <dt>Why is the test coverage so low?</dt>
   <dd>Test coverage for 'non-example' code is &gt;85%. Most of the not tested cases are either trivial OR lines that
   throw exceptions when the input format is broken. </dd>

   <dt>How can I contribute?</dt>
   <dd>Pullrequests are welcome! Please state in your PR that you put your code under the LICENSE.</dd>
   
   <dt>I am getting 'org.bouncycastle.openpgp.PGPException: checksum mismatch ..' exceptions</dt>
   <dd>The passphrase to your private key is very likely wrong (or you did not pass a passphrase).</dd>
   
   <dt>I am getting 'java.security.InvalidKeyException: Illegal key size' / 'java.lang.SecurityException: Unsupported keysize or algorithm parameters'</dt>
   <dd>The unrestricted policy files for the JVM are <a href="http://www.bouncycastle.org/wiki/display/JA1/Frequently+Asked+Questions">probably not installed</a>.</dd>

   <dt>Where is 'secring.pgp'?</dt>
   <dd>'secring.gpg' has been <a href="https://gnupg.org/faq/whats-new-in-2.1.html#nosecring">removed in gpg 2.1</a>. Use the other methods to read private keys.</dd>
</dl>

Building
=======

The project is a basic gradle build. All the scripts use `./gradlew  installDist`

The coverage report (incl. running tests) is generated with `./gradlew check`.

CAVE
=====

* Only one keyring per userid ("sender@example.com") supported.
* Only one signing key per userid supported.
* [TODOs](TODO.md)

## LICENSE

This code is placed under the WTFPL. Don't forget to adhere to the BouncyCastle License (http://bouncycastle.org/license.html).
