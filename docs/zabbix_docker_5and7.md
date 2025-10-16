# サクッと複数のバージョンのZabbixをコンテナ起動

## 経緯
ありがちな「EOSとなってしまったバージョン」からの移行があって、検証用にDockerでサクッと作った。

## お品書き
imageを差し替えれば他のディストリビューションや他のDBも動作する（はず）。公式リポジトリなので英語だけどPostgreSQLはあんまり問題はでない。MySQLは地獄。

トリガー式やマクロはうまく互換を保って持っていけるけど、カスタムで作り込んだアイテムやスクリプトで外部監視している場合は要注意。OSの仕組みが変わってるしね（initd→systemdとか）。

docker-compose.ymlはこんな感じ
~~~yaml
version: "3.9"

name: zabbix50-70-lab

networks:
  zbx_net:
    driver: bridge

volumes:
  pgdata50:
  pgdata70:
  zbx50_alertscripts:
  zbx50_external:
  zbx70_alertscripts:
  zbx70_external:

services:
  # ===== Zabbix 5.0 系 =====
  pgsql50:
    image: postgres:15

    container_name: zbx-pgsql-50
    environment:
      POSTGRES_DB: zabbix
      POSTGRES_USER: zabbix
      POSTGRES_PASSWORD: zabbix_pass_50
    volumes:
      - pgdata50:/var/lib/postgresql/data
    networks:
      - zbx_net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U zabbix -d zabbix"]
      interval: 10s
      timeout: 5s
      retries: 5

  zbx-server-50:
    image: zabbix/zabbix-server-pgsql:ubuntu-5.0-latest
    container_name: zbx-server-50
    depends_on:
      - pgsql50
    environment:
      DB_SERVER_HOST: pgsql50
      DB_SERVER_PORT: "5432"
      POSTGRES_USER: zabbix
      POSTGRES_PASSWORD: zabbix_pass_50
      POSTGRES_DB: zabbix
      ZBX_LISTENPORT: "10051"              # サーバ(5.0)は標準 10051 を使用
      ZBX_ENABLE_SNMP_TRAPS: "false"
    ports:
      - "10051:10051"                      # LAN からも5.0へトラップ可能
    volumes:
      - zbx50_alertscripts:/usr/lib/zabbix/alertscripts
      - zbx50_external:/usr/lib/zabbix/externalscripts
    networks:
      - zbx_net
    restart: unless-stopped

  zbx-web-50:
    image: zabbix/zabbix-web-nginx-pgsql:5.0-ubuntu-latest
    container_name: zbx-web-50
    depends_on:
      - zbx-server-50
      - pgsql50
    environment:
      DB_SERVER_HOST: pgsql50
      DB_SERVER_PORT: "5432"
      POSTGRES_USER: zabbix
      POSTGRES_PASSWORD: zabbix_pass_50
      POSTGRES_DB: zabbix
      ZBX_SERVER_HOST: zbx-server-50
      ZBX_SERVER_PORT: "10051"
      PHP_TZ: Asia/Tokyo
      ZBX_SERVER_NAME: "Zabbix 5.0 (DockerLab)"
    ports:
      - "8050:8080"                         # 5.0 のWebをホスト8050へ
    networks:
      - zbx_net
    restart: unless-stopped

  # ===== Zabbix 7.0 系 =====
  pgsql70:
    image: postgres:17
    container_name: zbx-pgsql-70
    environment:
      POSTGRES_DB: zabbix
      POSTGRES_USER: zabbix
      POSTGRES_PASSWORD: zabbix_pass_70
    volumes:
      - pgdata70:/var/lib/postgresql/data
    networks:
      - zbx_net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U zabbix -d zabbix"]
      interval: 10s
      timeout: 5s
      retries: 5

  zbx-server-70:
    image: zabbix/zabbix-server-pgsql:ubuntu-7.0-latest
    container_name: zbx-server-70
    depends_on:
      - pgsql70
    environment:
      DB_SERVER_HOST: pgsql70
      DB_SERVER_PORT: "5432"
      POSTGRES_USER: zabbix
      POSTGRES_PASSWORD: zabbix_pass_70
      POSTGRES_DB: zabbix
      ZBX_LISTENPORT: "10061"              # 7.0 はホスト側 10061 に公開
      ZBX_ENABLE_SNMP_TRAPS: "false"
    ports:
      - "10061:10061"
    volumes:
      - zbx70_alertscripts:/usr/lib/zabbix/alertscripts
      - zbx70_external:/usr/lib/zabbix/externalscripts
    networks:
      - zbx_net
    restart: unless-stopped

  zbx-web-70:
    image: zabbix/zabbix-web-nginx-pgsql:ubuntu-7.0-latest
    container_name: zbx-web-70
    depends_on:
      - zbx-server-70
      - pgsql70
    environment:
      DB_SERVER_HOST: pgsql70
      DB_SERVER_PORT: "5432"
      POSTGRES_USER: zabbix
      POSTGRES_PASSWORD: zabbix_pass_70
      POSTGRES_DB: zabbix
      ZBX_SERVER_HOST: zbx-server-70
      ZBX_SERVER_PORT: "10061"
      PHP_TZ: Asia/Tokyo
      ZBX_SERVER_NAME: "Zabbix 7.0 (DockerLab)"
    ports:
      - "8070:8080"                         # 7.0 のWebをホスト8070へ
    networks:
      - zbx_net
    restart: unless-stopped
~~~
たまにimage名（リポジトリ内の命名ルールがルーズ）は変わるのでちょっと注意。pullでエラーが出たらDockerのリポジトリを確認してね。
~~~bash
docker compose up -d
~~~
で
~~~bash
[+] Running 6/6
 ✔ Container zbx-pgsql-70   Running                                                                                                           0.0s 
 ✔ Container zbx-pgsql-50   Running                                                                                                           0.0s 
 ✔ Container zbx-server-70  Running                                                                                                           0.0s 
 ✔ Container zbx-web-70     Running                                                                                                           0.0s 
 ✔ Container zbx-server-50  Started                                                                                                           0.3s 
 ✔ Container zbx-web-50     Started       
~~~
でOK。ね、簡単でしょ。
上記のyamlだと
Zabbix 5.0（Web）：http://localhost:8050
Zabbix 7.0（Web）：http://localhost:8070
で見えるよ。自ホスト名やIPにすればLANからも見える。

もし、
~~~bash
- Container zbx-server-50                      Starting                                                                                      1.6s 
 ✔ Container zbx-web-70                         Started                                                                                       1.4s 
 ✔ Container zbx-web-50                         Created                                                                                       0.1s 
Error response from daemon: failed to set up container networking: driver failed programming external connectivity on endpoint コンテナ名 (コンテナID): Bind for 0.0.0.0:10051 failed: port is already allocated
~~~
とか出たら、他のコンテナなり、ローカルのなにかがポートを使ってる。
このエラーみたいに「0.0.0.0:10051」がバッティングなんてさらにZabbixコンテナ使ってるくらいしかありえないけど（昔作ったコンテナを放置してた）。どんだけZabbixが好きなんだよ！
8080とか8000は衝突しがち。

## 今後の予定
新しく構築する本番のZabbixは冗長構成にするよ。公式のHAではなくPatroniとVIPを使った方式。今、検証中なので完成したら設定内容公開予定。YAMLでPostgreSQL制御して、KVSで状態管理とか今風でしょ？
