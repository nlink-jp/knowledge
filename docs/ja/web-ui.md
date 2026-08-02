# Web UI（CSS / HTML）

WebView フロントエンド（Wails 等）と自己完結 HTML レポート生成で踏んだ CSS/レイアウトの罠。
各項目は「事象 → なぜ → 適用方法」。

---

## 細い視覚要素にホバー反応を持たせるには hit area を拡張する

**事象:** 高さ 2 px のバーに `pointer-events: none` を付けた状態で hover tooltip を
実装したが、tooltip が一切出なかった。

**なぜ:** 2-3 px は通常のマウス精度で hover を維持できない。さらに
`pointer-events: none` は `:hover` 自体を殺すため二重に発火不能だった。

**適用方法:**
1. 要素高は少なくとも 6 px（理想 8 px）。
2. `linear-gradient(to bottom, transparent 0, transparent <pad>px, <color> <pad>px, ...)`
   で視覚は下端の細い帯だけにする（見た目は変えず hit area だけ拡張）。
3. `pointer-events: none` は付けない。
4. `cursor: help` で affordance を明示。
5. tooltip 用 `::after` には十分な `z-index` を付け、リスト下端の行でも上に重なるようにする。

## 行 transform を使う仮想テーブルでは、セルの z-index は行内でしか効かない

**事象:** 仮想スクロールテーブル（各行を `transform: translateY()` で配置）で、
auto-grow するセルエディタにいくら `z-index` を付けても、下の行のテキストが
オーバーレイの上に描画されて重なった。

**なぜ:** **transform は stacking context を生成する**ため、各行が独立 stacking context に
なる。セルの z-index はその行のコンテキスト内でしか効かず、DOM で後に来る兄弟（下の行）を
超えられない。背景透明の問題と誤認しやすいが、実際は paint order の問題。

**適用方法:**
- セル境界を超えるオーバーレイは、**オーバーレイを含む行ごと** z-index で持ち上げる
  （行は transform で既に stacking context なので `z-index` が効く）:
  `.vt-row-editing { z-index: 20; }`
- context menu / 編集行 / ヘッダなどの z-index 階層を全体で設計する（例: 100 / 20 / 2）。
- デバッグの教訓: 「下の要素が透けて見える」を背景透明と即断しない。opaque でも後の兄弟
  stacking context が上に paint されれば同じ見え方になる。

## undefined CSS variable は silent fallback で UI が壊れる

**事象:** 新規スタイルが参照した `var(--accent-primary)` がどのテーマにも未定義で、
dark テーマではたまたま見えていたが、light テーマで「クリック後にボタンが見えなくなる」。

**なぜ:** 未定義の custom property は initial value（background なら transparent、color なら
black 等）に fallback し、**エラーは出ない**。

**適用方法:**
- `var(--xxx)` を書くときはテーマ定義ファイルに存在するか必ず確認する。
- フォールバック付き（`var(--x, #5078c8)`）はテーマ切替が効かなくなるので非推奨。
- 複数テーマがあるなら**全テーマで**動作確認する（dark で見えているだけかもしれない）。
- 新しい変数を作る前に、既存の theme token から選ぶことを原則にする。

## 自己完結 HTML レポート生成の CSS 注意点

**事象:** CDN 依存なしの単一 HTML レポート生成で、レイアウトバグの修正に 7 回の
イテレーションが必要になった。繰り返し踏んだ点:

1. **overflow-x: auto と overflow-y: visible は共存不可** — CSS 仕様上、一方が
   auto/scroll だともう一方の visible は auto に強制変換される。タブバーで負マージンの
   インジケータ表示は不可 → ボーダーをボタン側に移す。
2. **grid カード内の margin は使わない** — `.card` に margin-bottom を付けると grid 内で
   打ち消しが要り、grid 外との間隔もゼロになる。スペーシングはコンテナ側の `gap` で統一
   （`display: flex; flex-direction: column; gap: 1rem`）。
3. **grid の align-items は stretch より start** — stretch は margin/padding との相互作用で
   段差が出やすい。start の自然な上端揃えが安定。
4. **LLM 出力を CSS クラスに使うなら正規化必須** — LLM は "Excellent" 等の大文字で返すが
   CSS クラスは小文字。テンプレート側で `|lower` を全 badge class に適用する。
5. **@ユーザー名は二重付与防止** — LLM が @ 付きで返す場合があるので、付与ヘルパーで
   先頭 @ をチェックする。
