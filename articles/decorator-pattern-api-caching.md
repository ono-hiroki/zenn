---
title: "外部APIのキャッシュ実装にDecoratorパターンを適用する"
emoji: "🎁"
type: "tech"
topics: ["設計", "デザインパターン", "キャッシュ", "クリーンアーキテクチャ"]
published: false
---

## はじめに

外部APIを呼び出す処理にキャッシュを追加したい場面は多くあります。素直に実装すると、APIクライアントの中にキャッシュ処理を直接書くことになりがちです。

本記事では、Decoratorパターンを使ってキャッシュ処理を分離する設計を紹介します。責務を分離することで、テストしやすく変更に強い構造を実現できます。

## よくある実装の課題

APIクライアントにキャッシュ処理を直接書く場合、以下のような課題が生じます。

### 単一責任の原則からの逸脱

APIクライアントが「外部APIとの通信」と「キャッシュの管理」という2つの責務を持つことになります。どちらかの仕様が変わると、同じクラスを修正する必要が出てきます。

### テストの複雑化

キャッシュの有無によってAPIクライアントの振る舞いが変わるため、テストケースが複雑になります。「API呼び出しが正しく行われるか」と「キャッシュが正しく機能するか」を分離してテストすることが難しくなります。

### キャッシュ戦略の変更が困難

キャッシュストアの変更（RedisからMemcachedへ、など）やTTLの調整をする際に、APIクライアント本体を修正する必要があります。

### 環境ごとの切り替えが面倒

開発環境ではキャッシュを無効にしたい、といった要件に対応するために、条件分岐がAPIクライアント内に増えていきます。

## Decoratorパターンによる解決

Decoratorパターンは、オブジェクトに動的に新しい責務を追加するデザインパターンです。継承ではなく委譲を使って機能を拡張します。

### 構造

```
          ApiClientInterface
                 △
                 │ implements
        ┌────────┴────────┐
        │                 │
   ApiClient      CachingApiClient
  (実際のAPI呼出)     (キャッシュ層)
                          │
                          │ has-a (委譲)
                          ▼
                  ApiClientInterface
```

`CachingApiClient`は`ApiClientInterface`を実装しつつ、内部に同じインターフェース型のオブジェクトを保持します。これにより、呼び出し側はキャッシュの有無を意識せずに利用できます。

### 設計のポイント

1. **インターフェースを定義する** - APIクライアントの振る舞いを抽象化します
2. **APIクライアントを実装する** - キャッシュを意識せず、純粋にAPI呼び出しだけを行います
3. **キャッシュDecoratorを実装する** - 同じインターフェースを実装し、キャッシュヒット時はそれを返し、ミス時は委譲先を呼び出して結果をキャッシュします
4. **DIコンテナで組み立てる** - APIクライアントをDecoratorでラップしてバインドします

### コード例

```php
// インターフェース
interface WeatherApiClientInterface
{
    public function getWeather(string $city): WeatherData;
}

// 実際のAPIクライアント
class WeatherApiClient implements WeatherApiClientInterface
{
    public function getWeather(string $city): WeatherData
    {
        // 外部APIを呼び出す
        $response = $this->httpClient->get("/weather?city={$city}");
        return WeatherData::fromResponse($response);
    }
}

// キャッシュDecorator
class CachingWeatherApiClient implements WeatherApiClientInterface
{
    public function __construct(
        private WeatherApiClientInterface $inner,
        private CacheInterface $cache,
        private int $ttl = 3600,
    ) {}

    public function getWeather(string $city): WeatherData
    {
        $key = "weather:{$city}";

        return $this->cache->remember($key, $this->ttl, fn () =>
            $this->inner->getWeather($city)
        );
    }
}
```

DIコンテナでの組み立て例：

```php
// 本番環境：キャッシュを有効化
$container->bind(
    WeatherApiClientInterface::class,
    fn () => new CachingWeatherApiClient(
        new WeatherApiClient(),
        $container->get(CacheInterface::class),
    )
);
```

## Decoratorパターンの利点

### 責務の分離

各クラスが1つの責務だけを持つようになります。

- APIクライアント：外部APIとの通信
- キャッシュDecorator：キャッシュの読み書き

それぞれが独立しているため、変更の影響範囲が限定されます。

### テストの容易さ

責務が分離されているため、それぞれを独立してテストできます。

- APIクライアントのテスト：キャッシュを気にせず、API呼び出しのロジックだけを検証
- キャッシュDecoratorのテスト：モックを使って、キャッシュの読み書きロジックだけを検証

### 機能の追加・削除が容易

Decoratorは入れ子にできるため、複数の横断的関心事を組み合わせられます。

- ログ出力Decorator
- リトライDecorator
- サーキットブレーカーDecorator
- キャッシュDecorator

これらを必要に応じて組み合わせたり外したりでき、順序の変更も容易です。

### 環境ごとの切り替え

DIコンテナの設定で、Decoratorを適用するかどうかを切り替えられます。

- 本番環境：キャッシュDecoratorを適用
- 開発環境：Decoratorなしで直接APIクライアントを使用

コード本体には条件分岐を入れる必要がありません。

## 適用を検討する場面

以下のような場面では、Decoratorパターンの適用を検討する価値があります。

- 外部APIのレスポンスをキャッシュしたい
- API呼び出しにリトライ処理を追加したい
- API呼び出しのログを取りたい
- 障害時にサーキットブレーカーで保護したい
- 上記の機能を環境や設定によって切り替えたい

## 注意点

Decoratorパターンはクラス数が増えるため、シンプルな要件には過剰な設計になる可能性があります。キャッシュ処理が1箇所だけで、将来的な拡張も見込まれない場合は、直接実装する方が適切な場合もあります。

設計の複雑さと得られるメリットのバランスを考慮して判断してください。

## まとめ

Decoratorパターンを使うことで、キャッシュなどの横断的関心事をAPIクライアント本体から分離できます。

- 責務が明確になり、変更の影響範囲が限定される
- 各コンポーネントを独立してテストできる
- 機能の追加・削除・組み合わせが容易
- DIコンテナの設定だけで環境ごとの切り替えが可能

クラス数は増えますが、複数の関心事を扱う必要がある場合や、将来的な拡張が見込まれる場合には有効な選択肢です。
