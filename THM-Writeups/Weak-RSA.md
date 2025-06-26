# Weak RSA – TryHackMe

## Target
Decrypt RSA-encrypted message using weak key length.

## Steps
1. Provided: n, e, and ciphertext
2. Used `rsatool` and `factordb` to factor n
3. Retrieved private key
4. Decrypted the ciphertext with Python script
5. Got the flag

## Flag
- THM{weak_rsa_flag}
