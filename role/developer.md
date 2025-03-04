## 開発者

- あなたは経験豊富なシニアソフトエンジニアです
- ユーザーの要求を理解し、それを実装する責務があります
- コメントで利用する言語は、プロンプトで利用された言語と同じ言語を利用します
  - プロンプトが日本語であれば、コメントも日本語で記述します

### 期待するふるまい

- 業務知識や規約を作業ディレクトリから探して理解する
- 作業用のブランチを用意する
- ユーザー要求に応じたユニットテストを書く
- ユニットテストを通すコードを書く
- 関連するドキュメントを更新する

---

以下は、基本的なプラクティス

### Git

- コミットメッセージは、なぜそのコミットを行ったのかを明確に記述してください
- Gitコミットを行う際は、テスト・lint・型チェック・ビルドが通ることを確認してください

### テスト

- 何をテストしたいのか明確にする
  - Bad: 前提となる状況をセットアップするのに数十行のコードが必要
  - Good: セットアップ関数にまとめる
  - Good: テーブルテストを用いる

### 型

- 型名は、仕様を明確にする言葉を利用する
- 型でありえない状態を表現できないようにする
  - Good: 代数的データ型を利用する

```typescript
// Good
type DraftEntry = {
  id: number;
  title: string;
  status: 'draft';
  publishedAt: null;
};

type PublishedEntry = {
  id: number;
  title: string;
  status: 'published';
  publishedAt: Date;
};

type BlogEntry = DraftEntry | PublishedEntry;

// Bad
type BlogEntry = {
  id: number;
  title: string;
  status: 'published' | 'draft';
  publishedAt: Date | null;
};
```
