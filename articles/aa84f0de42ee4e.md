---
title: "DDDなSymfony + Api Platformでファイルをuploadする"
emoji: "🐡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Symfony", "ApiPlatform", "php"]
published: true
---
# はじめに
仕事でphpのフレームワークのSymfonyとApiPlatformを利用してバックエンドAPI開発を行なっています。
今回はファイルアップロードについて書きます。

# アーキテクチャについて
ApiPlatformの設定はEntityファイルに直接書くことができますが、これはDomain層がInfrastructure層に依存している状態であり、アーキテクチャの基本指針である「依存の方向を揃える」を無視してしまいます。
そこで、ApiPlatformに関係するロジックは全てInfrastructure層に配置するような構造を取っています。
またCQRSを取り入れており、そのためにApiPlatform側でもProvider, ProcessorとしてそのそれぞれをQueryとCommandに対応させています。
具体的なアーキテクチャは[mtarld/apip-ddd](https://github.com/mtarld/apip-ddd)を参考にしています。
このアーキテクチャではApiPlatformのroutingをResourceファイルと呼ばれる、いわゆるDTOクラスに記述しています。

```php :UserResource.php

<?php
...

#[ApiResource(
  shortName: 'User',
  operations: [
    new GetCollection(
      '/api/users',
      provider: UserCollectionProvider::class
    )     
  ]
)]
final class UserResource
{
  public function __construct(
    public ?int $id = null,
    public ?string $name = null,
    public ?string $email = null
  ){
  }

  public function fromEntity(User $user): static
  {
    return new self(
       $user->id,
       $user->name,
       $user->email,
    );
  }
}
```

ちなみに、この例ではCQRSのQueryに該当するのでProviderを下記のように書きます
```php :UserCollectionProvider.php
<?php
declare(strict_types=1);

use ApiPlatform\Metadata\Operation;
use ApiPlatform\State\Pagination\Pagination;
use ApiPlatform\State\ProviderInterface;
...

final readonly class UserCollectionProvider implements ProviderInterface
{
    public function __construct(
        private QueryBusInterface $queryBus,
        private Pagination $pagination
    ){

    }

    public function provide(Operation $operation, array $uriVariables = [], array $context = []): Paginator|array
    {
        ...
    }

}
```
このようにしてDomain層がInfrastructure層に依存しないようにしています。

# ファイルアップロードについて
さて、googleで「api platform file upload」と検索すると[https://api-platform.com/docs/core/file-upload/](https://api-platform.com/docs/core/file-upload/) がヒットします。
基本的にはここのコードをそのまま書けばファイルアップロードは実現するのですが、上記のようなアーキテクチャを採用している場合は工夫が必要です。
方針は以下です。
- 上記ドキュメントではMediaObjectというDomain層のEntityがInfrastructureのApiPlatformに依存しているので,ResourceファイルにApiPlatformの設定を書く
- Controllerを利用するのではなく、Processorを利用する

# 実装
Resourceクラスは以下のような感じ
```php : FileUploadResource.php
<?php

declare(strict_types=1);

namespace App\Hogehoge\Presentation\Api\Resource;

use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\Post;
use ApiPlatform\OpenApi\Model;
use App\Hogehoge\Presentation\Api\State\Processor\FileUploadProcessor;
use Symfony\Component\HttpFoundation\File\File;

use Vich\UploaderBundle\Mapping\Annotation as Vich;

#[Vich\Uploadable]
#[ApiResource(
    shortName: 'FileUpload',
    types: ['https://schema.org/MediaObject'],
    operations: [
        new Post(
            openapi: new Model\Operation(
                requestBody: new Model\RequestBody(
                    content: new \ArrayObject([
                        'multipart/form-data' => [
                            'schema' => [
                                'type'       => 'object',
                                'properties' => [
                                    'file' => [
                                        'type'   => 'string',
                                        'format' => 'binary',
                                    ],
                                ],
                            ],
                        ],
                    ]),
                ),
            ),
            validationContext: ['groups' => ['Default', 'media_object_create']],
            processor: FileUploadProcessor::class,
        ),
    ],
    normalizationContext: ['groups' => ['media_object:read']],
)]
class FileUploadResource
{
    public function __construct(
        #[Vich\UploadableField(mapping: 'media_object', fileNameProperty: 'fileName')]
        public ?File $file = null,
        public ?string $fileName = null,
    ) {
    }
}
```

Processorクラスは以下のような感じ。
以下の例ではProcessorクラスでファイルのアップロードを行なっていますが、Application層で行なうのが良いかと思います。
```php : FileUploadProcessor.php
<?php
declare(strict_types=1);

namespace App\Hogehoge\Presentation\Api\State\Processor;

use ApiPlatform\Metadata\Operation;
use ApiPlatform\State\ProcessorInterface;

class FileUploadProcessor implements ProcessorInterface
{
    // アップロード先: webserverに保存することを前提にしている
    public const FILE_UPLOAD_PATH = 'media/';

    public function __construct()
    {
    }

    public function process(mixed $data, Operation $operation, array $uriVariables = [], array $context = []): void
    {
        // csv のアップロード
        // application側でやるなりよしなに
        $data->file->move(self::FILE_UPLOAD_PATH, $data->file->getClientOriginalName());

    }
}
```

これだけでは足りず、Normalizer, Denormalizer, Decoderを書く必要があります。
Normalizer等についてはSymfony公式のドキュメントで説明されています。 [https://symfony.com/doc/current/components/serializer.html](https://symfony.com/doc/current/components/serializer.html)
また、以下は[https://api-platform.com/docs/core/file-upload/](https://api-platform.com/docs/core/file-upload/)にソースがあります。
```php :MediaObjectNormalizer.php
<?php
declare(strict_types=1);

namespace App\Hogehoge\Infrastructure\ApiPlatform\Serializer;

use App\Hogehoge\Presentation\Api\Resource\FileUploadResource;
use Symfony\Component\Serializer\Normalizer\ContextAwareNormalizerInterface;
use Symfony\Component\Serializer\Normalizer\NormalizerAwareInterface;
use Symfony\Component\Serializer\Normalizer\NormalizerAwareTrait;
use Vich\UploaderBundle\Storage\StorageInterface;

final class MediaObjectNormalizer implements ContextAwareNormalizerInterface, NormalizerAwareInterface
{
    use NormalizerAwareTrait;

    private const ALREADY_CALLED = 'MEDIA_OBJECT_NORMALIZER_ALREADY_CALLED';

    public function __construct(private StorageInterface $storage)
    {
    }

    public function normalize($object, ?string $format = null, array $context = []): array|string|int|float|bool|\ArrayObject|null
    {
        $context[self::ALREADY_CALLED] = true;

        $object->contentUrl = $this->storage->resolveUri($object, 'file');

        return $this->normalizer->normalize($object, $format, $context);
    }

    public function supportsNormalization($data, ?string $format = null, array $context = []): bool
    {
        if (isset($context[self::ALREADY_CALLED])) {
            return false;
        }

        return $data instanceof FileUploadResource;
    }
}
```

```php : MultipartDecoder.php
<?php

declare(strict_types=1);

namespace App\Hogehoge\Infrastructure\ApiPlatform\Serializer;

use Symfony\Component\HttpFoundation\RequestStack;
use Symfony\Component\Serializer\Encoder\DecoderInterface;

final class MultipartDecoder implements DecoderInterface
{
    public const FORMAT = 'multipart';

    public function __construct(private RequestStack $requestStack)
    {
    }

    /**
     * {@inheritdoc}
     */
    public function decode(string $data, string $format, array $context = []): ?array
    {
        $request = $this->requestStack->getCurrentRequest();

        if (!$request) {
            return null;
        }

        return \array_map(static function(string $element) {
            // Multipart form values will be encoded in JSON.
            $decoded = \json_decode($element, true);

            return \is_array($decoded) ? $decoded : $element;
        }, $request->request->all()) + $request->files->all();
    }

    /**
     * {@inheritdoc}
     */
    public function supportsDecoding(string $format): bool
    {
        return self::FORMAT === $format;
    }
}
```

```php : UploadedFileDenormalizer.php
<?php
declare(strict_types=1);


namespace App\Hogehoge\Infrastructure\ApiPlatform\Serializer;

use Symfony\Component\HttpFoundation\File\UploadedFile;
use Symfony\Component\Serializer\Normalizer\DenormalizerInterface;

final class UploadedFileDenormalizer implements DenormalizerInterface
{
    /**
     * {@inheritdoc}
     */
    public function denormalize($data, string $type, ?string $format = null, array $context = []): UploadedFile
    {
        return $data;
    }

    /**
     * {@inheritdoc}
     */
    public function supportsDenormalization($data, $type, $format = null): bool
    {
        return $data instanceof UploadedFile;
    }
}
```

最後にapi_platform.yamlに以下のように追記します。
```yaml : api_platform.yaml
api_platform:
  title: Hello API Platform
  version: 1.0.0

  formats:
    ...
    multipart: [ 'multipart/form-data' ]
...
```

# 最後に
Symfonyはただでさえ情報が少ないのに、ApiPlatformと組み合わせた場合の実装例は情報が更に少なく大変です。
とはいっても公式ドキュメントにはあるのですが。
