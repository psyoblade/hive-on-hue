# Docker for HUE, Hive, Hadoop
> 도커를 통해서 Hadoop, Hive 기반의 Hue 설정을 해보자


## 1. 휴(Hue) 및 하이브 초기 환경 구성
* "hue/conf/hue.ini" 파일에서 아래의 항목은 필요에 따라 수정될 수 있습니다
```ini
[desktop]
  ...
  [[database]]
    engine=mysql
    host=<host>
    port=3306
    user=<user>
    password=<password>
    name=<database>
   ...
[beeswax]
  ...
  hive_server_host=<host>
  hive_server_port=10000
  hive_server_http_port=10001
  ...
  hive_metastore_host=<host>
  hive_metastore_port=9083
  hive_conf_dir=/etc/hive/conf
  ...
  thrift_version=7
  ...
```


## 2. 하이브에서 데이터 로딩 시에 필요한 데이터 경로를 위한 볼륨 지정
> hive/data 경로에 하이브 데이터 로딩에 필요한 데이터 파일을 /tmp/data 경로에 볼륨 마운트가 됩니다
```bash
cat hive/data/input.csv
1,suhyuk
2,psyoblade
```
* Hue 최초 접속 시에는 root 계정과 패스워드를 생성하고, 아래와 같이 테이블 생성을 합니다
```sql
drop table if exists foo;
create table foo (id int, name string) row format serde 'org.apache.hadoop.hive.serde2.OpenCSVSerde'
WITH SERDEPROPERTIES (
   "separatorChar" = ",",
   "quoteChar"     = "'",
   "escapeChar"    = "\\"
) ;
load data local inpath '/tmp/data/input.csv' into table foo;
show tables;
select * from foo;
```
* 좀 더 간단한 명령을 통해 테이블 생성도 가능합니다
```sql
drop table if exists bar;
create table bar(id int, name string) row format delimited fields terminated by ',' stored as textfile;
load data local inpath '/tmp/data/input.csv' into table bar;
select id, name from bar;
```


## 3. 도커 컴포즈를 통한 컨테이너 실행
* 환경변수를 지정하여 bind volume 의 경우에도 전체경로를 docker-compose.yml 파일에 포함하지 않아도 됩니다 (-d, --detach 옵션도 사용 가능합니다)
* 도커 컴포즈의 경우 현재 경로의 이름을 그대로 컨벤셔을 따라가기 때문에 반드시 hue 라는 이름으로 checkout 받아야 한다
```bash
git clone https://github.com/psyoblade/hive-on-hue.git hue
cd hue
export PROJECT_HOME=`pwd` ; docker-compose up
```


## 4. 도커 컴포즈 실행 대기 스크립트 적용
> 실행 시에 depends\_on 명령만으로 대상 컨테이너가 Ready 상태임을 알 수는 없기 때문에 별도의 스크립트 작업이 필요하다. 단, wait-for-it.sh 스크립트는 리눅스 환경에서만 제대로 동작한다 (-\_-;) 이 또한 해당 이미지에 포함되어야 하므로 매번 넣기는 귀찮기 때문에 나이브하게 30~60초 정도 대기 후에 실행하도록 작성되었다

