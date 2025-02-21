Подключение к хосту:

```sh
ssh team@176.109.91.7
```

Создание ключа на jump node и копирование его публичной части на остальные ноды, чтобы можно было подключаться к ним без пароля
```sh
ssh-keygen
cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys
scp ~/.ssh/authorized_keys 192.168.1.23:/home/team/.ssh/
scp ~/.ssh/authorized_keys 192.168.1.24:/home/team/.ssh/
scp ~/.ssh/authorized_keys 192.168.1.25:/home/team/.ssh/
```

На каждой виртуальной машине в /etc/hosts закомментировать все адреса и вставить такие хосты:
```
192.168.1.22 team-5-jn
192.168.1.23 team-5-nn
192.168.1.24 team-5-dn-00
192.168.1.25 team-5-dn-01
```

Создать пользователя hadoop на всех виртуальных машинах

```sh
ssh team@176.109.91.7

ssh team@192.168.1.22
sudo adduser hadoop
exit

ssh team@192.168.1.23
sudo adduser hadoop
exit

ssh team@192.168.1.24
sudo adduser hadoop
exit

ssh team@192.168.1.25
sudo adduser hadoop
exit
```

Создание ключа на jump node и копирование его публичной части на остальные ноды, чтобы можно было подключаться к ним без пароля
(от лица пользователя hadoop)
```sh
ssh-keygen
cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys
scp -r ~/.ssh 192.168.1.23:/home/hadoop/
scp -r ~/.ssh 192.168.1.24:/home/hadoop/
scp -r ~/.ssh 192.168.1.25:/home/hadoop/
```

Скачать дистрибутив hadoop
```sh
wget https://dlcdn.apache.org/hadoop/common/hadoop-3.4.1/hadoop-3.4.1.tar.gz
```

Скопировать архив с hadoop на остальные ноды
```sh
scp hadoop-3.4.1.tar.gz team-5-nn:/home/hadoop/
scp hadoop-3.4.1.tar.gz team-5-dn-00:/home/hadoop/
scp hadoop-3.4.1.tar.gz team-5-dn-01:/home/hadoop/
```

Распаковать архив с дистрибутивом hadoopЖ
```sh
tar -xvzf hadoop-3.4.1.tar.gz
```

Команда `readlink -f "$(which java)"` возвращает `/usr/lib/jvm/java-11-openjdk-amd64/bin/java`,
путь `/usr/lib/jvm/java-11-openjdk-amd64` запишем в env переменную `JAVA_HOME`

В `~/.profile` добавить пути для hadoop и добавить 2 папки в `PATH`:
```sh
export HADOOP_HOME=/home/hadoop/hadoop-3.4.1
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
export PATH="$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin"
```

Применить изменения в текущей сессии:
```sh
source ~/.profile
```

Скопировать обновлённый .profile с jump node на остальные ноды
```sh
scp ~/.profile team-5-nn:/home/hadoop/
scp ~/.profile team-5-dn-00:/home/hadoop/
scp ~/.profile team-5-dn-01:/home/hadoop/
```

Добавить `JAVA_HOME` в скрипт, задающий переменные окружения hadoop:
```sh
vim ~/hadoop-3.4.1/etc/hadoop/hadoop-env.sh
```
Добавить `JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64`

Добавить
```xml
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://team-5-nn:9000</value>
    </property>
</configuration>
```
в `~/hadoop-3.4.1/etc/hadoop/core-site.xml` (конфигурация адреса, по которому будет доступна файловая система hdfs)

Добавить
```xml
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>3</value>
    </property>
</configuration>
```
в `~/hadoop-3.4.1/etc/hadoop/hdfs-site.xml` (конфигурация фактора репликации)

Обновить конфигурацию воркеров:
```sh
echo "team-5-nn
team-5-dn-00
team-5-dn-01" > ~/hadoop-3.4.1/etc/hadoop/workers
```

Скопировать конфиги с jump node на остальные ноды:
```sh
pushd ~/hadoop-3.4.1/etc/hadoop
scp hadoop-env.sh core-site.xml hdfs-site.xml workers team-5-nn:/home/hadoop/hadoop-3.4.1/etc/hadoop
scp hadoop-env.sh core-site.xml hdfs-site.xml workers team-5-dn-00:/home/hadoop/hadoop-3.4.1/etc/hadoop
scp hadoop-env.sh core-site.xml hdfs-site.xml workers team-5-dn-01:/home/hadoop/hadoop-3.4.1/etc/hadoop
popd
```

Подключиться к виртуальной машине, на которой будет запущен демон name node:
```sh
ssh team-5-nn
```

Отформатировать файловую системы
```sh
hdfs namenode -format
```

Запустить файловую систему:
```sh
start-dfs.sh
```

В качестве проверки корректности работы запущенно hdfs были выполнены следующие действия:

У данной команды ожидаемо пустой вывод (ещё не создавалось никаких файлов с момента форматирования fs)
```sh
hdfs dfs -ls /
```
После создания папки:
```sh
hdfs dfs -mkdir /testdir
```
Вывод команды `hdfs dfs -ls /` (выполнена на jn):
```sh
Found 1 items
drwxr-xr-x   - hadoop supergroup          0 2025-02-21 19:54 /testdir
```
Вывод команды `hdfs dfs -ls /` (выполнена на dn-01):
```sh
Found 1 items
drwxr-xr-x   - hadoop supergroup          0 2025-02-21 19:54 /testdir
```

В логах не было логов с severity ERROR
```sh
tail -n 20 ~/hadoop-3.4.1/logs/hadoop-hadoop-namenode-team-5-nn.log
tail -n 20 ~/hadoop-3.4.1/logs/hadoop-hadoop-datanode-team-5-nn.log
```
