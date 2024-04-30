# Zenn CLI

* [📘 How to use](https://zenn.dev/zenn/articles/zenn-cli-guide)

## how to use

### 記事ファイルの作成

各オプションは省略可．

* `--slug` はファイル名であり，記事ID/URLである．省略した場合はランダム列
* `--title` は記事のタイトル．省略した場合は空文字列
* `--type` は `tech`で技術記事，`idea`でアイデア記事を示す．省略した場合は`tech`
* `--emoji` はアイキャッチとして使われる絵文字（1文字）．省略した場合はランダム
* `--published` は公開設定．省略した場合は`false`（非公開）
* `--help` でヘルプを表示

```sh
npm run article -- --slug <slug> --title <タイトル> --emoji <絵文字>
```

### プレビュー

```sh
npm run preview
```

### 記事の公開

`published`を`true`にした上で`published_at` (`YYYY-MM-DD` or `YYYY-MM-DD hh:mm`) を指定する．

> [!CAUTION]
> 公開日時の指定は一度のみで**変更不可**