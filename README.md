# Search Link Generator

指定したキーワードに基づいて、複数の検索サイトへのリンクをまとめて生成するツール。

カンマ区切りで複数のキーワードを入力すると、キーワードごとにGoogle・YouTube・Amazon・X・ニコニコ動画など各サイトの検索リンクをカード形式で一覧表示します。

## 機能

- 複数キーワードの一括リンク生成
- 14サイト対応（Google / YouTube / Amazon / DMM / メルカリ / Wikipedia / Google Scholar / X / Bluesky / ニコニコ動画 / 青空文庫 / 国会図書館DC / CiNii）
- サイトごとの絞り込みフィルタ（新着順・高評価順など）
- Google検索を信頼性の高いドメイン（ac.jp / go.jp 等）に限定するオプション
- シャッフル機能・検索履歴の保存
- ダーク/ライトモード切替

## デモ

https://mayholiholi.github.io/search-link-generator/

## 開発

```bash
npm install
npm run dev
```

Vue.js + Vite で構築しています。
