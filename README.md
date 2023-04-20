# gpg_cheatsheet

* Listing all the public keys that the local gpg knows about:
```
gpg --list-keys
```

* List all the private keys that the local gpg knows about (note that this will include private keys that were moved onto yubikeys by this gpg):
```
gpg --list-secret-keys
```
Note that this shows the key ID under the `sec` line for each key - e.g. `2693B4B355BBDB34DC39F000BCBABE22C605A88D` for this key:
```
sec   ed25519 2023-01-12 [SC]
      2693B4B355BBDB34DC39F000BCBABE22C605A88D
uid           [ultimate] a b <a@b.c>
ssb   cv25519 2023-01-12 [E]
```

* Generate new key pair:
```
gpg --full-generate-key
```

  And follow the instructions displayed.

* Export public key to a backup file:
```
gpg --armor --export <KEY_ID> > my-pub-keys.asc
```

  Note: can export all the pub keys listed by `--list-secret-keys`) by ommiting the key ID

* Export private key to a backup file:
```
gpg --armor --export <KEY_ID> > my-priv-key.asc
```


## Creating new keys

```
gpg --full-generate-keys
```

## Setting up yubikey with pgp private key

Useful links:
* https://developer.okta.com/blog/2021/07/07/developers-guide-to-gpg
* https://support.yubico.com/hc/en-us/articles/360013790259-Using-Your-YubiKey-with-OpenPGP

1. Generate a pub/priv key pair to use the yubi key with. (Have only tested this with rsa 4096 keys - unsure if other key types/sizes work)

**Note** YOu can generate the key on the yubikey - however this means you cannot backup the key (as it's not possible to remove the private key from the yubikey so I would not recommend!)

You can see the generated key using
```
gpg --list-secret-keys
```

We'll need the `key ID` later which is the long hex number listed at the start of the key

2. (Optional) We'll want to change the pins on the yubi keys if they are the still defaults:
**Notes on yubikey pins:**
  * The default admin pin is `12345678`
  * The default user pin in `123456`
  * The reset pin (which is used to reset the number of retries on the user pin) is not set initially - meaning you need the admin pin to reset the user pin retries
  * You get 3 attempts at the user pin before the card is locked and you need to _unblock_ the pin using the admin pin
  * You get 3 attempts at the admin pin before the card is _really_ locked and you have to reset the whole card (i.e. wipe all the keys etc. off the card)

  a. Insert the yubikey and run:
```
gpg --card-status
```
    to check gpg see the yubikey

  b. Enter the `card-edit` menu:
```
gpg --card-edit
```
    This menu is quiet unintuitiive - particularly the `admin` commands mode (which works like a toggle); i.e. you type `admin` to enable admin commands and type `admin` again to disable them.

  c. Enter `admin` mode by typing `admin` and then change the pins by typing `passwd`.
     Follow the menu for changing user pin, admin pin, reset pin...

3. Take backups of the generated key:
  a. First the public key:
```
gpg --armor --export <KEY_ID> > mykey.pub.asc
```

  b. Then the private key:
```
gpg --armor --export-secret-keys <KEY_ID> > mykey.priv.asc
```

*Note* the `--armor` option exports the key in ascii `PEM` format - seems like a more sensible option

4. Move the new key to the yubi key.

**Note** This is probably the most error prone step as you have to copy the encryption key across separately from the signing key - strongly recommend testing decryption of the newly setup yubikey after the private keys have been moved!!

  a. Enter the `key-edit` gpg mode for the key pair created in step 1:
```
gpg --key-edit <KEY_ID>
```
  Key-edit mode is similary janky to the card-edit mode - specifically it has the toggling mode for which _key_ you are working on...
  A newly generated (rsa4096) key pair has two keys - a signing key (`key 0`) and an encryption key (`key 1`).
  (You can additionally add an authentication key as `key 2`).
  `key 0` is selected by default, and you can toggle to select `key 1` by typing `key 1` (the second key labeled `ssb` will now display with an asterisk next to it (`ssb*`)

  b. Move the signing (`key 0`) to the yubikey by typing `keytocard`

  c. Select the encryption key by typing `key 1` (`ssb*` should not show up)

  d. move the encryption key to the yubikey by typing `keytocard`

You can see if both keys are on the yubikey by looking at the card status:
```
gpg --card-status
```
And look ad the `Encryption key` entry - should be not `[none]`

Also the last entry `General key info...` will show two keys (`sec` and `ssb`) and importantly underneath each it will show `card-no` and then the serial number of the yubi key - indicating that both those keys are on this yubikey. Something like:
```
General key info...: ...
sec> rsa4096/ABCD... created: 1234-01-01
                     card-no: 0006 12345678
ssb> rsa4096/ABCD... created: 1234-01-01
                     card-no: 0006 12345678
```
If card number is not displayed underneath both keys then they are not both on the yubikey!


## Encrypting data with the yubikey

The yubikey only stores the private keys - not the public key, and the public key is whats needed to encrypt.

It's possible to set a URL on the yubikey of where the public key is hosted (e.g. on github) by using the `url` command in `gpg --card-edit` - then using `fetch` in `gpg --card-edit` will get gpg to download the public key form that url.

Alternatively you can just import the public key form the pub key backup file:
```
gpg --import my_key.pub.asc
```

From there you can encrypt with
```
gpg --encrypt --recipient <KEY_ID> file_to_encrypt
```
which will create a `file_to_encrypt.gpg` file.
(Or, short form `gpg -e -r <KEY_ID> file`)

## Using the yubikey to decrypt on a new machine

Gpg needs to know about the public key before if will detect the private key (encryption key) on the yubikey - so we need to import the pub key first:
```
gpg --import my_key.pub.asc
```

Next insert the yubikey and run
```
gpg --card-status
```

This should be enough to get gpg to connect the yubikey as the private key for the pub key that was imported. This can be checked by running
```
gpg --list-secret-keys
```
and the imported public key should now show up as a secret (i.e. private) key as well... as part of the secret key info it should display the `card-no` (i.e. serial number) of the yubikey containing the private key.

Now to decrypt
```
gpg -d file_to_decrypt.gpg
```
and you'll be prompted to enter the user pin for the yubikey.

If the yubikey isn't plugged into the machine then running the decrpyt command will prompt to insert the yubikey with the serial number that gpg thinks the private key is on.

This association of a private key to a yubikey serial will be stored in gpg's keyring so it only needs to be done once.

### Note on using multiple yubikeys with the same private key on

Gpg's connecting of a private key to a particular yubikey is quiet _sticky_ and seems to rely on the serial number of a yubikey.

In practice, if you have a two yubikeys, each loaded with the same privatekey, and gpg has associated the private key to one of the yubikeys, plugging in the other yubikey and typing `gpg --card-status` seems to switch the association to the other yubikey... but it then get's stuck there (gpg seems to stop seeing the private key on the other yubikey).

To fix, you can delete the association of a private key to a yubikey serial number running
```
gpg --list-secret-keys <KEY_ID>
```
**Be sure to insert the key ID** - Even though this commands prompts for deletion I think it will delete all keys if a key id is not specified!

Now just plug in a yubikey and run
```
gpg --card-status
```
and gpg will associate the private key on the yubikey to respective public key in it's key-chain.
