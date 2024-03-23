<!-- headingDivider: 2 -->

# NixOSの話

## Nixの復習

- Nixパッケージマネージャ
- Nixパッケージマネージャのパッケージレシピを記述するためのNix言語
- Nixではソースコード, パッチ, 中間ビルド生成物, 設定ファイル, derivation, そしてパッケージは全てパッケージとして扱われる。
- パッケージの同一性はコンテンツから計算したハッシュによって決定される。
- 明示的にインストールされたパッケージや依存パッケージはNixストア内にフラットに格納される。
- 明示的にインストールされたパッケージはプロファイルにシンボリックリンクが作成される。
- 有効なプロファイルは特定のパスにシンボリックリンクが作成され、そのシンボリックリンクを指す環境変数 (`PATH`, `LD_LIBRARY_PATH`) を設定することで使用可能になる。

## NixOSとは

- NixのエコシステムをベースとしたLinuxのディストリビューションの1つ
- 環境をNixパッケージとし、Nix言語で宣言的に管理する。(以降、環境をNixパッケージで表現したものをsystemと呼ぶ)
- Nixパッケージのビルドは再現性がある (ことを期待されているので)、宣言した設定から環境を再現可能。
- そのため、systemがGCに回収されていなければ、任意の時点の環境に戻すことも可能。

## NixOSの設定例

`/etc/nixos/configuration.nix`:

```nix
{ config, pkgs, ... }: {
  imports = [ ./hardware-configuration.nix ];

  # systemd-boot を有効化
  boot.loader.systemd-boot.enable = true;

  # OpenSSHサービスを有効化
  services.sshd.enable = true;
}
```

## NixOSの設定から環境をビルド

以下を実行すると `/etc/nixos/configuration.nix` に記述した設定からsystemをビルドする。

```sh
sudo nixos-rebuild build
```

## NixOSの設定からビルドした環境に切替え

以下を実行すると、現在の環境をビルドした環境に切り替える。
また、`/etc/nixos/configuration.nix` からsystemがビルドされていない、あるいはsystemがoutdatedであれば、自動的にリビルドを行う。
そのため、基本的には `switch` だけ覚えておけばよい。

```sh
sudo nixos-rebuild switch
```

<!-- TODO: `nixos-rebuild switch` の実行例 -->
<!-- TODO: OpenSSHサービスが起動しているかの確認 -->

## NixOSはどのように環境を更新するのか

- 先に結論: **Nixパッケージマネージャ + アクティベーションスクリプト**
- NixパッケージのビルドはサンドボックスとNixストアの外に副作用をもたないため、実際の環境を更新するための何かが必要。
- その何かが**アクティベーションスクリプト**。

## アクティベーションスクリプト

- 実体はシェルスクリプト。
- 環境を切り替える際に、新しい環境のsystemに含まれるアクティベーションスクリプトが実行されることによって、実際の環境が更新される。
- シェルスクリプトなので冪等性は保証されておらず、自力で冪等になるように設計する必要がある。
- とはいえ、自分でアクティベーションスクリプトを書く機会はほぼない。

## NixOSモジュール

- Nix言語上でアクティベーションスクリプトあるいは他のNixOSモジュールをラップしたもの。
- NixOSモジュールを通して環境の設定を宣言的に記述できる。その冪等性はNixOSモジュールの裏のアクティベーションスクリプトが担保する。
- 設定例の `boot.loader.systemd-boot.enable` や `services.sshd.enable` はNixOSモジュールによって提供されたオプション。

  ```nix
  boot.loader.systemd-boot.enable = true;
  ```

  ```nix
  services.sshd.enable = true;
  ```

## NixOSモジュールの仕組み

`options` にモジュールのオプションを定義する。


```nix
{ lib, config, pkgs }:

{
  options = {
    services.openssh = {
      enable = mkOption {
        type = types.bool;
        default = false;
        description = lib.mdDoc ''
          Whether to enable the OpenSSH secure shell daemon, which
          allows secure remote logins.
        '';
      };

      ports = mkOption {
        type = types.listOf types.port;
        default = [22];
        description = lib.mdDoc ''
          Specifies on which ports the SSH daemon listens.
        '';
      };
    };
  };
}
```

## NixOSモジュールの仕組み

`config` にモジュールの設定を定義する。
`options` のスキーマと `config` のスキーマは対応している。
モジュールの設定から別のモジュールを設定するか、アクティベーションスクリプトを実行することによってモジュールの機能を実現する。

```nix
{ lib, config, pkgs }:

{
  # enable オプションが true だった場合のみ設定する
  config = mkIf config.services.openssh.enable {
    # etc モジュールを設定して /etc/ssh/sshd_config を作成する
    environment.etc."ssh/sshd_config".text = ''
      ${lib.concatMapStrings (port: "Port ${toString port}\n") config.services.openssh.ports}
    '';
  };
}
```

## NixOSモジュールの仕組み

```nix
{ lib, config, pkgs }:

{
  config = mkIf config.services.openssh.enable {
    # systemd モジュールを設定して sshd.service を有効化する
    systemd.service.sshd = {
      description = "SSH Daemon";
      wantedBy = "multi-user.target";
      after = [ "network.target" ];
      serviceConfig = {
        ExecStart = "${pkgs.openssh}/bin/sshd";
        KillMode = "process";
        Restart = "always";
        Type = "simple";
      };
    };
  };
}
```

## NixOSモジュールの仕組み

`system.activationScripts.<任意のキー>` に記述されたスクリプトがsystemのアクティベーションスクリプトとして実行される。

etcモジュールはアクティベーションスクリプトで `setup-etc.pl` を呼び出し、Nixストア内に再現した `/etc` 以下の設定ファイルの内容を実際の `/etc` 以下に再現する。

```nix
{
  system.activationScripts.etc = ''
    # Set up the statically computed bits of /etc.
    echo "setting up /etc..."
    ${pkgs.perl.withPackages (p: [ p.FileSlurp ])}/bin/perl ${./setup-etc.pl} ${etc}/etc
  '';
}
```
