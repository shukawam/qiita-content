---
title: Nest.jsのDB接続周りでハマった話
tags:
  - JavaScript
  - Node.js
  - TypeScript
  - TypeORM
  - NestJS
private: false
updated_at: '2021-03-30T14:50:22+09:00'
id: 6c318d3e58cf19ad3f0c
organization_url_name: null
slide: false
ignorePublish: false
---
# 何を作成する？

Nest.js (with TypeORM) で環境ごとにデータベースの接続先を分けるために、データベースの接続情報を実行環境の環境変数から非同期で取得し作成します。

# 環境

- Node.js v12.14.1
- Nest.js v6.7.2
- TypeORM v0.2.22
- PostgreSQL v11.6

# 実装ログ

## 必要最小限の実装

参考）[Nest.js Document > TECHNIQUES >Database](https://docs.nestjs.com/techniques/database)

### ライブラリインストール

TypeORM, Database Driver (PostgreSQL)をインストールする。

```bash
npm install @nestjs/typeorm typeorm pg
```

### DB 接続情報を定義

**app.module.ts**にデータベースの接続情報を定義します。

```app.module.ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { ItemModule } from './item/item.module';
import { Connection } from 'typeorm';
import { join } from 'path';

@Module({
  imports: [
    ItemModule,
    // database connection setting
    TypeOrmModule.forRoot({
      type: 'postgres',
      host: 'localhost',
      port: 5432,
      username: 'postgres',
      password: 'postgres',
      database: 'postgres',
      entities: [join(__dirname + '/**/*.entity{.ts,.js}')],
      synchronize: false,
    }),
  ],
})
export class AppModule {}

```

一番シンプルな書き方です。学習用のアプリケーションで自分一人しか触らず、環境もこれだけ！ということであれば百歩譲ってこの書き方で良いでしょう。
しかし、実際の開発では開発環境、テスト環境、ステージング環境、本番環境と複数の環境が存在し、上記のような実装では環境ごとに接続情報をハードコードし直し → ビルド → デプロイという手順を踏む必要がありナンセンスです。
そのため、通常は [The Twelve-Factor App](https://12factor.net/ja/) にも記載があるように環境固有の情報(Database の接続定義など)は環境変数に定義し、そこから参照する形で作成します。
と、いうことで環境変数を参照するように**app.module.ts**を修正します。

## 環境変数を参照するように実装を修正

**※この方法では実行時に依存関係が解決できずエラーとなります。**

参考）[Nest.js Document > TECHNIQUES > Configuration](https://docs.nestjs.com/techniques/configuration)

### ライブラリインストール

環境変数を参照するために必要なライブラリをインストールします。

```bash
npm install @nestjs/config
```

### ダミーの環境変数を用意

本来は、環境変数に定義するのですがサンプル実装なので環境変数ファイル（**.env**）をプロジェクトルートに作成する。

```.env
DATABASE_HOST=localhost
DATABASE_PORT=5432
DATABASE_USERNAME=postgres
DATABASE_PASSWORD=postgres
DATABASE_NAME=postgres
```

### 環境変数を参照するように接続定義を修正

```app.module.ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { ItemModule } from './item/item.module';
import { Connection } from 'typeorm';
import { join } from 'path';

@Module({
  imports: [
    ItemModule,
    ConfigModule.forRoot({
      envFilePath: '.env',
      isGlobal: true,
      // ignoreEnvFile: true, // .envからではなく環境変数から直接取得する場合はコメントアウトを外す．
    }),
    // 非同期で環境変数から値を取得し、接続情報を作成する．
    TypeOrmModule.forRootAsync({
      imports: [ConfigModule],
      useFactory: async (configService: ConfigService) => ({
        type: 'postgres' as 'postgres',
        host: configService.get('DATABASE_HOST'),
        port: Number(configService.get('DATABASE_HOST')),
        username: configService.get('DATABASE_USERNAME'),
        password: configService.get('DATABASE_PASSWORD'),
        database: configService.get('DATABASE_NAME'),
        entities: [join(__dirname + '/**/*.entity{.ts,.js}')],
        synchronize: false,
      }),
      inject: [ConfigService],
    }),
  ],
})
export class AppModule {
  constructor(private readonly connection: Connection) {}
}

```

起動後、以下のエラーが発生。

```bash
2:25:34 PM - Found 0 errors. Watching for file changes.
[Nest] 19111   - 01/13/2020, 2:25:35 PM   [NestFactory] Starting Nest application...
[Nest] 19111   - 01/13/2020, 2:25:35 PM   [InstanceLoader] TypeOrmModule dependencies initialized +24ms
[Nest] 19111   - 01/13/2020, 2:25:35 PM   [InstanceLoader] ConfigModule dependencies initialized +1ms
[Nest] 19111   - 01/13/2020, 2:25:35 PM   [ExceptionHandler] Nest can't resolve dependencies of the TypeOrmModuleOptions (?). Please make sure that the argument ConfigService at index [0] is available in the TypeOrmCoreModule context.

Potential solutions:
- If ConfigService is a provider, is it part of the current TypeOrmCoreModule?
- If ConfigService is exported from a separate @Module, is that module imported within TypeOrmCoreModule?
  @Module({
    imports: [ /* the Module containing ConfigService */ ]
  })
 +1ms
Error: Nest can't resolve dependencies of the TypeOrmModuleOptions (?). Please make sure that the argument ConfigService at index [0] is available in the TypeOrmCoreModule context.

Potential solutions:
- If ConfigService is a provider, is it part of the current TypeOrmCoreModule?
- If ConfigService is exported from a separate @Module, is that module imported within TypeOrmCoreModule?
  @Module({
    imports: [ /* the Module containing ConfigService */ ]
  })

    at Injector.lookupComponentInExports (/home/kawamura/docker/docker-services/sample-app/sample-back/node_modules/@nestjs/core/injector/injector.js:185:19)
    at processTicksAndRejections (internal/process/task_queues.js:94:5)
    at async Injector.resolveComponentInstance (/home/kawamura/docker/docker-services/sample-app/sample-back/node_modules/@nestjs/core/injector/injector.js:142:33)
    at async resolveParam (/home/kawamura/docker/docker-services/sample-app/sample-back/node_modules/@nestjs/core/injector/injector.js:96:38)
    at async Promise.all (index 0)
    at async Injector.resolveConstructorParams (/home/kawamura/docker/docker-services/sample-app/sample-back/node_modules/@nestjs/core/injector/injector.js:111:27)
    at async Injector.loadInstance (/home/kawamura/docker/docker-services/sample-app/sample-back/node_modules/@nestjs/core/injector/injector.js:78:9)
    at async Injector.loadProvider (/home/kawamura/docker/docker-services/sample-app/sample-back/node_modules/@nestjs/core/injector/injector.js:35:9)
    at async Promise.all (index 3)
    at async InstanceLoader.createInstancesOfProviders (/home/kawamura/docker/docker-services/sample-app/sample-back/node_modules/@nestjs/core/injector/instance-loader.js:41:9)
```

環境変数を参照するための`ConfigService`が`TypeOrmModuleOptions`内で依存関係が解決できないことが原因らしい。

同じような事象が GitHub の Issue にあがっていたので参考までに載せておきます。
[Can't init TypeOrmModule using factory and forRootAsync](https://github.com/nestjs/nest/issues/1119#)

### 実装を修正

**app.module.ts**を以下のように修正します。

```app.module.ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { ConfigModule } from '@nestjs/config';
import { ItemModule } from './item/item.module';
import { TypeOrmConfigService } from './common/database/type-orm-config.service';

@Module({
  imports: [
    ConfigModule.forRoot({
      envFilePath: '.env',
      isGlobal: true,
      // ignoreEnvFile: true, <- 環境変数から取得する場合はコメントアウトを外す．
    }),
    TypeOrmModule.forRootAsync({
      imports: [ConfigModule],
      // 接続情報を作成するServiceクラスを定義
      useClass: TypeOrmConfigService,
    }),
    ItemModule,
  ],
})
export class AppModule {
}
```

```type-orm-config.service.ts
import { TypeOrmOptionsFactory, TypeOrmModuleOptions } from '@nestjs/typeorm';
import { Injectable } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { join } from 'path';

@Injectable()
export class TypeOrmConfigService implements TypeOrmOptionsFactory {

  createTypeOrmOptions(): TypeOrmModuleOptions {
    const configService = new ConfigService(); // ポイント
    return {
      type: 'postgres' as 'postgres',
      host: configService.get('DATABASE_HOST', 'localhost'),
      port: Number(configService.get('DATABASE_PORT', 5432)),
      username: configService.get('DATABASE_USERNAME', 'postgres'),
      password: configService.get('DATABASE_PASSWORD', 'postgres'),
      database: configService.get('DATABASE_NAME', 'postgres'),
      entities: [join(__dirname + '../**/*.entity{.ts,.js}')],
      synchronize: false,
    };
  }
}
```

`ConfigService`を DI するのではなく、自分で new するのがポイントです。

# 最後に

たまたま案件で使う機会があったのですが、非常に使いやすいフレームワークでした。特に自分と同じような経験を積んできた方(Java, Spring etc.)にはおすすめできるフレームワークです。

# 参考

- [NestJs 公式ドキュメント](https://docs.nestjs.com/)

- [Can't init TypeOrmModule using factory and forRootAsync](https://github.com/nestjs/nest/issues/1119#)
