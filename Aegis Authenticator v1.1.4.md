# [Aegis Authenticator](https://getaegis.app/) [v1.1.4](https://github.com/beemdevelopment/Aegis/releases/tag/v1.1.4)

A **free**, [**open source**](https://github.com/beemdevelopment/Aegis/blob/v1.1.4/LICENSE), and **easy to use** Android 2FA application developed on [GitHub](https://github.com/beemdevelopment/Aegis/tree/v1.1.4). Distributed on [F-Droid](https://f-droid.org/packages/com.beemdevelopment.aegis/), and [Google Play Store](https://play.google.com/store/apps/details?id=com.beemdevelopment.aegis).

Supports [TOTP](https://tools.ietf.org/html/rfc6238), [HOTP](https://tools.ietf.org/html/rfc4226), and Steam for token generation.

Supports importing files from other 2FA apps: Aegis, AndOTP, FreeOTP, FreeOTP+, Google Authenticator, Steam, and WinAuth.

## Security Features

1. Vault protection via local storage, password, and biometrics.
2. Prevent screenshots, and visibility when switching applications.
3. Tap to reveal tokens. (Disabled by Default)
4. The vault is automatically locked when the screen is off, or the application is closed.
5. Notification for tap to lock vault.
6. Remind users to type in their password occasionally if biometrics are enabled for vault protection.

### Vault Protection

Adversaries want to obtain the vault, the generated tokens, and the keys to generate the tokens. Aegis protects the vault in several ways.

The vault is stored locally, unless the user exports it to a different system, reducing the chances of adversaries obtaining it.

AEAD cipher [AES in GCM mode](https://github.com/beemdevelopment/Aegis/blob/v1.1.4/app/src/main/java/com/beemdevelopment/aegis/crypto/CryptoUtils.java#L30) with a [128-bit tag](https://github.com/beemdevelopment/Aegis/blob/v1.1.4/app/src/main/java/com/beemdevelopment/aegis/crypto/CryptoUtils.java#L32) is used to provide confidentiality, integrity, and authenticity of the master key, and the vault contents. Wrapper keys are used to protect the master key. The master key is used to protect the vault contents. The [256-bit keys](https://github.com/beemdevelopment/Aegis/blob/v1.1.4/app/src/main/java/com/beemdevelopment/aegis/crypto/CryptoUtils.java#L31) are [generated with a CPRNG](https://github.com/beemdevelopment/Aegis/blob/v1.1.4/app/src/main/java/com/beemdevelopment/aegis/crypto/CryptoUtils.java#L112), and [96-bit nonces](https://github.com/beemdevelopment/Aegis/blob/v1.1.4/app/src/main/java/com/beemdevelopment/aegis/crypto/CryptoUtils.java#L33) are [generated with a CPRNG as well](https://github.com/beemdevelopment/Aegis/blob/v1.1.4/app/src/main/java/com/beemdevelopment/aegis/crypto/CryptoUtils.java#L67).

For key derivation, Scrypt with an [output length of 256 bits](https://github.com/beemdevelopment/Aegis/blob/v1.1.4/app/src/main/java/com/beemdevelopment/aegis/crypto/CryptoUtils.java#L31) is used with the following parameters: [N=2^15](https://github.com/beemdevelopment/Aegis/blob/v1.1.4/app/src/main/java/com/beemdevelopment/aegis/crypto/CryptoUtils.java#L35), [r=8](https://github.com/beemdevelopment/Aegis/blob/v1.1.4/app/src/main/java/com/beemdevelopment/aegis/crypto/CryptoUtils.java#L36), [p=1](https://github.com/beemdevelopment/Aegis/blob/v1.1.4/app/src/main/java/com/beemdevelopment/aegis/crypto/CryptoUtils.java#L37), and a [256-bit salt that is generated with a CPRNG](https://github.com/beemdevelopment/Aegis/blob/v1.1.4/app/src/main/java/com/beemdevelopment/aegis/crypto/CryptoUtils.java#L122).


#### Password

When creating a vault, a new wrapper key is derived from the user provided password; a [generated salt](https://github.com/beemdevelopment/Aegis/blob/v1.1.4/app/src/main/java/com/beemdevelopment/aegis/vault/slots/PasswordSlot.java#L38) along with [N](https://github.com/beemdevelopment/Aegis/blob/v1.1.4/app/src/main/java/com/beemdevelopment/aegis/vault/slots/PasswordSlot.java#L35), [r](https://github.com/beemdevelopment/Aegis/blob/v1.1.4/app/src/main/java/com/beemdevelopment/aegis/vault/slots/PasswordSlot.java#L36), and [p](https://github.com/beemdevelopment/Aegis/blob/v1.1.4/app/src/main/java/com/beemdevelopment/aegis/vault/slots/PasswordSlot.java#L37) are [stored (JSON format)](https://github.com/beemdevelopment/Aegis/blob/v1.1.4/app/src/main/java/com/beemdevelopment/aegis/vault/slots/PasswordSlot.java#L32) in the [password slot](https://github.com/beemdevelopment/Aegis/blob/v1.1.4/app/src/main/java/com/beemdevelopment/aegis/vault/slots/PasswordSlot.java), so it is possible to derive the _same_ wrapper key every time the user goes to unlock the vault.

##### Slot

When attempting to open a vault with a password, Aegis will use the parameters stored in the password slot. If `key`, `nonce`, `tag`, `n`, `r`, `p`, or `salt` is modified, it is not possible to open the vault with this slot.

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

A new wrapper key and UUID(version 4) is generated, and put in Android's Key Store. Only when Android determines that the user is authentic via biometrics, will it give Aegis Authenticator the wrapper key.

Assuming you remember your password, and you still have your vault: you will not lose access to your vault should Android's Key Store be wiped, or biometrics be disabled on your device.

##### Slot

The biometric slot is identical to the [raw slot](https://github.com/beemdevelopment/Aegis/blob/v1.1.4/app/src/main/java/com/beemdevelopment/aegis/vault/slots/RawSlot.java). If `uuid`, `key`, `nonce`, or `tag` is modified, it is not possible to unlock the vault with this slot.

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

No obscure, vulnerable, or broken cryptographic primitives are present in the design. Reasonable parameters were chosen for the cryptographic primitives. Implementation of the design is solid.

Key management is superb, preventing loss of the vault should Android's Key Store be wiped, or biometrics disabled prior to disabling the feature in the app. The UI is minimal, easy to use, and supports importing 2fa tokens from other apps, resulting in a good UX.

### Possible Improvements

1. Use Libsodium instead of BouncyCastle/SpongyCastle.

[They are](https://github.com/beemdevelopment/Aegis/issues/360) [already](https://github.com/beemdevelopment/Aegis/pull/302) [aware](https://github.com/alexbakker/Aegis/commit/e8e32ef7361ccd58fbba35dbdd290dadc8dc6147):

> This drops the dependency we have on SpongyCastle for the implementation of scrypt in favor of the one in libsodium. With this, we fix crashes on low-end devices (#114, #300), get better scrypt performance and pave the way for switching to better crypto primitives in the future as discussed in
vault.md.

2. Scrypt has one parameter that scales for _both_ CPU _and_ memory cost. [Argon2](https://en.wikipedia.org/wiki/Argon2)__id__ is potentially a better KDF to since it resistant to side channel and TMTO attacks. Not only that, but it scales better with _imbalanced_ resources, such as little CPU _or_ memory due to having separate parameters for resources.

[They are already aware](https://github.com/beemdevelopment/Aegis/blob/v1.1.4/docs/vault.md#kdf):

> Argon2 is a more modern KDF that provides an advantage over scrypt because it allows tweaking the memory-hardness parameter and CPU-hardness parameter separately, whereas scrypt ties those together into one cost parameter (N). As many applications have started using Argon2 in production, it seems that it has withstood the test of time. It will be considered as an alternative option to switch to in the future.

3. Use an AEAD cipher with a larger nonce size, or SIV.

[They are already aware](https://github.com/beemdevelopment/Aegis/blob/v1.1.4/docs/vault.md#aead):

> Switching to a nonce misuse-resistant cipher like AES-GCM-SIV or a cipher with a larger (192 bits) nonce like XChaCha-Poly1305 will be explored in the future.

4. Make sure users are not using compromised passwords via popular wordlists used in password cracking programs during vault creation and unlocking.
