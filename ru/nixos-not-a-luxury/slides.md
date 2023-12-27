<img src="nixos_logo.svg" width="30%" height="30%">

## NixOS

Не роскошь, а новый подход к построению дистрибутивов Linux

###### Иван Бущик "ivabus", 2023

---

## Что такое "обычный" дистрибутив?

---

## Что такое NixOS?

Декларативный, атомарный и воспроизводимый дистрибутив Linux

Создан на базе менеджера пакетов Nix и коллекции Nixpkgs <!-- .element: class="fragment" data-fragment-index="1" -->

--

## Что такое Nix?

Пакетный менеджер? Функциональный язык?

Всё и сразу! <!-- .element: class="fragment" data-fragment-index="1" -->

--

## О Nixpkgs

- Две ветки (unstable и release)
- 80000+ пакетов
- Содержит "суть" NixOS - основные модули и опции
- Не только Linux (Darwin)

--

## Каналы Nixpkgs

- Стабильные / нестабильные (23.11, 23.04, unstable)
- По платформе (NixOS - nixos-\*, Darwin - nixpkgs-\*-darwin)
- По времени обновления (обычные, small)

---

## Nix как пакетный менеджер

- Использует Nixpkgs как "репозиторий"
- При необходимости собирает пакет на месте
- Является интерпретатором языка Nix
- Написан на C++

--

## Установка без установки

`nix-shell -p <package>`

---

## О конфигурации NixOS

--

Система целиком описывается конфигурационным файлом(-ами).

Множество *разных* систем могут быть заданы через единую конфигурацию.

--

## Что можно описывать конфигурацией

- Установленное ПО
- Настройки служб
- Настройки сети
- Параметры загрузчика / ядра
- Пространство пользователя (home-manager)

--

## Минимальная конфигурация NixOS

```nix
{
  boot.loader.grub.device = "/dev/sda";
  fileSystems."/".device = "/dev/sda1";
}
```

--

## А если не минимально?

```nix[1-18|1|3|4-6|7-11|12|13-17|16]
{ config, pkgs, ... }:
{
  imports = [ ./hardware-configuration.nix ];
  environment.systemPackages = with pkgs; [
    firefox
  ];
  services.xserver = {
    enable = true;
    displayManager.sddm.enable = true;
    desktopManager.plasma5.enable = true;
  };
  services.sshd.enable = true;
  users.mutableUsers = false;
  users.users.root = {
    hashedPassword = null;
    openssh.authorizedKeys.keys = [ "ssh-ed25519 AAAaC1l....VfR user@host" ];
  };
}
```

--

## Сеть

```nix[1-19|1-6|8-14|16-19]
networking = {
  hostname = "hostname";
  useDHCP = true;
  nameservers =
    [ "1.0.0.1#cloudflare-dns.com" "8.8.8.8#dns.google" ];
};

services.resolved = {
  enable = true;
  dnssec = "false";
  extraConfig = ''
    DNSOverTLS=yes
  '';
};

services.avahi = {
  enable = true;
  nssmdns = true;
}
```

---

## Модульность конфигурации


```nix[1-19|4-5|2,6|7-14|16-17]
{ config, lib, pkgs, ... }:
let cfg = config.my.roles.server.nginx;
in {
  options.my.roles.server.nginx.enable =
    lib.mkEnableOption "Initial nginx setup";
  config = lib.mkIf (cfg.enable) {
    services.nginx = {
      enable = true;
      package = pkgs.nginxQuic;
      recommendedGzipSettings = true;
      recommendedOptimisation = true;
      recommendedProxySettings = true;
      recommendedTlsSettings = true;
    };
    security.acme = { acceptTerms = true; defaults.email = "user@host"; };
    networking.firewall.allowedTCPPorts = [ 80 443 ];
    networking.firewall.allowedUDPPorts = [ 80 443 ];
  };
}
```

--

```nix[]
{ pkgs ? import <nixpkgs> { }, ... }:
let
  version = "0e82fce377c71b5535448ec00243a580e235393d";
  repo = builtins.fetchGit {
    url = "https://github.com/ivabus/slides.ivabus.dev";
    rev = version;
  };
in pkgs.stdenv.mkDerivation {
  inherit version;
  name = "slides-ivabus-dev";
  src = repo;

  buildInputs = [ ];
  nativeBuildInputs = with pkgs; [ rsync ];

  installPhase = ''
    mkdir -p $out
    rsync -a . $out/ --exclude README.md --exclude LICENSE
  '';
}

```

--

```nix[1-20|2,4-6|7|8-18|13|14-16]
{ config, lib, pkgs, ... }:
let cfg = config.my.roles.server.slides-ivabus-dev;
in {
  options.my.roles.server.slides-ivabus-dev.enable =
    lib.mkEnableOption "Serve slides.ivabus.dev";
  config = lib.mkIf (cfg.enable) {
    my.roles.server.nginx.enable = true;
    services.nginx = {
      virtualHosts."slides.ivabus.dev" = {
        forceSSL = true;
        enableACME = true;
        http3 = true;
        root = pkgs.callPackage ../../pkgs/slides-ivabus-dev.nix { };
        extraConfig = ''
          error_page 404 /404.html;
        '';
      };
    };
  };
}
```

--

## Корень конфигурации

```nix[1-19|8|1,10,14]
{ config, pkgs, lib, secrets, ... }:

let
  my = import ../..;
in {
  imports = [ my.modules ./hardware.nix ];

  my.roles.server.slides-ivabus-dev.enable = true;

  my.features.secrets = true;

  services.nginx = {
    virtualHosts."music.ivabus.dev" = {
      locations."/".proxyPass = "http://${secrets.maas-address}:4533";
      enableACME = true;
      forceSSL = true;
      http3 = true;
    };
  };
}

```

---

## Хранение секретиков

```nix[]
{ config, ... }:
let
  canaryHash = builtins.hashFile "sha256" ./secrets/canary;
  expectedHash =
    "bc6f38a927602241c5e0996b61ebd3a90d5356ca76dc968ec14df3cd45c6612c";
in if (canaryHash != expectedHash && config.my.features.secrets) then
  abort
    "Secrets are enabled and not readable. Have you run `git-crypt unlock`?"
else {
  maas-address = builtins.readFile ./secrets/maas-address;
}
```

--

## Подключение модуля

```nix[]
let
  my = import ../.;
  secrets = my.secrets { inherit config; };
{
  _module.args.secrets = secrets;
}
```

--

## Взаимодействие с секретиками

```nix[|1,6,12,13]
{ config, lib, secrets, ... }:
let cfg = config.my.roles.yggdrasil-peer;
in {
  options.my.roles.yggdrasil-peer.enable = lib.mkEnableOption "Enable ...";
  config = lib.mkIf (cfg.enable) {
    my.features.secrets = lib.mkForce true;
    my.roles.yggdrasil-client.enable = true;
    services.yggdrasil = {
      settings = {
        Peers = lib.mkForce [];
        Listen = [
          "quic://[::]:60003?password=${secrets.yggdrasil-password}"
          "tls://[::]:60002?password=${secrets.yggdrasil-password}"
        ];
      };
    };
    networking.firewall.allowedTCPPorts = [ 60002 ];
    networking.firewall.allowedUDPPorts = [ 60003 ];
  };
}
```

---

## Спасибо за внимание

### Моя конфигурация

<img alt="https://ивабус.рф/nixos" src="qr.svg" style="max-width: 30%; width: 300px; height:300px;"/>