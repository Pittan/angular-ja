{@a glob}

# Service Workerの設定

#### 前提条件

次の基本的理解があること
* [プロダクションにおけるService Worker](guide/service-worker-devops)

<hr />

`src/ngsw-config.json`設定ファイルは、Angular Service WorkerがキャッシュすべきファイルとデータのURLと、キャッシュされたファイルとデータをどのように更新すべきかを指定します。CLIは`ng build --prod`中に設定ファイルを作成します。手動で`ngsw-config`ツールで作成することもできます。

```sh
ngsw-config dist src/ngsw-config.json /base/href
```

設定ファイルはJSON形式を使用します。すべてのファイルパスは`/`で始まらなければなりません。これはCLIプロジェクトでの展開ディレクトリであり、通常は `dist`です。

パターンは制限されたglobフォーマットを使います。

* `**`は、0個以上のパスセグメントに一致します。
* `*`は、厳密に1つのパスセグメントまたはファイル名セグメントに一致します。
* `!`接頭辞は、パターンを否定的なものとしてマークします。つまり、パターンに一致しないファイルのみが含まれます。

パターン例

* `/**/*.html`は、すべてのHTMLファイルを指定します。
* `/*.html`は、ルートのHTMLファイルのみを指定します。
* `!/**/*.map`は、すべてのソースマップを除外します。

設定ファイルの各セクションについて後述します。

## `appData`

このセクションでは、この特定のバージョンのアプリケーションを記述するために、任意のデータを渡すことができます。`SwUpdate`サービスは、更新通知にそのデータを含めます。このセクションを使用して、UIポップアップの表示のための追加情報を提供し、ユーザーに利用可能なアップデートを通知します。

## `index`

ナビゲーション要求を満たすためにインデックスページとして機能するファイルを指定します。通常これは`/index.html`です。

## `assetGroups`

*Assets*は、アプリケーションとともに更新される、アプリケーションバージョンの一部であるリソースです。ページのオリジンドメインからロードされたリソースだけでなく、CDNや他の外部URLからロードされたサードパーティのリソースを含めることができます。ビルド時にこのような外部URLをすべて知っているわけではないので、URLパターンを照合することができます。

このフィールドには、アセットリソースのセットとそれらがキャッシュされるポリシーを定義する、一連のアセットグループが含まれます。

```json
{
  "assetGroups": [{
    ...
  }, {
    ...
  }]
}
```

各アセットグループには、リソースグループとそれらを管理するポリシーの両方を指定します。このポリシーは、リソースがフェッチされるタイミングと、変更が検出されたときに発生する振る舞いを決定します。

アセットグループは、ここに示すTypeScriptインターフェイスに従います。

```typescript
interface AssetGroup {
  name: string;
  installMode?: 'prefetch' | 'lazy';
  updateMode?: 'prefetch' | 'lazy';
  resources: {
    files?: string[];
    versionedFiles?: string[];
    urls?: string[];
  };
}
```

### `name`

`name`は必須です。これにより設定のバージョン間で特定のアセットグループを識別します。

### `installMode`

`installMode`は、これらのリソースが最初にどのようにキャッシュされるかを決定します。`installMode`は次の2つの値のいずれかです。

* `prefetch`は、AngularService Workerに、現在のバージョンのアプリケーションをキャッシュしている間にリストされたすべてのリソースをフェッチするように指示します。これは帯域幅を大量に消費しますが、ブラウザが現在オフラインであっても、要求されたときはいつでもリソースを利用できるようにします。

* `lazy`はフロントにリソースをキャッシュしません。代わりに、Angular Service Workerはリクエストを受け取ったリソースのみをキャッシュします。これはオンデマンドキャッシングモードです。要求されないリソースはキャッシュされません。これは異なる解像度のイメージのようなものに役立ちます。そのため、Service Workerは特定の画面と向きの正しいアセットだけをキャッシュします。

### `updateMode`

すでにキャッシュにあるリソースの場合、`updateMode`は新しいバージョンのアプリケーションが発見されたときのキャッシングの動作を決定します。以前のバージョン以降に変更されたグループ内のリソースは、`updateMode`にしたがって更新されます。

* `prefetch`は、変更されたリソースをすぐにダウンロードしてキャッシュするようにService Workerに指示します。

* `lazy`は、Service Workerにそれらのリソースをキャッシュしないように指示します。代わりに、再度それらがリクエストされるまで、それらは要求されていないものとして扱われ、アップデートを待機します。`lazy`の`updateMode`は、`installMode`も`lazy`になっている場合にのみ有効です。

### `resources`

このセクションでは、キャッシュするリソースを3つのグループに分けて説明します。

* `files`は、配布ディレクトリ内のファイルと一致するパターンをリストします。これらは、単一のファイルまたは複数のファイルに一致するglobのようなパターンです。

* `versionedFiles`は、`files`と似ていますが、ファイル名にすでにハッシュを含んだビルド成果物のために使用するものです。これは、キャッシュ無効化に使われます。AngularService Workerは、ファイル内容が不変であると想定できる場合、その操作のいくつかの側面を最適化できます。

* `urls`は、実行時に照合されるURLとURLパターンの両方が含まれます。これらのリソースは直接取得されず、コンテンツハッシュもありませんが、HTTPヘッダーにしたがってキャッシュされます。これは、Google FontsサービスなどのCDNでもっとも便利です。

## `dataGroups`

アセットリソースとは異なり、データリクエストはアプリケーションとともにバージョン管理されません。これらは、手動で構成されたポリシーにしたがってキャッシュされます。このポリシーは、API要求やその他のデータの依存関係などの状況に役立ちます。

データグループはこのTypeScriptインターフェイスに従います。

```typescript
export interface DataGroup {
  name: string;
  urls: string[];
  version?: number;
  cacheConfig: {
    maxSize: number;
    maxAge: string;
    timeout?: string;
    strategy?: 'freshness' | 'performance';
  };
}
```

### `name`
`assetGroups`と同様に、すべてのデータグループはそれを一意に識別する`name`を持っています。

### `urls`
URLパターンのリスト。これらのパターンに一致するURLは、このデータグループのポリシーにしたがってキャッシュされます。

### `version`
時には、APIは下位互換性のない形式でフォーマットを変更します。新しいバージョンのアプリケーションは古いAPI形式と互換性がなく、そのAPIの既存のキャッシュされたリソースと互換性がない可能性があります。

`version`は、キャッシュされているリソースが下位互換性のない方法で更新されたこと、古いキャッシュエントリ(以前のバージョンからのキャッシュエントリ）を破棄するべきであることを示すためのメカニズムを提供します。

`version`は、整数フィールドで、デフォルトは「0」です。

### `cacheConfig`
このセクションでは、一致したリクエストをキャッシュするポリシーを定義します。

#### `maxSize`
（必須）キャッシュ内のエントリまたはレスポンスの最大数。オープンエンドのキャッシュは無限に成長しますが、最終的にはストレージクォータを超えたら、ストレージから追い出します。

#### `maxAge`
（必須）`maxAge`パラメータは、レスポンスが無効であるとみなされる前にキャッシュに残ることが許される期間を示します。`maxAge`は、次の単位サフィックスを使用した継続時間文字列です。

* `d`: days
* `h`: hours
* `m`: minutes
* `s`: seconds
* `u`: milliseconds

たとえば、文字列 `3d12h`はコンテンツを3日半までキャッシュします。

#### `timeout`
この継続時間文字列は、ネットワークタイムアウトを指定します。ネットワークのタイムアウトは、キャッシュされたレスポンスが構成されている場合に、キャッシュされたレスポンスを使用する前にAngular Service Workerがネットワークが応答するまで待機する時間です。

#### `strategy`

Angular Service Workerは、データリソース用の2つのキャッシング戦略のいずれかを使用できます。

* デフォルトの`performance`はできるだけ速いレスポンスのために最適化します。リソースがキャッシュに存在する場合、キャッシュされたバージョンが使用されます。これにより、よりよいパフォーマンスと引き換えに、maxAgeに依存して多少の古さを許容します。これは頻繁に変更されないリソースに適しています。たとえば、ユーザーのアバター画像です。

* `freshness`は、データをリアルタイム性で最適化し、ネットワークから要求されたデータを優先的に取り出します。`timeout`にしたがってネットワークがタイムアウトした場合にのみ、要求はキャッシュにフォールバックされます。これは、頻繁に変更されるリソースに役立ちます。たとえば、勘定残高などです。

