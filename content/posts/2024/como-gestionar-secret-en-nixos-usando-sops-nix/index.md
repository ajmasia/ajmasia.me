---
title: "Cómo gestionar secrets en NixOS usando sops-nix"
summary: "Aprende cómo gestionar los secretes de tus configuraciones de Nixos con sops-nix, un módulo atómico, declarativo y facilmente reproducible basado en sops."
canonicalURL: https://ajmasia.me/posts/2024/como-gestionar-secrets-en-nixos-usando-sops-nix
author: Antonio José Masiá
tags:
    - nix
    - nixos
date: 2024-02-06
draft: false
cover:
  image: https://plus.unsplash.com/premium_photo-1677702162621-f61d44e676eb?q=80&w=1470&auto=format&fit=crop&ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D
---

Una de las principales ventajas de usar [NixOS](https://nixos.org/) es que nos permite tener un sistema operativo altamente reproducible. Esto se debe, en mayor medida, a que toda la configuración se hace de forma declarativa usando [Nix](https://nixos.org/manual/nix/stable/installation/installing-binary). En toda configuración, ya sea para una máquina para uso profesional, personal, servidor o servicio, acabamos teniendo que gestionar *tokens*, *variables de entorno*, o bien configuraciones con información personal, apis, etc, que como es lógico queremos mantener de forma privada.

Hasta ahora había estado gestionando esta información de forma local sin integrarla en el repositorio de mis configuraciones, lo cual es un verdadero incordio ya que te obliga a mantener backups de dicha información para poder restaurar en caso de algún problema. Me puse a investigar y pude ver que una gran parte de la comunidad suele usar [sops-nix](https://github.com/Mic92/sops-nix). Existen otras alternativas aunque la verdad, [sops-nix](https://github.com/Mic92/sops-nix) parace estable y sobre todo escalable por lo que he decidido implementarla en mis configuraciones para hacer algunas pruebas y de esa forma poder sacar conclusiones.

Puedes implementar [sops-nix](https://github.com/Mic92/sops-nix) tanto en la configuración de NixOS como a nivel de usuario usando [home-manager](https://github.com/nix-community/home-manager).

## Cómo funciona sops-nix

[Sops-nix](https://github.com/Mic92/sops-nix) nos permite encriptar nuestros **secrets** dentro de la definición de nuestra configuración para posteriormente desencriptarlos en la fase de build. Esto permite definirlos de fomra declarativa. 

Para poder hacer esto, usa por lo general claves de tipo [age](https://github.com/FiloSottile/age) o [gpg](https://www.gnupg.org/). En el caso que voy a explicar usaré dos claves, una para el usario y normalmente la clave SSH de nuestra máquina (esta suele habilitarse con el servicio SSH). Como veremos, a nivel de home-manager, con tan sólo la clave de usuario será suficiente. También podría usarse una sola clave de tipo [age](https://github.com/FiloSottile/age).

Al quedar nuestros secrets totalmente encriptados, podremos subirlos de forma privada, tanto a nuestro repositorio de dotfiles como al hacer cualquier tipo de despligues en servidores, etc. Como veremos es bastante sencillo.

## Implementación

### Generación de claves principales

Podremos usar tanto claves de tipo **age** como claves **gpg**.

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

Una vez generadas la clave de usuario tendremos que obtener las claves públicas para poder configurar [sops](https://github.com/getsops/sops). Si estas usando una clave PGP, necesitaras el **fingerprint**. En el caso de usar una clave **age** puedes ver directamente la clave pública visualizando el fichero de la clave o bien anotarla tras su creación.

> IMPORTANTE: Si usas una clave GPG defínela sin password, de lo contrario cuando se haga el build del sistema fallará.

### Obtención de la clave pública de la clave SSH de nuestra máquina

Podemos obtener esta clave ejecutando el siguiente comando:

```bash
nix-shell -p ssh-to-age --run 'cat /etc/ssh/ssh_host_ed25519_key.pub | ssh-to-age'
age1rgffpespcyjn0d8jglk7km9kfrfhdyev6camd3rck6pn8y47ze4sug23v3
```

### Configuración de sops y definción de los scretes 

Crearemos un fichero denominado `.sops.yaml` en la raíz de nuestra configuración donde definiremos la ruta hacia nuestros **secrets** y las claves públicas que permitirán llevar a cabo el proceso de encriptado/desencriptado:

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

Si todo esta bién definido, pasaremos a definir nuestros **secrets**. Para ello usaremos la herramienta [sops](https://github.com/getsops/sops). Cuando abramos nuestro fichero de secrets por primera vez, es decir. al crearlo, [sops](https://github.com/getsops/sops) se encargará de desencriptar su contenido para que podamos verlo y definirlo/modificarlo. Cuando lo cerremos volverá a encriptar su contenido usando nuestras claves.

```bash
nix-shell -p sops --run "sops secrets/secrets.yaml"
```

La primera vez que abrmos el fichero nos crea unos datos de ejemeplo. Los borraremos y definiremos nuestro primer secret a nivel de sistema operativo:

```yaml
nisxos_secret: my_first_secret
```

Una vez guardado el fichero, si lo abrimos directamente con cualquier editor, podremos comprobar que está encriptado:

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

### Implementación de sops-nix en NixOS

La implementación ideal es usando el *flake* disponible. Lo añadiremos como input a nuestro flake de configuración y luego lo importaremos como módulo en NixOS:

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

### Acceso a los secretos desde nuestra configuración

Con el módulo ya disponible, añadiremos la siguiente configuración a nuestro `configuration.nix`:

```nix
{
  sops.gnupg.home = "~/.gnupg";
  sops.gnupg.sshKeyPaths = [ ];
  sops.age.sshKeyPaths = [ "/etc/ssh/ssh_host_ed25519_key" ];
}
```

Estas declaraciones lo que permiten que [sops-nix](https://github.com/Mic92/sops-nix) sepa donde encontrar las claves privatas para los procesos de desencriptado.

Para acceder a los secrets desde la configuración añadiremos la siguiente definición:

```nix
{
  sops.secrets.nixos_secret = {
    sopsFile = ./secrets.yaml;
  };
}

```

En esta definición le estamos diciendo a NixOS que haga disponible el secret **nixos_secret** para nuestra configuración quedadno accesible desde la propiedad `config.soap.secrets.nixos_secret.path` que podremos usar en cualquier parte de nuestra configuración. 

### Implementación de sops-nix en home-manager

El procedimiento es muy similar. Tan sólo tendremos que hacer disponible el módulo para la configuración de **home-manager**:

```nix
{
  # Configuration via home.nix
  imports = [
    inputs.sops-nix.homeManagerModules.sops
  ];
}
```

### Definición de secretos a nivel de usuario para home-manager

En primer lugar actualizaremos el fichero `.sops.yaml` con un nuevo path para los **secrets** del usuario:

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

Ahora podremos generar el fichero de **screts** para su uso desde home-mangaer usando **sops**:

```bash
nix-shell -p sops --run "sops home/secrets/secrets.yaml"
```

De nuevo, la primera vez que habramos el fichero nos creará unos datos de ejemplo. Los borraremos y definiremos nuestro primer secret a nivel de usuario:

```yaml
home_manager_secret: my_first_secret
```

Como vimos en el caso anterior, una vez que se guarde y cierre el fichero quedará totalmente encriptado.

### Acceso a los secretos desde home-manager

Con el módulo ya disponible, añadiremos la siguiente configuración a nuestro `home.nix`:

```nix
{
  sops.gnupg.home = "~/.gnupg";
  sops.gnupg.sshKeyPaths = [ ];
  sops.defaultSymlinkPath = "/run/user/1000/secrets";
  sops.defaultSecretsMountPoint = "/run/user/1000/secrets.d";
}
```

Estas declaraciones lo que permiten que [sops-nix](https://github.com/Mic92/sops-nix) sepa donde encontrar las claves privatas que vamos a usar, indicándose además dónde se volcarán los secrets en el sistema.

Para acceder a los secrets desde la configuración añadiremos la siguiente definición:

```nix
{
  sops.secrets.home_manager_secret = {
    sopsFile = ./secrets/secrets.yaml;
  };
}
```

Para probar el acceso desde la configuración de home-manager vamos a crearnos un script que acceda a dicho secret y nos devuelva el valor:

```nix
{ pkgs, config }:

let
  secret = config.sops.secrets.home_manager_secret.path;
in

pkgs.writeShellScriptBin "hello-world" ''
  echo "Hello World $(cat ${secret})";
''
```

Añadiremos dicho script a nuestra configuración como paquete:

```nix
{ config, pkgs, ... }:

with pkgs; [
  (import ../bin/hello-world.nix { inherit pkgs config; })
]
```

Si ejecutamos desde la consola `hello-world` podremos comprobar que se ha accedido al secret en fase de build y sesu valor se ha quemado en el script.

Respecto a **home-manager** hay una salvedad. La lectura de los secretos se hace a través de un servicio que define el propio módulo de [sops-nix](https://github.com/Mic92/sops-nix). Si se cambia algún secret habrá que reinicar el servicio una vez obtenida la nueva generación del usuario:

```bash
systemctl --user restart sops-nix.service
```

## Refelexión

Mantener segura tú información puede resultar clave desde el punto de vista de la propia privacidad como de la seguridad. Tú decides que secrets expones y cuales no, es tu decisión. Siempre se puede volver a dar una vuleta de tuerca, encriptando lo ya encriptado para tener una nueva capa de protección. En próximos posts veremos cómo podemos hacer usando [git-crypt](https://github.com/AGWA/git-crypt).

---
Enlaces de interés
- [Mic92/sops-nix: Atomic secret provisioning for NixOS based on sops](https://github.com/Mic92/sops-nix)


