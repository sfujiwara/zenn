---
title: "VSCode で程々に Emacs 風キーバインド"
emoji: "👌"
type: "tech"
topics:
  - "vscode"
published: false
---

# はじめに

VSCode の元々のショートカットをなるべく潰さない程度に Emacs ライトユーザーの自分が使いやすいようにキーバインドの設定をいじったのでメモ。

たぶん Emacs を使わない人でも便利に使えるはず。
ただし Control キーを Capslock と入れ替えていない人が真似すると、指がもげて死ぬので注意されたし。

# カーソルの移動

Emacs のカーソル移動をほぼそのまま再現できるようにしている。
Mac の場合はわざわざ設定しなくても大丈夫だが、業務以外では Ubuntu を使っているので。

```json
    {
        "key": "ctrl+f",
        "command": "cursorRight"
    },
    {
        "key": "ctrl+b",
        "command": "cursorLeft"
    },
    {
        "key": "ctrl+p",
        "command": "cursorUp"
    },
    {
        "key": "ctrl+n",
        "command": "cursorDown"
    },
    {
        "key": "ctrl+a",
        "command": "cursorHome"
    },
    {
        "key": "ctrl+e",
        "command": "cursorEnd"
    },
```

# 削除関連

カーソルの右側の文字を削除するやつ。
`ctrl+d` は最悪なくても死なないが、`ctrl+k` はめちゃくちゃ使うのでないと死ぬ。

```json
    {
        "key": "ctrl+d",
        "command": "deleteRight"
    },
    {
        "key": "ctrl+k",
        "command": "deleteAllRight"
    },
```

# 補完候補の選択

Emacs と同じように `ctrl+p` `ctrl+n` で補完候補の選択をできるようにした。
個人的には必須の設定。
めちゃくちゃ頻繁にやる操作なので、そのためにカーソルキーに手を伸ばしたくない。

```json
{
        "key": "ctrl+n",
        "command": "selectNextSuggestion",
        "when": "suggestWidgetMultipleSuggestions && suggestWidgetVisible && textInputFocus || suggestWidgetVisible && textInputFocus && !suggestWidgetHasFocusedSuggestion"
    },
    {
        "key": "ctrl+p",
        "command": "selectPrevSuggestion",
        "when": "suggestWidgetMultipleSuggestions && suggestWidgetVisible && textInputFocus || suggestWidgetVisible && textInputFocus && !suggestWidgetHasFocusedSuggestion"
    },
```

# ターミナルへの移動

これは Emacs とは特に関係なし。

ターミナルとエディタの移動はキーボードで完結させたい。
ターミナルからエディタへの移動デフォルトで `ctrl+1` `ctrl+2` などが割り当てられているので、それに合わせて `ctrl+0` でエディタからターミナルへ移動できるようにした。

```json
    {
        "key": "ctrl+0",
        "command": "workbench.action.terminal.focus"
    },
```

ちなみに Exploere への移動はデフォルトで `ctrl+shift+e` というショートカットが用意されている。
こっちに合わせるなら `ctrl+t` や `ctrl+shift+t` でも良いかもしれない。

# 競合するやつ

Mac の場合はデフォルトショートカットが cmd キーを使うことが多いので大丈夫だが、Ubuntu の場合はここまでの設定でいくつかの重要なデフォルトショートカットが潰されてしまう。

## 全選択

`ctrl+a` がカーソル移動に取られているので、全選択のショートカットが潰されてしまった。
仕方がないので `ctrl+shift+a` に割り当てている。
まあ頻繁にやる操作ではないので、それほど困ってはいない。

```json
    {
        "key": "ctrl+shift+a",
        "command": "editor.action.selectAll"
    },
```


## 検索

`ctrl+f` がカーソル移動に取られているので、検索のショートカットも潰されてしまっている。
`ctrl+shift+f` にすると、今度はワークスペース内の検索とぶつかってしまうので困った。
Emacs キーバインドと合わせるなら `ctrl+s` でも良いが、これはこれで保存のショートカットと被ってしまう。

結局どうしたかというと、`ctrl+h` の置換で代用できるので別になくても良いことに気付いた。

一応申し訳程度に `ctrl+shift+s` に割り当ててはいる。

```json
    {
        "key": "ctrl+shift+s",
        "command": "actions.find",
        "when": "editorFocus || editorIsOpen"
    },
```

# まとめ

元々の VSCode のショートカットを潰さない程度に Emacs のキーバインドが効くようになって概ね満足。

Mac は cmd キーがあるのでショートカットキーが競合しないの偉いんだよな。
Ubuntu にもくれ。
