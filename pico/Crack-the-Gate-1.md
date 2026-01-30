# 問題名
Crack the Gate 1

# 概要
ログイン画面のHTMLに残ったコメントからバイパス用ヘッダが判明。ヘッダを付けてログインAPIへ送るとフラグが返る。

# 確認
- ページ表示はメール/パスワード/ログインボタンのみ
- 適当なPWで送信すると「Invalid credentials」

# 手順
1) `htmlを参照` 怪しいコメントを発見
2) コメント内の文字列をROT13で復号
3) ヘッダ `X-Dev-Access: yes` を付けて `/login` にPOST
4) 返却レスポンスの `flag` を取得

# 参考
`source.html` 内コメント:
```
<!-- ABGR: Wnpx - grzcbenel olcnff: hfr urnqre "K-Qri-Npprff: lrf" -->
```
ROT13結果:
```
<!-- NOTE: Jack - temporary bypass: use header "X-Dev-Access: yes" -->
```

# ペイロードを送信
burpでリクエストをインターセプトしヘッダを追加してフォワード。
フラグがリークされる。

```
picoCTF{brut4_f0rc4_cbb8faa7}
```
