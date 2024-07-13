---
title: "symfonyでカスタムなmaker-bundleを作成する"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["symfony", "php"]
published: false
---

# はじめに
symfonyでは`symfony console make:migration`のようにコマンドでいろんなファイルを作成することができます。
これはmaker-bundleという仕組みを利用しています。
このmaker-bundleは割と簡単に自分で新たなコマンドを追加することができます。
その方法を紹介します。

# 準備
Symfony公式では「AbstractMakerを継承すればいいよ！」とは書いてあるが具体的な説明されていません。 https://symfony.com/bundles/SymfonyMakerBundle/current/index.html#creating-your-own-makers
なのでmaker-bundleの実装 (https://github.com/symfony/maker-bundle/tree/main/src/Maker) を参考に作りました。

# 全体の方針
- `symofony console make:hoge`を叩くと`src/Foo/Application/Service`にFooServceクラスが作成されるようなコマンドを作成する。
- 作成するファイルのテンプレートを作成する。

# 実装
コマンドの実装は以下の通りです。./src/Maker配下に配置しました。
```php
class MakeTest extends AbstractMaker
{

	public function __construct(private KernelInterface $kernel)
    {
        $this->templatePath = \sprintf('%s/%s/%s', $this->kernel->getProjectDir(), 'Maker', 'skeletons'); // テンプレートファイルを指定。もっといい方法があるような気もする。
    }

	public function configureCommand(Command $command, InputConfiguration $inputConfig): void
    {
        $command->addArgument('foo', InputArgument::OPTIONAL, 'Foo')
            ->addArgument('name', InputArgument::OPTIONAL, 'Name');
        $inputConfig->setArgumentAsNonInteractive('foo');
        $inputConfig->setArgumentAsNonInteractive('name');
    }

	// コマンドの引数が渡されなかったときに実行するロジック（よくわかっていない）
    public function interact(InputInterface $input, ConsoleStyle $io, Command $command): void
    {
	    $select = ['Foo', 'Hoge'];

        if ($input->getArgument('foo') === null) {
            $input->setArgument('foo', $io->choice('選んでね！', $select)); // foo引数に値が渡されなかったときにインタラクトに選択させる。選択内容は$selectのもの。選択された値は$input->getArgument('foo')で取り出せる。
        }

        if ($input->getArgument('name') === null) {
            $input->setArgument('name', $io->ask('名前を教えて')); // name引数が渡されていない場合、ユーザに入力させる。入力された値は$input->getArgument('name')で取り出せる。
        }
    }

	// コマンドの名前. この場合、symfony console make:hogeでコマンドが実行される。
    public static function getCommandName(): string
    {
        return 'make:hoge';
    }

	// コマンドの説明。symfony console make:hoge -hで表示される説明文
	public static function getCommandDescription(): string
    {
        return 'This is hoge !';
    }

	// コマンド実行でファイルを作成するロジック。
	public function generate(InputInterface $input, ConsoleStyle $io, Generator $generator): void
	{
		// クラスファイルの名前などの設定を行う
		$classNameDetail = $generator->createClassNameDetails(
	        $input->getArgument('name'), // class名のPrefix
            \sprintf('%s%s', $input->getArgument('foo'), '\\Application\\Service'), // Namespace
            'Service', // class名のsuffix
        );

		// ファイルを作成する
		$generator->generateClass(
            $interfaceDetail->getFullName(), // 上記の設定を利用する
            \sprintf('%s/%s', $this->templatePath, 'hoge/temp.tpl'), // 利用するテンプレートファイルを指定
            [
                'name'       => $input->getArgument('name'),
                'foo' => $input->getArgument('foo'),
            ], // 第三引数にはテンプレートファイルに渡す変数を指定する。
        );

		$generator->writeChanges(); // ファイルをwrite
	}
}
```
テンプレートファイルはhttps://github.com/symfony/maker-bundle/blob/main/src/Resources/skeleton/test/ApiTestCase.tpl.php を参考にしました。
テンプレートファイルは以下のように書きます。上記クラスで指定したテンプレートファイルのパス（./Maker/skeletons/hoge）に配置します。
```
<?="<?php\n"; ?>

declare(strict_types=1);

namespace <?= $namespace; ?>;


class <?= $class_name; ?> extends <?= $foo ?>

{
    public function __construct()
    {

    }
}
```

services.yamlにクラスをサービスとして登録します（不要かもしれない）。
```yaml
# ...
    Maker\:
        resource: '../Maker/'
# ...
```


以上で完了です。
`symfony console make:foo`とコマンドを叩くことでFoo, Hogeの選択肢が表示されます。どちらかを選択すると対応したクラスが生成されます。
