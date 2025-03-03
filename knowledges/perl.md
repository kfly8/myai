## Perl

Perlでのコーディングにおけるプラクティス

## 方針

- 最新のモダンPerlを利用する
- 値の型制約を宣言する
- パッケージに、NAME,SYNOPSIS,DESCRIPTIONを記載する
- 関数の冒頭に、その関数が何をするかを最低限コメントする
- アダプタパターンで副作用を抽象化する
- 純粋関数を優先する
- 例外よりもエラーを返す

## 環境

- miseでインストールしたperlを利用する
- 作業ディレクトリに「perl 5.40.0」と書いた.tool-versionsファイルを作成し、mise activateを行ってからスクリプトを実行する

## コーディングスタイル

- プラグマは、v5.40を利用する
- utf8を利用する
- subroutine signaturesを利用する
- パッケージの最後の行に1;の記載は不要

```perl
# Good
package MyModule;
use v5.40;
use utf8;

sub add($x, $y) {
    return $x + $y;
}

# Bad
package MyModule;
use strict;
use warnings;
use utf8;

sub add {
    my ($x, $y) = @_;
    return $x + $y;
}

1;
```

## テストの書き方

- ユニットテストは、`Test2::V0` を利用する
- テストの説明は、"前提となる状況"、"実行する操作"、"期待する結果"を書く
- 何をテストしたいのか明確にする
  - Bad: 前提となる状況をセットアップするのに数十行のコードが必要
  - Good: セットアップ関数にまとめる
  - Good: テーブルテストを用いる

```perl
use v5.40;
use Test2::V0;

subtest 'add' => sub {
    my @cases = (
        # x, y => expected
        1, 2 => 3
        2, 3 => 5
        3, 4 => 7
    );

    for my ($x, $y, $expected) (@cases) {
        is add($x, $y), $expected, "add $x and $y";
    }
};

done_testing;
```

- テストのmatcherは、`is` を利用する
- `like`, `unlike` は用いず、`match` を利用する

```perl
sub hello($name) { return "Hello, $name!" }

# Good
is hello('bar'), match qr/^Hello, /, 'say hello to bar';

# Bad
like hello('bar'), qr/^Hello, /, 'say hello to bar';
```

## 値の型制約を宣言する

- `Types::Standard` と `kura` を利用して型制約を宣言する
- 型制約は、仕様を明確にする言葉を利用する

```perl
# Good
use Types::Standard -types;

use kura UserId => Str;
use kura UserName => Str;

use kura UserData => Dict[
    id => UserId,
    name => UserName,
];

# Bad
use Types::Standard -types;

use kura Data => HashRef;
```

- `Syntax::Keyword::Assert` を利用して、値のチェックを行う

```perl
use Syntax::Keyword::Assert;

assert( UserData->check($data) );
```


## エラー処理

1. 想定しているエラーは、エラーメッセージを持つエラークラスを継承した子クラスをエラーケースごとに用意する

```perl
use v5.40;
use experimental 'class';

class My::Error {
    field $message :params :reader;
}

class My::Error::InvalidUserName :isa(My::Error);
class My::Error::ForbiddenUserName :isa(My::Error);
```

2. `Result::Simple` でResult型を返す

```perl
use Result::Simple;

use Types::Standard -types;

use kura User => InstanceOf['My::User'];
use kura ErrorInvalidUserName => InstanceOf['My::Error::InvalidUserName'];
use kura ErrorForbiddenUserName => InstanceOf['My::Error::ForbiddenUserName'];
use kura Error => ErrorInvalidUserName | ErrorForbiddenUserName;

sub new_user :Result(User, Error) ($data) {
    assert( UserData->check($data) );

    if (my $err = UserName->validate($data->{name})) {
        return Err(ErrorInvalidUserName(message => 'invalid user name'));
    }

    state $uuid = Data::UUID->new;

    my $user = My::User->new(
        id => $uuid->create_str,
        name => $data->{name},
    );

    return Ok($user);
}

my ($user, $err) = new_user({name => 'foo'});
if ($err isa ErrorInvalidUserName) {
    say $err->message;
}

say "Hello, @{[$user->name]}!";
```

3. 想定外のエラーは、組み込みのtry/catchを利用し、Result::Simpleでエラーを返す。Try::Tinyは使わない

```perl
use v5.40;

use Result::Simple;

sub foo {
    try {
        die 'Unexpected error';
    }
    catch($e) {
        return Err(ErrorUnexpected(message => $e));
    }
}
```

## 関数のエクスポート

- 純粋関数は、Exporterを利用してエクスポートする
- `@EXPORT_OK` にエクスポートする関数を列挙する

```perl
package MyModule;
use v5.40;
use utf8;

use Exporter 'import';

our @EXPORT_OK = qw(add);

sub add($x, $y) {
    return $x + $y;
}
```

## 依存モジュールの管理

- `cpanfile` で依存モジュールを管理し、バージョンを指定する
- `Carmel` でインストールをする

