---
name: release
description: 新しい版をリリースする。前回のタグからの変更をまとめたパッチノートを書いて、GitHub Release を公開する。「リリースして」「1.0.5 を出して」などで使う。
---

# リリース

コマンドは Mac・Windows どちらでも動く git / gh だけを使う。シェル固有の書き方(`$(...)` やパイプ前提の処理)に頼らない。

## 0. 公開のしかたを調べる

プロジェクトに専用のリリース手順があればそれを使う。次の順に探す。

1. CLAUDE.md / README / CONTRIBUTING にリリース手順が書いてあれば、それに従う。
2. `package.json` の `scripts.release`、`Makefile` の `release` ターゲット、`scripts/release.*` などがあれば、それを使う。引数(版・ノートのファイル)の渡し方はスクリプトを読んで確かめる。
3. どちらもなければ、6. の「専用の手順がないとき」で公開する。

版をファイル(`package.json`、`manifest.json`、`Cargo.toml`、`pyproject.toml` など)に書いているかも見ておく。

## 1. 版を決める

- 既定のブランチ: `gh repo view --json defaultBranchRef -q .defaultBranchRef.name`
- 前回のタグ: `git fetch --tags` のあと `git describe --tags --abbrev=0 origin/<既定のブランチ>`
- タグがまだなければ初回リリースとして扱い、全履歴を対象にする。
- 指定がなければ、機能追加ありならマイナー、修正だけならパッチを上げる案を出してユーザーに確認する。

## 2. 前回のタグからの変更を集める

```bash
git log <前回のタグ>..origin/<既定のブランチ> --merges --pretty=format:%s%n%b
gh pr list --state merged --base <既定のブランチ> --search "merged:>=<前回のタグの日付>" --json number,title,body
```

マージコミットがない運用(squash / rebase)なら `--merges` を外してコミット一覧を見る。

PR のタイトルだけで分からないときは本文や差分(`gh pr diff <番号>`)を見る。

ノートに書く項目は、PR の説明をうのみにせず、既定のブランチのコードで本当に入っていることを確かめる(該当の処理・文言を Grep して見る)。途中で戻された・別 PR で消えたものは書かない。

## 3. パッチノートを書く

使う人向けに日本語で書く。PR 番号・作者・コミットの型(feat/fix)は書かない。使う人から見て何が変わったかで書く。

```md
## 追加
- 〇〇を追加

## 変更
- 〇〇を変更

## 削除
- 〇〇を削除(理由があれば一言)

## バグ修正
- 〇〇のとき△△になる問題を修正

## その他
- 表示を高速化 など
```

- 文末は見出しごとに揃える(追加は「〇〇を追加」、変更は「〇〇を変更」、削除は「〇〇を削除」、バグ修正は「〇〇を修正」)。
- 当てはまる項目がない見出しは消す。
- ドキュメント・テスト・リファクタ・CI だけの変更は書かない(使う人に関係あるなら「その他」)。
- 画面や機能のまとまりがいくつかあるプロジェクトなら、その名前を頭に付ける(例「設定画面: 〇〇を追加」)。

ファイルは版ごとに `release-notes/v<version>.md` に新しく作る。プロジェクトに別の置き場所の決まりがあればそれに従う。既にあれば上書きせずユーザーに確認する。

ノートのファイルが git 管理下にあると「作業ツリーが汚れている」で止まるリリーススクリプトがある。`.gitignore` に入っているか確かめ、入っていなければノートを一時フォルダ(スクラッチパッド)に置くか、ユーザーにどうするか聞く。

## 4. ユーザーに確認する

**必ず**ノートの全文と、6. で実行するコマンドをユーザーに見せ、はっきり OK をもらうまで公開しない(push と公開は取り消せない)。全文を返答に書いたうえで AskUserQuestion で「この内容で公開する / 直す」を聞く。直しを頼まれたらファイルを直してもう一度全文を見せて聞き直す。

## 5. 公開の前に

```bash
git switch <既定のブランチ>
git pull
git status --porcelain
```

作業ツリーが汚れていたら止めてユーザーに聞く。

## 6. 公開する

### 専用の手順があるとき

0. で見つけた手順で、版とノートのファイルを渡して実行する(例: `bun run release <version> release-notes/v<version>.md`)。

### 専用の手順がないとき

版をファイルに書いているプロジェクトなら、その版を書き換えてコミットする(`chore: release v<version>`)。それから:

```bash
git tag -a v<version> -m v<version>
git push origin <既定のブランチ> --follow-tags
gh release create v<version> --title v<version> --notes-file release-notes/v<version>.md
```

プレリリース(`-alpha` / `-beta` / `-rc` 付き)なら `gh release create` に `--prerelease` を付ける。

失敗したら、どこまで済んだか(コミット・タグ・push・Release)を `git log` / `git ls-remote --tags origin` / `gh release view v<version>` で確かめ、残りの手順だけをユーザーに示す。push 済みのタグを消したり作り直したりはしない。
