---
title: "Spring Batch"
author: "jacob"
date: 2022-02-22T12:25:40+09:00
categories: ["Spring"]
tags: ["SpringBatch"]
archives: ["2022-02"]
draft: true
archives: ["2022-02"]
---

## Spring Batch Architecture

![Spring Batch Architecture](/images/spring-batch-layers.png)

### Applicationレイアー

* all batch jobs
* custom code by developers

<!--more-->

### Batch Coreレイアー

* バッチジョブの起動
* バッチジョブの制御
* `JobLauncher`,`Job`,`Step`の実装

### Batch Infrastructureレイアー

* common readers
  * such as ItemReader
* common writers
  * such as ItemWriter
* services
  * such as the RetryTemplate
* core framework

## Spring Batchの概念

![batch model](/images/spring-batch-reference-model.png)

### JobRepository

* JobやStepの実行状況や実行結果を保存しておくところ。
* 管理情報は、SpringBatchが規程するテーブルスキーマを元にデータベース上に永続化される。

  |Entityクラス|テーブル名|説明|
  |-|-|-|
  |JobExecution|BATCH_JOB_EXECUTION|ジョブの状態・実行結果の保持する。</br>1回のジョブ実行|
  |JobExecutionContext|BATCH_JOB_EXECUTION|ジョブ内部のコンテキストを保持する。</br>1回のジョブ実行|
  |JobExecutionParams|BATCH_JOB_EXECUTION_PARAMS|起動時に当られたジョブパラメータを保持する。</br>1回のジョブ実行|
  |StepExecution|BATCH_STEP_EXECUTION|ステップの状態・実行結果、コミット・ロールバック件数を保持する。</br>1回のステップ実行|
  |StepExecutionContext|BATCH_STEP_EXECUTION_CONTEXT|ステップ内部のコンテキストを保持する。</br>1回のステップ実行|
  |JobInstance|BATCH_JOB_INSTANCE|ジョブ名、及びジョブパラメータをシリアライズした文字列を保持する。</br>ジョブ名とジョブパラメータの組み合わせ|

* 上記のテーブル手動で作るが必要です。
  * [DDL Scripts](https://docs.spring.io/spring-batch/docs/current/reference/html/schema-appendix.html)
* `@EnableBatchProcessing`annotation利用すると、frameworkからJobRepositoryを提供します。
* カスタマイズの場合、`BatchConfigurer`インターフェースをimplementsしてカスタマイズができます。

### JobLauncher

* ジョブを起動するためのインタフェース。
* 基も基本的な実装は`SimpleJobLauncher`です.
  * `注意`: Httpリクエストから起動しようとする時、`SimpleJobLauncher`が呼び出し元にすぐに戻るように、起動を非同期で行う必要がありあります。

### Job

* SpringBatchにおけるバッチアップリケーションの一連の処理をまとめた１実行単位。
* ジョブの実行
  * `JobLauncher`一つと`job`一つが必要です。

### Step

* ジョブを構成する処理の単位。1つのジョブに1~N個のStepを持たせることが可能です。
* 一つのジョブを複数のStepに分割して処理することにより、処理の再利用、並列化、条件分岐が可能です。
* チャンクモデルまだタスクレットモデルのいずれがで実装する。

### ItemReader,ItemProcessor,ItemWriter

* `チャンクモデル`を実装する際に、データの入力/加工/出力の三つに分割するためのインタフェース。
* `タスクレットモデル`では,タスクレット内に入出力、入力チェック、ビジネスロジックのすべてを実装する必要がある。

### Listener

* `Job`、`Step`、`ItemReader`、`ItemProcessor`、`ItemWriter`各段階の実行状況について、Listenerで監視ができます。
* 利用頻度が高いのは下記の二つです。
  * `JobListener`: ジョブの実行に対して処理を挟み込むためのインターフェース
  * `StepListener`: ステップの実行に対して処理を挟み込むためのインターフェース

* その他のリスナー
  * `ChunkListener`、`ItemReadListener`、`ItemProcessListener`、`ItemWriteListener`
* リスナーの実装
  * 各種リスナーインターフェースをimplementsして実装する。
  * 各種リスナーインターフェースに対応したアノテーションを付与する。

  |リスナーインターフェース|アノテーション|
  |-|-|
  |JobExecutionListener|@beforeJob</br>@afterJob|
  |StepExecutionListener|@BeforeStep</br>@AfterStep|
  |ChunkListener|@BeforeChunk</br>@AfterChunk</br>@afterChunkError|
  |ItemReadListener|@BeforeRead</br>@AfterRead</br>@OnReadError|
  |ItemProcessListener|@beforeProcess</br>@afterProcess</br>@onProcessError|
  |ItemWriteListener|@BeforeWrite</br>@AfterWrite</br>@OnWriteError|
