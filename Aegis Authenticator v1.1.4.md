# (First  Draft) [Aegis Authenticator](https://getaegis.app/) v1.1.4

A free, and open source Android 2FA application distributed on [F-Droid](https://f-droid.org/packages/com.beemdevelopment.aegis/), and [Google Play Store](play.google.com/store/apps/details?id=com.beemdevelopment.aegis).

Supports TOTP, HOTP, and Steam for token generation.

Supports importing files from other 2FA apps: Aegis, AndOTP, FreeOTP, FreeOTP+, Google Authenticator, Steam, and WinAuth.

[Aegis uses these libraries](https://github.com/alexbakker/Aegis/commit/e8e32ef7361ccd58fbba35dbdd290dadc8dc6147):
> * TextDrawable by Amulya Khare
> * FloatingActionButton by Dmytro Tarianyk
> * AppIntro by Paolo Rotolo
> * Krop by Avito Technology
> * SpongyCastle by Roberto Tyley
> * CircleImageView by Henning Dodenhof
> * barcodescanner by Dushyanth
> * libsu by John Wu

## Security Features

1. Vault protection via local storage, password, and biometrics.
2. Prevent screenshots, and visibility when switching applications.
3. Tap to reveal tokens. (Disabled by Default)
4. Notification for tap to lock vault.
5. The vault is automatically locked when the screen is off, or the application is closed.
6. Remind users to type in their password once in a while if biometrics are enabled for vault protection.

### Vault Protection

Adversaries want to obtain the vault, the generated tokens, and the keys to generate the tokens. Aegis protects the vault in several ways.

The vault is stored locally, unless the user exports it to a different system. It reduces the chances of adversaries obtaining the vault.

AEAD cipher [AES in GCM mode](https://github.com/beemdevelopment/Aegis/blob/09065c705b33bc52498733ca8a95362eb653b909/app/src/main/java/com/beemdevelopment/aegis/crypto/CryptoUtils.java#L30) with a [128-bit tag](https://github.com/beemdevelopment/Aegis/blob/09065c705b33bc52498733ca8a95362eb653b909/app/src/main/java/com/beemdevelopment/aegis/crypto/CryptoUtils.java#L32) is used to provide confidentiality, integrity, and authenticity of the master key, and the vault contents. Wrapper keys are used to protect the master key. The master key is used to protect the vault contents. The [256-bit keys](https://github.com/beemdevelopment/Aegis/blob/09065c705b33bc52498733ca8a95362eb653b909/app/src/main/java/com/beemdevelopment/aegis/crypto/CryptoUtils.java#L31) are [generated with a CSPRNG](https://github.com/beemdevelopment/Aegis/blob/09065c705b33bc52498733ca8a95362eb653b909/app/src/main/java/com/beemdevelopment/aegis/crypto/CryptoUtils.java#L112), and [96-bit nonces](https://github.com/beemdevelopment/Aegis/blob/09065c705b33bc52498733ca8a95362eb653b909/app/src/main/java/com/beemdevelopment/aegis/crypto/CryptoUtils.java#L33) are [generated with a CSPRNG as well](https://github.com/beemdevelopment/Aegis/blob/09065c705b33bc52498733ca8a95362eb653b909/app/src/main/java/com/beemdevelopment/aegis/crypto/CryptoUtils.java#L67).

For key derivation, Scrypt with an [output length of 256 bits](https://github.com/beemdevelopment/Aegis/blob/09065c705b33bc52498733ca8a95362eb653b909/app/src/main/java/com/beemdevelopment/aegis/crypto/CryptoUtils.java#L31) is used with the following parameters: [N=2^15](https://github.com/beemdevelopment/Aegis/blob/09065c705b33bc52498733ca8a95362eb653b909/app/src/main/java/com/beemdevelopment/aegis/crypto/CryptoUtils.java#L35), [r=8](https://github.com/beemdevelopment/Aegis/blob/09065c705b33bc52498733ca8a95362eb653b909/app/src/main/java/com/beemdevelopment/aegis/crypto/CryptoUtils.java#L36), [p=1](https://github.com/beemdevelopment/Aegis/blob/09065c705b33bc52498733ca8a95362eb653b909/app/src/main/java/com/beemdevelopment/aegis/crypto/CryptoUtils.java#L37), and a [256-bit salt that is generated with a CSPRNG](https://github.com/beemdevelopment/Aegis/blob/09065c705b33bc52498733ca8a95362eb653b909/app/src/main/java/com/beemdevelopment/aegis/crypto/CryptoUtils.java#L122).

PINs are not supported, as users may pick predictable ones, and the permutations are dangerously low, even with a 24-digit PIN. Making it trivial for adversaries to obtain the vault's contents.

Assuming an adversary can attempt 100,000 (which is not unreasonable considering mobile devices typically do not have much resources) guesses per second, and the user has a 24-digit PIN, they have a 50% chance of discovering the correct PIN in approximately 116 days. `sqrt(10^24) / (100,000 * 60 * 60 * 24)`


#### Password

When creating a vault, a new wrapper key is derived from the user provided password; a [generated salt](https://github.com/beemdevelopment/Aegis/blob/master/app/src/main/java/com/beemdevelopment/aegis/vault/slots/PasswordSlot.java#L38) along with [N](https://github.com/beemdevelopment/Aegis/blob/master/app/src/main/java/com/beemdevelopment/aegis/vault/slots/PasswordSlot.java#L35), [r](https://github.com/beemdevelopment/Aegis/blob/master/app/src/main/java/com/beemdevelopment/aegis/vault/slots/PasswordSlot.java#L36), and [p](https://github.com/beemdevelopment/Aegis/blob/master/app/src/main/java/com/beemdevelopment/aegis/vault/slots/PasswordSlot.java#L37) are [stored (JSON format)](https://github.com/beemdevelopment/Aegis/blob/09065c705b33bc52498733ca8a95362eb653b909/app/src/main/java/com/beemdevelopment/aegis/vault/slots/PasswordSlot.java#L32) in the [password slot](https://github.com/beemdevelopment/Aegis/blob/master/app/src/main/java/com/beemdevelopment/aegis/vault/slots/PasswordSlot.java), so it is possible to derive the _same_ wrapper key every time the user goes to unlock the vault.

##### Slot

When attempting to open a vault with a password, Aegis will use the parameters stored in the password slot. If the `key`, `nonce`, `tag`, `n`, `r`, `p`, or `salt` is modified, it is not possible to open the vault.

```json
{
	// Password slot is type 2.
	"type": 2,

	"uuid": "...",

	// The master key encrypted with the wrapper key derived from the password.
	"key": "... encrypted master key ...",

	"key_params": {
		// The nonce needed to produce the same key stream.
		"nonce": "...",

		// The tag needed to authenticate the encrypted master key.
		"tag": "..."
	},

	// Parameters needed to derive the same wrapper key.
	"n": 32768,
    "r": 8,
    "p": 1,
	"salt": "..."
}
```

#### Biometric

A new wrapper key and UUID(version 4) is generated, and put in Android's Key Store. Only when Android determines that the user is authentic via biometrics, will it give Aegis Authenticator the wrapper key. To prevent losing access to the vault (in the event Android's Key Store is cleared, or the stored key is deleted somehow), a password is required.

##### Slot

The biometric slot is identical to the [raw slot](https://github.com/beemdevelopment/Aegis/blob/master/app/src/main/java/com/beemdevelopment/aegis/vault/slots/RawSlot.java). If `uuid`, `key`, `nonce`, or `tag` is modified, it is not possible to unlock the vault.

```json
{
	// Biometric slot is type 1.
	"type": 1,

	// The UUIDv4 to look up in Android's Key Store to obtain the wrapper key.
	"uuid": "...",

	// The master key encrypted with the wrapper key.
	"key": "... encrypted master key ...",

	// Information needed to decrypt and verify the master key.
	"key_params": {
		"nonce": "...",
		"tag": "..."
	}
}
```

## Conclusion

No obscure, vulnerable, or broken cryptographic primitives are present in the design. Reasonable parameters were chosen for the cryptographic primitives. Implementation of the design is solid; I have found no issues.

Key management is superb, preventing loss of the vault by enforcing a password even if biometrics are enabled. The UI and UX is great, meaning the software is easier to use.

Aegis also makes it easy to transition over via a wide range of support for importing third-party 2FA app files.

### Possible Improvements

1. Use Libsodium instead of BouncyCastle, or SpongyCastle.

[They are](https://github.com/beemdevelopment/Aegis/issues/360) [already](https://github.com/beemdevelopment/Aegis/pull/302) [aware](https://github.com/alexbakker/Aegis/commit/e8e32ef7361ccd58fbba35dbdd290dadc8dc6147):

> This drops the dependency we have on SpongyCastle for the implementation of scrypt in favor of the one in libsodium. With this, we fix crashes on low-end devices (#114, #300), get better scrypt performance and pave the way for switching to better crypto primitives in the future as discussed in
vault.md.

__NOTE:__ v1.1.4 is still using SpongyCastle, migration to Libsodium is in development.

2. Scrypt has one parameter that scales for _both_ CPU _and_ memory cost. [Argon2](https://en.wikipedia.org/wiki/Argon2)__id__ is potentially a better KDF to since it resistant to side channel and TMTO attacks. Not only that, but it scales better with _imbalanced_ resources, such as little CPU _or_ memory due to having separate parameters for resources.

[They are already aware](https://github.com/beemdevelopment/Aegis/blob/master/docs/vault.md#kdf):

> Argon2 is a more modern KDF that provides an advantage over scrypt because it allows tweaking the memory-hardness parameter and CPU-hardness parameter separately, whereas scrypt ties those together into one cost parameter (N). As many applications have started using Argon2 in production, it seems that it has withstood the test of time. It will be considered as an alternative option to switch to in the future.

3. Use a AEAD cipher with a larger nonce size, or SIV.

[They are already aware](https://github.com/beemdevelopment/Aegis/blob/master/docs/vault.md#aead):

> Switching to a nonce misuse-resistant cipher like AES-GCM-SIV or a cipher with a larger (192 bits) nonce like XChaCha-Poly1305 will be explored in the future.

4. Make sure users are not using compromised passwords via popular wordlists used in password cracking programs during vault creation and unlocking.
