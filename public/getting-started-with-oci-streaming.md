---
title: OCI StreamingをSDK経由で触ってみた
tags:
  - oracle
  - streaming
  - oci
  - oraclecloud
private: false
updated_at: '2021-12-14T13:00:33+09:00'
id: 24b6daf192357ebb2b0c
organization_url_name: oracle
slide: false
ignorePublish: false
---
# はじめに

こちらの記事は、[Oracle Cloud Infrastructure Advent Calendar 2020 その2](https://qiita.com/advent-calendar/2020/oci2) の Day 11 として書かれています。（もう17日だけど、、）その 2 は結構余裕があるので滑り込ませていただきました。今回は、[OCI Streaming](https://docs.cloud.oracle.com/ja-jp/iaas/Content/Streaming/Concepts/streamingoverview.htm) を OCI SDK 経由で色々触ってみたいと思います。この辺りのコンテンツが中々見つけられなくて私自身困ったので、この記事が誰かの助けになればと思います。



# OCI Streaming とは？

一言でいうと、大量のデータストリームを処理するための分散メッセージングサービスです。
細かい用語なんかは[公式ドキュメント](https://docs.cloud.oracle.com/ja-jp/iaas/Content/Streaming/Concepts/streamingoverview.htm)を参考にしてください。（動かしてみてから読んだ方が分かりやすいかも？）

さっそく触っていきます。



# 手順

## インスタンス・プリンシパルの設定

インスタンス・プリンシパルの説明や設定方法は、こちらの記事が参考になります。（[[Oracle Cloud] インスタンスプリンシパル を OCI Go SDK で使ってみた](https://qiita.com/sugimount/items/159852749851282a33e6)）作成した動的グループに対して、ポリシーを以下のようにアタッチします。作成するポリシーは、[公式ドキュメント](https://docs.cloud.oracle.com/ja-jp/iaas/Content/Identity/Reference/streamingpolicyreference.htm)を参照し適切に設定します。

```
Allow dynamic-group <dynamic-group-name> to manage streams in compartment <compartment-name>
Allow dynamic-group <dynamic-group-name> to use stream-pull in compartment <compartment-name>
Allow dynamic-group <dynamic-group-name> to use stream-push in compartment <compartment-name>
```



## 実装例

実際に作成したコードを少し解説します。ソースコードの全量や動作方法を参照したい場合は、[リポジトリ](https://github.com/shukawam/streaming-demo)をご参照ください。

```javascript
const st = require('oci-streaming');
const common = require('oci-common');

const compartmentId = process.env.COMPARTMENT_ID;
const exampleStreamName = 'demo-streaming';

let provider;
let adminClient;
let client;
let waiters;

(async () => {
    // Use Instacne Principals
    provider = await new common.InstancePrincipalsAuthenticationDetailsProviderBuilder().build(); // ... 1
    adminClient = new st.StreamAdminClient({ authenticationDetailsProvider: provider });
    client = new st.StreamClient({ authenticationDetailsProvider: provider });
    waiters = adminClient.createWaiters();
    const partitions = 3; // ... 2
    console.log('Get or Create the stream.');
    let stream = await getOrCreateStream(compartmentId, partitions, exampleStreamName);
    // Streams are assigned a specific endpoint url based on where they are provisioned.
    // Create a stream client using the provided message endpoint.
    client.endpoint = stream.messagesEndpoint;
    const streamId = stream.id;

    // publish some messages to the stream
    await publishMessages(client, streamId); // ... 3

    // give the streaming service a second to propagate messages
    await delay(1);

    // Use a cursor for getting messages; each getMessages call will return a next-cursor for iteration.
    // There are a couple kinds of cursors.
    // A cursor can be created at a given partition/offset.
    // This gives explicit offset management control to the consumer.

    console.log('Starting a simple message loop with a partition cursor');
    const partitionCursor = await getCursorByPartition(client, streamId, '0'); // ... 4
    await simpleMessageLoop(client, streamId, partitionCursor); // ... 5

    // A cursor can be created as part of a consumer group.
    // Committed offsets are managed for the group, and partitions
    // are dynamically balanced amongst consumers in the group.

    console.log('Starting a simple message loop with a group cursor');
    const groupCursor = await getCursorByGroup(client, streamId, 'exampleGroup', 'exampleInstance-1'); // ... 6
    await simpleMessageLoop(client, streamId, groupCursor); // ... 7

    // Cleanup; remember to delete streams which are not in use.
    await deleteStream(adminClient, streamId);

    // Stream deletion is an asynchronous operation, give it some time to complete.
    const getStreamRequest = { streamId: streamId };
    await waiters.forStream(getStreamRequest, st.models.Stream.LifecycleState.Deleted);
})();

// ... 省略
```

実行すると、以下のような出力を得られます。

```
Get or Create the stream.
{
  compartmentId: 'your compartment id',
  name: 'demo-streaming',
  lifecycleState: 'ACTIVE'
}
No active stream named demo-streaming was found.
Creating stream with partitions3 // ... 2
Create Stream executed successfully.
Publishing 10 messages to stream <your stream ocid>. // ... 3
Published messages to parition 0, offset 0
Published messages to parition 2, offset 0
Published messages to parition 1, offset 0
Published messages to parition 2, offset 1
Published messages to parition 2, offset 2
Published messages to parition 2, offset 3
Published messages to parition 1, offset 1
Published messages to parition 0, offset 1
Published messages to parition 1, offset 2
Published messages to parition 2, offset 4
Starting a simple message loop with a partition cursor
Creating a cursor for partition 0. // ... 4
Read 2 messages. // ... 5
messageKey0: messageValue0
messageKey7: messageValue7
Read 0 messages.
Read 0 messages.
Read 0 messages.
Read 0 messages.
Read 0 messages.
Read 0 messages.
Read 0 messages.
Read 0 messages.
Read 0 messages.
Starting a simple message loop with a group cursor
Creating a cursor for group exampleGroup, instance exampleInstance-1. // ... 6
Read 3 messages. // ... 7
messageKey2: messageValue2
messageKey6: messageValue6
messageKey8: messageValue8
Read 0 messages.
Read 0 messages.
Read 5 messages.
messageKey1: messageValue1
messageKey3: messageValue3
messageKey4: messageValue4
messageKey5: messageValue5
messageKey9: messageValue9
Read 0 messages.
Read 2 messages.
messageKey0: messageValue0
messageKey7: messageValue7
Read 0 messages.
Read 0 messages.
Read 0 messages.
Read 0 messages.
```



### 1. AuthenticationProviderとしてインスタンス・プリンシパルを使用する  
今回は、インスタンス・プリンシパルを使用しましたが、端末に保存されているクレデンシャルを使うには、以下のようにします。  

   ```javascript
   const provider = new common.ConfigFileAuthenticationDetailsProvider();
   ```  
### 2. Streamのセクション数を指定する
Messageを複数ノードに分割することでStreamの分散読み出しが実現できます。

### 3. Messageを書き込む

   ```javascript
   (async () => {
   	// ... 省略
       // publish some messages to the stream
       await publishMessages(client, streamId); // ... 3
   	// ... 省略
   })();
   
   async function publishMessages(client, streamId) {
       // build up a putRequest and publish some messages to the stream
       let messages = [];
       for (let i = 0; i < 10; i++) {
           let entry = {
               key: Buffer.from('messageKey' + i).toString('base64'),
               value: Buffer.from('messageValue' + i).toString('base64')
           };
           messages.push(entry);
       }
       console.log('Publishing %s messages to stream %s.', messages.length, streamId);
       const putMessageDetails = { messages: messages };
       const putMessagesRequest = {
           putMessagesDetails: putMessageDetails,
           streamId: streamId
       };
       const putMessageResponse = await client.putMessages(putMessagesRequest);
       putMessageResponse.putMessagesResult.entries.forEach(entry => console.log('Published messages to parition %s, offset %s', entry.partition, entry.offset));
   }
   ```

   Base64エンコードしてから書き込んでいるのがポイントです。

### 4. Messageを読み出すためにカーソルをPartitionを指定して作成する

   ```javascript
   (async () => {
       // ... 省略
       // Use a cursor for getting messages; each getMessages call will return a next-cursor for iteration.
       // There are a couple kinds of cursors.
       // A cursor can be created at a given partition/offset.
       // This gives explicit offset management control to the consumer.
       
       console.log('Starting a simple message loop with a partition cursor');
       const partitionCursor = await getCursorByPartition(client, streamId, '0'); // ... 4
       // ... 省略
   })();
   
   async function getCursorByPartition(client, streamId, partition) {
       console.log('Creating a cursor for partition %s.', partition);
       let cursorDetails = {
           partition: partition,
           type: st.models.CreateCursorDetails.Type.TrimHorizon
       };
       const createCursorRequest = {
           streamId: streamId,
           createCursorDetails: cursorDetails
       };
       const createCursorResponse = await client.createCursor(createCursorRequest);
       return createCursorResponse.cursor.value;
   }
   ```

   2 で指定した通り、今回はPartitionを3つ作成しています。上記の処理は、3つ作成したPartitionのうち、0番目のPartitionに対してカーソル（StreamからMessageを読み出すためのポインタ）を`TRIM_HORIZON`で作成しています。他にもサポートされているカーソル・タイプとしては以下のようなものが存在します。

   > - `TRIM_HORIZON` - ストリーム内で使用可能な最も古いメッセージから消費を開始します。ストリーム内のすべてのメッセージを消費する場合に、カーソルをTRIM_HORIZONで作成します。
   > - `AT_OFFSET` -指定されたオフセットで消費を開始します。オフセットは、最も古いメッセージのオフセット以上かつ最新の公開済オフセット以下である必要があります。
   > - `AFTER_OFFSET`:指定されたオフセット後に使用を開始します。このカーソルには、AT_OFFSETカーソルと同じ制限があります。
   > - `AT_TIME` - 指定された時間から消費を開始します。戻されるメッセージのタイムスタンプは、指定された時間以降になります。
   > - `LATEST` -カーソルの作成後に公開されたメッセージの消費を開始します。

### 5. Stream内のMessageを読み出し

   ```javascript
   (async () => {
   	// ... 省略
       await simpleMessageLoop(client, streamId, partitionCursor); // ... 5
   	// ... 省略
   })();
   
   async function simpleMessageLoop(client, streamId, initialCursor) {
       let cursor = initialCursor;
       for (var i = 0; i < 10; i++) {
           const getRequest = {
               streamId: streamId,
               cursor: cursor,
               limit: 10
           };
           const response = await client.getMessages(getRequest);
           console.log('Read %s messages.', response.items.length);
           for (var message of response.items) {
               console.log(
                   '%s: %s',
                   Buffer.from(message.key, 'base64').toString(),
                   Buffer.from(message.value, 'base64').toString()
               );
           }
           // getMessages is a throttled method; clients should retrieve sufficiently large message
           // batches, as to avoid too many http requests.
           await delay(2);
           cursor = response.opcNextCursor;
       }
   }
   ```

   0番目のPartitionに対して書き込まれたMessageが読み出されます。3 でStreamに対してMessageを書き込んでいますが、Partition0には0番目と7番目のMessageが書き込まれていることが確認できます。

   ```
   // ... 省略
   Publishing 10 messages to stream <your stream ocid>. // ... 3
   Published messages to parition 0, offset 0 // <- これ
   Published messages to parition 2, offset 0
   Published messages to parition 1, offset 0
   Published messages to parition 2, offset 1
   Published messages to parition 2, offset 2
   Published messages to parition 2, offset 3
   Published messages to parition 1, offset 1
   Published messages to parition 0, offset 1 // <- これ
   Published messages to parition 1, offset 2
   Published messages to parition 2, offset 4
   // ... 省略
   ```

   Partition0に対して作成したカーソルを使用して、Messageを読み出してみると確かに0番目と7番目のMessageが書き込まれていることが確認できました。

   ```
   // ... 省略
   Starting a simple message loop with a partition cursor
   Creating a cursor for partition 0. // ... 4
   Read 2 messages. // ... 5
   messageKey0: messageValue0
   messageKey7: messageValue7
   // ... 省略
   ```

### 6. Messageを読み出すためにグループカーソルを作成する

   ```javascript
   (async () => {
   	// ... 省略
       console.log('Starting a simple message loop with a group cursor');
       const groupCursor = await getCursorByGroup(client, streamId, 'exampleGroup', 'exampleInstance-1'); // ... 6
   	// ... 省略
   })();
   
   async function getCursorByGroup(client, streamId, groupName, instanceName) {
       console.log('Creating a cursor for group %s, instance %s.', groupName, instanceName);
       const cursorDetails = {
           groupName: groupName,
           instanceName: instanceName,
           type: st.models.CreateGroupCursorDetails.Type.TrimHorizon,
           commitOnGet: true
       };
       const createCursorRequest = {
           createGroupCursorDetails: cursorDetails,
           streamId: streamId
       };
       const response = await client.createGroupCursor(createCursorRequest);
       return response.cursor.value;
   }
   ```

   複数のConsumerで論理的なグループを作成します。今回の実装例だと、Partition0, 1, 2それぞれに対して読み出すためのConsumerが存在しますがそれらを一つのグループとして捉えて、Messageの読み出しを行います。

### 7. Stream内のMessageを読み出し

   以下のように読み出されています。

   ```
   // ... 省略
   Starting a simple message loop with a group cursor
   Creating a cursor for group exampleGroup, instance exampleInstance-1. // ... 6
   Read 3 messages. // ... 7
   messageKey2: messageValue2 // ... Partition1から読み出し
   messageKey6: messageValue6
   messageKey8: messageValue8
   // ... 省略
   Read 5 messages.
   messageKey1: messageValue1 // ... Partition2から読み出し
   messageKey3: messageValue3
   messageKey4: messageValue4
   messageKey5: messageValue5
   messageKey9: messageValue9
   // ... 省略
   Read 2 messages.
   messageKey0: messageValue0 // ... Partition0から読み出し
   messageKey7: messageValue7
   // ... 省略
   ```

   上の出力結果を見て気づいたかもしれませんが、Consumer Group内のどのPartitionから読み出されるかは自動的に決定されるので制御することができません。（今回は、1 -> 2 -> 0の順番で読み出されています）

   

# 終わりに

ドキュメントとにらめっこするよりも、まずは触ってみる・動かしてみる方がその後、ドキュメントを読んだ時に理解しやすいと思います。
また、今回の例に限らずSDKを使用した実装は、OracleがGitHubで公開している実装例が非常に役に立ちます。今回は、Node.jsで実装しましたが、SDKとして使用可能な言語の実装例が提供されているので、是非参考にしてみてください！

- [Java Examples](https://github.com/oracle/oci-java-sdk/tree/master/bmc-examples/src/main/java)
- [JavaScript/TypeScript Examples](https://github.com/oracle/oci-typescript-sdk/tree/master/examples)

- [Python Examples](https://github.com/oracle/oci-python-sdk/tree/master/examples)

- [Go Examples](https://github.com/oracle/oci-go-sdk/tree/master/example)
- [Ruby Examples](https://github.com/oracle/oci-ruby-sdk/tree/master/examples-oci)

- etc ...



# 参考

- [Oracle Clous Infrastructure ドキュメント - Streaming](https://docs.cloud.oracle.com/ja-jp/iaas/Content/Streaming/Concepts/streamingoverview.htm)

- [OCI TypeScript SDK - Examples(Streaming)](https://github.com/oracle/oci-typescript-sdk/blob/master/examples/javascript/streaming.js)
