# メモ: Neovim の lsp.log が 232GB になってストレージを食い尽くした話

## 想定タイトル案

- 「Neovim の lsp.log が 232GB になってた話と対処法」
- 「ストレージが 300GB 圧迫されていた原因は Neovim の LSP ログだった」
- 「rust-analyzer のパニックループで lsp.log が爆発した話」

## 想定読者

- Neovim + rust-analyzer を使っている開発者
- Mac のストレージ圧迫の原因がわからず困っている人
- LSP 周りのトラブルシューティングに興味がある人

---

## 事象

Mac のストレージが 300GB 以上圧迫されていることに気づいた。
「書類」カテゴリでの使用量が異常に大きく表示されており、原因がわからない状態。

ディスク構成：

- SSD 500GB
- Data ボリューム: 362GB 使用中（77.8%）

---

## 原因調査の流れ

### 1. まずホームディレクトリの大まかな内訳を確認

```bash
du -sh ~/Documents ~/Downloads ~/Library ~/Pictures ~/Movies ~/Music
```

結果：Library が 25GB、その他は数GB 程度。合計しても 300GB に届かない。

### 2. 隠しフォルダに目を向ける

```bash
for d in /Users/<name>/.*; do
  size=$(du -sk "$d" 2>/dev/null | awk '{print $1}')
  if [ -n "$size" ] && [ "$size" -gt 500000 ]; then
    printf "%.2fG\t%s\n" "$(echo "$size / 1024 / 1024" | bc -l)" "$d"
  fi
done
```

**`~/.local` が 236GB** というとんでもない数字が出た。

### 3. .local の内訳を掘り下げる

```bash
du -sk ~/.local/* | sort -rn
# → .local/state  232GB
# → .local/share  3.75GB

du -sk ~/.local/state/* | sort -rn
# → .local/state/nvim  232GB

du -sk ~/.local/state/nvim/* | sort -rn
# → lsp.log  232GB  ← 犯人
```

### 4. ログの中身を確認

```bash
tail -50 ~/.local/state/nvim/lsp.log
```

```
[ERROR][2026-02-28 09:50:09] ...p/_transport.lua:36  "rpc"  "rust-analyzer"  "stderr"
"WARN Propagating panic for cycle head that panicked in an earlier execution in that revision"
```

**rust-analyzer が同じエラーを何百万行も垂れ流していた。**

---

## 原因の詳細

### なぜ lsp.log が肥大化したか

1. **rust-analyzer の内部バグ** により、循環参照の解析中にパニックが発生
   - salsa（インクリメンタルコンパイルのクエリエンジン）の計算グラフ内でサイクルを検出した際、前回の実行でパニックした「サイクルの先頭ノード」のパニック状態が現在の実行に伝播するバグ
   - バージョン 0.3.2786 のリグレッション。保存のたびに salsa のサイクル再計算がトリガーされ、そのたびにパニック伝播の警告が出続ける
   - 参照: [rust-analyzer Issue #21610](https://github.com/rust-lang/rust-analyzer/issues/21610), [rust-analyzer Issue #21617](https://github.com/rust-lang/rust-analyzer/issues/21617), [rust-analyzer Issue #2446](https://github.com/rust-lang/rust-analyzer/issues/2446)

2. `"Propagating panic for cycle head..."` というエラーが**ループして大量出力**された
   - Issue #21617 では「ログに同メッセージが繰り返し出力される」と複数ユーザーが報告
   - 参照: [rust-analyzer Issue #21617](https://github.com/rust-lang/rust-analyzer/issues/21617)

3. Neovim は LSP の通信内容をデフォルトで `lsp.log` に **WARN レベル以上** 書き出す
   - デフォルトのログレベルは `WARN`。今回の rust-analyzer のエラーも WARN として stderr に出力されているため、デフォルト設定でそのまま記録される
   - 参照: [neovim/runtime/lua/vim/lsp/log.lua](https://github.com/neovim/neovim/blob/master/runtime/lua/vim/lsp/log.lua)

4. ログのローテーション機能がないため、無制限に肥大化し続けた
   - log.lua のソースコード上、1GB を超えると警告通知が出るが、ローテーション・自動削除は一切行われない。追記モードで書き続けるだけ
   - 250GiB に達したユーザーの実例報告もある
   - 参照: [LunarVim Issue #4537](https://github.com/LunarVim/LunarVim/issues/4537), [Neovim Discourse: "LSP log file grows infinitely"](https://neovim.discourse.group/t/lsp-log-file-grows-infinitely/3596), [neovim Issue #32106](https://github.com/neovim/neovim/issues/32106)

### lsp.log の場所

```
~/.local/state/nvim/lsp.log
# = vim.fn.stdpath("state") .. "/lsp.log"
```

### rust-analyzer のエラーメッセージ

```
Propagating panic for cycle head that panicked in an earlier execution in that revision
```

rust-analyzer が内部の依存グラフ解析（salsa というクエリエンジン）で循環を検出した際に
パニックが伝播し、そのワーニングが止まらなくなるというバグ。

---

## 対処

### 即時対応：ログファイルを削除

```bash
rm ~/.local/state/nvim/lsp.log
```

これだけで **232GB をそのまま回収**できた。

### 再発防止：起動時に古いログを自動削除

`~/.config/nvim/lua/config/options.lua` に追加：

```lua
-- LSPログの肥大化防止: 起動時に1日以上古いlsp.logを削除
local lsp_log = vim.fn.stdpath("state") .. "/lsp.log"
local stat = vim.uv.fs_stat(lsp_log)
if stat and (os.time() - stat.mtime.sec) > 86400 then
  os.remove(lsp_log)
end
```

`vim.uv` は Neovim 0.10 以降で使える標準の libuv バインディング。
`stdpath("state")` は OS に依存せず正しいパスを返すので、移植性も高い。

### rust-analyzer 側の根本対処

```bash
# rust-analyzer を最新版に更新
rustup update

# 問題が起きているプロジェクトのビルドキャッシュをリセット
cargo clean
```

---

## 教訓・まとめポイント

- Mac の「書類」ストレージ圧迫は **隠しフォルダ (dotfiles) が原因**のことがある
- Neovim の LSP ログ (`lsp.log`) は**無制限に増加する**ため注意が必要
- rust-analyzer はバグでエラーをループ出力することがある
- `du` コマンドで隠しフォルダも含めて調査するのが有効
- ログファイルは定期的に確認・削除する仕組みを入れておくべき

---

## 記事に入れると良さそなもの

- [ ] `diskutil apfs list` でボリューム単位の使用量を確認する方法
- [ ] `df -h` と `du` の違いと使い分け
- [ ] Neovim の `stdpath` の各ディレクトリ一覧（state / cache / data / config）
- [ ] LSP ログレベルを変更する方法 (`vim.lsp.set_log_level("WARN")` など）
- [ ] rust-analyzer の issue リンク（該当バグがあれば）

---

## 参考

- Neovim ドキュメント: `:help lsp-log`
- rust-analyzer: salsa cycle handling

## タグ候補（Zenn）

`neovim`, `rust`, `rust-analyzer`, `lsp`, `mac`, `troubleshooting`
