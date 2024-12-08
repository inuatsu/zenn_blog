---
title: dotfiles を育て始めた話
emoji: "\U0001F6E0️"
type: tech
topics:
  - dotfiles
  - Neovim
published: true
publication_name: simpleform_blog
---

本記事は [SimpleForm Advent Calendar 2024](https://qiita.com/advent-calendar/2024/simpleform) の 1 日目の記事です。

2024 年 10 月ころから dotfiles をパブリックリポジトリで管理し始めました。
この記事では、作り始めたきっかけや導入しているツールなどについて紹介します。

そんな話よりも早くコードを見たいという方は、以下のリンクからご参照ください。

@[card](https://github.com/inuatsu/dotfiles)

## dotfiles を管理し始めたきっかけ

筆者は 2023 年 5 月から Neovim を使っているのですが、当時いくつかのディストリビューションを調べた上で、[NvChad](https://nvchad.com/) を利用して Neovim を使い始めました。
最初に設定してから特に手を入れずに使っていましたが、2023 年 8 月にアーカイブされた null-ls.nvim をずっと使っていることに危機意識を持ち始めました。
そろそろ脱 null-ls せねばと重い腰を上げて、2024 年 9 月末から移行先の候補を調査していました。
その際に、[2024 年 3 月に NvChad の新しいバージョンがリリースされている](https://github.com/NvChad/NvChad/releases/tag/v2.5)ことに気付いたのでアップデートを試みました。
しかし、少しカスタマイズしていたことや、新しいバージョンでは [starter template](https://github.com/nvchad/starter) が使われるようになっていたことなどから、アップデートでつまずいて環境を壊してしまいました...。

使い始めた当初は Neovim の知識もあまりなくそれほど凝ったカスタマイズをしていたわけでもないので、アップデートは諦めて最初から設定し直すことにしました。
せっかく作り直すなら、今後違うマシンを使う時にも簡単に移行できるよう dotfiles 管理もしたいなと考えたことが、上のリポジトリを作り始めたきっかけです。

## dotfiles 管理こと始め

dotfiles 管理をするにあたり、ほかの人はどう管理しているのかなということで Google で検索して上位にヒットした記事をいくつか読んでみました。
以下の記事を割と参考にした記憶があります。

@[card](https://qiita.com/reireias/items/b33b5c824a56dc89e1f7)

とりあえず以下はやりたいなと方針を立てました。

- Ubuntu と macOS どちらでも導入できるようにする
  - 筆者は [Pop!\_OS](https://pop.system76.com/) ユーザーだが、とりあえず macOS でも使えるようにしたいなくらいのモチベーション
- CI を回すようにする
  - CI を回して Ubuntu でも macOS でもインストールできることを保証したいというモチベーション
- PR 以外からの main ブランチへのコミットを禁止し、CI が通ることを PR マージの必須条件とする
  - 個人開発とはいえ開発プロセスはちゃんとしておこうかなというモチベーション

最初は最低限の Neovim の設定と CI の整備を進め、10 月中旬には上の方針を満たす形で開発が進められるようになりました。

## 現在 dotfiles で導入しているツールなど

この 2 ヵ月 dotfiles を育ている中で、新しいツールなども導入したのでざっと紹介します。

### シェルプロンプト : [Starship](https://starship.rs/)

もともと [Powerlevel10k](https://github.com/romkatv/powerlevel10k) を使っていたのですが、Starship は toml ファイルで設定ができカスタマイズしやすいと知り、乗り換えてみました。
Powerlevel10k はウィザードでインタラクティブに初期セットアップでき導入は簡単ですが、生成される設定ファイルが 1,000 行以上の長大なファイルで、設定をいじりづらい印象がありました。
そんなに凝ったカスタマイズはしていませんが、toml ファイルなのでサクッとカスタムの設定を追加でき良いです。

### 開発ツール管理 : [mise](https://mise.jdx.dev/)

筆者は普段 Python、Ruby、Node.js を書くことが多く、これまでは pyenv、rbenv、nodeenv などで各言語のバージョン管理をしていました。
そろそろ asdf や mise などあらゆる言語を管理できるツールを試してみたいなと思い、mise を使ってみることにしました。
以前は Neovim で使う LSP、Linter/Formatter は Mason でインストールしていましたが、これを機に mise や pnpm でのインストールを基本とすることにしました。

### シェルプラグインマネージャー : [Sheldon](https://sheldon.cli.rs/)

もともと [Zinit](https://github.com/zdharma-continuum/zinit) を使っていたのですが、そこまでこだわりはなかったのと Sheldon が早い & toml ファイルで設定を管理できると知り、使ってみることにしました。
Zinit を使っていた時 .zshrc にプラグインのインストールを書いていたことと比べると、toml ファイルにプラグインの管理をまとめられるのは .zshrc がスッキリして良いと感じています。
最大限遅延読み込みを効かせて、いまの zsh の起動時間はだいたい 50 ms 程度です。

![zsh の起動速度](/images/20241201-start_dotfiles_zsh_startup_time.png)

### ターミナルエミュレータ : [WezTerm](https://wezfurlong.org/wezterm/index.html)

Pop!\_OS では Terminator を使っていましたが、Ubuntu と macOS どちらでも導入できるようにクロスプラットフォームで使えるものを導入したいと考えました。
さらに言えば、ファイルで設定を管理できるとなお良いです。
いくつか候補はありましたが、Neovim ユーザーとしては Lua で設定ファイルを書けるというのに興味を惹かれて WezTerm を導入してみることにしました。
とはいえ、現時点では非常に簡素な設定しかしていないのであまり Lua で設定ファイルを書ける意味もない感はありますが、これからもう少しいろいろな設定に手を出したいなと感じるところです。

### テキストエディタ : [Neovim](https://neovim.io/)

Neovim の環境を作り直したのを機に、便利そうなプラグインを探して新しく導入してみました。
現時点で使っている主なプラグインを紹介します。

#### プラグイン管理 : [lazy.nvim](https://lazy.folke.io/)

Neovim を使い始めた当初から変わらず使っています。
最近 [rocks.nvim](https://github.com/nvim-neorocks/rocks.nvim) も気になっているものの、まだ手を出せていないです。

#### LSP 設定 : [nvim-lspconfig](https://github.com/neovim/nvim-lspconfig)

こちらも以前から使っていたのですが、Vue3 + TypeScript 向けの Vue の LSP の設定に少し工夫が必要でした。
筆者はいったん以下のような設定をしています。

```lua:lspconfig.lua
local lspconfig = require "lspconfig"

lspconfig[volar].setup {}

lspconfig.ts_ls.setup {
  init_options = {
    plugins = {
      {
        name = "@vue/typescript-plugin",
        location = "$HOME/.local/share/pnpm/global/5/node_modules/@vue/typescript-plugin",
        languages = { "vue" },
      },
    },
  },
  filetypes = {
    "javascript",
    "javascriptreact",
    "javascript.jsx",
    "typescript",
    "typescriptreact",
    "typescript.tsx",
    "vue",
  },
}
```

#### formatter : [conform.nvim](https://github.com/stevearc/conform.nvim)

NvChad をアップデートしたところ conform.nvim が使われていたので、null-ls.nvim から乗り換えて使うことにしました。
いまのところ自分が実現したいことを設定できており、特に困っていません。
むしろ、以前 null-ls を使っていた時にはあまり理解せず formatter や linter を設定していたので、いまの方がより良い感じに使えています。

たとえば、Python や Node.js の場合、各プロジェクトディレクトリの環境に formatter をインストールすることがあります。
そのため、プロジェクトディレクトリの環境の formatter を優先的に使う設定をしています。
どのように設定しているかを書くと長くなるので、具体的な設定については別で記事を書きます。

#### linter : [nvim-lint](https://github.com/mfussenegger/nvim-lint)

conform.nvim は formatter しか対応していないので、linter はどうしようかなと調査してこちらを使うことにしました。
conform.nvim と同様、Python や Node.js でプロジェクトディレクトリにインストールされている linter を優先的に使うよう設定しています。
また、conform.nvim だと `ConformInfo` というコマンドで以下の画像のようにどのパスにある formatter が有効化されているかなどを確認する画面が表示されます。

![ConformInfo](/images/20241201-start_dotfiles_conforminfo.png)

conform.nvim を設定する際、想定通りの formatter が有効化されているか上の画面で確認でき非常に便利だったのですが、nvim-lint には残念ながら同様の画面がありません...。
nvim-lint の Issue も見てみましたが、<https://github.com/mfussenegger/nvim-lint/issues/559#issuecomment-2010049274> のやりとりを見る限りそのような画面の実装も予定されていなさそうですので、自分で以下のような画面を作ってみました。

![LinterInfo](/images/20241201-start_dotfiles_linterinfo.png)

#### 通知 : [fidget.nvim](https://github.com/j-hui/fidget.nvim)

Ruby LSP など起動に少し時間がかかる LSP もあるので、起動状況を通知で見られると良いなと思い導入しました。
LSP の起動状況が通知されるのを見ると、formatter や linter の起動状況も通知で見たいなと感じるようになり、以下のように右下に通知が出るように設定しました。

![fidget.nvim](/images/20241201-start_dotfiles_fidget.png)

#### コード解析結果表示 : [tiny-inline-diagnostic.nvim](https://github.com/rachartier/tiny-inline-diagnostic.nvim)

デフォルトでの LSP や linter でのコード解析結果表示がやや見づらいと感じていたので導入しました。
以下のように複数行にまたがっての表示や、2 件以上の警告やエラーの表示もされるので、ほかのツールを使わなくてもエラーや警告の内容が把握できるようになり快適です。

![tiny-inline-diagnostic.nvim](/images/20241201-start_dotfiles_diagnostic.gif)

#### そのほか : [hardtime.nvim](https://github.com/m4xshen/hardtime.nvim)

"Establish good command workflow and quit bad habit" と紹介されているこちらのプラグイン。
Vim で非効率なコマンド操作をできなくすることで、効率的なコマンド操作ができるようにユーザーを矯正してくれるものです。
上下左右の移動で hjkl のキーを 4 回以上連続でたたくと一定期間移動できなくなる、キーボードの上下左右のキーでは移動できなくなるなど、非効率なキー操作をやめるのに一定効果があるかなと感じます。
このプラグインを使い始めて、相対的な行番号表示を使う理由が良く理解できました。

## dotfiles を用意して良かったこと

2024 年 10 月上旬から dotfiles を GitHub で管理し始めた矢先、何と愛用していた Pop!\_OS のマシンが 10 月下旬に故障で電源が入らなくなってしまいました...。
会社にあった代替機の macOS のマシンを使うことができたので事なきを得ましたが、dotfiles を用意していたおかげでかなりスムーズに以前同様の開発環境を再現できました。
クロスプラットフォームで dotfiles 用意しておいたことの効能をこんなに早く体験することになるとは... (笑) という出来事でした。

また、CI を整備していたことの効能を実感する出来事もありました。
ある日いつも通りコミットして PR を作成すると、CI が [`mise ERROR aws-sso-cli not found in mise tool registry` というエラー](https://github.com/inuatsu/dotfiles/actions/runs/12001098349/job/33451134094#step:3:225)で失敗しました。
エラーメッセージ内の[aws-sso-cli](https://github.com/synfinatic/aws-sso-cli) は、筆者が mise でインストールしているツールの 1 つです。
ただ、その PR で修正した箇所とまったく関係ないのにな...と思い調べたところ、[mise のこの PR](https://github.com/jdx/mise/pull/3179) で aws-sso-cli の名前が aws-sso に変わっていたことが原因と判明しました。
ローカル環境ではすでにインストール済みだったためエラーにならず気付いていなかったのですが、CI がエラーになったので新規インストール時にはエラーになってしまうことに気付けた好例でした。

## 終わりに

最後まで読んでいただきありがとうございました。
もし興味を持っていただけましたら、ぜひリポジトリも覗いていただけると幸いです (リンク再掲します)。

@[card](https://github.com/inuatsu/dotfiles)

今回は使っているツールを網羅的に紹介しましたが、いくつかもっと詳しい実装上のこだわりを書きたい部分もあるので、そのあたりはまた別記事を書きます。
