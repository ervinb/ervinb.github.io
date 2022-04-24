---
layout: article
title: From Zero to Encyrpted Secrets in 2 Minutes with SOPS and GPG
tags: devops
key: sops-gpg-encrypted-secrets
---

You probably heard about [mozilla/sops](https://github.com/mozilla/sops), but even if the readme is amazingly detailed, a from-scratch example is always nice to have.

`sops`, in a nutshell, bridges the gap between various key management services (PGP, AWS KMS, GCP KMS, Azure Key Vault) and you.
This post will attempt to get you on your feet as fast as possible, in 3 simple steps: from "*I have no idea what to do with my hands*" to "*No way it's that easy!*".

Install the dependencies:
```sh
$ brew install sops gnupg
```

And run these 3-ish commands to convince yourself:
```sh
# Clone the example repository
$ git clone https://github.com/ervinb/sops-gpg-example.git
$ cd sops-gpg-example

# Import the encryption key
$ gpg --import <(curl -L https://gist.githubusercontent.com/ervinb/288c44a45cf2614a0684bea333b3aa36/raw/sops-gpg-example.asc)

# Decrypt and open the file
$ sops secrets/mysecrets.dev.enc.yaml
```

Your day-to-day interaction with this would be only the last line. 

`gpg --import` has to be executed only once, after which the key will be part of the local keychain
(persists reboots as well).
That's literally all there is to it, after following the below steps.

## Do it yourself

Start the stopwatch - we have 2 minutes.

1. Generate a PGP key
   ```sh
   $  gpg --batch --generate-key <<EOF
     %no-protection
     Key-Type: DSA
     Subkey-Type: ECDSA
     Subkey-Curve: nistp256
     Subkey-Type: default
     Name-Real: Foo Bar
     Expire-Date: 0
   EOF
   ```
   The key is created without a passphrase because of the `%no-protection` option. Otherwise a `Passphrase: <pass>` would be required.

2. Create a sops configuration file with the key's fingeprint. This is the ✨ magic ✨ ingredient,
which makes the onboarding so frictionless.
   ```sh
   $ gpg --list-keys
   pub   rsa2048 2020-12-06 [SC]
         7E6DC556C66C43D928A95EA3715A56B718EAF0B6
   uid           [ultimate] Foo Bar
   sub   rsa2048 2020-12-06 [E]
   
   $ cat .sops.yaml
   creation_rules:
     - path_regex: secrets/.*\.dev\.enc\.yaml$
       pgp: 7E6DC556C66C43D928A95EA3715A56B718EAF0B6
   ```

   This is also perfect if you want more control over the secrets, like using different keys for different environments.
   For example `secrets/*.dev.enc.yaml` could use one key, and `secrets/*.prod.enc.yaml` another one.
   More details on this [here](https://github.com/mozilla/sops#using-sops-yaml-conf-to-select-kms-pgp-for-new-files).

3. Use `sops` to edit and create new secrets
```sh
$ sops secrets/mysecrets.dev.enc.yaml
```

   Then it just a question of distributing the keys to the right people and/or environment.
   Which brings us to Keybase.
   
#### Note for Linux users
   
I've found that both on Fedora and Ubuntu, for whatever reason, creating a new file with `sops` throws the following cryptic error:

```sh
$ sops secrets/new.dev.enc.yaml
<save the file in the editor>
File has not changed, exiting.
```

The solution is to create the file first and encrypt it in-place afterwards:
```sh
$ vi secrets/new.dev.enc.yaml
$ sops -i -e secrets/new.dev.enc.yaml
```


## Distributing the key to firends and family

To extract the PGP key from your local keychain, use:
```sh
$ gpg --list-keys
-------------------------------
pub   rsa2048 2020-12-06 [SC]
      7E6DC556C66C43D928A95EA3715A56B718EAF0B6
uid           [ultimate] Foo Bar
sub   rsa2048 2020-12-06 [E]

$ gpg --armor --export-secret-keys 7E6DC556C66C43D928A95EA3715A56B718EAF0B6 > key.asc
```

`--armor` makes it so that the output is ASCII (`.asc`) formatted, and not in binary (default).

One of the most seamless ways to distribute keys and other sensitive files is Keybase.
It has a low barrier of entry, and you can control the granularity of access with "teams".
It also integrates nicely with the filesystem.

1. Install Keybase
   ```sh
   $ brew install keybase
   ```
2. Create an account
3. Create a new team and store the secret key under the team's folder

After that, you grab the universal path and import the key to anywhere with `gpg` installed. Your peers
can also grab the key after they join your team.

The command below imports the same PGP key we used at the beginning of the post. The `sopsgpg` team is open, so you can join if you want to test it out.

```
## The path is Keybase specific and it will work on any platform - no need to use your local filesystem path.
## Join the 'sopsgpg' team in Keybase first.
$ gpg --import <(keybase fs read /keybase/team/sopsgpg/pgp/key.asc)
```

## Use it in your applications

To use the decrypted values in your application, you can just add a line to your setup scripts
to run
```sh
sops -d secrets/mysecret.dev.enc.yaml > configuration.yaml
```
(make sure to add the decrypted files to `.gitignore`)

For Terraform projects use [terraform-sops](https://github.com/carlpett/terraform-provider-sops), and if you're into Terragrunt,
it has a built-in [sops_decrypt_file](https://terragrunt.gruntwork.io/docs/reference/built-in-functions/#sops_decrypt_file) function.

You will be running `sops` only to create or edit secrets, otherwise, it will be invisible (and incredible).
