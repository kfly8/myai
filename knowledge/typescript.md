# TypeScript

TypeScriptでのコーディングにおけるプラクティス

## 方針

- 最初に型と、それを処理する関数のインターフェースを考える
- 型安全性を最大限に確保する
- コードのコメントとして、そのファイルがどういう仕様かを可能な限り明記する
- 実装が内部状態を持たないとき、classによる実装を避けて関数を優先する
- 副作用を抽象するために、アダプタパターンで外部依存を抽象化する
- エラーハンドリングを明示的に行う

## 環境

- miseでインストールしたBunを利用する。システムのBunを絶対に利用しない
- `eval "$(mise activate bash)" && bun --version` のようにmise activateを行ってからスクリプトを実行する
- mise.toml には次が記載されていることを確認する

```toml
[tools]
bun = "1.0.24"
```

## コーディングスタイル

- TypeScript 5.0 以上の機能を積極的に利用する
- Strict モードを必ず有効にする
- 暗黙的な `any` 型を許容しない

```typescript
// Good
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "target": "ES2022",
    "lib": ["ESNext"],
    "module": "ESNext",
    "moduleResolution": "Bundler"
  }
}

// Bad
{
  "compilerOptions": {
    "strict": false,
    "target": "ES6"
  }
}
```

## コメント

- TSDoc 形式でドキュメントを記載する
- 関数の冒頭に、その関数が何をするかをコメントする
- *していないこと* に注意が必要な場合は、していない理由もコメントする
- 関数の仕様を簡潔で読みやすい形式に書く
    - 機能説明、使用例、パラメータ説明、戻り値、アルゴリズム（任意）の順に記述

```typescript
/**
 * 指定された上限以下の素数をすべて見つける
 * 
 * エラトステネスのふるいアルゴリズムを使用して素数を検索します
 * 
 * @example
 * ```
 * const primes = findPrimes(10);
 * // primes = [2, 3, 5, 7]
 * ```
 * 
 * @param limit - 検索する上限値
 * @returns 素数のリストまたはエラー
 */
function findPrimes(limit: number): Result<number[], InvalidInputError> {
  // 実装
}
```

## テスト

- Bun:test を使用してユニットテストを書く
- 関数のふるまいをテストする場合、`describe`の説明は関数名でグルーピングし、その配下に`it`でふるまいのテストを書く
- テーブル駆動テストを活用する

```typescript
import { describe, expect, it } from 'bun:test';

describe('findPrimes', () => {
  it('有効な入力の場合、素数リストを返す', () => {
    const testCases = [
      { limit: 10, expected: [2, 3, 5, 7] },
      { limit: 20, expected: [2, 3, 5, 7, 11, 13, 17, 19] },
    ];
    
    for (const { limit, expected } of testCases) {
      it(`${limit}以下の素数は${expected}`, () => {
        const result = findPrimes(limit);
        expect(result.isOk()).toBe(true);
        expect(result.value).toEqual(expected);
      });
    }
  });
  
  it('無効な入力の場合、エラーを返す', () => {
    const testCases = [
      { limit: -1 },
      { limit: 0 },
    ];
    
    for (const { limit } of testCases) {
      it(`${limit}は無効な入力`, () => {
        const result = findPrimes(limit);
        expect(result.isErr()).toBe(true);
        expect(result.error).toBeInstanceOf(InvalidInputError);
      });
    }
  });
});
```

## 値の型定義

- 意図を明確に伝える型名を使用する
- 実用的なユーティリティ型を活用する
- 型の再利用を促進する

```typescript
// Good
type UserId = string;
type UserName = string;

type UserData = {
  id: UserId;
  name: UserName;
  createdAt: Date;
};

// Bad
type Data = any;
```

- `type` キーワードを優先して型を定義する
  - インターフェースの継承や実装よりも型合成を優先する

```typescript
// Good
type User = {
  id: UserId;
  name: UserName;
};

// Bad
interface User {
  id: UserId;
  name: UserName;
}

class UserImpl implements User {
  id: UserId;
  name: UserName;

  constructor(id: UserId, name: UserName) {
    this.id = id;
    this.name = name;
  }
}
```

## エラーハンドリング

エラーを表現する場合、タグ付きユニオン型を利用する

```typescript
import { err, ok, Result } from 'neverthrow';

type ApiError =
  | { type: "network"; message: string }
  | { type: "notFound"; message: string }
  | { type: "unauthorized"; message: string };

async function fetchUser(id: string): Promise<Result<User, ApiError>> {
  try {
    const response = await fetch(`/api/users/${id}`);
    if (!response.ok) {
      switch (response.status) {
        case 404:
          return err({ type: "notFound", message: "User not found" });
        case 401:
          return err({ type: "unauthorized", message: "Unauthorized" });
        default:
          return err({
            type: "network",
            message: `HTTP error: ${response.status}`,
          });
      }
    }
    return ok(await response.json());
  } catch (error) {
    return err({
      type: "network",
      message: error instanceof Error ? error.message : "Unknown error",
    });
  }
}
```

### Result型

`neverthrow` パッケージを使用して Result 型でエラーハンドリングを実装する

```typescript
import { ok, err, Result } from 'neverthrow';
import { z } from 'zod';

const UserNameSchema = z.string().min(3).max(20);
type UserName = z.infer<typeof UserNameSchema>;

type CreateUserError = 
     | { type: "invalidName"; message: string }
     | { type: "forbidden"; message: string };

function createUser(name: string): Result<User, CreateUserError> {
  const nameResult = UserNameSchema.safeParse(name);
  
  if (!nameResult.success) {
    return err({ type: "invalidName", message: '名前は3文字以上20文字以下である必要があります' });
  }
  
  if (name.toLowerCase() === 'admin') {
    return err({ type: "forbidden", message: 'この名前は予約されています' });
  }
  
  const newUser = {
    id: crypto.randomUUID(),
    name: nameResult.data as UserName,
  };
  
  return ok(newUser);
}

// 使用例
const userResult = createUser('johndoe');

if (userResult.isOk()) {
  console.log(`ユーザー作成成功: ${userResult.value.name}`);
} else if (userResult.error instanceof InvalidUserNameError) {
  console.error(`無効な名前: ${userResult.error.message}`);
} else {
  console.error(`エラー: ${userResult.error.message}`);
}
```

## 関数のエクスポート

- 明示的に関数をエクスポートする
- 名前付きエクスポートを優先する

```typescript
// Good
export function add(x: number, y: number): number {
  return x + y;
}

// Acceptable for utility/helper functions
export const multiply = (x: number, y: number): number => x * y;

// Bad
export default function(x: number, y: number): number {
  return x + y;
}
```

## 依存モジュールの管理

- `package.json` でバージョンを明示する
- Bun を使用してロックファイルを維持する
- 開発依存性と実行時依存性を適切に区別する

```json
{
  "dependencies": {
    "neverthrow": "^6.0.0",
    "zod": "^3.22.4"
  },
  "devDependencies": {
    "typescript": "^5.2.2",
    "bun-types": "^1.0.11"
  }
}
```

- 依存モジュールは `bun add` で追加する

```bash
bun add neverthrow zod
bun add -d typescript bun-types
```

## アーキテクチャパターン

### 副作用の抽象化

副作用を持つコードは、アダプタパターンを使用して抽象化します。

```typescript
// インターフェース定義
type UserRepository = {
  findById(id: string): Promise<Result<User, RepositoryError>>;
  save(user: User): Promise<Result<User, RepositoryError>>;
};

// 実装
const PostgresUserRepository: UserRepository = {
  async findById(id: string): Promise<Result<User, RepositoryError>> {
    try {
      const user = await db.query('SELECT * FROM users WHERE id = $1', [id]);
      if (!user) {
        return err(new UserNotFoundError(id));
      }
      return ok(user);
    } catch (error) {
      return err(new DatabaseError('ユーザー検索中にエラーが発生しました', { cause: error }));
    }
  },
  
  async save(user: User): Promise<Result<User, RepositoryError>> {
    // 実装
    return ok(user);
  }
};

// テスト用モック
const InMemoryUserRepository: UserRepository = {
  users: new Map<string, User>(),
  
  async findById(id: string): Promise<Result<User, RepositoryError>> {
    const user = this.users.get(id);
    if (!user) {
      return err(new UserNotFoundError(id));
    }
    return ok(user);
  },
  
  async save(user: User): Promise<Result<User, RepositoryError>> {
    this.users.set(user.id, user);
    return ok(user);
  }
};
```
