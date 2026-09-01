---
title: "Gmail APIには「送信だけ」の権限はあるのに、「下書きだけ」の権限が無い"
emoji: "✉️"
type: "tech"
topics: ["gmail", "oauth", "googleapi", "ai", "業務自動化"]
published: true
published_at: 2026-08-13 07:30
---

常駐案件の月次請求で、報告メールをPythonで組み立ててGmailの下書きに入れる、というスクリプトを書いたときの話です。毎月の流れはこうなっています。

```
[毎日] 日報のExcelに作業時間を手入力
  ↓
[月初] 集計して請求金額を出す → 報告メールを書いて日報を添付 → 送信
  ↓
       先方の承認をもらい、請求書を発行
```

集計まではPythonで済むようになっていたので、次に手を付けたのが「報告メールを書いて日報を添付」のところでした。ただし作るのは下書きまでで、送信ボタンは人間が押す。

そう決めたのには、譲れない要件が1つありました。**間違って送信されることだけは絶対に避けたい。**金額の書かれたメールが、確認前に先方へ飛ぶのが最悪の事故だからです。下書きまでは機械に、送信ボタンは自分で押す。そこは人間の仕事として残したい。

素直に考えれば「下書きの作成だけを許可するスコープ」を選べばいいはずです。ところが、それがGmail APIには無い。「なんで無いんだろう」と思って調べていくと、単に用意されていないというだけでなく、ちょっと不思議な非対称が見えてきました。

(スコープまわりの記述は2026年8月時点の確認です。出典は記事末尾にまとめました)

## 1. スコープ一覧を眺めても、それらしいものが無い

まずGmail APIのスコープ一覧を見ます。メール本体に関わるものを抜き出すと、こんな並びでした。

| スコープ | 説明(公式ドキュメントの原文) | 区分 |
|---|---|---|
| `gmail.readonly` | View your email messages and settings. | 制限付き |
| `gmail.metadata` | View your email message metadata such as labels and headers, but not the email body. | 制限付き |
| `gmail.send` | Send email on your behalf. | 機密 |
| `gmail.insert` | Add emails into your Gmail mailbox. | 制限付き |
| `gmail.compose` | Manage drafts and send emails. | 制限付き |
| `gmail.modify` | Read, compose, and send emails from your Gmail account.(以下略) | 制限付き |
| `https://mail.google.com/` | Read, compose, send, and permanently delete all your email from Gmail. | 制限付き |

(出典: [Choose Google Workspace API scopes | Gmail](https://developers.google.com/workspace/gmail/api/auth/scopes) — 説明文は原文ママ、区分は同ページの Sensitive / Restricted の分類。表記は`https://www.googleapis.com/auth/`を省いています。`https://mail.google.com/`だけは形が違うので、そのまま書いています)

目当ての「下書き」に一番近いのは`gmail.compose`です。ところが説明文が **"Manage drafts and send emails."** 。下書きの管理と、メールの送信。1つのスコープに両方が同居しています。

`gmail.readonly`や`gmail.metadata`は読み取り専用なので下書きを作れません。`gmail.modify`はもっと広い。`gmail.insert`は説明だけ見ると惜しいのですが、これはメールボックスにメッセージを直接放り込むためのもので、下書きを作る用途ではありません。

つまり、下書きを作りたければ、送信の権限も一緒に受け取るしかない。これがこの記事の出発点です。

## 2. メソッド単位ならもっと細かいのでは、と思ったら完全に一致していた

スコープ一覧の説明文がざっくりしているだけで、実際に呼ぶメソッドの単位では区別されているのかもしれない。そう思って、APIリファレンスのメソッドごとに書かれている「Authorization scopes」を突き合わせてもらいました。結果がこれです。

| メソッド | 必要なスコープ(いずれか1つ) |
|---|---|
| `users.drafts.create`(下書きを作る) | `https://mail.google.com/` / `gmail.modify` / `gmail.compose` |
| `users.drafts.send`(下書きを送信する) | `https://mail.google.com/` / `gmail.modify` / `gmail.compose` |
| `users.messages.send`(メールを送信する) | `https://mail.google.com/` / `gmail.modify` / `gmail.compose` / `gmail.send` |

(出典: 各メソッドのAPIリファレンスの「Authorization scopes」 —
[users.drafts.create](https://developers.google.com/workspace/gmail/api/reference/rest/v1/users.drafts/create) /
[users.drafts.send](https://developers.google.com/workspace/gmail/api/reference/rest/v1/users.drafts/send) /
[users.messages.send](https://developers.google.com/workspace/gmail/api/reference/rest/v1/users.messages/send))

上2行を見比べてください。**「下書きを作れるスコープの集合」と「下書きを送信できるスコープの集合」が、完全に一致しています。**

これは「たまたま細かいスコープが用意されていない」という話ではありません。集合が同じである以上、`drafts.create`を呼べる権限を持っているなら、その権限は必ず`drafts.send`も呼べます。どのスコープを選んでも、この関係は変わりません。設計としてそうなっている、ということです。

## 3. 逆に「送信だけ」はちゃんと表現できる

面白いのは反対方向です。

`gmail.send`は`users.messages.send`の行にだけ登場して、`drafts.create`の行にはいません。つまり`gmail.send`だけを渡されたアプリは、メールを送信できるけれど、下書きは作れない。

「送信だけできて下書きは作れない権限」は用意されているのに、その裏返しである「下書きだけ作れて送信はできない権限」は用意されていない。ここが妙だな、と思ったところです。

しかも`gmail.send`は区分が「機密スコープ」で、下書きを作るために必要な`gmail.compose`は一段重い「制限付きスコープ」です。公式ドキュメントによると、制限付きスコープは「Googleユーザーデータへの広範なアクセス」を提供するものとして、制限付きスコープ向けのOAuthアプリ審査が必要とされていて、さらにそのデータをサーバーに保存・送信する場合はセキュリティ評価も求められるとのことでした(これも[スコープ一覧のページ](https://developers.google.com/workspace/gmail/api/auth/scopes)に書かれています)。

整理すると、こうなります。

- 送信だけしたい人(危ないことができる) → 軽い区分のスコープで済む
- 下書きだけ作りたい人(慎重にやりたい) → 重い区分のスコープを取らされ、おまけに送信権限も付いてくる

安全側に倒そうとした方が、要求される権限も審査も重くなる。直感とは逆向きです。

## 4. なぜこうなっているのか推測して、自分で崩した

ここからは、公式の説明を見つけられなかったので推測です。

まず、Googleが雑なわけではないだろう、という擁護の理屈は立ちます。下書きというのは「まだ送っていないメール」であって、置き場所はユーザーのメールボックスです。下書きを作れるということは、そのメールボックスに任意の宛先と本文を書き込めるということでもある。作られた下書きを人間が見て「あ、これでいいのか」と送信ボタンを押してしまえば、結果は送信されたのと変わりません。そう考えると`drafts.create`と`drafts.send`を地続きの権限として扱うのは、防御としてむしろ筋が通っている——

と、ここまで書いたところで気づいたのですが、**この理屈は第1章の表の中にある反例で崩れます。**

`gmail.insert`("Add emails into your Gmail mailbox.")です。これはメールボックスにメールを追加できるけれど、送信はできないスコープでした。つまりGoogleは「メールボックスへの書き込みだけを許して、送信は許さない」というスコープを作る能力も意志も持っていて、現に作っています。「メールボックスに書けるのは危険だから分離できない」という説明にはなりません。下書きのところだけ、そうなっていない。

もう1つ、判断が付かないものもありました。Gmailのアドオン向けに`gmail.addons.current.action.compose`というスコープがあります。これが厄介で、公式の説明が2か所にあって内容が食い違っています。

[スコープ一覧のページ](https://developers.google.com/workspace/gmail/api/auth/scopes)ではこうなっています(区分はNon-sensitive)。

> Manage drafts and send emails when you interact with the add-on.

一方、[アドオン側のドキュメント](https://developers.google.com/workspace/add-ons/concepts/workspace-scopes)ではこうです。

> Allows the add-on to temporarily create new drafts messages and replies.

前者は送信を含むと読めますが、後者は下書きの作成しか書いていません。同じスコープなのに、読み手が受け取る権限の範囲が変わります。**どちらが実際の挙動かは、ドキュメントを読んだだけでは確定できませんでした。**

いずれにせよ、これはアドオンの実行中だけ使える別枠のスコープで、今回のようなスタンドアロンのスクリプトからは使えません。`users.drafts.create`の「Authorization scopes」にも載っていません(ここは何度か踏み直して確認しました)。

結局、なぜこうなっているのかは分かりませんでした。

## 5. 同じことで困っている人は、ちゃんといた

自分の要件が特殊なだけかとも思ったのですが、調べてもらったところGoogleのIssue Trackerに、まさにこれを求める起票がありました。[Request for drafts-only Gmail API scope for AI integrations](https://issuetracker.google.com/issues/442444733)です。

2025年9月2日の起票で、内容も自分とほぼ同じでした。AIにGmailの下書きを作らせたい、でも送信の権限は渡したくない、という話です。

> Until Google provides a dedicated "drafts-only" scope, the closest option is the gmail.compose scope, which technically includes sending.

(拙訳: Googleが専用の「下書き専用」スコープを提供するまでは、一番近い選択肢は gmail.compose で、これは技術的には送信も含んでしまう)

その後に付いたコメントも、ほとんどがAIエージェント絡みでした。2026年5月には「AIツールが増えてきたので調べてみたが、"draft only" のスコープが無いことに驚いた。ガードレールとしてこのスコープが必要だ」という趣旨のコメントが、2026年7月にも「エージェントがGmailの作業を手伝ううえで、意図しない送信のリスクなしにやるには、これがどうしても必要」というコメントが入っています。

一方で、Google側からの実質的な回答は見当たりません。起票直後に自動で担当者が割り当てられ、2026年5月にGoogleの担当者がステータスを「New」と記録したのが最後です。この記事を書いている2026年8月時点でもステータスは New のまま、画面上部の STATUS UPDATE は "No update yet."、種別は Feature Request、優先度はP2。起票から11か月以上が経っています。

とはいえ責める気にはならなくて、スコープを1つ増やすというのは同意画面の文言も、審査の区分も、既存アプリへの影響も全部絡んでくる話なので、そう簡単ではないんだろうと想像します(これも推測です)。ただ少なくとも、「Googleが理由を説明したことは公には無さそうだ」とは言えそうで、だから第4章の推測は推測のままです。

## 6. 結局、要件とAPIの「軸」が違う

Googleの意図は分からないままですが、自分の中で整理がついたのはこの見方でした。

自分がやりたかったのは「機械には下書きまでを、送信は人間に」という**工程の分割**です。一方でGmail APIが提供しているのは、**データへのアクセス範囲の分割**でした。読めるのか、書けるのか、消せるのか。

軸が違うので、いくら一覧表を眺めても目当てのものは出てきません。「下書きまで」は範囲ではなく工程だからです。これは推測ではなく、スコープ一覧を見れば分かることでした。Issue Trackerに集まっているのが揃いも揃ってAIエージェント絡みの要望なのも、たぶん理由は同じです。AIに仕事を任せるときに引きたい線は、だいたいデータの範囲ではなく工程の側にあるので。

## 7. で、どうしたか

権限で止められないなら、コードで止めるしかありません。実際にやったのは、地味ですがこの3つです。

**1. 送信APIを実装しない。そう決めたことをファイルの先頭に書く**

スクリプトの冒頭のdocstringに、方針として明記しました。

```python
"""out/ に生成済みのメール下書きを、Gmailの下書きとして保存する。

方針:
    - このスクリプトはメールを【送信しない】。
      users.messages.send / users.drafts.send は呼ばない。送信は人間がGmailの画面で行う。
    - 下書き作成に必要な gmail.compose スコープは、仕様上「送信」も許可してしまう。
      権限では送信を止められないため、送信APIを実装しないことで担保している。
"""
```

「なぜこう書いてあるのか」を残すのが大事だと思っています。理由を書いておかないと、半年後の自分が「これ何で送信しない作りにしたんだっけ」と言って、平気で足してしまうので。スコープを宣言している箇所にも、同じ趣旨のコメントを1行付けました。

```python
# 下書きの作成・参照に必要な最小のスコープ。
# 「下書きは作れるが送信はできない」スコープはGmail APIに存在しない。
SCOPES = ["https://www.googleapis.com/auth/gmail.compose"]
```

**2. `send`という文字列が入っていないことを、レビューで見る**

止めているのが「送信APIを書かない」という自分ルールだけなので、確認する方法も自分で用意しないと意味がありません。`grep -n "send"`をレビュー項目にしました。実際にコードレビューを頼んだときにも、この観点で見てもらっています。

**3. 下書きを作る前に、全部表示して人間に確認させる**

宛先・CC・件名・添付ファイル名とサイズ・本文を全部ターミナルに出してから、`[y/N]`で聞きます。デフォルトはNoです。

```
  宛先  : xxxxx@example.com
  CC    : (なし)
  件名  : 2026年7月度 作業報告
  添付  : 日報.xlsx (48 KB)
----------------------------------------------------------
(本文がそのまま表示される)
----------------------------------------------------------

この内容でGmailの下書きを作成します。よろしいですか? [y/N]:
```

これは権限の話とは別の目的(添付ファイルの取り違えや、集計後に日報を編集してしまったケースの検知)もあるのですが、結果として「機械が勝手に何かを完了させない」という形にはなっています。

## 8. 「APIに止めてもらう」のと「自分で止める」のは別物

ここまでやっても、最初に欲しかったものとは違うな、という感覚は残っています。

APIに止めてもらう場合、こちらが何をしようが止まります。スコープが無ければ、コードに送信処理を書いてもAPIが`403`を返して終わりです。うっかりミスでは破れません。

自分で止める場合は、うっかりで破れます。`service.users().messages().send()`と1行書いた瞬間に、成立しなくなります。しかも、破れたことがどこにも出ません。テストは通るし、スコープも足りているので、普通に動いてしまいます。

つまり自分がやったのは、「送信されない」という保証の担い手を、**Googleから自分に移した**ということです。それが悪いというより、そうするしかない場面がある、という話です。大事なのは今どちらに乗っているかを分かっておくことかなと思っています。止まるはずだと思い込んで運用するのが、一番危ないので。

この話はGmailに限らないはずです。いま請求書の発行側もfreee連携で自動化しようとしていて、そこでも同じことを考えることになりました(そちらは実装してから別途書きます)。「これだけできる権限」が欲しいのに用意されていない、という場面は、これから増えていく気がします。特に、ルールを守る側が人間ではなくAIになるときは。Issue Trackerに並んでいた要望が全部AI絡みだったのも、その入り口な気がしています。

## まとめ

- Gmail APIには「下書きは作れるが送信はできない」スコープが存在しない。下書き作成に必要な`gmail.compose`の説明は "Manage drafts and send emails." で、送信が同居している
- メソッド単位で見ても、`users.drafts.create`と`users.drafts.send`の必要スコープは完全に同一。どれを選んでも送信権限は付いてくる
- 逆方向の「送信だけできて下書きは作れない」(`gmail.send`)はちゃんと用意されている。しかも区分は`gmail.compose`より軽い。慎重にやりたい側の方が重い権限を要求される
- なぜそうなっているのかは、公式の説明が見つからなかった。「メールボックスに書けるのは危険だから分離できない」という擁護を考えてみたが、`gmail.insert`(メールボックスに追加できるが送信はできない)という反例が同じ表の中にあって成り立たなかった。アドオン向けのスコープは、公式の説明が2か所で食い違っていて(片方は送信を含み、片方は下書き作成のみ)、ドキュメントからは範囲を確定できなかった
- 同じ要望はGoogleのIssue Trackerに2025年9月から出ていて、コメントはほぼ全部AIエージェント絡み。2026年8月時点でステータスは「New」のまま、Google側の実質的な回答は見当たらない
- 腑に落ちたのは、「工程で分けたい」というこちらの要件と、「データへのアクセス範囲で分ける」というプラットフォームの区切り方で、そもそも軸が違うという整理。「下書きまで」は範囲ではなく工程なので、スコープ一覧をいくら眺めても出てこない
- 権限で止められないものは、コードと運用で止めるしかない。ただしAPIに止めてもらうのはうっかりでは破れないが、自分で止めるのは1行書いた瞬間に破れる。今どちらに乗っているかは分かっておきたい

## 追記：アドオン用スコープの記述を訂正しました(2026-08-22)

公開時、第4章に「アドオン用スコープの公式の説明は下書き作成だけで、送信には触れていない」と書いていました。**これは2か所ある公式の説明のうち、片方しか見ていませんでした。** スコープ一覧のページには "Manage drafts and send emails when you interact with the add-on." とあり、そちらは送信を含みます。該当箇所を、2つの説明が食い違っている旨に差し替えました。

見落とした原因は、最初にスコープ一覧を確認したときの聞き方でした。確認したいスコープを7つ名指しして問い合わせたので、それ以外(アドオン系)が返ってこなかったんです。質問が答えの範囲を決めていました。この話は別途書きます。

なお、記事の主題である「`users.drafts.create`と`users.drafts.send`の必要スコープが完全に一致している」という点は変わりません。こちらは日を空けて4回確認しています。

## 参考(この記事で確認に使った一次資料)

いずれも2026年8月時点で確認した内容です。仕様は変わりうるので、実装時はご自身で最新版を確認してください。

- [Choose Google Workspace API scopes | Gmail](https://developers.google.com/workspace/gmail/api/auth/scopes) — スコープの一覧、説明文、機密/制限付きの区分
- [Method: users.drafts.create](https://developers.google.com/workspace/gmail/api/reference/rest/v1/users.drafts/create)
- [Method: users.drafts.send](https://developers.google.com/workspace/gmail/api/reference/rest/v1/users.drafts/send)
- [Method: users.messages.send](https://developers.google.com/workspace/gmail/api/reference/rest/v1/users.messages/send)
- [Google Workspace アドオンのスコープ](https://developers.google.com/workspace/add-ons/concepts/workspace-scopes) — `gmail.addons.current.action.compose`の説明
- [Request for drafts-only Gmail API scope for AI integrations](https://issuetracker.google.com/issues/442444733) — Google Issue Tracker(閲覧にGoogleアカウントでのサインインが必要です)

## 関連記事

- [Claude Codeのログで棚卸しする前に。その「直近1ヶ月」は選んだ期間じゃない](https://zenn.dev/kitamura65/articles/claude-code-log-audit-pitfalls) — この記事で触れている初期設定の作業そのものが、後日 Claude Code のセッションログを棚卸ししたときに「毎回やっているのに手順化していない作業」として浮かび上がりました。
