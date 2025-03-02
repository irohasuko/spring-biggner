# apprication.propertiesの使い方
## `application.properties` の役割  
前回の講義で、DB接続情報などをapprication.propertiesに記述しました。以前も紹介しましたが、`application.properties`（または `application.yml`）は、アプリケーションの設定を管理するためのファイルです。以下のような設定を記述できます。  

- **データベース接続情報**  
- **サーバー設定**
- **ログ設定**  
- **カスタム設定（環境変数の管理）**  

## ① データベース接続
前回も行った通り、DB接続情報などはWEBアプリ共通で持つべき設定なので、このファイルに記述します。
使用するライブラリによって若干記述が変わることもありますが、基本的に以下のような記述は必須です。

### **1. `application.properties` の記述**
```properties
# データベース接続情報（SQLServer）
spring.datasource.url=jdbc:sqlserver://localhost:1433;databaseName=master;
spring.datasource.username=(ユーザ名)
spring.datasource.password=(パスワード)
spring.datasource.driver-class-name=com.microsoft.sqlserver.jdbc.SQLServerDriver
```

### **2. JDBC を使ったコード例**
```java
@Repository
public class UserRepository {
    private final JdbcTemplate jdbcTemplate;

    public UserRepository(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    public List<User> findAll() {
        return jdbcTemplate.query("SELECT * FROM users", (rs, rowNum) -> 
            new User(rs.getLong("id"), rs.getString("name"), rs.getString("email"))
        );
    }
}
```

## ② サーバー設定
SpringBootの内部tomcatを扱う場合、そのtomcatの設定を一部プロパティファイル内で定義することが可能です。tomcatの設定はいろいろめんどくさい部分も多いのですが、SpringBoot内部のtomcatを使う場合のみはまとめてここに記述することが可能なので、とっても楽に設定できます。
```properties
# サーバーのポート設定
server.port=8081

# コンテキストパスの設定（例: http://localhost:8081/api）
server.servlet.context-path=/api
```

## ③ ログ設定
研修ではあまり扱っていませんが、適切なログを出力することは実務において必須の考えです。
ログに関してもアプリ全体で扱う話ではあるので、その設定をプロパティファイルにて記述することが可能です。
いろいろ設定はありますが、ここではログレベルの設定を行っている例を示します。ログレベルとは、そのログが単なる情報なのか、デバッグ目的なのか、エラーなのかなど、管理しやすくするためのログの段階です。
開発中はどのレベルのログを出力し、運用中はどのレベルのログを出力するなどの使い分けをするために、ログにレベルを設定して管理しやすくします。

```properties
# ログレベルの設定
logging.level.root=INFO
logging.level.com.example=DEBUG
```

**利用例**
```java
@Service
public class UserService {
    private static final Logger logger = LoggerFactory.getLogger(UserService.class);

    public void processUser(String name) {
        logger.info("Processing user: {}", name);
    }
}
```
この場合、`com.example` パッケージ内のログは `DEBUG` レベル、それ以外は `INFO` レベルでログが出力されます。

---

## ④ 環境別設定（`application.properties` と `application-<profile>.properties`）
環境（開発・本番）ごとに異なる設定を適用したい場合、`application-dev.properties` や `application-prod.properties` を作成し、Spring にどのプロファイルを適用するか指定できます。
開発中はこちらのDB、本番環境は別のDBなど切り替える場合に、プロパティファイルを分けて管理しておくことで、切り替えを楽にすることが可能です。

```properties
# application.properties
spring.profiles.active=dev
```

```properties
# application-dev.properties
spring.datasource.url=jdbc:mysql://localhost:3306/devdb
spring.datasource.username=dev_user
spring.datasource.password=dev_pass
```

```properties
# application-prod.properties
spring.datasource.url=jdbc:mysql://prod-server:3306/proddb
spring.datasource.username=prod_user
spring.datasource.password=prod_pass
```

## ⑤ カスタム設定値を定義し、アプリ内で利用
その他、Javaコード内には記述したくないが、Javaコード内で使う必要がある、みたいな値をプロパティファイル内で設定し、Javaコード内で呼び出す方法があります。
以下のように記述して、Javaコード内で利用可能です。
```properties
# カスタムプロパティ
app.title=My Spring Boot App
app.version=1.0.0
```

### **利用コード**
```java
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

@Component
public class AppConfig {
    @Value("${app.title}")
    private String title;

    @Value("${app.version}")
    private String version;

    public void printAppInfo() {
        System.out.println("App: " + title + " (Version: " + version + ")");
    }
}
```

- **`@Value`アノテーション**
これを用いてプロパティファイル内で定義したプロパティ名を指定すると、そこで指定した文字列が変数内に代入されます。

「Javaコード内には記述したくないが、Javaコード内で使う必要がある」っていう場合が思いつきにくいかもしれません。これは実務に当たらないとイメージしにくい部分かもしれませんが、プロパティファイル内に定義した値は`@Value`で取り出すことができる！ということだけは理解しておきましょう。
