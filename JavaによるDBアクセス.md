# JavaによるDBアクセス
DB講座でDBの基礎を学んだと思いますので、ここではJava経由でDBへのアクセス/操作を行う方法を学んでいきます。

## 復習：DBについて

## 復習：ライブラリについて

## JDBCについて
上で述べたとおり、DBにアクセスするのも自前で実装することは少なく、どなたかが用意してくれたライブラリを用いて行います。
DB接続というのはもう基本的にはどこでも使うものになるので、実はJava（JDK）をインストールしたときに標準で接続や操作用のライブラリが用意されています（ArrayListとかと同じ）。
ただし、接続するDBの種類（MySQL、SQL Server）などによって、接続や操作に必要な命令も違うはず。この違いをうまく吸収するための仕組みが、JDBCになります。

### JDBCとは
JDBC（Java DataBase Connectivity）は、Javaのプログラムが簡単にDBにアクセスするためのプログラム群です。大きく以下の3つの部品で構成されています。

1. JDBC API
DB操作のクラス/インタフェースをまとめた、 Javaの標準ライブラリに含まれるAPI群。
実際にDBの操作をしたい場合などは、このライブラリのクラスなどを利用します。

2. JDBCドライバマネージャ
複数のJDBCドライバを管理するためのクラス。実際にその時に使いたいJDBCドライバを選択し、ドライバ経由でDBに接続するために利用します。

3. JDBCドライバ
特定のDBと接続するためのJavaクラス群。MySQL、SQLServerなど、接続したいDBごとにjarファイルで提供していくれているので、利用する場合はそれをダウンロードして使います。

```
    なんかしらの画像
```

この3種類のライブラリがそろって初めてJavaからDBを操作することが可能になります。先ほども述べたとおり、接続するDBによって接続のやり方や実際の操作命令などが微妙に違うはず。そこの違いをうまいこと吸収するために、ライブラリが3段構造に分かれていると認識しておいてください。
このおかげで、もし接続するDBが変わっても、JDBCドライバ部分を変えるだけで、実際のDB操作の記述部分を変えることなく同じ動作が可能になります。

言葉だけで説明されてもわかりにくいので、実際に記述してみましょう。

## JDBCを用いてDBに接続してみよう
### githubからプロジェクトをダウンロード
（ダウンロード手順などは割愛、詳細はgitリポジトリが完成してから追記）

### mavenの設定を確認
JDBCのうち、JDBC APIやJDBCドライバマネージャなどはJava標準ライブラリに含まれていますので、追加で必要になるのは接続するDBに応じたJDBCドライバです。これはjarファイルの形式で各社から配布されており、もちろんmavenなどのライブラリ管理ツールからもダウンロード可能です。
配布したプロジェクトの`pom.xml`を確認してみましょう。以下のように、SQL Server用のJDBCドライバに関する記述があります。このおかげで、JDBCドライバが勝手に導入されている状態になっています。

```xml
  <dependency>
    <groupId>com.microsoft.sqlserver</groupId>
    <artifactId>mssql-jdbc</artifactId>
    <version>12.8.1.jre11</version>
  </dependency>
```
当然、別の種類のDBを使いたい場合は、ここに別のDB用のJDBCドライバを記述するだけで利用することが可能です。

### JDBCドライバマネージャを確認
JDBCドライバマネージャは、簡単に言うとJDBCドライバを使えるように設定している部分です。今回は、`MyDriverManager.java`というコードに切り分けてこの設定部分を記述していますので、このコードを確認してみましょう。

```java
public class MyDriverManager {
    // 接続に必要な情報を先に定数として定義しておく
    private static final String URL = "jdbc:sqlserver://localhost:1433;databaseName=(DB名);";
    private static final String USER = "(ユーザ名)";
    private static final String PASSWORD = "(パスワード)";

    public static Connection getConnection() throws SQLException {

        // ①使用したいJDBCドライバのクラス名を指定
        try {
            Class.forName("com.microsoft.sqlserver.jdbc.SQLServerDriver");
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }

        // ②DriverManagerの接続情報を返却
        return DriverManager.getConnection(URL, USER, PASSWORD);
    }
}
```

このコードで行っていることは大まかに2つです。
①の部分で使用したいドライバを指定しています。今回は先ほどmaven経由で取得したSQLServer用のJDBCドライバのクラスを記述することで指定しています。
②の部分で実際に接続し、その接続情報を返却しています。ここで`java.sql.DriverManager`というクラスの`getConnection`という関数を使っています。これがJava標準クラスに含まれる、JDBCドライバマネージャのクラスの実態になっており、先ほど指定したJDBCドライバのクラス、またここではURL、ユーザ名、パスワードを使って接続を行っています。

### 実際にDBに接続しているコードを確認
先ほど取得した接続情報を使って、実際にSQLを実行する部分です。例として、`DBOperationExample.java`というクラスに切り分けています。

```java
public class DBOperationExample {
    public void fetchData() {
        // ① try-with-resourcesでDB接続情報をドライバマネージャ経由で取得
        try (Connection connection = MyDriverManager.getConnection()){

            // ② ② ステートメントを作成（SQLを実行するための関数が詰め込まれたクラス）
            Statement statement = connection.createStatement();

            // ③ SQLを実行して結果を取得
            ResultSet resultSet = statement.executeQuery("SELECT * FROM NewEmployee");

            // ③ 結果を出力
            while (resultSet.next()) {
                // ③-1 カラム名をもとにデータを取得
                int id = resultSet.getInt("ID");
                String name = resultSet.getString("NAME");

                // ③-2 取得したデータを標準出力
                System.out.println("ID: " + id + ", Name: " + name);
            }

        } catch (SQLException e) {
            e.printStackTrace();
        }
    }
}
```

ここでの`Statement`や`ResultSet`などが、一番上で説明したJava APIであり、Java標準ライブラリに含まれているJavaでのSQL操作を行うためのクラスです。もし接続するDBの種類が変わっても、このJava APIの命令は変えずに利用することが可能な作りになっています。

### 再度JDBCの仕組みを確認する。
ここで、改めて先ほどの図を確認してみましょう。
DB接続を担っている部分はJDBCドライバ（DBごとに接続するための設定などを記述）とJDBCドライバマネージャ（接続情報などを設定し、実際の接続をする）です。
なので、もしDBを変更したくなれば、JDBCドライバを別のものに変え、JDBCドライバマネージャで使いたいJDBCドライバのクラスを指定するだけで、実際のSQLや取得する記述は変更する必要がありません。
またDBの種類はそのまま、別のユーザでDBを扱いたいなどがあれば、JDBCドライバマネージャでの設定を変えればいいだけになっています。
また接続情報はそのままに、別のSQLを実行したいのであれば、当然同じJDBCドライバマネージャを用いて記述することが可能です。
このような形で、DBの接続周りの切り替えが発生しても、最低限のコードを変更するだけでいいような仕組みになっています。

## JavaAPIの記述方法
サンプルコードで、`DriverManager`, `Connection`などの標準ライブラリで実装されているDB接続/操作のためのクラスを一部紹介しましたが、ここでそれぞれの意味も合わせてまとめて紹介します。

### DriverManagerクラス
JDBCドライバを管理して、データベース接続を確立するためのクラスです。このクラスのみDB操作ではなくDB接続に用いるので分けて説明していましたが、これもJavaAPIに含まれる標準ライブラリです。

**主なメソッド**
- getConnetion(String url, String user, String password)
指定されたURL、ユーザー名、およびパスワードを使用してデータベース接続を確立します。

### Connectionインターフェース
Connectionインターフェースは、データベースとの接続情報を表します。確立されたデータベース接続情報を持っており、SQLクエリを実行するためのメソッドを提供します。

主なメソッド
- createStatement()
SQLクエリを実行するためのStatementオブジェクトを作成します。
- prepareStatement(String sql)
事前にコンパイルされたSQLクエリを実行するためのPreparedStatementオブジェクトを作成します。Statementとの違いは、下で記述しています。
- setAutoCommit(boolean autoCommit)
自動コミットモードを設定します。
- commit()
トランザクションをコミットします。
- rollback()
トランザクションをロールバックします。
- close()
データベース接続を閉じます。try-with-resource文であれば、使い終わった後に自動的にこの関数を呼び出します。

### Statementインターフェース
Statementインターフェースは、SQLクエリをデータベースに送信し、実行するためのメソッドを提供します。

**主なメソッド**
- executeQuery(String sql)
SQLクエリを実行し、結果セットを返します。SELECT文を実行するときに使います。
- executeUpdate(String sql)
データベースを更新するSQL文を実行し、影響を受けた行数を返します。INSERT/UPDATE/DELETE文を実行するときに使います。

```java
Statement statement = connection.createStatement();
ResultSet resultSet = statement.executeQuery("SELECT * FROM users");
```

### PreparedStatementインターフェース
PreparedStatementインターフェースは、事前に用意したSQLクエリを実行するためのインターフェースです。ここまでだとStatementと何も変わらないような気がしますが、パラメータ化されたクエリをサポートし、SQLインジェクション攻撃を防ぐのに役立ちます。

**主なメソッド**
- setString(int parameterIndex, String value)
指定されたパラメータインデックスに文字列値を設定します。
- setInt(int parameterIndex, int value)
指定されたパラメータインデックスに整数値を設定します。
- executeQuery(String sql)
SQLクエリを実行し、結果セットを返します。SELECT文を実行するときに使います。
- executeUpdate(String sql)
データベースを更新するSQL文を実行し、影響を受けた行数を返します。INSERT/UPDATE/DELETE文を実行するときに使います。

```java
// 後でセットしたい部分を"?"で用意しておく
String sql = "INSERT INTO users (name, email) VALUES (?, ?)";
PreparedStatement preparedStatement = connection.prepareStatement(sql);
// 一つ目の?に"John Doe"をセットしている
preparedStatement.setString(1, "John Doe");
// 一つ目の?に"john.doe@example.com"をセットしている
preparedStatement.setString(2, "john.doe@example.com");
int rowsAffected = preparedStatement.executeUpdate();
```

### 演習：DBのCRUD操作をやってみよう
- DBに任意のデータを追加してみよう
- DBに登録されているデータを更新してみよう
- DBに登録したデータを削除してみよう
- (応用)Javaの変数に応じてSQLを変更させてみよう