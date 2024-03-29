---
title: "InnoDB Storage Engine"
author: "jacob"
date: 2022-03-03T16:06:46*09:00
draft: true
categories: ["databases"]
tags: ["Mysql"]
archives: ["2022-03"]
---

## InnoDBの利点

* DML(データ操作言語)操作は、トランザクションにユーザーデータを保護するためのコミット、ロールバック、およびクラッシュリカバリ機能が備わっている ACID モデルに従っています
  * ACID モデル
  * A: 原子性
  * C: 一貫性
  * I: 分離性
  * D: 持続性
* 行レベルのロックと一貫性読み取りを使用すると、複数ユーザーの並列性およびパフォーマンスが向上します。
* InnoDB テーブルでは、主キーに基づいてクエリーが最適化されるように、ディスク上のデータが整列されます。
* FOREIGN KEY 制約がサポートされています

## InnoDB のアーキテクチャ

* インメモリー構造
* ディスク構造

![InnoDB architecture](/images/innodb-architecture.png)

## インメモリー構造

* バッファプール
  * 大容量読み取り操作の効率を高めるため、バッファープールは複数行を保持できるページに分割されます。
  * 最低使用頻度 (LRU) アルゴリズムのバリエーションを使用してリストとして管理されます。
  * バッファプールサイズ
    * SET ステートメントを使用して動的に設定でき、サーバーを再起動せずにバッファープールのサイズを変更できます
    * 常に innodb_buffer_pool_chunk_size * innodb_buffer_pool_instances と等しいか倍数である必要があります
      * innodb_buffer_pool_chunk_size は 1MB (1048576 バイト) 単位で増減できますが、起動時、コマンドライン文字列または MySQL構成ファイルでのみ変更できます。
      * innodb_buffer_pool_chunk_size変更する場合、innodb_buffer_pool_sizeに影響があります。（要注意）
* 変更バッファ
  * secondary index ページが buffer pool にない場合に、そのページに対する変更をキャッシュする特別なデータ構造です。
  * セカンダリインデックスは通常一意ではなく、セカンダリインデックスへの挿入は比較的ランダムな順序で行われます。(削除および更新も)
  * セカンダリインデックスページをディスクからバッファープールに読み込むために必要な大量のランダムアクセス I/O が回避されます。
  * 変更バッファの最大サイズの構成
    * 変更バッファの最大サイズをバッファプールの合計サイズに対する割合として構成できます。
    * innodb_change_buffer_max_size
      * デフォルトでは 25 に設定されます。 最大設定は 50 です。
* 適応型ハッシュインデックス
  * innodb_adaptive_hash_index オプションによって有効化され、または skip_innodb_adaptive_hash_index オプションにより無効化されます。
* ログバッファ
  * ディスク上のログファイルに書き込まれるデータを保持するメモリー領域です
  * innodb_log_buffer_size 変数によって定義されます。
    * デフォルトのサイズは 16M バイトです。
  * ログバッファの内容は定期的にディスクにフラッシュされます。
    * innodb_flush_log_at_trx_commit 変数
      * ログバッファの内容をディスクに書き込む方法およびフラッシュする方法を制御します
    * innodb_flush_log_at_timeout 変数
      * ログのフラッシュ頻度を制御します

## InnoDB オンディスク構造

* テーブル
  * InnoDB テーブルの作成
    * 略
  * 外部でのテーブルの作成
    * DATA DIRECTORY 句の使用
    * CREATE TABLE ... TABLESPACE 構文の使用
    * 外部一般テーブルスペースでのテーブルの作成
  * InnoDB テーブルのインポート
    * 前提条件
      * innodb_file_per_table 変数は、デフォルトで有効になっている必要があります。
      * 略

* インデックス
  * クラスタインデックスとセカンダリインデックス
    * すべての InnoDB テーブルは、行のデータが格納されているクラスタ化されたインデックスと呼ばれる特別なインデックスを持っています。
    * テーブル上で PRIMARY KEY を定義すると、InnoDB ではそれがクラスタ化されたインデックスとして使用されます。
    * PRIMARY KEYが定義されていない場合、MySQL はすべてのキーカラムが NOT NULL の UNIQUE インデックスを最初に検索し、InnoDB はそれをクラスタ化されたインデックスとして使用します。
    * PRIMARY KEY インデックスまたは適切な UNIQUE インデックスがない場合
      * GEN_CLUST_INDEX という名前の非表示のクラスタインデックスを、行 ID 値を含む合成カラムに内部的に生成します。

  * InnoDB インデックスの物理構造
    * B*tree データ構造です
  * ソートされたインデックス構築
    * インデックス作成には 3 つのフェーズがあります
      * clustered index がスキャンされ、インデックスエントリが生成されてソートバッファに追加されます。 sort buffer がいっぱいになると、エントリはソートされ、一時中間ファイルに書き込まれます
      * 1 つ以上の実行が一時中間ファイルに書き込まれ、ファイル内のすべてのエントリに対してマージソートが実行されます
      * ソートされたエントリが B*tree に挿入されます。
  * InnoDB FULLTEXT インデックス
    * InnoDB 全文インデックスの設計
      * 転置インデックスの設計が使用されています
* テーブルスペース
  * システムテーブルスペース
    * データファイルのサイズと数は、innodb_data_file_path 起動オプションによって定義されます。
    * innodb_data_file_path
      * データファイルの名前、サイズおよび属性を定義します。
    *
  * file-per-table テーブルスペース
  * 一般テーブルスペース
  * undo テーブルスペース
  * 一時テーブルスペース
  * サーバーがオフラインのときのテーブルスペースファイルの移動
  * テーブルスペースパス検証の無効化
  * linux でのテーブルスペースの領域割当ての最適化
  * テーブルスペースの autoextend_size 構成

* 二重書き込みバッファー
* redo ログ
* undo ログ
