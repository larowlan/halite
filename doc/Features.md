# Halite Features

In addition to its [core functionality](Basics.md), Halite offers some useful
APIs for solving common problems.

* `Cookie` - Authenticated encryption for your HTTPS cookies
* `File` - Cryptography library for working with files
* `Password` - Secure password storage and password verification API

## Cookie Encryption/Decryption

Unlike the core Halite APIs, the Cookie class is not static. You must create an
instance of `Cookie` and work with it.

```php
$enc_key = \ParagonIE\Halite\Symmetric\EncryptionKey::fromFile('/path/to/key');
$cookie = new \ParagonIE\Halite\Cookie($enc_key);
```

From then on, all you need to do is use the `fetch()` and `store()` APIs.

**Storing** data in an encrypted cookie:

```php
$cookie->store(
    'auth',
    ['s' => $selector, 'v' => $verifier],
    time() + 2592000
);
```

**Fetching** data from an encrypted cookie:

```php
$token = $cookie->fetch('auth');
var_dump($token); // array(2) ...
```

## File Cryptography

Halite's `File` class provides streaming file cryptography features, such as
authenticated encryption and digital signatures. `File` allows developers to
perform secure cryptographic operations on large files with a low memory
footprint.

The `File` API looks like this:

* Lazy Mode
  * `File::checksum`(`lazy`, [`AuthenticationKey?`](Classes/Symmetric/AuthenticationKey.md), `bool?`): `string`
  * `File::encrypt`(`lazy`, `lazy`, [`EncryptionKey`](Classes/Symmetric/EncryptionKey.md))
  * `File::decrypt`(`lazy`, `lazy`, [`EncryptionKey`](Classes/Symmetric/EncryptionKey.md))
  * `File::seal`(`lazy`, `lazy`, [`EncryptionPublicKey`](Classes/Asymmetric/EncryptionPublicKey.md))
  * `File::unseal`(`lazy`, `lazy`, [`EncryptionSecretKey`](Classes/Asymmetric/EncryptionSecretKey.md))
  * `File::sign`(`lazy`, [`EncryptionSecretKey`](Classes/Asymmetric/EncryptionSecretKey.md)): `string`
  * `File::verify`(`lazy`, [`EncryptionPublicKey`](Classes/Asymmetric/EncryptionPublicKey.md)): `bool`
* Filenames
  * `File::checksumFile`(`string`, [`AuthenticationKey?`](Classes/Symmetric/AuthenticationKey.md), `bool?`): `string`
  * `File::encryptFile`(`string`, `string`, [`EncryptionKey`](Classes/Symmetric/EncryptionKey.md))
  * `File::decryptFile`(`string`, `string`, [`EncryptionKey`](Classes/Symmetric/EncryptionKey.md))
  * `File::sealFile`(`string`, `string`, [`EncryptionPublicKey`](Classes/Asymmetric/EncryptionPublicKey.md))
  * `File::unsealFile`(`string`, `string`, [`EncryptionSecretKey`](Classes/Asymmetric/EncryptionSecretKey.md))
  * `File::signFile`(`string`, [`EncryptionSecretKey`](Classes/Asymmetric/EncryptionSecretKey.md)): `string`
  * `File::verifyFile`(`string`, [`EncryptionPublicKey`](Classes/Asymmetric/EncryptionPublicKey.md)): `bool`
* Resources
  * `File::checksumResource`(`resource`, [`AuthenticationKey?`](Classes/Symmetric/AuthenticationKey.md), `bool?`): `string`
  * `File::encryptResource`(`resource`, `resource`, `EncryptionKey`)
  * `File::decryptResource`(`resource`, `resource`, `EncryptionKey`)
  * `File::sealResource`(`resource`, `resource`, [`EncryptionPublicKey`](Classes/Asymmetric/EncryptionPublicKey.md))
  * `File::unsealResource`(`resource`, `resource`, [`EncryptionSecretKey`](Classes/Asymmetric/EncryptionSecretKey.md))
  * `File::signResource`(`resource`, [`EncryptionSecretKey`](Classes/Asymmetric/EncryptionSecretKey.md)): `string`
  * `File::verifyResource`(`resource`, [`EncryptionPublicKey`](Classes/Asymmetric/EncryptionPublicKey.md)): `bool`

The `lazy` type indicates that the argument can be either a `string` containing
the file's path, or a `resource` (open file handle). Don't mix and match types.
If one is a `resource`, both must be. If the other is a `string`, both must be.

Each of feature is designed to work in a streaming fashion.

> In each case, any call to `::*File` is just a friendly wrapper for 
> the identical `::*Resource` endpoint. We're documenting the File steps, but if
> you have an open file handle, feel free to use the resource methods too.

### Calculating the Checksum of a File

Basic usage:

```php
$checksum = \ParagonIE\Halite\File::checksum('/source/file/path');
```

If for some reason you desire to use a keyed hash rather than just a plain one,
you can pass an [`AuthenticationKey`](Classes/Symmetric/AuthenticationKey.md) to
an optional second parameter, or `null`.

```php
$keyed = \ParagonIE\Halite\File::checksum('/source/file/path', $auth_key);
```

Finally, you can pass `true` as the optional third argument if you would like a
raw binary string rather than a hexadecimal string.

```php
$keyed_checksum = \ParagonIE\Halite\File::checksum('/source/file/path', null, true);
```

### Symmetric-Key File Encryption / Decryption

If you need to encrypt a file larger than the amount of memory available to PHP,
you'll run into problems with just the basic `\ParagonIE\Halite\Symmetric\Crypto`
API. To work around these limitations, use `File::encryptFile()` instead.

For example:

```php
\ParagonIE\Halite\File::encrypt(
    $inputFilename,
    $outputFilename,
    $enc_key
);
```

This will encrypt the contents of the file located at `$inputFilename` and write
its contents to `$outputFilename`.

Decryption is straightforward as well:

```php
\ParagonIE\Halite\File::decryptFile(
    $inputFilename,
    $outputFilename,
    $enc_key
);
```

### Asymmetric-Key File Encryption / Decryption

This feature encrypts files with a public key so that they can be  decrypted 
offline with a secret key.

```php
$seal_keypair = \ParagonIE\Halite\EncryptionKeyPair::generate();
$seal_secret = $seal_keypair->getSecretKey();
$seal_public = $seal_keypair->getPublicKey();
```

**Sealing** the contents of a file using Halite:

```php
\ParagonIE\Halite\File::seal(
    $inputFilename,
    $outputFilename,
    $seal_public
);
```

**Opening** the contents of a sealed file using Halite:

```php
\ParagonIE\Halite\File::unseal(
    $inputFilename,
    $outputFilename,
    $seal_secret
);
```

### Asymmetric-Key Digital Signatures

First, you need a key pair.

```php
$sign_keypair = \ParagonIE\Halite\SignatureKeyPair::generate();
    $sign_secret = $sign_keypair->getSecretKey();
    $sign_public = $sign_keypair->getPublicKey();
```

**Signing** the contents of a file using your secret key:

```php
$signature = \ParagonIE\Halite\File::sign(
    $inputFilename,
    $sign_secret
);
```

**Verifying** the contents of a file using a known public key:

```php
$valid = \ParagonIE\Halite\File::verify(
    $inputFilename,
    $sign_public,
    $signature
);
```

Like `checksumFile()`, you can pass an optional `true` to get a raw binary
signature instead of a hexadecimal-encoded string.

## Secure Password Storage

This feature serves a very narrow use case: You have the webserver and database
on separate hardware, and would like to prevent a database compromise from 
leaking the actual password hashes.

If your webserver and database server are the same machine, there is no
advantage to using this feature over [libsodium's scrypt implementation](https://paragonie.com/book/pecl-libsodium/read/07-password-hashing.md#crypto-pwhash-scryptsalsa208sha256-str).

**Hashing then Encrypting** a password:

```php
$stored_hash = \ParagonIE\Halite\Password::hash(
    $plaintext_password, // string
    $encryption_key      // \ParagonIE\Halite\Symmetric\EncryptionKey
);
```

**Validating a password**:

```php
try {
    if (\ParagonIE\Halite\Password::verify(
        $plaintext_password, // string
        $stored_hash,        // string
        $encryption_key      // \ParagonIE\Halite\Symmetric\EncryptionKey
    )) {
        // Password matches
    }
} catch (\ParagonIE\Halite\Alerts\InvalidMessage $ex) {
    // Handle an invalid message here. This usually means tampered ciphertext.
}
```