## Perl

Perlでのコーディングにおけるプラクティス

## 方針

- 最新のモダンPerlを利用する
- 値の型制約を宣言する
- 理解を助けるコメントを書く
- アダプタパターンで副作用を抽象化する
- 純粋関数を優先する
- 例外よりもエラーを返す

## 環境

- miseでインストールしたperlを利用する。システムのperlを絶対に利用しない
- `eval "$(mise activate bash)" && perl -v` のようにmise activateを行ってからスクリプトを実行する
- mise.toml には次が記載されていることを確認する

    ```toml
    [tools]
    perl = "5.40.0"
    ```

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

## コメント

- POD形式で、NAME,SYNOPSIS,DESCRIPTIONを記載する
   - AUTHORS, LICENSE はユーザーが記載するため不要
- encoding utf8を記載する
- 関数の冒頭に、その関数が何をするかをコメントする
- *していないこと* に注意が必要な場合は、していない理由もコメントする
- 関数の仕様をより簡潔で読みやすい形式に書く
    - 機能説明、使用例、パラメータ説明、戻り値、アルゴリズム（任意）の順に記述
    - 例: find_primes($limit) -> Result(PrimeList, InvalidInputError)
    - `@params` や `@return` などのタグを使用する

```perl
package MyModule;

=pod

=encoding utf8

=head1 NAME

MyModule - This is a sample module

=head1 SYNOPSIS

    use MyModule qw(add);
    add(1, 2); # => 3

=head1 DESCRIPTION

This is a sample module

=head1 FUNCTIONS

=cut

use v5.40;

=pod

=head2 add($x, $y)

Add two numbers. 

=cut

sub add($x, $y) {
    return $x + $y;
}
```

## テスト

- ユニットテストは、`Test2::V0` を利用する
- `subtest` でテストをグループ化する
    - 1つ目は、`subtest '関数名'` として、テスト対象の関数名を書く
    - 2つ目は、`subtest 'どんな時に、何を期待するか'` を書く
- テーブルテストを書く際は、次のコードのようにMultiple-alias syntax for foreach を利用する
  - `while (my ($x, $y, $expected) = splice(@cases, 0, 3)) { ... }` は*使わない*

```perl
use v5.40;
use utf8;
use Test2::V0;

subtest 'add' => sub {
    subtest '2つの数値を与えた時、その和を返す' => sub {
        my @cases = (
            # x, y => expected
            1, 2 => 3
            2, 3 => 5
            3, 4 => 7
        );

        for my ($x, $y, $expected) (@cases) {
            my ($got, $err) = add($x, $y);
            is $got, $expected, "add $x and $y";
            is $err, undef, "no error";
        }
    };

    subtest '数値以外を与えた時、InvalidInputErrorを返す' => sub {
        my @cases = (
            # x, y
            'a', 2
            2, 'b'
            'a', 'b'
        );

        for my ($x, $y) (@cases) {
            my ($got, $err) = add($x, $y);
            is $got, undef, "no result";
            is $err, 'InvalidInputError', "$x, $y is invalid";
        }
    };
};

done_testing;
```

- `is` を利用する。ユーザーはmatcherに注力するため。
- `like`, `unlike` は用いない。代わりに`match` を利用する

```perl
sub bye($name) { die "Bye, $name!" }

# Good
is dies { bye('foo') }, match qr/^Bye, foo!/, 'say bye to foo';

# Bad
like dies { bye('foo') }, qr/^Bye, foo!/, 'say bye to foo';
```

- テストは、`prove` で実行する
- テストファイルに `use lib 'lib'` の指定は不要

```sh
prove -lvr t/path/to/test.t
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

- `Syntax::Keyword::Assert` の assert キーワードを利用して、値のチェックを行う
  - unless ... croak は使わない

```perl
use Syntax::Keyword::Assert;
use kura PositiveInt => Int & sub { $_ > 0 };

# Good
sub add($x, $y) {
    assert( PositiveInt->check($x) );
    assert( PositiveInt->check($y) );

    return $x + $y;
}

# Bad
sub add($x, $y) {
    unless ( PositiveInt->check($x) ) {
        croak 'x is not a positive integer';
    }
    unless ( PositiveInt->check($y) ) {
        croak 'y is not a positive integer';
    }

    return $x + $y;
}
```

## エラーハンドリング

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
  - 記載するモジュール名は、依存しているモジュールの名前を記載し、配布名は記載しない
- `Carmel` でインストールをする

```perl
# Example of cpanfile
requires 'Types:Standard', '== 2.006000';
```

