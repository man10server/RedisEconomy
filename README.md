# RedisEconomy
Economy plugin made with redis

## Man10 fork notes

この fork では、大規模サーバーで問題になりやすいメモリ使用量を抑えるために、以下の修正を入れています。

- 起動時に全プレイヤーの残高を JVM メモリへロードしない
- プレイヤーの残高はログイン時、または明示的な残高操作時だけ Redis から読む
- ログアウト時にそのプレイヤーのローカル残高キャッシュを破棄する
- 取引履歴表示で `HGETALL` を使わず、`HSCAN` で分割して読む
- Redis の `KEYS` を使う履歴削除をやめ、`SCAN` で分割処理する
- `/archive-transactions <file>` はデフォルトでは Redis の履歴を削除しない
- Redis の履歴削除は `/archive-transactions <file> --delete-after-archive` を明示した場合だけ実行する

### この修正で解決すること

主に **Minecraft サーバー側 JVM のメモリ爆発** と、履歴表示・アーカイブ時の **一時的な巨大メモリ使用** を抑えます。

修正前は、通貨ごとに全ユーザーの残高を `ConcurrentHashMap<UUID, Double>` に常駐させていました。

ユーザー数や通貨数が増えると、ログインしていない過去ユーザーの残高まで JVM に載るため、サーバー側メモリを大きく消費します。

この fork では、基本的にオンラインユーザーと操作中ユーザーだけをローカルキャッシュします。

### この修正だけでは解決しないこと

**Redis 自体の `used_memory` が大きい問題は、この修正だけではほぼ減りません。**

Redis はメモリ上にデータを保持する DB です。

すでに Redis に保存されている取引履歴、名前 UUID 対応表、残高 Sorted Set などは、この修正を入れても自動では消えません。

特に `rediseco:transactions:<uuid>` のような取引履歴キーが大量にある場合、Redis のメモリ使用量は履歴数に比例して増えます。

この fork は履歴を読む時の一時メモリを減らしますが、保存済み履歴そのものを自動削除するものではありません。

Redis の常駐メモリを本当に減らすには、以下のような別対応が必要です。

- Redis のバックアップを取る
- `redis-cli --bigkeys` や `MEMORY USAGE <key>` で大きいキーを特定する
- 必要な履歴をファイルや別 DB に退避する
- Redis 7.4+ で `transactionsTTL` を有効にする
- 取引履歴を MySQL / PostgreSQL / MongoDB などの永続 DB に移す
- Redis にはオンラインキャッシュ、Pub/Sub、ランキングなど短命・軽量な用途だけを残す

大きな Redis キーを削除する場合は、必ず RDB/AOF などのバックアップを取ってから行ってください。Redis は後から安全に内容を書き戻す用途には向いていないため、履歴削除は慎重に扱う必要があります。

### Maven
Add this repository to your `pom.xml`:
```xml
<repository>
  <id>jitpack.io</id>
  <url>https://jitpack.io</url>
</repository>  
```

Add the dependency and replace `<version>...</version>` with the latest release version:
```xml
<dependency>
  <groupId>com.github.Emibergo02</groupId>
  <artifactId>RedisEconomy</artifactId>
  <version>4.5.0</version>
  <scope>provided</scope>
</dependency>
```

### Gradle
Add it in your root `build.gradle` at the end of repositories:
```gradle
allprojects {
  repositories {
    ...
    maven { url 'https://jitpack.io' }
  }
}
```

Add the dependency and replace `master-SNAPSHOT` with the latest release version:
```gradle
dependencies {
  compileOnly 'com.github.Emibergo02:RedisEconomy:4.5.0'
}
```
## API usage
Use the Vault API only for the Vault-linked main currency, and for other currencies use RedisEconomy’s API directly
```java
// Access Point
RedisEconomyAPI api = RedisEconomyAPI.getAPI();
if(api==null){
    Bukkit.getLogger().info("RedisEconomyAPI not found!");
}

//get a Currency
Currency currency = api.getCurrencyByName("vault"); //Same as api.getDefaultCurrency()
api.getCurrencyBySymbol("€");//Gets the currency by symbol

//Currency is a Vault Economy https://github.com/MilkBowl/VaultAPI/blob/master/src/main/java/net/milkbowl/vault/economy/Economy.java, 
//same methods and everything
currency.getBalance(offlinePlayer);
currency.withdrawPlayer(offlinePlayer, 100, "Reason of withdrawal");

//Modify a player balance (default currency)
api.getDefaultCurrency().setPlayerBalance(player.getUniqueId(), 1000);

//Get all accounts from currency cache
api.getDefaultCurrency().getAccounts().forEach((uuid, account) -> {
    Bukkit.getLogger().info("Account: "+uuid+", Balance: "+account);
});

//Direct data from redis. (Not recommended)
api.getDefaultCurrency().getOrderedAccounts().thenAccept(accounts -> {
    accounts.forEach(account -> {
        Bukkit.getLogger().info("UUID: "+account.getElement()+", Balance: "+account.getScore());
    });
});
api.getDefaultCurrency().getAccountRedis(uuid).thenAccept(account -> {
    Bukkit.getLogger().info("Balance: "+ account);
});
```
