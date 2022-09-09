# Gereksinimler

Not: Bu yazı https://pganalyze.com/docs adresindeki pganalyze resmi dökümanından derlenmiştir.

Collector, PgAnalyze servislerine her 10 dakikada bir normalize edilmiş veri ve diğer istatistikleri gönderir. Desteklenen minimum PostgreSQL sürümü 9.2'dir.
Kurulum
## Postgres Kullanıcısı Oluşturma ve Erişim Haklarının Sağlanması

İstatistik tablolarına tam olarak erişebilmek için aşağıdaki sorguları Postgres süper kullanıcısı olarak çalıştırmanın önemli olduğunu unutmayın.
```
CREATE USER pganalyze WITH PASSWORD ‘mypassword’ CONNECTION LIMIT 5;GRANT pg_monitor TO pganalyze;CREATE SCHEMA pganalyze;
GRANT USAGE ON SCHEMA pganalyze TO pganalyze;
REVOKE ALL ON SCHEMA public FROM pganalyze;CREATE OR REPLACE FUNCTION pganalyze.get_stat_replication() RETURNS SETOF pg_stat_replication AS
$$
 /* pganalyze-collector */ SELECT * FROM pg_catalog.pg_stat_replication;
$$ LANGUAGE sql VOLATILE SECURITY DEFINER;
```
postgres user ile aşağıdaki komutu çalıştırıp db’e bağlanalım ve oluşturduğumuz kullanıcının erişimin gerçekleştiğini kontrol edelim.
```
postgres@debian:~$ PGPASSWORD=aYuk71bSj1 psql -h localhost -d kasa -U pganalyze
psql (11.5 (Debian 11.5–1+deb10u1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type “help” for help.
```
## PostgreSQL Konfigürasyonlarını Yapılandırma

Bu aşamada postgres istatistiklerini tutan pg_stat_statement ayarlamaları yapacağız.

Postgresin koştuğu OS seçip devam edelim.

Aşağıdaki paketin yüklü olduğundan emin olalım,

Dikkat ! Yüklü versiyonunuz farklı ise dikkat! Kopyala yapıştır derseniz yeni bir postgresql cluster’nız olur :)

sudo apt-get install postgresql-contrib-11

pg_stat_statement extension aktif etmek için superuser ile sorguyu çalıştıralım,

postgres=# SHOW shared_preload_libraries;
-[ RECORD 1 ] — — — — — — + — — — — — — — — — -
shared_preload_libraries | pg_stat_statementspostgres=# \dx;List of installed extensions
 Name | Version | Schema | Description
 — — — — — — — — — — + — — — — -+ — — — — — — + — — — — — — — — — — — — — — — — — — — — — — — — — — — — — -
 pg_stat_statements | 1.6 | public | track execution statistics of all SQL statements executed
 plpgsql | 1.0 | pg_catalog | PL/pgSQL procedural language
(2 rows)
postgres=# ALTER SYSTEM SET shared_preload_libraries = ‘pg_stat_statements’;
ALTER SYSTEM

Daha önce pg_stat_statements kullanmadıysanız, ilk defa etkinleştirmek için Postgresi yeniden başlatmaya ihtiyacınız olacak;

sudo service postgresql restart

Pg_stat_statements öğesinin veri döndürdüğünü doğrulayalım,

postgres=# CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
CREATE EXTENSIONpostgres=# SELECT calls, query FROM pg_stat_statements LIMIT 1;
 calls | query
 — — — -+ — — — — — — — — — — — — — — — — — — — — — — — — — -
 1 | CREATE EXTENSION IF NOT EXISTS pg_stat_statements
(1 row)

Tamamdır çalışıyor.

Eğer bir hata alıyorsan sunucuyu başlatmayı unutmuş olman muhtemeldir.

## PgAnalyze Collector Yükleme

Bu adımda, pganalyze’e istatistik bilgileri gönderen toplayıcıyı yükleyeceğiz.

Öncelikle, hangi Ubuntu / Debian sürümünü çalıştırdığınızı bilmemiz gerekir.

postgres@debian:~$ lsb_release -a
No LSB modules are available.
Distributor ID: Debian
Description: Debian GNU/Linux 10 (buster)
Release: 10
Codename: buster

Benim işletim sistemim Debian 10 olduğu için aşağıdaki şekilde komut çalıştıracağım.

Bu komutu çalıştırabilmek için curl ve gnupg paketlerinin yüklü olması gerekiyor.

Aşağıdaki komutla key’i indirelim

curl -L https://packages.pganalyze.com/pganalyze_signing_key.asc | sudo apt-key add -
echo “deb [arch=amd64] https://packages.pganalyze.com/debian/buster/ stable main” | sudo tee /etc/apt/sources.list.d/pganalyze_collector.list

wget ile de yapabilirsiniz;

wget — quiet -O — https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
wget — quiet -O — https://packages.pganalyze.com/pganalyze_signing_key.asc | sudo apt-key add -

Update işleminden sonra yüklemeye hazırız.

sudo apt-get update
sudo apt-get install pganalyze-collector

## PgAnalyze Konfigürasyon Dosyasını Yapılandırma

İstatistik toplayıcının konfigürasyon dosyası şu dizinde;

/etc/pganalyze-collector.conf

Dosya içeriği aşağıdaki gibi, burada ki parametreleri adım adım dolduralım.

[pganalyze]
api_key: YAL3B2XJDTM2H4X7 #pganalyze trial ve satın almada veriyor[server1]
db_name: postgres, kasa ** #buraya yazdığın DB’e,pganalyze kullanıcısına — erişim vermeyi unutma
   #GRANT CONNECT ON DATABASE kasa TO pganalyze;
db_username: pganalyze #1.adımda oluşturulan parola ve kullaıcı adı 
db_password: mypassword
db_host: localhost #genelde localhost, farklı sunucuda ise ip yazılır
db_port: 5432

Konfigürasyonların çalışıp çalışmadığını kontrol edelim.

    pganalyze-collector — test

sami@debian:~$ sudo pganalyze-collector — test
2019/11/12 10:43:52 I [server1] Testing statistics collection…
2019/11/12 10:43:55 I [server1] Test submission successful (21.6 KB received, server 5kca67kbmvffdc3f7cbg5yldjm)
2019/11/12 10:43:55 I [server1] Testing activity snapshots…
2019/11/12 10:43:56 I [server1] Test submission successful (1.69 KB received, server 5kca67kbmvffdc3f7cbg5yldjm)

DİKKAT ! /etc/pganalyze-collector.conf dosyasına yazılan veritabanı için pganalyze kullanıcısına erişim verilmez ise aşağıdaki şekilde hata alırsınız!

sami@debian:/etc$ sudo pganalyze-collector — test
2019/11/11 14:28:47 I [server1] Testing statistics collection…
2019/11/11 14:28:48 I [server1] pg_stat_statements does not exist, trying to create extension…
2019/11/11 14:28:48 E [server1] Could not process server: Error collecting pg_stat_statements: pq: permission denied to create extension “pg_stat_statements”
2019/11/11 14:28:48 I [server1] Testing activity snapshots…
2019/11/11 14:28:49 I [server1] Test submission successful (1.69 KB received, server wg4zgyspebdzjniallmctgjslq)

PgAnalyze kullanıcımıza erişim izni verelim, ben burada tam erişim verdim.

GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO ROL_ADI;

Artık erişim hatası almayacağız;

sami@debian:~$ sudo vim /etc/pganalyze-collector.conf
sami@debian:~$ sudo pganalyze-collector — test
2019/11/16 22:38:28 I [server1] Testing statistics collection…
2019/11/16 22:38:29 I [server1] Test submission successful (23.4 KB received, server 5kca67kbmvffdc3f7cbg5yldjm)
2019/11/16 22:38:29 I [server1] Testing activity snapshots…
2019/11/16 22:38:37 I [server1] Test submission successful (1.69 KB received, server 5kca67kbmvffdc3f7cbg5yldjm)

UYARI ! Konfigürasyon güncellemesi için reload yapalım

Bu adımı yapılmazsa dashboard üzerinde database adı gözükür fakat veri gelmez.

sami@debian:/etc$ sudo pganalyze-collector — reload
Successfully reloaded pganalyze collector (PID 11249)

Test başarılı oldu, her şey tamam. Veritabanımızı artık daha konforlu bir şekilde izleyebiliriz.


