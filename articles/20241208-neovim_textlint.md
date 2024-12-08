---
title: Neovim でテックブログを書きながら校正できる環境を整備した話
emoji: ✍️
type: tech
topics:
  - Neovim
  - textlint
published: true
publication_name: simpleform_blog
---

本記事は [SimpleForm Advent Calendar 2024](https://qiita.com/advent-calendar/2024/simpleform) の 6 日目の記事です。

テックブログ執筆に没頭していると、うっかり表記揺れや誤字脱字などをしてしまいがちです。
書き終わった後に校正すればある程度気が付けると思われますが、書きながら気付けるとより執筆が捗りそうです。
というわけで、textlint を使って Neovim でテックブログを書きながら校正できる環境を整備したので、設定した内容を共有します。

なお、筆者は format に [conform.nvim](https://github.com/stevearc/conform.nvim)、lint に [nvim-lint](https://github.com/mfussenegger/nvim-lint) を使っているため、それらのプラグインでの設定方法を紹介します。
Google で検索すると、null-ls.nvim, efm-langserver, coc.nvim などでの導入の解説記事はあるものの、筆者が使う 2 プラグインについては見つかりませんでした。

## nvim-lint で textlint の lint 結果が表示されるようにする

nvim-lint が対応している linter は [README](https://github.com/mfussenegger/nvim-lint#available-linters) に記載されています。
残念ながら textlint は対応していませんが、カスタムで linter を追加する方法が同じく [README](https://github.com/mfussenegger/nvim-lint#custom-linters) に記載されています。
README の記載にしたがって追加してみます。

### カスタムの linter を登録する際に必要な要素

README によると、linter を登録する際には以下の要素が必要なようです。

- `cmd` (string) : linter の実行コマンド
- `stdin` (boolean) : lint 対象のテキストを標準入力で渡せるか
- `append_fname` (boolean) : `stdin=false` の場合に `args` へファイル名を自動的に追加するか
- `args` (list) : 実行コマンドに渡す引数のリスト
- `stream` ('stdout' | 'stderr' | 'both' いずれか) : lint 結果の出力先
- `ignore_exitcode` (boolean) : linter の実行コマンドの終了コードを無視するか
- `env` (`table`) : カスタム環境のテーブル
- `parser` (function) : lint 結果をパースする関数

カスタムの環境などは使わないので `env` は特に気にしないで良さそうです。
[texlint の CLI のオプション](https://github.com/textlint/textlint?#cli)を見ると、`--stdin` を使えば lint 対象のテキストを標準入力で渡せるようです。
したがって、`stdin` は true で良いので、 `append_fname` も特に気にする必要はありません。
textlint を実行すると、lint 結果は標準出力に出ているので `stream` は `stdout` で良さそうです。
終了コードについては textlint の README には詳細が書かれていないので、`ignore_exitcode` はいったん true にする方向で検討します。

linter の実行コマンドは、nvim-lint が対応しているほかの linter を参考にします。
たとえば、Biome は以下のように設定されています。

@[github](https://github.com/mfussenegger/nvim-lint/blob/6b46370d02cd001509a765591a3ffc481b538794/lua/lint/linters/biomejs.lua#L29-L35)

`node_modules` 内にインストールされていればそれを使い、ない場合はパスが通っている biome を使う実装になっており、これを踏襲する形で問題なさそうです。

### lint 結果のパース方法

残る `args` と `parser` がカスタマイズ要素の多い箇所です。
linter によって、lint 結果がどのように表示されるかは異なるので、表示内容に合わせて適切にパースする関数を実装する必要があります。

README によると、textlint の lint 結果は以下のフォーマットで出力可能です。

- `checkstyle`
- `compact`
- `jslint-xml`
- `json`
- `junit`
- `pretty-error`
- `stylish`
- `table`
- `tap`
- `unix`

キー構造が把握できれば一番実装しやすそうですので、今回は JSON フォーマットを選択します。
実際に JSON フォーマットで出力したところ、以下のような結果が返りました。

```json
[
  {
    "messages": [
      {
        "type": "lint",
        "ruleId": "ja-technical-writing/no-doubled-joshi",
        "message": "一文に二回以上利用されている助詞 \"で\" がみつかりました。\n\n次の助詞が連続しているため、文を読みにくくしています。\n\n- \"で\"\n- \"で\"\n\n同じ助詞を連続して利用しない、文の中で順番を入れ替える、文を分割するなどを検討してください。\n",
        "index": 411,
        "line": 18,
        "column": 21,
        "range": [411, 412],
        "loc": {
          "start": {
            "line": 18,
            "column": 21
          },
          "end": {
            "line": 18,
            "column": 22
          }
        },
        "severity": 2
      },
      {
        "type": "lint",
        "ruleId": "ja-technical-writing/sentence-length",
        "message": "Line 32 sentence length(106) exceeds the maximum sentence length of 100.\nOver 6 characters.",
        "index": 938,
        "line": 32,
        "column": 908,
        "range": [938, 1044],
        "loc": {
          "start": {
            "line": 32,
            "column": 908
          },
          "end": {
            "line": 32,
            "column": 1014
          }
        },
        "severity": 2
      },
      {
        "type": "lint",
        "ruleId": "ja-technical-writing/ja-no-weak-phrase",
        "message": "弱い表現: \"思います\" が使われています。",
        "index": 3530,
        "line": 108,
        "column": 117,
        "range": [3530, 3531],
        "loc": {
          "start": {
            "line": 108,
            "column": 117
          },
          "end": {
            "line": 108,
            "column": 118
          }
        },
        "severity": 2
      },
      {
        "type": "lint",
        "ruleId": "ja-technical-writing/no-exclamation-question-mark",
        "message": "Disallow to use \"！\".",
        "index": 3797,
        "line": 121,
        "column": 48,
        "range": [3797, 3798],
        "loc": {
          "start": {
            "line": 121,
            "column": 48
          },
          "end": {
            "line": 121,
            "column": 49
          }
        },
        "severity": 2
      }
    ],
    "filePath": "/Users/inuatsu/GitHub/zenn_blog/articles/20231203-lambda_graceful_shutdown.md"
  }
]
```

parser の実装も、nvim-lint がすでに対応しているほかの linter の実装を参考にします。
たとえば、actionlint は以下のようになっています。

@[github](https://github.com/mfussenegger/nvim-lint/blob/6b46370d02cd001509a765591a3ffc481b538794/lua/lint/linters/actionlint.lua#L6-L27)

linter の出力を output で受け取り、JSON をデコードしてループしています。
ループする際に、以下の要素を取り出しています。

- `lnum` : 指摘箇所の開始位置の行番号
- `end_lnum` : 指摘箇所の終了位置の行番号
- `col` : 指摘箇所の開始位置の列番号
- `end_col` : 指摘箇所の終了位置の列番号
- `severity` : 指摘箇所の重大度 ([`vim.diagnostic.severity`](https://neovim.io/doc/user/diagnostic.html#vim.diagnostic.severity) の値のいずれか)
- `source` : 指摘を出している linter 名
- `message` : 指摘内容

先に記載した JSON のキーを見て、上記のそれぞれの要素に対応する値を引っ張ってくれば良さそうです。
キー名からおおむね推測可能なので詳細な説明は省きますが、`severity` だけは数字になっており対応関係がこれを見ただけではわかりません。
これに関しては textlint のコードの以下の箇所を見れば対応関係が把握できます。

@[github](https://github.com/textlint/textlint/blob/e0038784a28dc87fe5ca638cd2d944f54393c354/packages/textlint/src/shared/type/SeverityLevel.ts#L8-L13)

最終的な実装は以下のようになりました。

```lua:.config/nvim/lua/configs/lint.lua
local lint = require "lint"

lint.linters_by_ft = {
    markdown = { "textlint" },
}

local severities = {
  [0] = vim.diagnostic.severity.INFO,
  [1] = vim.diagnostic.severity.WARN,
  [2] = vim.diagnostic.severity.ERROR,
}

local binary_name = "textlint"

lint.linters.textlint = {
  cmd = function()
    local local_binary = vim.fn.fnamemodify('./node_modules/.bin/' .. binary_name, ':p')
    return vim.loop.fs_stat(local_binary) and local_binary or binary_name
  end,
  stdin = true,
  args = {
    "--format",
    "json",
    "--stdin",
    "--stdin-filename",
    function()
      return vim.api.nvim_buf_get_name(0)
    end,
  },
  ignore_exitcode = true,
  stream = "stdout",
  parser = function(output)
    if output == "" then
      return {}
    end
    local decoded = vim.json.decode(output)
    if decoded == nil then
      return {}
    end
    local diagnostics = {}
    for _, value in ipairs(decoded) do
      for _, item in ipairs(value.messages) do
        table.insert(diagnostics, {
          message = item.message,
          col = item.loc.start.column - 1,
          end_col = item.loc["end"].column - 1,
          lnum = item.loc.start.line - 1,
          end_lnum = item.loc["end"].line - 1,
          severity = severities[item.severity],
          source = "textlint",
        })
      end
    end
    return diagnostics
  end,
}
```

nvim-lint で textlint が利用できるようになると、以下の GIF 画像のように textlint の指摘が表示されます。

![nvim-lint](/images/20241208-neovim_textlint_nvim_lint.gif)

<!-- textlint-disable ja-technical-writing/no-exclamation-question-mark -->

これでだいぶ執筆が捗りそうです！

<!-- textlint-enable ja-technical-writing/no-exclamation-question-mark -->

## conform.nvim で修正可能な lint 結果が修正されるようにする

textlint の lint 結果には、漢字とひらがなの表記統一など自動修正が可能なものもあります。
保存して即座に指摘してもらえるだけでもありがたいですが、自動修正可能なものは自動で修正してもらえるとより価値がありそうです。
そこで、conform.nvim で自動修正可能な lint 結果を修正するようにしてみます。
conform.nvim が対応している formatter は [README](https://github.com/stevearc/conform.nvim#formatters) に記載されていますが、やはりこちらも textlint 未対応です。
ただし、conform.nvim も[カスタムの formatter を追加可能](https://github.com/stevearc/conform.nvim#customizing-formatters)ですので、textlint を formatter として追加してみます。

### カスタムの formatter を登録する際に必要な要素

今回も、対応済みの formatter の実装からカスタム formatter を登録するのに必要な要素を確認します。
Biome は以下のように実装されています。

@[github](https://github.com/stevearc/conform.nvim/blob/02fd64fb3d4b18ec029c0e0683c3dc3ec6d2c5b8/lua/conform/formatters/biome.lua)

これを見ると、以下の要素が必要そうです。

- `command` (string) : formatter の実行コマンド
- `stdin` (boolean) : format 対象のテキストを標準入力で渡せるか
- `args` (list) : 実行コマンドに渡す引数のリスト
- `cwd` (string) : カレントディレクトリ

`command`, `stdin`, `cwd` は Biome の実装をほぼ流用できそうです。
実行コマンドに渡す引数を検討します。

### textlint コマンドに渡す引数

textlint の README を見て必要な引数を探します。
まず、自動修正するので `--fix` は必要です。
また、標準入力でテキストを渡すので、nvim-lint 同様 `--stdin`, `--stdin-filename` も必要です。
format については、`--fix` を使う場合以下の選択肢があるようです。

- `compats`
- `diff`
- `fixed-result`
- `json`
- `stylish`

CLI の docs に以下の記述があり、`fixed-result` を選ぶと自動修正された結果が返るようです。

@[github](https://github.com/textlint/textlint/blob/e0038784a28dc87fe5ca638cd2d944f54393c354/docs/cli.md#L85-L94)

ということは、以下の引数を渡せば良さそうと考えて実装してみました。

```lua
{ "--fix", "--stdin", "--stdin-filename", "$FILENAME", "--format", "fixed-result" }
```

ところが、以下の GIF 画像のように保存して更新される際に `WARNING: The file has been changed since reading it!!!` という警告が出てしまいます。

![conform.nvim](/images/20241208-neovim_textlint_conform_nvim_without_dry_run.gif)

なぜだろうと思い conform.nvim の Issue を眺めていたところ、以下のコメントを発見しました。

@[card](https://github.com/stevearc/conform.nvim/issues/498#issuecomment-2246439579)

なるほど、標準出力に書き込まれていないからかということに気付き、であれば引数 `--dry-run` を追加すると良いのではと思い至りました。
これがビンゴで、保存時に上の GIF 画像で出ている警告が出なくなりました。

最終的な実装は以下のようになりました。

```lua:.config/nvim/lua/configs/conform.lua
local options = {
  formatters_by_ft = {
    markdown = { "textlint" },
  },
  formatters = {
    textlint = {
      meta = {
        url = "https://github.com/textlint/textlint",
        description = "The pluggable natural language linter for text and markdown.",
      },
      command = require("conform.util").from_node_modules("textlint"),
      stdin = true,
      args = {
        "--fix",
        "--stdin",
        "--stdin-filename",
        "$FILENAME",
        "--format",
        "fixed-result",
        "--dry-run",
      },
      cwd = require("conform.util").root_file({
        "package.json",
      })
    }
  }
}
```

最終的に、以下の GIF 画像のように自動修正可能な指摘 (以下では `他` → `ほか`) が保存時に自動修正されるよう設定できました。

![conform.nvim](/images/20241208-neovim_textlint_conform_nvim.gif)

<!-- textlint-disable ja-technical-writing/no-exclamation-question-mark -->

執筆体験が非常に良くなりました！

<!-- textlint-enable ja-technical-writing/no-exclamation-question-mark -->

## 終わりに

Neovim の nvim-lint と conform.nvim で textlint の lint 指摘表示および自動修正を可能にする設定方法を紹介しました。
執筆しながら指摘修正ができるのでなくても困らないのですが、一応 commit 前と GitHub Actions の CI でも textlint を実行して、指摘がないことを確認しています。
そちらの設定は以下のリポジトリで公開しているので、気になる方はご覧ください。

@[card](https://github.com/inuatsu/zenn_blog)

