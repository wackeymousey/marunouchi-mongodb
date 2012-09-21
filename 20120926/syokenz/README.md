MongoDB 2.2.0 新機能紹介
=================
#コンテンツ
- 並列処理の強化（DBのレベルロック、PageFaultアーキテクチャの改善）
- Aggregation Framework
- Readに関する設定
- Tagを使用したShardingが可能に
- TTL(Time To Live) Collections
- その他

出所:[New Features in 2.2](http://kumoya.com/wordpress/wp-content/uploads/2012/09/New-Features-2.2.0.pdf)


## 並列処理の強化

### DBレベルのロック
Globalロックを排除し、DBレベルのロックへ 

![DB Lock](http://www.fedc.biz/~fujisaki/img/db_lock.jpg)  
出所:[MongoDBの開発元、サンフランシスコの10gen社のエリックさんにインタビューしてきました](http://enterprisezine.jp/dbonline/detail/4177)


今後、ロックの粒度を細かくして行く予定  
[collection level locking](https://jira.mongodb.org/browse/SERVER-1240)はチケットがある、リリースバージョンは未定  
ロックの状態は以下のコマンドで確認可能  
```
currentOp()
serverStatus()
```

### Page faultアーキテクチャの改善
ロック中にPageFaultが発生することを避ける仕組み
```
If 書き込みオペレーションで、  
・アクセス先がメモリ上になくて、mutationが起こりそうで  
・Page faultしようとしている時  
Then  
・PageFaultEceptionを発生させます  
・ロックを解除して、pageをメモリに読み込み
・書き込みオペレーションを再実行  
```
![concurrency-internals-mongodb-2-2](http://www.fedc.biz/~fujisaki/img/concurrency-internals-mongodb-2-2.png)  
参考：[MongoDB Concurrency Internals in v2.2](http://www.slideshare.net/mongodb/mongosf-mongodb-concurrency-internals-in-v22)  
書き込みオペレーションで、PageFaultが発生することが分かってる場合は、ロックする前にPageFaultExceptionを発生させてオペレーションを実行  
参考：[MongoDB v2.2に含まれる予定のConcurrency改善](http://d.hatena.ne.jp/matsukaz/20120528/1338201757)  
同一コレクション内の同時実効性(特にupdate?)が向上  
参考：[Goodbye global lock – MongoDB 2.0 vs 2.2](http://blog.serverdensity.com/goodbye-global-lock-mongodb-2-0-vs-2-2/)


## Aggregation Framework

SQLでいう、GROUP BY機能に似たもの。

<pre>
Aggregation Frameworkは保存されたデータに対しさまざまな処理や操作を行うもので、
従来はJavaScriptで実装していたような集計処理をMongoDBにコマンドを発行することで
実行できるようになる
</pre>
出所: [「MongoDB 2.2」リリース、データの集計・操作機構など多数の新機能を追加](http://sourceforge.jp/magazine/12/08/30/0423241)より引用

```
- $match    //集計処理を行う条件を指定する（SQLのHAVING区）
- $project  //集計処理を行うフィールドの選択/除外、リネーム（SQLのAS区）、計算結果のinsertを行える
- $unwind   //配列を引数にとり、展開して返す
- $group    //$sum, $avgなどを使い集計処理を実施する
- $sort     //sortキーを指定
- $skip     //数字を引数にとり、数字分スキップする
- $limit    //数字を引数にとり、数字分の集計結果を返す
```

[Aggregation Framework Reference -MongoDB マニュアル- ](http://docs.mongodb.org/manual/reference/aggregation/)

特に$projectと$unwindについて説明します。
### $project
$projectを使用するとつぎの集計処理に渡す/取り除くフィールドを指定できます。デフォルトで_idは含まれているので、取り除くためには明示的に0をセットする必要があります。
```
db.article.aggregate(
    { $project : {
        _id : 0 ,
        title : 1 ,
        author : 1
    }}
);
```
計算結果をフィールドに追加して、次の処理に渡します。
```
db.article.aggregate(
    { $project : {
        title : 1,
        doctoredPageViews : { $add:["$pageViews", 10] }
    }}
);
```
フィールド名を変更して、次の処理に渡します。
```
db.article.aggregate(
    { $project : {
        title : 1 ,
        page_views : "$pageViews" ,
        bar : "$other.foo"
    }}
);
```
サブドキュメントを作成して、次の処理に渡します。
```
db.article.aggregate(
    { $project : {
        title : 1 ,
        stats : {
            pv : "$pageViews",
            foo : "$other.foo",
            dpv : { $add:["$pageViews", 10] }
        }
    }}
);
```


### $unwind
配列を展開して、次の処理に渡します。
```js
db.article.insert({"author":"fujisaki","title":"tech mongo","tags":["Database","mongo","NoSQL"]})

db.article.aggregate(
    { $project : {
        author : 1 ,
        title : 1 ,
        tags : 1
    }},
    { $unwind : "$tags" }
);

//結果
{
        "result" : [
                {
                        "_id" : ObjectId("5059813fb70184b5bcb4bcc7"),
                        "author" : "fujisaki",
                        "title" : "tech mongo",
                        "tags" : "Database"
                },
                {
                        "_id" : ObjectId("5059813fb70184b5bcb4bcc7"),
                        "author" : "fujisaki",
                        "title" : "tech mongo",
                        "tags" : "mongo"
                },
                {
                        "_id" : ObjectId("5059813fb70184b5bcb4bcc7"),
                        "author" : "fujisaki",
                        "title" : "tech mongo",
                        "tags" : "NoSQL"
                }
        ],
        "ok" : 1
}
```

## Pipeline

![Aggregation Framework](http://www.fedc.biz/~fujisaki/img/af01.png)  
出所:[New Features in 2.2](http://kumoya.com/wordpress/wp-content/uploads/2012/09/New-Features-2.2.0.pdf)

### クラスの平均点を集計するサンプル
sample document
```js
> use classdb
> year = ["freshman", "junior", "senior"];
> for(var i=1; i<=100; i++) db.scores.insert({"name":"quiz","score":Math.floor(Math.random()*100+1),"student":i,"year":year[Math.floor(Math.random()*year.length)]})
> db.scores.count()
100
> db.scores.findOne()
{ 
  "_id" : ObjectId("5029e745a0988a275aefd0c0"),
  "name" : "quiz",	
  "score"	:	99,	
  "student"	:	7,	
  "year"	:	"junior"
}

```

集計処理
```js
>db.scores.aggregate(  { $match   : { "year"  : "junior" } },
                       { $project : { "name"  : 1, "score" : 1 } },
                       { $group   : { "_id"     : "$name", 
                                      "average" : {"$avg" : "$score" } } }
)

{
 "result" : [
		{
			"_id" : "quiz",
			"average" : 65.41666666666667
		}
	],
	"ok" : 1
}

```

SQLで同じ処理
```SQL
SELECT name, AVG(score) FROM scores
GROUP BY name
HAVING year = 'junior' 
```

## Readに関する設定

### Read時のConsistency強度を指定可能に
- PRIMARY
- PRIMARY PREFERRED
- SECONDARY
- SECONDARY PREFERRED
- NEAREST  
![StrongConsistency](http://www.fedc.biz/~fujisaki/img/StrongConsistency.png)  
![EventualConsistency](http://www.fedc.biz/~fujisaki/img/EventualConsistency.png)  
出所:[New Features in 2.2](http://kumoya.com/wordpress/wp-content/uploads/2012/09/New-Features-2.2.0.pdf)

Rubyでの例
```ruby
@collection.find({:doc => 'foo'}, :read => :primary)    # read from primary only
@collection.find({:doc => 'foo'}, :read => :secondary)  # read from secondaries only
```
出所:[Read Preference in Ruby](http://api.mongodb.org/ruby/1.7.0.rc0/file.READ_PREFERENCE.html)


### NEAREST
- ドライバからReplicaSetsにpingを打って、15ms以内で返ってきたサーバ群から1台選択し読み込む
- 1台に偏ることはない
- ドライバが一定間隔（10秒？）ごとにpingを送り、ステータスを更新している  
出所:[Member Selection](http://jp.docs.mongodb.org/manual/applications/replication/#replica-set-read-preference-behavior-nearest)


## Tagを使用したShardingが可能に

###Tagベースでのレンジパーティション
- sharding keyによる書き込み先の制御
- uid=1〜100は東京データセンターのノード、uid=100〜200はNewYorkデータセンターのノード、という設定が可能に

```js
sh.addShardTag(shard, tag)
sh.addTagRange(namespace, minimum, maximum, tag)
sh.removeShardTag(shard, tag)

//例
//login mongs
use admin
sh.addShardTag("shard0000", "TokyoDC")
sh.addShardTag("shard0001", "NewYorkDC")
sh.addShardTag("shard0002", "ParisDC")
sh.addTagRange("logdb.logs", { "uid" : 1  }, { "uid" : 10 }, "TokyoDC")
sh.addTagRange("logdb.logs", { "uid" : 10 }, { "uid" : 20 }, "NewYorkDC")
sh.addTagRange("logdb.logs", { "uid" : 20 }, { "uid" : 30 }, "ParisDC")
//tagsの削除はconfigサーバに接続して、config.tagsを変更する

use logdb
db.logs.ensureIndex( { uid : 1 } );
for(var i=1; i<=30; i++) db.logs.insert({"uid":i, "value":Math.floor(Math.random()*100000+1)})

use admin
db.runCommand( { enablesharding : "logdb" });
db.runCommand( { shardcollection : "logdb.logs" , key : { uid : 1 } } );

db.printShardingStatus(); //まだ1chunkなので書き込み先は1つになっている

//1chunk 1uidに分割
for(var i=1; i<=30; i++) db.runCommand( { split : "logdb.logs" , middle : { uid : i } } )

db.printShardingStatus();
```
参考:[mongo/jstests/slowNightly/balance_tags1.js](https://github.com/mongodb/mongo/blob/master/jstests/slowNightly/balance_tags1.js) 


## TTL(Time To Live) Collections
### 期限付きコレクションを作成できる

```js
//eventsコレクションのデータを、statusフィールドを起点に30秒後に削除されるように設定
db.events.ensureIndex( { "status": 1 }, { expireAfterSeconds: 30 } )

//statusにはdate-type informationを入れる。new Date()でOK。
db.events.insert( { "status" : new Date(), "value" : 1 } );
db.events.insert( { "status" : new Date(), "value" : 2 } );
db.events.insert( { "status" : "not Date", "value" : 3 } );
db.events.insert( { "no-status" : "blank", "value" : 4 } );

db.events.find();
//30秒後に
db.events.find();

```
参考:[Expire Data from Collections by Setting TTL](http://docs.mongodb.org/manual/tutorial/expire-data/)

### 制限
- date-type フィールドが必須
- capped collectionでは使用できない

## その他
### 認証方式が変更されました
認証付きsharding環境を構成している場合は、以下を確認してください。
- sharding環境での2.0のmogos インスタンスは、2.2で構成された認証付きsharding環境との互換性がありません。[upgrade-shard-cluster](http://docs.mongodb.org/manual/release-notes/2.2/#upgrade-shard-cluster)を参考にupgradeしてください。
- 最新ドライバのリリースノートを確認してください。

### upsert オペレーションでnullが返るようになりました。
2.0では```{}```が返ってましたが、2.2から、```null```が返ります。
```
MongoDB shell version: 2.0.6
connecting to: test
> db.test.findAndModify({query: {'_id': 1}, update: {'$inc': {'i': 1}}, upsert: true})
{ }
 
MongoDB shell version: 2.2.0
connecting to: test
> db.test.findAndModify({query: {'_id': 1}, update: {'$inc': {'i': 1}}, upsert: true})
null
```

### 2.2のMongoDBインスタンスで作成したmongodumpは2.2でしたリストアできません


### Windowsで以下の文字列がDatabase Nameで使用できなくなりました
```
/\. "*<>:|?
```

### Capped Collectionsに_idフィールドとindexが追加されました

### $elemMatch(projection)が追加されました
$elemMatchで表示するフィールドを制御できるようになりました。
詳細は[こちら](http://docs.mongodb.org/manual/reference/projection/elemMatch/)

### Windowsに関する修正
- Windows XPのサポート終了しました
- mongos.exeがWindows Serviceとしてサポートされました
- Windowsでログローテートコマンドがサポートされました
- 64bit版のWindows7,Windows Server 2008 R2 のバイナリは、並列処理に関するパフォーマンスが向上しました

### mongodump,mongorestoreでindex定義を扱えるようになりました
mongodumpで[--collection](http://docs.mongodb.org/manual/reference/mongodump/#cmdoption-mongodump--collection)を使うとindex定義をバックアップできます。
mongorestoreに[--noIndexRestore](http://docs.mongodb.org/manual/reference/mongorestore/#cmdoption-mongorestore--noIndexRestore)が追加されました。

### mongooplog コマンドが追加されました
mongooplogを使うと、レプリケーション環境でpoint-in-time backupができます。
[mongooplog Manual](http://docs.mongodb.org/manual/reference/mongooplog/)  
メモ:Oplogの役割
出所:http://www.slideshare.net/doryokujin/mongodb-oplog
```
・データが更新されるオペレーションが実行されるときは、その"オペレーション自身"をoplogというコレクションに保存
・oplogさえあれば異なるサーバでもオペレーションを実行してデータを再現できる
・"データ"ではなく、"オペレーション"を同期しあうことでレプリケーションを行う
```

### mongotopとmogostatに認証機能がサポートされました

### mongoimportとmongorestoreエラー検出オプションが追加されました
- mongoimportでは[--stopOnError](http://docs.mongodb.org/manual/reference/mongoimport/#cmdoption-mongoimport--stopOnError)をつけることで、最初のエラーが検出されたらimportを停止します。
- mongorestoreでは[--w](http://docs.mongodb.org/manual/reference/mongorestore/#cmdoption-mongorestore--w)をつけることで、[書き込み確認](http://docs.mongodb.org/manual/applications/replication/#write-concern)を行うことができます。

### mongodumpがレプリケーション環境のSecondaryサーバから取得できるようになりました

### mongoimportが16MB Documentsをサポートしました

### mongodumpなどにTimestamp()が使えるようになりました
```
mongodump --db local --collection oplog.rs --query '{"ts":{"$gt":{"$timestamp" : {"t": 1344969612000, "i": 1 }}}}'  --out oplog-dump
```
[Timestamp data type](http://www.mongodb.org/display/DOCS/Timestamp+data+type)

### shell機能改善
- Unicodeをフルサポートしました。
- mongo shellがbashライクに使えるようになりました。Ctrl-R の履歴検索などが使えるようになりました。
- 複数行コマンドの履歴が1行になりました。
- Windowsでeditコマンドが使えるようになりました。

### Server-Side Functionsをloadできるようになりました
db.system.jsに保存したfunctionをdb.loadServerScripts()でloadできるようになりました。

```js
> db.system.js.save({ "_id" : "echo", "value" : function(x){return x;} })
> echo(3)
Wed Sep 19 18:40:49 ReferenceError: echo is not defined (shell):1
> db.loadServerScripts()
> echo(3)
3

```
[Add feature to expose server-side functions in shell -SERVER-1651- ](https://jira.mongodb.org/browse/SERVER-1651)

### mongo shellでバルクインサートをサポートしました 
ドキュメントを配列形式で一括insertできます。
```js
> db.test.insert([{x:1},{x:2},{x:3}])
> db.test.find()
{ "_id" : ObjectId("505990c3845b73b3fae82ff8"), "x" : 1 }
{ "_id" : ObjectId("505990c3845b73b3fae82ff9"), "x" : 2 }
{ "_id" : ObjectId("505990c3845b73b3fae82ffa"), "x" : 3 }

```

### Verbose mode が追加されました
```
> set verbose true
> db.article.update({}, {$set : {"author" : "shoken"}})
Updated 1 existing record(s) in 1ms // <-この表示が出る

```


## 参考リンク
- [Release Notes for MongoDB 2.2](http://docs.mongodb.org/manual/release-notes/2.2/)
- [MongoDB 2.2 At the Silicon Valley MongoDB User Group](http://sssslide.com/speakerdeck.com/u/mongodb/p/mongodb-2-dot-2-at-the-silicon-valley-mongodb-user-group)  
- [「MongoDB 2.2」リリース、データの集計・操作機構など多数の新機能を追加 -sourceforge- ](http://sourceforge.jp/magazine/12/08/30/0423241)
- [MongoDB 2.2登場 - パフォーマンスや柔軟性を強化 -マイナビニュース- ](http://news.mynavi.jp/news/2012/09/03/010/index.html)
- [MongoDB 2.2 Aggregation Framework -IIJの最新技術- ](http://www.iij.ad.jp/company/development/tech/activities/mongodb/index.html)
- [MongoDB v2.2に含まれる予定のConcurrency改善 -matsukazの日記- ](http://d.hatena.ne.jp/matsukaz/20120528/1338201757)