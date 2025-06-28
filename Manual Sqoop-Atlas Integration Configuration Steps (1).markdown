# Manual Sqoop-Atlas Integration Configuration Steps

This document corrects and consolidates the manual configuration steps for integrating Sqoop with Apache Atlas on a multi-node Hadoop cluster, based on details provided from April 23, 2025. It configures Sqoop on `edge1` (192.168.22.17) to import data from a MySQL database (`test_sqoop_atlas`, table `products`) into a Hive table (`test_sqoop_atlas.products_hive`), with metadata published to Atlas on `master1` (192.168.22.10:21000). The steps incorporate your `sqoop-env.sh`, Ansible playbook, `hadoopZetta` user, and files from the GitHub repository (https://github.com/hamdi-trikii/ansible-atlas-setup), including `sqoop-1.4.7.jar` and `sqoop-atlas-patch.jar`. All paths and versions (e.g., Hive 3.1.3) are aligned with your environment.

## Cluster Details
- **Nodes**:
  - `edge1` (192.168.22.17): Sqoop 1.4.7, Hive 3.1.3, Hadoop 3.3.2, Tez 0.9.1, MySQL 8.0.
  - `master1` (192.168.22.10): Atlas 2.4.0 (UI at port 21000).
  - `master2` (192.168.22.11), `master3` (192.168.22.16): Hadoop HA NameNodes, ZooKeeper (port 2181).
  - `slave2` (192.168.22.14): Kafka (port 9092).
- **HDFS HA**: Nameservice `mycluster`.
- **Users**: `hadoopZetta` (for manual steps), `gadet` (in Ansible playbook).
- **Database**: MySQL on `edge1` (`test_sqoop_atlas`, table `products`, user `edge1`).
- **Software Paths**:
  - Hadoop: `/opt/hadoop-3.3.2`
  - Hive: `/opt/apache-hive-3.1.3-bin`
  - Sqoop: `/opt/sqoop-1.4.7.bin__hadoop-2.6.0`
  - Tez: `/opt/apache-tez-0.9.1-bin`
  - Atlas: `/opt/apache-atlas-2.4.0` (on `master1`, configs copied to `edge1`)
- **GitHub Repository**: https://github.com/hamdi-trikii/ansible-atlas-setup (for `sqoop-1.4.7.jar`, `sqoop-atlas-patch.jar`).

## Configuration Steps

### 1. Install Sqoop on `edge1`
Log in to `edge1` (192.168.22.17) as `hadoopZetta`:

```bash
ssh hadoopZetta@192.168.22.17
```

Download and extract Sqoop 1.4.7:

```bash
wget https://archive.apache.org/dist/sqoop/1.4.7/sqoop-1.4.7.bin__hadoop-2.6.0.tar.gz -P /tmp
sudo tar -xvzf /tmp/sqoop-1.4.7.bin__hadoop-2.6.0.tar.gz -C /opt
sudo chown -R hadoopZetta:hadoopZetta /opt/sqoop-1.4.7.bin__hadoop-2.6.0
```

### 2. Copy Hive JDBC JARs
Copy Hive JDBC JARs from Hive’s lib directory to Sqoop’s lib directory:

```bash
sudo cp /opt/apache-hive-3.1.3-bin/lib/hive-jdbc-3.1.3.jar /opt/sqoop-1.4.7.bin__hadoop-2.6.0/lib/
sudo cp /opt/apache-hive-3.1.3-bin/lib/hive-jdbc-handler-3.1.3.jar /opt/sqoop-1.4.7.bin__hadoop-2.6.0/lib/
sudo chown hadoopZetta:hadoopZetta /opt/sqoop-1.4.7.bin__hadoop-2.6.0/lib/hive-jdbc*.jar
```

### 3. Update .bashrc for HADOOP_CLASSPATH
Add Hive JARs to `HADOOP_CLASSPATH` in `/home/hadoopZetta/.bashrc`:

```bash
echo 'export HADOOP_CLASSPATH=$HADOOP_CLASSPATH:/opt/apache-hive-3.1.3-bin/lib/hive-jdbc-3.1.3.jar:/opt/apache-hive-3.1.3-bin/lib/hive-service-3.1.3.jar:/opt/apache-hive-3.1.3-bin/lib/hive-exec-3.1.3.jar' >> /home/hadoopZetta/.bashrc
source /home/hadoopZetta/.bashrc
```

### 4. Extract Atlas Sqoop Hook
Copy the Atlas Sqoop hook tarball from `master1` to `edge1`:

```bash
scp hadoopZetta@192.168.22.10:/home/hadoopZetta/apache-atlas-sources-2.4.0/distro/target/apache-atlas-2.4.0-sqoop-hook.tar.gz /home/hadoopZetta/
```

Extract the tarball:

```bash
cd /home/hadoopZetta
tar -xvzf apache-atlas-2.4.0-sqoop-hook.tar.gz
sudo chown -R hadoopZetta:hadoopZetta apache-atlas-sqoop-hook-2.4.0
```

### 5. Create Atlas Hook Directory
Create a directory for Sqoop hooks on `edge1`:

```bash
sudo mkdir -p /opt/apache-atlas-2.4.0/hook/sqoop
sudo chown hadoopZetta:hadoopZetta /opt/apache-atlas-2.4.0/hook/sqoop
```

### 6. Copy Sqoop Hook Files
Copy Sqoop hook files to the Atlas hook directory:

```bash
sudo cp -r /home/hadoopZetta/apache-atlas-sqoop-hook-2.4.0/hook/sqoop/* /opt/apache-atlas-2.4.0/hook/sqoop/
sudo chown -R hadoopZetta:hadoopZetta /opt/apache-atlas-2.4.0/hook/sqoop
```

### 7. Copy and Configure atlas-application.properties
Copy Atlas’s configuration file from `master1` to Sqoop’s conf directory:

```bash
scp hadoopZetta@192.168.22.10:/opt/apache-atlas-2.4.0/conf/atlas-application.properties /opt/sqoop-1.4.7.bin__hadoop-2.6.0/conf/
sudo chown hadoopZetta:hadoopZetta /opt/sqoop-1.4.7.bin__hadoop-2.6.0/conf/atlas-application.properties
```

Edit to include Kafka and Atlas settings:

```bash
vi /opt/sqoop-1.4.7.bin__hadoop-2.6.0/conf/atlas-application.properties
```

Add or verify:

```properties
atlas.kafka.bootstrap.servers=192.168.22.14:9092
atlas.kafka.zookeeper.connect=192.168.22.10:2181,192.168.22.11:2181,192.168.22.16:2181
atlas.notification.create.topics=true
atlas.hook.sqoop.synchronous=true
atlas.rest.address=http://192.168.22.10:21000
atlas.cluster.name=mycluster
```

### 8. Configure sqoop-site.xml
Create or edit `sqoop-site.xml`:

```bash
vi /opt/sqoop-1.4.7.bin__hadoop-2.6.0/conf/sqoop-site.xml
```

Add:


<configuration>
  <property>
    <name>atlas.rest.address</name>
    <value>http://192.168.22.10:21000</value>
  </property>
  <property>
    <name>sqoop.job.data.publish.class</name>
    <value>org.apache.atlas.sqoop.hook.SqoopHook</value>
  </property>
  <property>
    <name>atlas.cluster.name</name>
    <value>mycluster</value>
  </property>
</configuration>
```

Set permissions:

```bash
sudo chown hadoopZetta:hadoopZetta /opt/sqoop-1.4.7.bin__hadoop-2.6.0/conf/sqoop-site.xml
```

### 9. Symlink Atlas Sqoop Hook JARs
Link Atlas Sqoop hook JARs to Sqoop’s lib directory:

```bash
sudo ln -sf /opt/apache-atlas-2.4.0/hook/sqoop/*.jar /opt/sqoop-1.4.7.bin__hadoop-2.6.0/lib/
```

### 10. Update sqoop-env.sh
Edit `sqoop-env.sh` to match your provided configuration:

```bash
vi /opt/sqoop-1.4.7.bin__hadoop-2.6.0/conf/sqoop-env.sh
```

Set contents:

```bash
export HADOOP_HOME="/opt/hadoop-3.3.2"
export HADOOP_COMMON_HOME="/opt/hadoop-3.3.2"
export HADOOP_HDFS_HOME="/opt/hadoop-3.3.2"
export HADOOP_MAPRED_HOME="/opt/hadoop-3.3.2"
export HADOOP_YARN_HOME="/opt/hadoop-3.3.2"
export HIVE_HOME="/opt/apache-hive-3.1.3-bin"
export HBASE_HOME="/opt/hbase"
export HCAT_HOME="/opt/apache-hive-3.1.3-bin/hcatalog"
export ACCUMULO_HOME="/opt/accumulo"
export ZOOKEEPER_HOME="/opt/zookeeper"
export TEZ_CONF_DIR="/opt/apache-tez-0.9.1-bin/conf"
export TEZ_JARS="/opt/apache-tez-0.9.1-bin"
export HADOOP_CLASSPATH=$HADOOP_CLASSPATH:/opt/apache-atlas-2.4.0/hook/sqoop/atlas-sqoop-plugin-impl/*:/opt/sqoop-1.4.7.bin__hadoop-2.6.0/lib/atlas-plugin-classloader-2.4.0.jar:/opt/sqoop-1.4.7.bin__hadoop-2.6.0/lib/sqoop-bridge-shim-2.4.0.jar:${TEZ_CONF_DIR}:${TEZ_JARS}/*:${TEZ_JARS}/lib/*:/opt/apache-hive-3.1.3-bin/lib/hive-jdbc-3.1.3.jar:/opt/apache-hive-3.1.3-bin/lib/hive-service-3.1.3.jar:/opt/apache-hive-3.1.3-bin/lib/hive-exec-3.1.3.jar:/opt/sqoop-1.4.7.bin__hadoop-2.6.0/lib/mysql-connector-java-8.0.28.jar:/opt/sqoop-1.4.7.bin__hadoop-2.6.0/lib/sqoop-1.4.7.jar:/opt/sqoop-1.4.7.bin__hadoop-2.6.0/lib/sqoop-atlas-patch.jar
```

Create dummy directories to avoid warnings:

```bash
sudo mkdir -p /opt/hbase /opt/accumulo /opt/zookeeper
sudo chown hadoopZetta:hadoopZetta /opt/hbase /opt/accumulo /opt/zookeeper
```

Set permissions:

```bash
sudo chown hadoopZetta:hadoopZetta /opt/sqoop-1.4.7.bin__hadoop-2.6.0/conf/sqoop-env.sh
```

### 11. Download Dependencies
Download required JARs to Sqoop’s lib directory:

```bash
wget https://repo1.maven.org/maven2/commons-configuration/commons-configuration/1.10/commons-configuration-1.10.jar -P /opt/sqoop-1.4.7.bin__hadoop-2.6.0/lib/
wget https://repo1.maven.org/maven2/org/json/json/20210307/json-20210307.jar -P /opt/sqoop-1.4.7.bin__hadoop-2.6.0/lib/
sudo chown hadoopZetta:hadoopZetta /opt/sqoop-1.4.7.bin__hadoop-2.6.0/lib/*.jar
```

### 12. Copy Repository Files
Clone the GitHub repository and copy `sqoop-1.4.7.jar` and `sqoop-atlas-patch.jar`:

```bash
git clone https://github.com/hamdi-trikii/ansible-atlas-setup.git /tmp/ansible-atlas-setup
sudo cp /tmp/ansible-atlas-setup/files/sqoop-1.4.7.jar /opt/sqoop-1.4.7.bin__hadoop-2.6.0/lib/
sudo cp /tmp/ansible-atlas-setup/files/sqoop-atlas-patch.jar /opt/sqoop-1.4.7.bin__hadoop-2.6.0/lib/
sudo cp /tmp/ansible-atlas-setup/files/sqoop-atlas-patch.jar /opt/sqoop-1.4.7.bin__hadoop-2.6.0/conf/
sudo chown hadoopZetta:hadoopZetta /opt/sqoop-1.4.7.bin__hadoop-2.6.0/lib/sqoop*.jar
sudo chown hadoopZetta:hadoopZetta /opt/sqoop-1.4.7.bin__hadoop-2.6.0/conf/sqoop-atlas-patch.jar
```

### 13. Configure MySQL Table
Set up the MySQL table `products` in `test_sqoop_atlas` (from your document):

```bash
mysql -h 192.168.22.17 -u edge1 -p
```

```sql
USE test_sqoop_atlas;
CREATE TABLE products (
    id INT PRIMARY KEY,
    name VARCHAR(100)
);
INSERT INTO products (id, name) VALUES
(1, 'Laptop'),
(2, 'Phone'),
(3, 'Tablet');
GRANT ALL ON test_sqoop_atlas.* TO 'edge1'@'localhost' IDENTIFIED BY '<password>';
FLUSH PRIVILEGES;
```

### 14. Test Sqoop with Atlas Hook
Run a Sqoop job to import from MySQL to Hive with Atlas hook (aligned with your document and Feb 27, 2025, memory about `test_sqoop_atlas`):

```bash
source /opt/sqoop-1.4.7.bin__hadoop-2.6.0/conf/sqoop-env.sh
sqoop import \
  --connect "jdbc:mysql://192.168.22.17:3306/test_sqoop_atlas?useSSL=false&serverTimezone=UTC" \
  --username edge1 \
  --password <password> \
  --table products \
  --hive-import \
  --hive-database test_sqoop_atlas \
  --hive-table products_hive \
  --create-hive-table \
  --atlas-hook
```

## Notes
- **Hive Services**: You noted HiveServer2 and Metastore were not running, but no configuration steps were provided for starting them.
- **Atlas Source**: Assumed `/home/hadoopZetta/apache-atlas-sources-2.4.0` exists on `edge1` or is copied from `master1`.
- **MySQL Password**: Replace `<password>` with the actual `edge1` password.
- **Prior Context**: Incorporated your Feb 27, 2025, mention of a `SQOOP` table in `test_sqoop_atlas` and Apr 23, 2025, Hive hook setup on `edge1` for consistency.
- **Ansible Alignment**: Steps mirror your playbook’s tasks, correcting `atlas.rest.address` to `http://192.168.22.10:21000` and Hive version to 3.1.3.
- **Kafka Topics**: Omitted topic creation as you didn’t provide explicit commands, but included `atlas.notification.create.topics=true`.