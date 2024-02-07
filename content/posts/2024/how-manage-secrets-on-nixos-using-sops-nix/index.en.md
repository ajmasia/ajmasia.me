---
title: "How to manage secrets in NixOS using sops-nix"
summary: "Learn how to manage the secrets of your NixOS configurations with sops-nix, an atomic, declarative, and easily reproducible module based on sops."
canonicalURL: https://ajmasia.me/posts/2024/como-gestionar-secrets-en-nixos-con-sops-nix
author: Antonio José Masiá
tags:
    - nix
date: 2024-02-06
draft: false
cover:
  image: https://plus.unsplash.com/premium_photo-1677702162621-f61d44e676eb?q=80&w=1470&auto=format&fit=crop&ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D
---

One of the main advantages of using [NixOS](https://nixos.org/) is that it allows us to have a highly reproducible operating system. This is largely because all the configuration is done declaratively using [Nix](https://nixos.org/manual/nix/stable/installation/installing-binary). In any configuration, whether for a machine for professional, personal use, server, or service, we end up having to manage *tokens*, *environment variables*, or configurations with personal information, APIs, etc., which, logically, we want to keep private.

Until now, I had been managing this information locally without integrating it into my configuration repository, which is a real hassle as it forces you to maintain backups of this information to be able to restore in case of any problems. I started researching and saw that a large part of the community tends to use [sops-nix](https://github.com/Mic92/sops-nix). There are other alternatives, although truthfully, [sops-nix](https://github.com/Mic92/sops-nix) seems stable and especially scalable, so I have decided to implement it in my configurations to do some testing and thereby draw conclusions.

You can implement [sops-nix](https://github.com/Mic92/sops-nix) both in the NixOS configuration and at the user level using [home-manager](https://github.com/nix-community/home-manager).

## How sops-nix Works

[Sops-nix](https://github.com/Mic92/sops-nix) allows us to encrypt our **secrets** within our configuration definition to later decrypt them in the build phase. This allows for declarative definition. To do this, it generally uses at least two keys for the system configuration, one for the user and usually the SSH key of our machine (this is usually enabled with the SSH service). As we will see, at the home-manager level, just the user key will be enough.

Since our secrets are fully encrypted, we can upload them privately, both to our dotfiles repository and when deploying on servers, etc. As we will see, it's quite simple.

## Implementation

### Generation of Main Keys

We can use either **age** keys or **gpg** keys.

```bash
# for age..
mkdir -p ~/.config/sops/age
age-keygen -o ~/.config/sops/age/keys.txt
# or to convert an ssh ed25519 key to an age key
mkdir -p ~/.config/sops/age
nix-shell -p ssh-to-age --run "ssh-to-age -private-key -i ~/.ssh/id_ed25519 > ~/.config/sops/age/keys.txt"
# for GPG >= version 2.1.17
gpg --full-generate-key
# for GPG < 2.1.17
gpg --default-new-key-algo rsa4096 --gen-key
```

Once the user key is generated, we will need to obtain the public keys to configure [sops](https://github.com/getsops/sops). If you are using a PGP key, you will need the **fingerprint**. In the case of using an **age** key, you can directly see the public key by viewing the file of the key or noting it down after its creation.

> IMPORTANT: If you use a GPG key, define it without a password; otherwise, when the system build is performed, it will fail.

### Obtaining the Public Key of Our Machine's SSH Key

We can obtain this key by executing the following command:

```bash
nix-shell -p ssh-to-age --run 'cat /etc/ssh/ssh_host_ed25519_key.pub | ssh-to-age'
age1rgffpespcyjn0d8jglk7km9kfrfhdyev6camd3rck6pn8y47ze4sug23v3
```

### Configuration of sops and Definition of Secrets

We will create a file named `.sops.yaml` at the root of our configuration where we will define the path to our **secrets** and the public keys that will allow carrying out the encryption/decryption process:

```yaml
keys:
  - &user_alice 2504791468b153b8a3963cc97ba53d1919c5dfd4
  - &host_delta age12zlz6lvcdk6eqaewfylg35w0syh58sm7gh53q5vvn7hd7c6nngyseftjxl
creation_rules:
  - path_regex: secrets.(yaml|json|env|ini)$
    key_groups:
    - pgp:
      - *user_alice
      age:
      - *host_delta
```

If everything is well defined, we will proceed to define our **secrets**. For this, we will use the [sops](https://github.com/getsops/sops) tool. When we open our secrets file for the first time, that is, when creating it, [sops](https://github.com/getsops/sops) will take care of decrypting its content so that we can see and define/modify it. When we close it, it will re-encrypt its content using our keys.

```bash
nix-shell -p sops --run "sops secrets/secrets.yaml"
```

The first time we open the file, it creates some example data. We will delete them and define our first operating system-level secret:

```yaml
nixos_secret: my_first_secret
```

Once the file is saved, if we open it directly with any editor, we can verify that it is encrypted:


```bash
example-key: ENC[AES256_GCM,data:AB8XMyid4P7mXdjj+A==,iv:RRsZC+V+3w22pOi/2TCjBYn/0OYsNGCu5CT1ZBSKGi0=,tag:zT5mlujrSuA6KKxLKL8CMQ==,type:str]
#ENC[AES256_GCM,data:59QWbzCQCP7kLdhyjFOZe503MgegN0kv505PBNHwjp6aYztDHwx2N9+A1Bz6G/vWYo+4LpBo8/s=,iv:89q3ZXgM1wBUg5G29ROor3VXrO3QFGCvfwDoA3+G14M=,tag:hOSnEZ6DKycnF37LCXOjzg==,type:comment]
#ENC[AES256_GCM,data:kUuJCkDE9JT9C+kdNe0CSB3c+gmgE4We1OoX4C1dWeoZCw/o9/09CzjRi9eOBUEL0P1lrt+g6V2uXFVq4n+M8UPGUAbRUr3A,iv:nXJS8wqi+ephoLynm9Nxbqan0V5dBstctqP0WxniSOw=,tag:ALx396Z/IPCwnlqH//Hj3g==,type:comment]
myservice:
    my_subdir:
        my_secret: ENC[AES256_GCM,data:hcRk5ERw60G5,iv:3Ur6iH1Yu0eu2otcEv+hGRF5kTaH6HSlrofJ5JXvewA=,tag:hpECXFnMhGNnAxxzuGW5jg==,type:str]
sops:
    kms: []
    gcp_kms: []
    azure_kv: []
    hc_vault: []
    age:
        - recipient: age12zlz6lvcdk6eqaewfylg35w0syh58sm7gh53q5vvn7hd7c6nngyseftjxl
          enc: |
            -----BEGIN AGE ENCRYPTED FILE-----
            YWdlLWVuY3J5cHRpb24ub3JnL3YxCi0+IFgyNTUxOSB1dFYvSTRHa3IwTVpuZjEz
            SDZZQnc5a0dGVGEzNXZmNEY5NlZDbVgyNVU0Clo3ZC9MRGp4SHhLUTVCeWlOUUxS
            MEtPdW4rUHhjdFB6bFhyUXRQTkRpWjAKLS0tIDVTbWU2V3dJNUZrK1A5U0c5bkc0
            S3VINUJYc3VKcjBZbHVqcGJBSlVPZWcKqPXE01ienWDbTwxo+z4dNAizR3t6uTS+
            KbmSOK1v61Ri0bsM5HItiMP+fE3VCyhqMBmPdcrR92+3oBmiSFnXPA==
            -----END AGE ENCRYPTED FILE-----
        - recipient: age18jtffqax5v0t6ehh4ypaefl4mfhcrhn6ek3p80mhfp9psx6pd35qew2ww3
          enc: |
            -----BEGIN AGE ENCRYPTED FILE-----
            YWdlLWVuY3J5cHRpb24ub3JnL3YxCi0+IFgyNTUxOSBzT3FxcDEzaFRQOVFpNkg2
            Skw4WEIxZzNTWkNBaDRhcUN2ejY4QTAwTERvCkx2clIzT2wyaFJZcjl0RkFXL2p6
            enhqVEZ3ZkNKUU5jTlUxRC9Lb090TzAKLS0tIDBEaG00RFJDZ3ZVVjBGUWJkRHdQ
            YkpudG43eURPVWJUejd3Znk5Z29lWlkK0cIngn2qdmiOE5rHOHxTRcjfZYuY3Ej7
            Yy7nYxMwTdYsm/V6Lp2xm8hvSzBEIFL+JXnSTSwSHnCIfgle5BRbug==
            -----END AGE ENCRYPTED FILE-----
    lastmodified: "2021-11-20T16:21:10Z"
    mac: ENC[AES256_GCM,data:5ieT/yv1GZfZFr+OAZ/DBF+6DJHijRXpjNI2kfBun3KxDkyjiu/OFmAbsoVFY/y6YCT3ofl4Vwa56Veo3iYj4njgxyLpLuD1B6zkMaNXaPywbAhuMho7bDGEJZHrlYOUNLdBqW2ytTuFA095IncXE8CFGr38A2hfjcputdHk4R4=,iv:UcBXWtaquflQFNDphZUqahADkeege5OjUY38pLIcFkU=,tag:yy+HSMm+xtX+vHO78nej5w==,type:str]
    pgp: []
    unencrypted_suffix: _unencrypted
    version: 3.7.1
```

### Implementation of sops-nix in NixOS

The ideal implementation is using the available *flake*. We will add it as an input to our configuration flake and then import it as a module in NixOS:

```nix
{
  inputs.sops-nix.url = "github:Mic92/sops-nix";
  # optional, not necessary for the module
  #inputs.sops-nix.inputs.nixpkgs.follows = "nixpkgs";

  outputs = { self, nixpkgs }: {
    # change `yourhostname` to your actual hostname
    nixosConfigurations.yourhostname = nixpkgs.lib.nixosSystem {
      # customize to your system
      system = "x86_64-linux";
      modules = [
        ./configuration.nix
        inputs.sops-nix.nixosModules.sops
      ];
    };
  };
}
```

### Accessing Secrets from Our Configuration

With the module now available, we will add the following configuration to our `configuration.nix`:

```nix
sops.gnupg.home = "~/.gnupg";
sops.gnupg.sshKeyPaths = [ ];
sops.age.sshKeyPaths = [ "/etc/ssh/ssh_host_ed25519_key" ];
```

These declarations allow [sops-nix](https://github.com/Mic92/sops-nix) to know where to find the private keys for the decryption processes.

To access the secrets from the configuration we will add the following definition:

```nix
sops.secrets.nixos_secret = {
  sopsFile = ./secrets.yaml;
};
```

In this definition, we are telling NixOS to make the secret **nixos_secret** available for our configuration, accessible from the property `config.sops.secrets.nixos_secret.path` which we can use anywhere in our configuration.

### Implementation of sops-nix in home-manager

The procedure is very similar. We only have to make the module available for the **home-manager** configuration:

```nix
{
  # Configuration via home.nix
  imports = [
    inputs.sops-nix.homeManagerModules.sops
  ];
}
```

### User-Level Secrets Definition for home-manager

First, we will update the `.sops.yaml` file with a new path for the user **secrets**:

```yaml
keys:
  - &user_alice 2504791468b153b8a3963cc97ba53d1919c5dfd4
  - &host_delta age12zlz6lvcdk6eqaewfylg35w0syh58sm7gh53q5vvn7hd7c6nngyseftjxl
creation_rules:
  - path_regex: secrets.(yaml|json|env|ini)$
    key_groups:
    - pgp:
      - *user_alice
      age:
      - *host_delta

  - path_regex: home/secrets.(yaml|json|env|ini)$
    key_groups:
    - pgp:
      - *user_alice
```

Now we can generate the **secrets** file for use from home-manager using **sops**:

```bash
nix-shell -p sops --run "sops home/secrets/secrets.yaml"
```

Again, the first time we open the file, it will create some example data. We will delete them and define our first user-level secret:

```yaml
home_manager_secret: my_first_secret
```

As we saw in the previous case, once the file is saved and closed, it will be fully encrypted.

### Accessing Secrets from home-manager

With the module now available, we will add the following configuration to our `home.nix`:

```nix
sops.gnupg.home = "~/.gnupg";
sops.gnupg.sshKeyPaths = [ ];
sops.defaultSymlinkPath = "/run/user/1000/secrets";
sops.defaultSecretsMountPoint = "/run/user/1000/secrets.d";

```

These declarations allow [sops-nix](https://github.com/Mic92/sops-nix) to know where to find the private keys we are going to use, indicating also where the secrets will be dumped in the system.

To access the secrets from the configuration, we will add the following definition:

```nix
};
sops.secrets.home_manager_secret = {
  sopsFile = ./secrets/secrets.yaml;
};

```

To test access from the home-manager configuration, we will create a script that accesses said secret and returns its value:

```nix
{ pkgs, config }:

let
  secret = config.sops.secrets.home_manager_secret.path;
in

pkgs.writeShellScriptBin "hello-world" ''
  echo "Hello World $(cat ${secret})";
''
```

We will add this script to our configuration as a package:

```nix
{ config, pkgs, ... }:

with pkgs; [
  (import ../bin/hello-world.nix { inherit pkgs config; })
]
```

If we execute `hello-world` from the console, we can verify that the secret has been accessed during the build phase, and its value has been hardcoded into the script.

Regarding **home-manager**, there is a caveat. The reading of the secrets is done through a service defined by the [sops-nix](https://github.com/Mic92/sops-nix) module itself. If any secret is changed, the service must be restarted once the new user generation is obtained:

```bash
systemctl --user restart sops-nix.service
```

## Reflection

Keeping your information secure can be crucial from both a privacy and security standpoint. You decide which secrets to expose and which not, it's your decision. There's always room for further security measures, encrypting what's already encrypted for an additional layer of protection. In future posts, we'll see how we can do this using [git-crypt](https://github.com/AGWA/git-crypt).

---
Links of Interest
- [Mic92/sops-nix: Atomic secret provisioning for NixOS based on sops](https://github.com/Mic92/sops-nix)

