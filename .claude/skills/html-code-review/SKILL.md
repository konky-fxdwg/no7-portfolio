---
name: html-code-review
description: >
  HTML/CSS/JSファイルのコードレビューを体系的に実施し、問題点を修正してデプロイまで完結させる。
  「コードレビューして」「レビュー観点を洗い出して」「品質チェックして」と言われたとき、
  またはHTMLポートフォリオ・静的サイトの実装後に品質確認を求められたときに使うこと。
  修正のみ・観点洗い出しのみ・フルフローのいずれにも対応する。
---

# html-code-review

HTML/CSS/JS 静的サイトを対象に、**観点洗い出し → Finderフェーズ → 修正 → commit/push** を一連で実施するスキル。

---

## ステップ0: スコープ確認

ユーザーの指示を読み、以下を判断してから進める。

| 指示の内容 | 実施する範囲 |
|---|---|
| 「観点を洗い出して」のみ | Step1のみ。修正しない |
| 「レビューして」 | Step1〜Step3。修正はするがデプロイは確認 |
| 「レビューしながら進めて」 | Step1〜Step4をフルで実施 |

---

## ステップ1: レビュー観点の洗い出し

レビュー対象のファイルを Read して内容を把握したうえで、以下の**7カテゴリ**で観点を表にまとめる。修正はまだ行わない。

### カテゴリ一覧

| カテゴリ | 観点の例 |
|---|---|
| **セマンティクス** | セマンティックHTML（header/main/section/article/footer）、landmark構造、aria属性の有無 |
| **アクセシビリティ** | aria-hidden漏れ、:focus-visible未定義、prefers-reduced-motion未対応、コントラスト比 |
| **デッドコード** | 使われていないCSSクラス、未参照の変数・関数 |
| **UX・動作** | href="#"によるスクロール誤動作、リンク切れ、クリック範囲 |
| **パフォーマンス** | will-change常時適用、レンダリングブロック、不要なGPUレイヤー |
| **デザイン仕様との整合** | デザインYAMLや要件に対する実装差異 |
| **デプロイ品質** | .nojekyll有無、title/favicon/meta description設定、画像パス |

---

## ステップ2: Finderフェーズ（候補収集）

**3つのアングルを並列で Agent として実行する**（同一ターンで全てを起動する）。

### Angle A+B: 行単位スキャン & 削除された動作の監査
```
以下のファイルをコードレビューしてください。バグ・UX破壊・削除された動作の候補を最大6件、JSON配列で返してください。
観点A: 誤った動作・クラッシュ・UX破壊につながるコードを行単位で探す
観点B: 以前のバージョンにあって現バージョンで消えたもの（ガード・エラーパス・機能）を探す
[対象ファイルのパスを列挙]
```

### Angle C+D: セマンティクス・CSS重複
```
以下のファイルをコードレビューしてください。候補を最大6件、JSON配列で返してください。
観点C: HTMLセマンティクス誤用、aria属性の欠落、landmark構造の問題
観点D: 定義されているが使われていないCSSクラス、冗長・重複するプロパティ
[対象ファイルのパスを列挙]
```

### Angle E+F+G: 不必要な複雑さ・パフォーマンス・アクセシビリティ
```
以下のファイルをコードレビューしてください。候補を最大6件、JSON配列で返してください。
観点E: 冗長なCSS・JS、簡略化できる箇所
観点F: will-change過剰使用、レンダリングコスト、モバイル非効率
観点G: prefers-reduced-motion未対応、コントラスト懸念、キーボード操作問題
[対象ファイルのパスを列挙]
```

### 候補のJSON形式
```json
[
  {
    "file": "ファイルパス",
    "line": 行番号,
    "summary": "一文の問題説明",
    "failure_scenario": "具体的な状況→どう壊れるか"
  }
]
```

> **Note**: Agentがファイル読み取り権限を拒否された場合は、自分で直接ファイルを Read して同じ観点でインライン分析する。

---

## ステップ3: Verifyと修正

### 3-1: 候補の重複除去・優先順位付け

| 優先度 | 条件 |
|---|---|
| 🔴 高 | ユーザーに見えるUX破壊（href="#"誤動作、リンク切れ、コンテンツ非表示） |
| 🟡 中 | アクセシビリティ（aria-hidden、focus-visible、prefers-reduced-motion） |
| 🟢 低 | クリーンアップ（デッドCSS、冗長プロパティ、パフォーマンス最適化） |

### 3-2: 修正適用

優先度の高いものから順に Edit ツールで修正する。

**よく出る修正パターン:**

```css
/* will-changeをhover時のみに限定 */
.card { /* will-change: transform; 削除 */ }
.card:hover { will-change: transform; }

/* prefers-reduced-motion対応 */
@media (prefers-reduced-motion: reduce) {
  html { scroll-behavior: auto; }
  .reveal { opacity: 1; transform: none; transition: none; }
  .card  { transition: none; }
  .btn svg { transition: none; }
}

/* focus-visible */
.btn:focus-visible {
  outline: 2px solid var(--accent);
  outline-offset: 3px;
}
```

```html
<!-- SVGにaria-hiddenを追加 -->
<svg aria-hidden="true" ...>

<!-- href="#"を意味のある値に変更 -->
<a href="#works" aria-label="ツール名のデモを見る（準備中）">
```

### 3-3: 修正サマリーを表で出力

修正完了後、以下の形式で報告する:

| # | 問題 | 修正内容 | 対象ファイル |
|---|---|---|---|
| 1 | ... | ... | ... |

---

## ステップ4: commit & push

```bash
git add <変更ファイル>
git commit -m "Apply code review fixes: <カテゴリ一覧>\n\nCo-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>"
git push
```

GitHub Pagesへのデプロイが必要な場合は、Pages のビルド完了を待ってから Chrome で確認する。

---

## チェックリスト（デプロイ前の最終確認）

- [ ] セマンティックHTML（header/main/section/article/footer）が正しく使われている
- [ ] aria-label が各セクションに付与されている
- [ ] SVG装飾要素に `aria-hidden="true"` がある
- [ ] `:focus-visible` スタイルが定義されている
- [ ] `prefers-reduced-motion` 対応が入っている
- [ ] `will-change` がホバー時のみに限定されている
- [ ] デッドCSSクラスが存在しない
- [ ] `href="#"` のリンクが残っていない
- [ ] `.nojekyll` が存在する
- [ ] `<title>`、`<meta name="description">`、favicon が設定されている
- [ ] 画像の `alt` テキストが全て設定されている
