---
title: "Nestjs dev environment using Nix"
summary: "Learn how to generate a scalable and reproducible development environment for Nestjs using Nix flakes"
canonicalURL: https://ajmasia.me/en/posts/2024/nix-entorno-de-desarrollo-nestjs
author: Antonio José Masiá
tags:
    - nix
date: 2024-02-01
draft: false
cover:
  image: https://plus.unsplash.com/premium_photo-1677702162621-f61d44e676eb?q=80&w=1470&auto=format&fit=crop&ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D
  hiddenInList: true
  hiddenInSingle: false
---

One of the advantages of using [Nix](https://nixos.org/) is that it allows us to create customized and completely isolated environments for development on specific platforms. The advantage of this is that it saves you from having to install everything you need for development on your system. This can be as extensive as you want.

In this case, I'm going to show you how to create a development environment for learning Nest.js that could even be improved for deploying applications in production.

To do this, we start by creating a `flake.nix` file in the project directory with a basic configuration. We can do this manually or use the command `nix flake init`. If you use the command, you will get a flake that looks like this:

```nix
{
  description = "A very basic flake";

  outputs = { self, nixpkgs }: {

    packages.x86_64-linux.hello = nixpkgs.legacyPackages.x86_64-linux.hello;

    packages.x86_64-linux.default = self.packages.x86_64-linux.hello;

  };
}
```

Let's get started!

## Flake estructure

### description

Sets a description for the flake, serving as documentation to briefly explain its specific use.

### outputs

It is a function that defines the output the flake will produce. This output can receive certain parameters. The most common is `nixpkgs`, which represents a reference to the available Nix packages.


```nix
{
  description = "Nest.js dev environment";

  outputs = { }: { }
}
```

## Outputs definition

We'll need to return a specific shell with a set of dependencies that allow us to develop in Node.js and Nest.js. Additionally, we'll need some extra tools such as `nest-cli` or `httpie` for making HTTP requests from the console.

### Output structure
We will define that shell as follows:


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

The `let ... in` expression allows us to define or assign local variables. In this case, we define `pkgs` as the available packages, in this case for the Linux x86_64 platform.

### Dev shell (devShell.x86_64-linux):

`mkShell` is used to create a development shell environment.

- `name` defines the name of the shell.
- `builtInputs` defines the packages that will be available in the shell.
- `shellHook` is executed every time the shell is run. This property allows us to make previous configurations or show necessary information for the user.


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

In our case, we have defined the following dependencies for the shell:

- [lolcat](https://github.com/busyloop/lolcat) for highlighting texts in the console.
- [tmux](https://github.com/tmux/tmux/wiki) as a *terminal multiplexer*.
- [Yarn](https://classic.yarnpkg.com/lang/en/) as a package manager. Notice how a specific variable called `yarnWithNode20` has been created, which preconfigures *yarn* to be used with this version of Node.
- The specific version of Node.js we want to use.
- [HTTPie](https://httpie.io/) for making *http* requests from the console.
- and finally, the CLI for [NestJS](https://nestjs.com/).

## Improvements

We're going to include the use of a basic configuration for tmux and a welcome message for when we start the shell. We will do all this using the `shellHook` property:


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
The `welcomeMessage` variable defines the message that will be displayed when entering the shell.

In the `tmuxSessionName` variable, we define the name that will be used for the tmux session. This prevents us from having to change it in several places at once if we decide to rename it.

In the `shellHook`, we define everything that will be executed when starting the shell. In this case, a tmux session is started with two windows, the first named `code` for development, and the second `server` for starting the project, executing console commands, etc.

## Testing the Shell

Run the command `nix develop`.

This flake can serve as a guide to create your own development environments without limits. I hope you have found it useful.

---
Links of Interest
- [Nix & NixOS | Reproducible builds and deployments](https://nixos.org/)
- [Download Nix / NixOS | Nix & NixOS](https://nixos.org/download#download-nix)
- [Nix Reference Manual](https://nixos.org/manual/nix/stable/introduction)

