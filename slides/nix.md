<!-- headingDivider: 1 -->

# NixとNixOS

# Nixとは

- 2つ意味がある
  - Nix言語
  - Nixパッケージマネージャ

# Nix言語

- NixのビルドやNixOSの設定ファイルで使われるプログラミング言語
- **純関数型言語**
- **遅延評価**
- ここではNix言語の記法については触れない。

# 遅延評価

以下のスニペットで `z` の値を評価しようとするとエラーが発生するが、実際には `z` は評価されないため、以下のスニペットの式は正常に評価される。

```nix
let
  x = 1;
  y = 2;
  z = abort "error";
in
  x + y # => 3
```

# 文字列型とパス型

Nix言語には文字列型とは別にパスを表すパス型がある。

```nix
# 文字列リテラル
"foo"

# パスリテラル
./foo
```

# パス型

- パス型は記述したファイルが存在するディレクトリからの相対パスを表す。

  ```
  nix-repl> ./foo
  /path/to/foo
  ```

- パス型はstring interpolationによって文字列型に変換されるタイミングでそのパスが示すファイル or ディレクトリの存在がチェックされ、存在しない場合はその時点でエラーとなる。
  (パス型を文字列型に`builtins.toString` で変換した場合はチェックされない)

  ```
  nix-repl> "${./foo}"
  error: getting status of '/path/to/foo': No such file or directory
  ```

- 実はもう1つ重要な特徴があるが、後述する。

# `nix-repl`

`nix-repl` を使うと任意のNix言語の式がどのように評価されるのかを確認できる。

```sh
nix-repl
```

```
nix-repl> 1 + 2
3
```

# Nixパッケージマネージャ

- 実はNixOS以外でも使える。
- パッケージのレシピは**derivation**と呼ばれ、Nix言語の関数として記述される。

# サンドボックス

サンドボックスを有効にすると、Nixは隔離された環境でビルドを実行する。
これにより、ビルドスクリプトは `/usr/bin` などビルドの依存物以外を参照できなくなるため、ビルドプロセスが冪等性をもつようになる。(ことを期待する)

# サンドボックス

- Linuxではデフォルトで有効
- Linuxでフルサポートされ、macOSでは部分的にサポートされる。
- **ファイルシステムの隔離**
  - ビルド依存物と `sandbox-paths` オプションで指定したパス群のみアクセス可能。
  - `/proc`, `/dev`, `/dev/shm`, `/dev/pts` はホストから隔離される。
- **PID, マウント, ネットワーク, IPC, UTSの名前空間の隔離**
  - ネットワークの名前空間が隔離されるということは、ホストのインターネットアクセスが使えない。

参照: https://nixos.org/manual/nix/stable/command-ref/conf-file.html#conf-sandbox

# フェッチ処理

- Nixがサンドボックスで実行されている場合、derivationで実行されるbuilderなどからはインターネットにアクセスできない。
- 例外的に、`builtins.fetchurl` などの各種フェッチャーはハッシュが判明しているコンテンツであればインターネットからフェッチすることができる。
- ソースコードやパッチをフェッチする場合は、事前にそのファイルのハッシュを事前に計算しておく必要がある。

# フェッチ処理 (余談)

フェッチャー `fetch*` は単に `build-support` 下の `nix-prefetch-*` を動かしているだけだが、なぜこれらのプログラムはインターネットにアクセスできているのか、Nixpkgsのソースコードを読んだが分からなかった。誰か教えてほしい。

```sh
# Perform the checkout.
clone_user_rev "$tmpFile" "$url" "$rev"

# Compute the hash.
hash=$(nix-hash --type $hashType --base32 "$tmpFile")
# Add the downloaded file to the Nix store.
finalPath=$(nix-store --add-fixed --recursive "$hashType" "$tmpFile")

if test -n "$expHash" -a "$expHash" != "$hash"; then
    echo "hash mismatch for URL \`$url'. Got \`$hash'; expected \`$expHash'." >&2
    exit 1
fi
```

# フェッチ処理 (余談)

`clone_user_rev` 内部:

```sh
# Perform the checkout.
case "$rev" in
    HEAD|refs/*)
        clone "$dir" "$url" "" "$rev" 1>&2;;
    *)
        if test -z "$(echo "$rev" | tr -d 0123456789abcdef)"; then
            clone "$dir" "$url" "$rev" "" 1>&2
        else
            # if revision is not hexadecimal it might be a tag
            clone "$dir" "$url" "" "refs/tags/$rev" 1>&2
        fi;;
esac
```

# フェッチ処理 (余談)

`clone` 内部:

```sh
# Initialize the repository.
init_remote "$url"

# Download data from the repository.
checkout_ref "$hash" "$ref" ||
checkout_hash "$hash" "$ref" || (
    echo 1>&2 "Unable to checkout $hash$ref from $url."
    exit 1
)
```

# フェッチ処理 (余談)

`checkout_ref` 内部:

```sh
clean_git fetch ${builder:+--progress} --depth 1 origin +"$ref" || return 1
clean_git checkout -b "$branchName" FETCH_HEAD || return 1
```

# ハッシュの事前計算

事前計算したコンテンツのハッシュはSRI表記をすることが推奨されている。

# ハッシュの事前計算

コンテンツのハッシュを事前計算するには以下の方法がある:

1. `nix-prefetch-url` コマンドでハッシュを事前計算し、`nix hash convert` コマンドでSRI表記に変換する
1. 一時的に `lib.fakeHash` でダミーのハッシュを指定し、ハッシュ不一致のエラーメッセージから本物のハッシュを知る
1. `nurl` コマンドを使う

# `nix-prefetch-url` & `nix hash convert`

`nix-prefetch-url` を使うと、指定したURLからファイルをフェッチし、そのファイルのハッシュ値を計算することができる。

```sh
nix hash convert --hash-algo sha256 \
    "$(nix-prefetch-url https://example.com/foo.tar.gz --type sha256 --unpack)"
```

# `lib.fakeHash` を使う

一時的に `lib.fakeHash` でダミーのハッシュを指定することで、エラーメッセージから本物のハッシュを知ることもできる。

| 変数名           | 値                                                                                                                                   |
| :--------------- | :----------------------------------------------------------------------------------------------------------------------------------- |
| `lib.fakeHash`   | `"sha256-AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA="`                                                                              |
| `lib.fakeSha256` | `"0000000000000000000000000000000000000000000000000000000000000000"`                                                                 |
| `lib.fakeSha512` | `"00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000"` |

# `lib.fakeHash` を使う

```
hash mismatch for URL `https://example.com/foo.tar.gz'.
Got `sha256-AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=';
expected `sha256-7HXJspebluQeejKYmVA7sy/F3dtU1gc4eAbKiPexMMA='.
```

# `nurl` を使う

[nurl](https://github.com/nix-community/nurl) というツールをインストールして使うと、事前計算されたコンテンツのハッシュを含むフェッチャーの式を自動生成できる。

```shell-session
$ nurl https://github.com/nix-community/nurl.git
$ nix flake prefetch \
            --extra-experimental-features 'nix-command flakes' \
            --json github:nix-community/nurl/7f789ea2da9ff52724efb38df01ecda87704fc87
fetchFromGitHub {
  owner = "nix-community";
  repo = "nurl";
  rev = "7f789ea2da9ff52724efb38df01ecda87704fc87";
  hash = "sha256-QzvpuikAHHHN91pDkBDoUx3x8LJVHbk2JDwYXe87WCc=";
}
```

# `nix-init`

[nix-init](https://github.com/nix-community/nix-init) というツールをインストールして使うと、ソースコードの種別を自動判別してフェッチャーを含むderivation (後述) を自動生成してくれる。

# `nix-init`

**対応しているソースコード種別:**

- Go
- Python (アプリケーション, パッケージ)
- Rust

**対応しているフェッチャー:**

- GitHub
- GitLab
- Gitea
- PyPI
- crates.io

# ハッシュの事前計算まとめ

以下の優先度で試すとよい:

1. `nix-init`
1. `nurl`
1. `nix-prefetch-url` & `nix hash convert`
1. `lib.fakeHash` (中間ビルド生成物のハッシュ確認に有用)

# derivation

derivationとは、パッケージのビルド手順やインストール手順をまとめた情報である。
Arch LinuxのパッケージマネージャPacmanの `PKGBUILD`, あるいはGentoo LinuxのパッケージマネージャPortageの `ebuild` に相当する。

一般的に、パッケージごとにderivationの依存を引数にとりderivationを返す**Nix言語の関数**が定義されることが多い。

# derivationのビルドフェーズ

derivationでは以下のフェーズに分けてパッケージのビルド手順やインストール手順を記述する。

| フェーズ  | 説明                                                                                                  |
| :-------- | :---------------------------------------------------------------------------------------------------- |
| unpack    | パッケージのソースコードを含むアーカイブを解凍するフェーズ                                            |
| patch     | パッケージのソースコードにパッチを適用するフェーズ                                                    |
| configure | ビルドの準備をするフェーズ                                                                            |
| build     | ビルドを実行するフェーズ                                                                              |
| check     | テストを実行し、パッケージが正しくビルドされたかを検証するフェーズ                                    |
| install   | ビルド成果物を再配置するためのフェーズ。このフェーズでNixストアに残すファイルを `$out` にコピーする。 |
| fixup     | `$out` に配置されたファイルに対して後処理を行うフェーズ                                               |

# derivationを返す関数の例

`sl` のパッケージのderivation:

```nix
{ lib, stdenv, fetchFromGitHub, ncurses }:

stdenv.mkDerivation rec {
  pname = "sl";
  version = "5.05";

  src = fetchFromGitHub {
    owner = "eyJhb";
    repo = "sl";
    rev = version;
    sha256 = "11a1rdgb8wagikhxgm81g80g5qsl59mv4qgsval3isykqh8729bj";
  };

  buildInputs = [ ncurses ];

  makeFlags = [ "CC:=$(CC)" ];

  installPhase = ''
    runHook preInstall

    install -Dm755 -t $out/bin sl
    install -Dm644 -t $out/share/man/man1 sl.1{,.ja}

    runHook postInstall
  '';

  meta = with lib; {
    # パッケージのメタ情報 (省略)
  };
}
```

# derivationを返す関数

```
nix-repl> :t import ./sl.nix
a function
```

# derivationを返す関数の引数

```nix
{ lib, stdenv, fetchFromGitHub, ncurses }:
```

ビルドに必要なライブラリ関数や依存パッケージなど (以下、ビルド依存物) はすべて引数として受け取る。

# derivationを返す関数

derivationを取得するには関数の引数を指定する必要がある:

```nix
(import ./sl.nix) {
  inherit lib;
  inherit (pkgs) fetchFromGitHub ncurses stdenv;
}
```

これは面倒くさい。

# `pkgs.callPackage`

Nixpkgsに含まれる `pkgs.callPackage` はderivationを返す関数のビルド依存物を自動注入 (DI, dependency injection) する。

```nix
pkgs.callPackage ./sl.nix { }
```

# `pkgs.callPackage`

特定の依存パッケージだけを上書きすることもできる:

```nix
pkgs.callPackage ./sl.nix {
  ncurses = pkgs.ncurses5;
}
```

# `pkgs.callPackage`

Flake (後述) でNixプロジェクトを管理する場合は、同じFlakeに異なるバージョンのNixpkgsを複数もつことができる。

その場合、注入したいビルド依存物が含まれるNixpkgsの `pkgs.callPackage` を使う:

```nix
let
  pkgs = import inputs.nixpkgs { system = "x86_64-linux"; };
  pkgs' = import inputs.nixpkgs-unstable { system = "x86_64-linux"; };
in
  # ビルド依存物としてnixpkgs-unstableに含まれるライブラリ関数やパッケージが注入される
  pkgs'.callPackage ./sl.nix { };
```

# `nativeBuildInputs` と `buildInputs`

`nativeBuildInputs` と `buildInputs` にはビルド時に必要なライブラリやパッケージを指定する。
これらのパッケージに含まれる実行可能ファイルはderivationのビルド時のみ `PATH` が通り、ビルドコマンドとして呼び出せる。

# `nativeBuildInputs` と `buildInputs` の違い

- `nativeBuildInputs` はビルド時にのみ必要なパッケージ。
- `buildInputs` はビルド時あるいは実行時に必要なパッケージ。

# `nativeBuildInputs` と `buildInputs` の違い

- `buildInputs` と異なり `nativeBuildInputs` はビルド時にしか存在が保証されない。
- そのため、`nativeBuildInputs` にしか含まれないパッケージはGC (garbage collection) によってNixストアから消される可能性がある。
- Java (slim-jar) やPython, Rubyのライブラリはランタイムでロードされるため、`buildInputs` に含める。

# ランタイムで `buildInputs` に指定したパッケージを呼ぶ

- `buildInputs` に含まれるビルド依存物の実行可能ファイルも、ビルド時以外は `PATH` が通っていない。
- そのため、依存元であるパッケージからランタイムで `buildInputs` に含まれるビルド依存物の実行可能ファイルを呼ぶには次スライドで示すいずれかを行う。

# ランタイムで `buildInputs` に指定したパッケージを呼ぶ

1. 依存元パッケージのソースコードにパッチを当て、依存先パッケージのコマンドをNixストア内の実行可能ファイルまでのフルパスに置換する。
   (一旦パッチでコマンドをプレイスホルダーに置換 (例: `foo` -> `@foo@`) し、その後にプレイスホルダーを依存先パッケージのフルパスに置換するのがよい)
1. 依存元パッケージの実行可能ファイルをラップし、依存先パッケージの実行可能ファイルがあるディレクトリに `PATH` を通す。

```sh
# オリジナルの実行可能ファイルは自動的にリネームされる
wrapProgram $out/bin/sl --prefix PATH : ${ncurses}/bin
```

# buildフェーズ

buildフェーズではソースコードからビルド成果物のビルドを行う。
デフォルトでは `make` が実行される。

```nix
stdenv.mkDerivation {
  # (略)

  buildPhase = ''
    make
  '';

  # (略)
}
```

# installフェーズ

installフェーズではbuildフェーズでビルドしたビルド成果物をパッケージに再配置する。
具体的にはinstallフェーズで環境変数 `$out` が示すディレクトリにコピーされたファイル群がパッケージの内容になる。

# derivationで役に立つNixライブラリ関数 or シェルスクリプト

1. `escapeShellArg`, `escapeShellArgs`
1. `patchShebangs`
1. `wrapProgram`
1. `makeBinPath`
1. `installManPage`
1. `installShellCompletion`

# `escapeShellArg`, `escapeShellArgs` (Nix言語の関数)

文字列をシェルの引数としてエスケープするための関数。
Nix言語の文字列をコマンドライン引数として渡す場合に便利。

`escapeShellArg` は単一の文字列を単一のトークンにエスケープし、`escapeShellArgs` は複数の文字列を複数のトークンにエスケープする。

Nixの文字列をコマンドライン引数として渡す場合に便利。

```
nix-repl> lib.escapeShellArg "foo'bar"
"'foo'\\''bar'"

nix-repl> lib.escapeShellArgs [ "foo", "bar" ]
"'foo' 'bar'"
```

# `patchShebangs` (シェルスクリプト)

指定したディレクトリから実行可能ファイルをスキャンし、そのファイルのshebang (`#!/usr/bin/env bash` みたいなやつ) が `nativeBuildInputs` or `buildInputs` に含まれるパッケージの実行可能ファイルで置き換え可能であれば、そのパッケージの実行可能ファイルのフルパスで置き換えるシェルスクリプト。

```sh
patchShebangs $out/libexec
```

例: `#!/usr/bin/env bash` → `#!/nix/store/x88ivkf7rmrhd5x3cvyv5vh3zqqdnhsk-bash-interactive-5.2-p15/bin/bash`

これによりプログラムの環境依存が減る。

参照: https://github.com/NixOS/nixpkgs/blob/master/pkgs/build-support/setup-hooks/patch-shebangs.sh

# `patchShebangs` (シェルスクリプト)

とはいえ、デフォルトのfixupフェーズで実行されるので、`fixupPhase` を上書きしない限り明示的に実行する機会は少ない。

> The default fixupPhase does the following:
> (中略)
> It rewrites the interpreter paths of shell scripts to paths found in PATH. E.g., /usr/bin/perl will be rewritten to /nix/store/some-perl/bin/perl found in PATH. See the section called “patch-shebangs.sh” for details.

参照: https://nixos.org/manual/nixpkgs/stable/#ssec-fixup-phase

# `wrapProgram` (シェルスクリプト)

実行可能ファイルをラップし、以下のようなことを行える:

- 実行可能ファイルが実行されている間のみ有効な環境変数の設定
  - 環境変数のデフォルト値を設定
  - `PATH` のような `:` 区切りの環境変数に値をappend, prepend
- コマンドライン引数を自動的に追加

参照: https://github.com/NixOS/nixpkgs/blob/master/pkgs/build-support/setup-hooks/make-wrapper.sh

# `wrapProgram` (シェルスクリプト)

先程の `sl` で指定した `ncurses` の実行可能ファイルのディレクトリに `PATH` を通す例:

```sh
# オリジナルの実行可能ファイルは自動的にリネームされる
wrapProgram $out/bin/sl --prefix PATH : ${ncurses}/bin
```

# `makeBinPath` (Nix言語の関数)

指定したパッケージのリストから、それらのパッケージの `bin` ディレクトリを `:` 区切りで返す。
実行可能ファイルが実行されている間に `PATH` を通したい場合に `wrapProgram` と併用するのが便利。

# `installManPage` (シェルスクリプト)

シェルの補完ファイルを然るべきディレクトリ `$out/share/man/man<n>` にインストールするための関数。

```nix
installManPage share/doc/foobar.1
```

参照: https://github.com/NixOS/nixpkgs/blob/master/pkgs/build-support/setup-hooks/install-shell-files.sh

# `installShellCompletion` (シェルスクリプト)

シェルの補完ファイルをインストールするための関数。

```sh
installShellCompletion --bash --name foobar.bash share/completions.bash
installShellCompletion --fish --name foobar.fish share/completions.fish
installShellCompletion --zsh --name _foobar share/completions.zsh
```

参照: https://github.com/NixOS/nixpkgs/blob/master/pkgs/build-support/setup-hooks/install-shell-files.sh

# `installShellCompletion` (シェルスクリプト)

最近ではプログラムにシェルの補完ファイルを生成するサブコマンドが含まれることが多いが、これをビルド時に生成してパッケージに含めておくと便利。

```sh
installShellCompletion --zsh --name _foobar <(foobar --generate-completions zsh)
```

# チャンネル (channel)

Nixパッケージマネージャではパッケージのderivationを含むレポジトリをチャンネル (channel) として管理する。

# nix-channel

`nix-channel` で登録されているチャンネルを確認したり、編集 (追加 or 削除) したりできる:

```sh
$ sudo nix-channel --list
nixos "https://nixos.org/channels/nixos-unstable
```

```sh
$ sudo nix-channel --add https://nixos.org/channels/nixos-unstable nixos-unstable
```

# Nixpkgs

NixpkgsはNixパッケージマネージャのデフォルトのパッケージ (のderivation) のレポジトリである。

https://github.com/NixOS/nixpkgs

# Nixpkgs

Nixpkgsに含まれるパッケージは**NixOS Search**というページで検索できる:
https://search.nixos.org/packages

# Nixストア内のチャンネルのコピーへの参照

Nix言語で `<チャンネル名>` のようにカンマでチャンネル名を囲むと、Nixストア内のそのチャンネルのコピーへのパス型を返す。

```
nix-repl> <nixpkgs>
/nix/store/j8han9cf3g8vba52yhiklaa6a500pcbv-source
```

# Nixpkgsのインポート

これを利用して、`nix-repl` でNixpkgsを参照することができる。

```sh
nix-repl -f '<nixpkgs>'
```

```
nix-repl>:t pkgs.hello
a set
```

# 既定のderivationの上書き

Nixpkgsに含まれるderivationのような、既定のderivationを上書きすることもできる。

# `.override`

derivationの `.override` メソッドはderivationを返す関数の引数の値を上書きすることができる。

```nix
pkgs.sl.override {
  ncurses = pkgs.ncurses5;
}
```

# `.overrideAttrs`

`.overrideAttrs` メソッドはderivationの属性を上書きすることができる。

具体的には、古いderivationの属性 (attribute) から新しいderivationの属性の差分を返す関数を`.overrideAttrs` の引数に渡す。

```nix
pkgs.sl.overrideAttrs (oldAttrs: {
  installPhase = ''
    ${oldAttrs.installPhase}

    # installフェーズで追加のインストールコマンドを実行
  '';
})
```

# Nixストア

- Nixでパッケージをインストールすると、そのパッケージやビルド依存物はNixストア以下に格納される。
  - パッケージやビルド依存物のNixストア内のパスを便宜上Nixストアパスと呼ぶ。
- Nixストアは通常 `/nix/store` にある。
- Nixストアは通常read-onlyで、`nixbld` グループに所属するユーザーのみ書き込みが可能。
  - つまり、パッケージのインストールは内部的には `nixbld` に所属するユーザーが行っている。

# Nixストア

Nixストアは**フラットなディレクトリ構造**になっており、明示的にインストールしたパッケージやそれらのビルド依存物はNixストア直下に `<ハッシュ値>-<パッケージ名>-<バージョン>` という名前のディレクトリに格納される。

例:

```
/nix/store
- x88ivkf7rmrhd5x3cvyv5vh3zqqdnhsk-bash-interactive-5.2-p15
- ph9ibc3b7l76iggsddqfzrhfr4f5ys9n-sl-5.05
- ...
```

# Nixストア

Nixストアに格納されるのはパッケージだけではない。
パッケージのビルドに使われたソースコードやパッチ, 中間ビルド生成物, そしてderivation自身もNixストアに格納される。

# パス型

Nix言語のパス型はstring interpolationによって文字列に変換されるタイミングで、そのパスが指すファイル or ディレクトリが存在するかを確認する。
そして、そのパスが指すファイル or ディレクトリをNixストアにコピーし、Nixストア内のコピーのパスを返す。

```
nix-repl> "${./foo}"
"/nix/store/cwabdpqn4g6g8cszs8g9ry6ycz2igzm1-foo"
```

# パス型

これにより、あるパッケージのビルド成果物から任意のファイルをパス型で参照しても、そのファイルの変更によってパッケージのビルド成果物から見えるファイルの内容は変わることがない。

```nix
stdenv.mkDerivation {
  # (略)

  installPhase = ''
    # ビルド後に .vimrc が変更されても vim から見える .vimrc は変わらない
    wrapProgram $out/bin/vim --add-flags -u ${./.vimrc}
  '';

  # (略)
}
```

# derivationのハッシュのチェーン

derivation自身やderivationのビルド依存物に少しでも変更されると、必ずderivationのハッシュも変わる。

- derivationの属性を変更 → **derivationのハッシュが変わる**
- ソースコードを変更 → ソースコードのハッシュが変わる → ソースコードを格納するNixストア内のパスが変わる → derivationの属性が変わる → **derivationのハッシュが変わる**
- ビルド依存物を変更 → ビルド依存物のハッシュが変わる → ビルド依存物を格納するNixストア内のパスが変わる → **derivationのハッシュが変わる**

パッケージの依存ツリーを木構造と捉えると、木の節や葉のハッシュの変更が必ず根のハッシュまで影響するイメージ。(Merkle Treeに近い)

# derivationのハッシュのチェーン

- あるパッケージのderivationのハッシュが変わると、そのパッケージのNixストア内のパス (`/nix/store/<ハッシュ値>-<パッケージ名>-<バージョン>`) も変わるため、変更前のパッケージと変更後のパッケージは別物として扱われる。
- そのため、変更されたパッケージ or ビルド依存物はNixストア内に存在しない状態なので、次にビルドされたタイミングでインストールされる。(差分が更新される)

# derivationのハッシュのチェーン

このように、Nixはパッケージのハッシュはビルド依存物の完全性の確認だけではなく、差分の検出手段としてもワークする。

# パッケージのインストール

パッケージをNixストアに格納しただけではインストールしたとは言えない。

- `/bin` 以下の実行可能ファイルに `PATH` が通っていないため、コマンドとして実行できない。
- `/lib` 以下のライブラリに `LD_LIBRARY_PATH` が通っていないため、動的ライブラリとして参照できない。
- `/share/applications` 以下のデスクトップファイル ( `.desktop` ) に `XDG_DATA_DIRS` が通っていないため、デスクトップアイテムとして表示されない。
- などなど

# パッケージのインストール

- Nixパッケージマネージャはシンボリックリンクを用いてインストールするパッケージ群を1つのディレクトリに集約し、Nixストアに格納する。
- 集約先のディレクトリを**プロファイル (profile)** と呼ぶ。
- マルチユーザーモードでNixパッケージマネージャをインストールする場合は、ユーザーごとにプロファイルが作成される。

# パッケージのインストール

- パッケージ群の集約は内部的に `lib.symlinkJoin` というライブラリ関数を使うことで実現されている。
  参照: https://github.com/NixOS/nixpkgs/blob/master/pkgs/build-support/trivial-builders/default.nix
- `lib.symlinkJoin` は `ignoreCollisions` という引数を持っており、これを `false` にすると、異なるパッケージで同じファイルが存在する (名前衝突) 場合にエラーとなる。
- Nixパッケージマネージャのデフォルトの動作は `ignoreCollisions = false` 相当の挙動となっており、異なるパッケージで同じファイルが存在する場合にエラーとなる。

# パッケージのインストール

- 最新のプロファイルへのシンボリックリンクが `/etc/profiles/per-user/<ユーザー名>` に作成される。(NixOSの場合)
- このパスを `PATH` や `LD_LIBRARY_PATH`, `XDG_DATA_DIRS` などに追加することで、初めてNixストアに格納したパッケージを利用できるようになる。
- Nixインストーラは `/etc/profile` でこれらの環境変数の設定を行うスクリプト `set-environment` を読み込むように設定する。
  (NixOSではデフォルトで `/etc/profile` で `set-environment` を読み込むになっている)

# nix-shell

nix-shellはNixストアの性質を利用して、任意のパッケージをプロファイルにインストールすることなく使用できるシェル環境を提供する。

```shell-session
$ nix-shell -p cowsay
this path will be fetched (0.01 MiB download, 0.05 MiB unpacked):
  /nix/store/rc7kpqwb8z5ch37ysv5yk9yg5hl5bkdj-cowsay-3.7.0
copying path '/nix/store/rc7kpqwb8z5ch37ysv5yk9yg5hl5bkdj-cowsay-3.7.0' from 'https://cache.nixos.org'...
```

# nix-shell

確認すると、プロファイルの他にNixストア内の `cowsay` パッケージの `bin` ディレクトリに直接 `PATH` が通っている。

```shell-session
[nix-shell:~]$ echo $PATH
/nix/store/x88ivkf7rmrhd5x3cvyv5vh3zqqdnhsk-bash-interactive-5.2-p15/bin
:(中略)
:/nix/store/rc7kpqwb8z5ch37ysv5yk9yg5hl5bkdj-cowsay-3.7.0/bin
:(中略)
:/run/wrappers/bin
:/home/sei40kr/.nix-profile/bin
:/home/sei40kr/.local/state/nix/profile/bin
:/etc/profiles/per-user/sei40kr/bin
:/nix/var/nix/profiles/default/bin
:/run/current-system/sw/bin

[nix-shell:~]$ which cowsay
/nix/store/rc7kpqwb8z5ch37ysv5yk9yg5hl5bkdj-cowsay-3.7.0/bin/cowsay
```

# End
