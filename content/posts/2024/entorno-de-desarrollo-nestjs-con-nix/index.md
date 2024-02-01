---
title: "Entorno de desarrollo para Nestjs con Nix"
canonicalURL: https://ajmasia.me/posts/2024/entorno-de-desarrollo-nestjs-con-nix
author: Antonio José Masiá
tags:
    - nix
date: 2024-02-01
draft: false
---

Una de las ventajas de usar [Nix](https://nixos.org/) es que nos permite crear entorno personalizados y totalmente aislados para poder hacer desarrollo para plataformas específicas. La ventaja que esto tiene es que te evitas de tener que instalar en tu sistema todo aquello que necesitas para desarrollar. Esto puede tener el alcance que quieras.

En este caso voy a mostrarte como crear un entorno desarrollo para aprendizaje de Nest.js que podría mejorarse includo para el despliegue de aplicaciones en producción.

Para ello comenzamos por crear el fichero `flake.nix` dentro del directorio del proyecto con una configuración básica. Podemos hacerlo de forma manual u usar el comando `nix flake init`. Si usas el comando obtendrás un flake con este aspecto:

```nix
{
  description = "A very basic flake";

  outputs = { self, nixpkgs }: {

    packages.x86_64-linux.hello = nixpkgs.legacyPackages.x86_64-linux.hello;

    packages.x86_64-linux.default = self.packages.x86_64-linux.hello;

  };
}
```

Manos a la obra!

## Estrctura general del flake

### description

Establece una descripción para el flake, esto sirve a modo de documentación a explicar brevemente para que se usa de forma específica.

### outputs

Es una función que define la salida que producirá el flake. Dicha podrá recibir ciertos parámetros. El más común es `nixpkgs` que representa una referencia a los paquetes disponibles de Nix.

```nix
{
  description = "Nest.js dev environment";

  outputs = { }: { }
}
```

## Definición del outputs

Vamos a necesitar devolver un shell específico con una serie de dependencias que nos permitan desarrollar en Node.js y Nest.js. Además necesitaremos algunas herramientas extra como por ejemplo el `nest-cli` o `httpie` para poder hacer peticiones http desde la consola.

### Estructura del output
Definiremos ese shell de la siguiente forma:

```nix
{
  description = "Nest.js dev environment";

  outputs = { nixpkgs }:
    let
      pkgs = nixpkgs.legacyPackages.x86_64-linux;
    in
    { 
      devShell.x86_64-linux = pkgs.mkShell {
        name = "dev-env";
        buildInputs = with pkgs; [ ];
        shellHook = '''';
      };
    };
}
```

La expresión `let ... in` nos permite definir o hacer asignaciones de variables locales. En este caso definimos `pkgs` como los paquetes disponibles, en este caso para la plataforma Linux x86_64.

 
### Shell de Desarrollo (devShell.x86_64-linux):

Se usa mkShell para crear un entorno de shell de desarrollo. 

- `name` define el nombre que tendrá el shell
- `builtInputs` define los paquetes que estarán disponibles en el shell
- `shellHook` se ejecutará cada vez que se ejecute el shell. Esta propiedad nos permitirá hacer configuraciones previas o mostrar información necesaria para el usuaria

```nix
{
  description = "Nest.js dev environment";

  outputs = { self, nixpkgs }:
    let
      pkgs = nixpkgs.legacyPackages.x86_64-linux;

      yarnWithNode20 = pkgs.yarn.overrideAttrs (oldAttrs: rec {
        buildInputs = with pkgs; [
          nodejs_20
        ];
      });
    in
    {
      devShell.x86_64-linux = pkgs.mkShell {
        name = "dev-env";
        buildInputs = with pkgs; [
          lolcat
          tmux
          yarnWithNode20
          nodejs_20
          httpie
          nest-cli
        ];
        shellHook = '' '';
      };
    };
}
```

En nuestro caso, hemos definido las siguientes dependencias para el shell:

- [lolcat](https://github.com/busyloop/lolcat) para poder hacer highlight de textos en la consola
- [tmux](https://github.com/tmux/tmux/wiki) como *terminal multiplexer*
- [Yarn](https://classic.yarnpkg.com/lang/en/) como gestgor de paquetes. Observa como se ha creado una variable específica denominada `yarnWithNode20` que preconfigura *yarn* para usarse con esta versión de Node
- La propia versión de Node.js que queremos usar
- [HTTPie](https://httpie.io/) para poder hacer peticiones *http* desde la consola
- y por último el CLI para [NestJS](https://nestjs.com/)

## Mejoras

Vamos a incluir el uso una configuración básica para tmux y un mensaje de bienvenida para cuando iniciemos el shell. Todo esto lo haremos usando la propiedad `shellHook`:

```nix
{
  description = "Nest.js dev environment";

  outputs = { self, nixpkgs }:
    let
      pkgs = nixpkgs.legacyPackages.x86_64-linux;

      welcomeMessage = pkgs.writeShellScript "welcome_message" ''
        #!/usr/bin/env bash
        clear
        echo "Welcome to the $(node --version) environment!" | lolcat
        echo "Nest CLI version: $(nest --version)" | lolcat
        echo "****************************************"
        echo -e "https://nodejs.org/docs/latest-v20.x/api/"
        echo -e "https://nixos.wiki/wiki/Node.js"
        echo -e "https://nixos.wiki/wiki/Development_environment_with_nix-shell#direnv"
        echo "*********************************************************************"
      '';

      tmuxSessionName = "dev-env";

      yarnWithNode20 = pkgs.yarn.overrideAttrs (oldAttrs: rec {
        buildInputs = with pkgs; [
          nodejs_20
        ];
      });
    in
    {
      devShell.x86_64-linux = pkgs.mkShell {
        name = "dev-env";
        buildInputs = with pkgs; [
          lolcat
          tmux
          yarnWithNode20
          nodejs_20
          httpie
          nest-cli
        ];
        shellHook = ''
          # Start tmux seesion if not already in one
          if [ -z "$TMUX" ]; then
            tmux new-session -d -s ${tmuxSessionName}
            tmux rename-window "code"
            tmux send-keys "bash ${welcomeMessage}" C-m
            tmux new-window -n "server"

            tmux select-window -t dev-env:code
            tmux attach-session -d -t ${tmuxSessionName}
          else
            bash ${welcomeMessage}
          fi
        '';
      };
    };
}
```

La variable `welcomeMessage` define el mensaje que se mostrara al entran en el shell.

En la variable `tmuxSessionName` definiremos el nombre que usará la sessión de tmux. Esto nos evita tener que cambiarla en varios sitios a la vez en el caso que decidamos renombrarla.

Ya en el `shellHook` definimos todo lo que se ejecutará al iniciar el sehll. En este caso se inicia una sesión de tmux con dos ventanas, la primera denominada `code` para desarrollar y la segunda `server` para poder arrancar el proyecto, ejecutar comandos de consola, etc.

## Probando el shell

Ejecutamos el comando `nix develop`.

Este flake pude servirte de guía para crear tus propios entorno de desarrollo sin límite. Espero que te haya resultado útil.

---
Enlaces de interés
- [Nix & NixOS | Reproducible builds and deployments](https://nixos.org/)
- [Download Nix / NixOS | Nix & NixOS](https://nixos.org/download#download-nix)
- [Nix Reference Manual](https://nixos.org/manual/nix/stable/introduction)
