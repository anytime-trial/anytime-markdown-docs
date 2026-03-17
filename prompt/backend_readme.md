## docker command

ボリューム一覧
docker volume ls
IP アドレス一覧
docker network inspect shared_network
Dokcer のログ確認
docker compose logs nginx
未使用イメージ一括削除
docker rmi `docker images -q`

### VSCode セットアップ(拡張機能)

- Extension Pack for Java
- Spring Boot Extension Pack
- Gradle for Java
- Checstyle for Java
- Thunder Client

### 削除用

drop table users CASCADE;
drop table profiles;
drop table accounts;
drop table verifications;
drop table signin_historys;
drop table flyway_schema_history;

###

```wsl
#タスク ID を取得する
aws ecs list-tasks --cluster DevCluster --service-name ECS-DevBackend
#コンテナインスタンス ID を取得する
aws ecs describe-tasks --cluster DevCluster --tasks 8e6e0b0ba4ba40ada277c76994065662
#ECS Exec を有効にする
aws ecs update-service \
  --cluster DevCluster \
  --service ECS-DevBackend \
  --enable-execute-command
#ECS Exec を実行する
aws ecs execute-command \
  --cluster DevCluster \
  --task 59096a174975482da8523bb784b7cf84 \
  --container traefik \
  --interactive \
  --command "sh"
```

aws ecs update-service --cluster DevCluster --service ECS-DevBackend --force-new-deployment

### java fomatter

1. java.format.settings.url https://raw.githubusercontent.com/google/styleguide/gh-pages/eclipse-java-google-style.xml
2. java.format.settings.profile GoogleStyle
3. editor.formatOnSave true

### PowerBI 接続

1. ODBC 接続用のドライバーをダウンロードする
2. PowerBI -> データの取得 -> odbc -> DSN (なし) -> 接続文字列に移動します。
3. ドライバー={PostgreSQL ANSI(x64)}; サーバー= [DBLINK]; ポート= [DBPORT]; データベース= postgres
   （例）
   Driver={PostgreSQL ANSI(x64)}; Server= aws-0-ap-northeast-1.pooler.supabase.com; Port= 5432; Database= postgres
