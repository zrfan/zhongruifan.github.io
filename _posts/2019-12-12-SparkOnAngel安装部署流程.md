Spark on Angel同时支持yarn和local两种运行模型，从而方便用户在本地调试程序。
spark on Angel本质上是一个spark的application，但是多了一个附属的application。在任务提交成功后，集群上会出现两个独立的application，一个是spark application，一个是angel-PS application，两个application不关联，一个spark on Angel的作业删除，需要用户或者外部系统同时kill两个。

### Sona部署流程

#### 安装spark
- 参考spark安装

#### 安装sona

##### 编译sona

```
编译环境依赖
* Jdk >= 1.8
* Maven >= 3.0.5
* Protobuf >= 2.5.0 需要和hadoop环境自带的protobuf版本保持一致。目前hadoop官方发布包使用的是2.5.0版本，所以推荐使用2.5.0版本，除非你自己使用更新的protobuf版本编译了hadoop。
```

- 1. git clone https://github.com/Angel-ML/sona.git

- 2. 编译：进入源码根目录

```
mvn clean package -Dmaven.test.skip=true
```
编译完成后，在源码根目录dist/target目录下会生成一个发布包：sona-0.1.0-bin.zip
编译时如果没有安装protobuf或者protobuf版本与angle的2.5.0不一致，会出现错误：
```
我安装官方文档要求安装的Protobuf版本为3.3>2.5.0，但是在编译的时候出现错误，提示如下：
Failed to execute goal com.github.igor-petruk.protobuf:protobuf-maven-plugin:0.6.3:run (default) on project angel-ps-core: Protobuf installation version does not match Protobuf library version
```
这里235服务器上是3.3版本的protobuf，因此在本机mac上编译，然后把发布包上传到服务器上供集群使用
如果是没有安装protobuf：
```
mac环境下做如下安装，我就解决了：
wget https://github.com/google/protobuf/releases/download/v2.5.0/protobuf-2.5.0.tar.gz
tar -xzvf protobuf-2.5.0.tar.gz
pushd protobuf-2.5.0 && ./configure --prefix=/usr/local && make && sudo make install && popd
```
因为官方sona支持的spark版本是2.3.0，与我们所使用的集群版本2.2.1不一致，跑sona样例出现问题，找不到类，因此修改sona源码中spark版本为2.2.1，代码中有一处编译错误，修改如下
```
object DataTypeUtil {
def sameType(left: DataType, right: DataType): Boolean =
// if (SQLConf.get.caseSensitiveAnalysis) {
if (SQLConf.CASE_SENSITIVE.defaultValue.get){
equalsIgnoreNullability(left, right)
} else {
equalsIgnoreCaseAndNullability(left, right)
}
...
}
```
pom文件修改如下：
```
<properties>
<scala.binary.version>2.11</scala.binary.version>
<scala.version>2.11.8</scala.version>
<!-- <spark.version>2.3.0</spark.version>-->
<spark.version>2.2.1</spark.version>
<angel.version>3.0.1</angel.version>
<breeze.version>0.13</breeze.version>
...
</properties>
```
- 3. 解压并解压发布包sona-<version>-bin.zip
```
发布包解压后，根目录下有四个子目录：
* bin：Angel任务提交脚本
* conf：系统配置文件
* data：简单测试数据
* lib：Angel jar包 & 依赖jar包
```
##### 配置环境变量
- 解压后的路径就是SONA_HOME路径
- 把解压后的文件夹sona-<version>-bin上传到HDFS，具体就是放入你设置的SONA_HDFS_HOME路径下
- 在sona项目/bin/spark-on-angel-env.sh脚本中设置环境变量：JAVA_HOME，HADOOP_HOME，SPARK_HOME，SONA_HOME，SONA_HDFS_HOME
```
其中，JAVA_HOME，HADOOP_HOME,SPARK_HOME，参考/etc/bashrc中的设置:
export JAVA_HOME=/usr/local/jdk1.8.0_131
export JRE_HOME=/usr/local/jdk1.8.0_131/jre
export SPARK_HOME=/data0/spark/spark-2.2.1-bin
export CLASSPATH=.:$JAVA_HOME/lib:$JAVA_HOME/jre/lib
export HADOOP_HOME=/usr/local/hadoop-2.7.3
export HIVE_HOME=/usr/local/hive-0.13.0
export SONA_HOME=/data10/home/XXX_bigdata_push/zhongrui3/rsync/sona/sona-0.1.0
export SONA_HDFS_HOME=viewfs://c9/user_ext/XXX_bigdata_push/zhongrui3/depend/sona-0.1.0
```
- 执行source ./spark-on-angel-env.sh，导入环境变量
```
注意：： 这个脚本会设置上述环境变量，同时重置系统的SONA_ANGEL_JARS以及SONA_SPARK_JARS的变量值
```
- 配置jar包路径：spark.ps.jars=$SONA_ANGEL_JARS和--jars $SONA_SPARK_JARS
#### 提交sona任务
Spark on angel 的任务本质上是一个spark的Application，完成Spark on Angel的程序编写打包后，
通过spark-submit脚本提交任务。
不过spark on angel提交的脚本有几点不同：
- source ./spark-on-angel-env.sh
- 配置spark.ps.jars=$SONA_ANGEL_JARS 和--jars $SONA_SPARK_JARS
- spark.ps.instance，spark.ps.cores，spark.ps.memory是配置Angel PS的资源参数
任务提交后，YARN会出现两个Application，一个是Spark Application，一个是Angel-PS Application
##### 运行样例（LR模型）
```
#! /bin/bash
- cd sona-<version>-bin/bin;
- ./SONA-example
```
SONA-example脚本的内容如下
```
source ./spark-on-angel-env.sh
$SPARK_HOME/bin/spark-submit \
--master yarn-cluster \
--conf spark.ps.jars=$SONA_ANGEL_JARS \
--conf spark.ps.instances=10 \
--conf spark.ps.cores=2 \
--conf spark.ps.memory=6g \
--jars $SONA_SPARK_JARS\
--name "LR-spark-on-angel" \
--files <lr.json_file_path> \
--driver-memory 10g \
--num-executors 10 \
--executor-cores 2 \
--executor-memory 4g \
--class org.apache.spark.angel.examples.JsonRunnerExamples \
./../lib/angelml-${SONA_VERSION}.jar \
data:<input_path> \
modelPath:<output_path> \
jsonFile:./lr.json \
lr:0.1
```
其中，spark.ps.instance，spark.ps.cores,spark.ps.memory是angel PS的参数
--files <lr.json_file_path>是用于上传本地json文件的路径，<lr.json_file_path>就是json文件的本地路径
jsonFile:./lr.json，这个参数表示使用你上传的json文件
--class参数用于指定要执行程序的主类和执行的jar包位置，后面的路径可以是本地路径
#### sona开发
需要引入的maven包如下：
- spark相关的依赖包
```
<dependency>
<groupId>org.apache.spark</groupId>
<artifactId>spark-core_2.11</artifactId>
<version>2.2.1</version>
</dependency>
<dependency>
<groupId>org.apache.spark</groupId>
<artifactId>spark-sql_2.11</artifactId>
<version>2.2.1</version>
</dependency>
<dependency>
<groupId>org.apache.spark</groupId>
<artifactId>spark-hive_2.11</artifactId>
<version>2.2.1</version>
<scope>provided</scope>
</dependency>
<dependency>
<groupId>org.apache.spark</groupId>
<artifactId>spark-mllib_2.11</artifactId>
<version>2.2.1</version>
<scope>runtime</scope>
</dependency>
```
- sona的依赖包
```
<dependency>
<groupId>com.tencent.angel</groupId>
<artifactId>sona-dist</artifactId>
<version>0.1.0</version>
<scope>provided</scope>
</dependency>
```
编写代码，注意，scala的main入口函数定义在object对象里,执行：
```
mvn clean package
```
提交spark执行：
```
单机版：
spark-submit --class com.sina.XXX.dsp.SonaTest ./dsp_sona-1.0-SNAPSHOT.jar
集群版：
spark-submit --master yarn --deploy-mode cluster --driver-memory 12g --executor-memory 2g --num-executors 10 --files $SPARK_HOME/conf/hive-site.xml --class com.sina.XXX.dsp.SonaTest ./dsp_sona-1.0-SNAPSHOT.jar
```

