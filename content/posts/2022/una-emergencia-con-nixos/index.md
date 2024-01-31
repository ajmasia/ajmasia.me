---
title: "Una emergencia con NixOS"
canonicalURL: https://ajmasia.me/posts/2022/una-emergencia-con-nixos
author: Antonio José Masiá
tags:
    - nixos
date: 2022-11-06
draft: false
---

## Antecedentes

Esta tarde haciendo algo de limpieza en mi equipo del trabajo, vi que me quedaba poco espacio en la partición del sistema, y es que una de las ventajas de NixOS es que cada vez que vas  generando cambios en la configuración, se van quedando dependencias enlazadas para en cualquier momento poder volver a ellas según sea necesario. El caso es que accidentalmente acabé eliminando todas los ficheros del boot loader del sistema que apuntan a cada una de estas generaciones.

Mi intención era sólo quedarme con la última y sin embargo las eliminé todas. La solución hubiera sido sencilla en ese mismo momento de haberme dado cuenta de que esto había ocurrido: sudo nixos-rebuild switch. Sin embargo de forma anticipada, reinicie el sistema y .... momento pánico, no aparecía ninguna versión del sistema por lo que era imposible arrancar.

> No hay nada como dejar reposar las cosas para verlas con perspectiva.

Dada la situación y tras intentar recurrir a varios recursos sin fortuna, decidí dejarlo temporalmente con la tranquilidad que sabía que encontraría la solución y que el sistema seguía estando ahí.

## Fase de enfriamiento 

Cuando te quedas atascado en algo y no paras de dar vueltas sin resolver nada, lo mejor es enfriar. Tras este periodo de calma, me puse a leer documentación al respecto y enseguida vi la luz. La solución era aparentemente sencilla, volver a construir una generación desde la configuración existente en la partición. Y, cómo hacer esto cuando no puedes acceder a ninguna TTY del sistema? Afortunadamente tenemos los Live USB.

## Guía de resolución

1. En primer lugar debes disponer un USB con la imagen de NixOS. En mi caso siempre tengo una con la instalación minimal. Me gusta usar la terminal ;-)
2. Una vez arrancado el Live USB y para evitar tener que estar usando constantemente sudo, ejecutaremos sudo -i
3. Ahora necesitamos conexión a internet para que NixOS pueda traerse todo aquello que necesite para construir una nueva generación del sistema. Para ello seguiremos los siguientes pasos:

```bash
# habilitamos la conexión wifi
systemctl start wpa_supplicant

# conectamos a nuestra red wifi mediante el cli
wpa_cli

# ejecutamos la siguiente secuencia de comandos
add_network
set_network 0 ssid "myhomenetwork"
set_network 0 psk "mypassword"
set_network 0 key_mgmt WPA-PSK
enable_network 0

# salimos del cli y verificamos nuestra conexión a la red
ping 8.8.8.8
```

4. Procedemos amontar nuestras particiones, en mi caso:

```bash
mount /dev/nnvm0n1p1 /mnt/boot
mount /dev/nnvm0n1p3 /mnt
mount /dev/nnvm0n1p4 /mnt/home
swapon /dev/nvm0n1p2
```

5. Ahora ya podemos entra en nuestro sistema ejecutando $(which nixos-enter). Este comando nos permite acceder a una instalación de NixOS desde el Live USB.
6. Una vez dentro nos dirigiremos a nuestro usuario su <user> y sencillamente ejecutaremos un `sudo nixos-rebuid switch` para poder reconstruir nuestro sistema y por tanto disponer de una nueva generación, y magia .....
Una vez construida esta nueva generación del sistema, volveremos a tener acceso desde el boot loader.

## Aprendizajes

> Las experiencias sin aprendizajes resulta inútiles.

En este caso hubiera sido muy útil haber conocido con claridad las consecuencias de un error en la operación que estaba realizando. De haber sido así la cautela hubiera brillado mucho más. Cuando se trata de operaciones delicadas del sistema toda precaución es poca. Afortunadamente NixOS no es cualquier distribución de Linux y cuenta con solución para cualquier problema ;-).

Tras este episodio, me reafirmo en que NixOS es la distribución Linux que más me ha sorprendido con diferencias. Mi intención es seguir profundizando e ir contándote poco a poco mis experiencias y aprendizajes. Mientras tantos por si te anima, te dejo por aquí enlace hacia la documentación de NixOS por si te pica la curiosidad :-).
