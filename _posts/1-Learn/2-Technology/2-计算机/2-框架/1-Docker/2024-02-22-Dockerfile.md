---
categories: [Dockerfile]
tags: Dockerfile
---


# 编写Dockerfile
~~~dockerfile
#声明变量
ARG zeeplinVersion=0.10.1
#ARG kinitPrincipal=hive/**-**-cm-**-17@HADOOP.JJWORLD.TECH


FROM apache/zeppelin:${zeeplinVersion}

USER root

#升级并添加一些软件
RUN set -ex && \
    mkdir -p /var/lib/apt/lists/partial && \
    sed -i s@/archive.ubuntu.com/@/mirrors.aliyun.com/@g /etc/apt/sources.list && \
    apt-get update && \
    apt-get install -y telnet curl inetutils-ping  wget vim sudo net-tools lsof bash tini libc6 libpam-modules krb5-user libpam-krb5 libpam-ccreds libkrb5-dev libnss3 procps && \
    rm -rf /var/lib/apt/lists/*

#修改时区
RUN /bin/cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && \
    echo 'Asia/Shanghai' > /etc/timezone

#创建必要目录 && \
RUN mkdir -p /opt/kerberos && \
    mkdir -p /opt/kubernetes && \
    mkdir -p /etc/.kube && \
    mkdir -p /opt/java


#配置java环境
RUN  ln -s /usr/lib/jvm/java-8-openjdk-amd64 /opt/java/openjdk


#配置kerberos认证
RUN wget -P /opt/kerberos/ http://10.11.**.**/yum/keytab/hive.keytab && \
    wget -P /opt/kerberos/ http://10.11.**.**/yum/krb5.conf&& \
    cat /opt/kerberos/krb5.conf > /etc/krb5.conf && \
    kinit -kt /opt/kerberos/hive.keytab hive/hb11-bd-cm-**-**@HADOOP.**.**


#配置k8s环境
RUN wget -P /opt/kubernetes/ http://10.11.**.**/yum/BigdataMoudles/K8s/kubectl && \
    wget -P /opt/kubernetes/ http://10.11.**.**/yum/BigdataMoudles/K8s/config/hb11-k8s-bd-cluster02 && \
    mv /opt/kubernetes/hb11-k8s-bd-cluster02 /etc/.kube/config && \
    chmod 755 /opt/kubernetes/kubectl  && \
    mv /opt/kubernetes/kubectl /usr/bin/

ENV KUBECONFIG=/etc/.kube/config


#配置hadoop环境
RUN wget --user=admin --password=admin --auth-no-challenge  'http://10.11.**.**:**/api/v32/clusters/landlord_spade/services/hive/clientConfig' -O /opt/landlord_spade_hive.zip && \
    unzip -d /opt/ /opt/landlord_spade_hive.zip && \
    mv /opt/hive-conf /opt/hadoop-conf


#配置spark
RUN wget -P /opt/ http://10.11.**.**/yum/BigdataMoudles/Spark/spark-3.3.2-bin-hadoop3.gz  && \
    tar -zxf /opt/spark-3.3.2-bin-hadoop3.gz -C /opt/ && \
    rm -rf /opt/spark-3.3.2-bin-hadoop3.gz && \
    mv /opt/spark-3.3.2-bin-hadoop3 /opt/spark && \
    ln -s /opt/spark /spark && \
    ln -s /opt/hadoop-conf/core-site.xml /opt/spark/conf/ && \
    ln -s /opt/hadoop-conf/hdfs-site.xml /opt/spark/conf/ && \
    ln -s /opt/hadoop-conf/yarn-site.xml /opt/spark/conf/ && \
    ln -s /opt/hadoop-conf/hive-site.xml /opt/spark/conf/ && \
    cp /opt/spark/conf/spark-env.sh.template /opt/spark/conf/spark-env.sh && \
    echo "export JAVA_HOME=/opt/java/openjdk" >> /opt/spark/conf/spark-env.sh && \
    echo "export HADOOP_CONF_DIR=/opt/hadoop-conf:/opt/hadoop-conf" >> /opt/spark/conf/spark-env.sh && \
    echo "export YARN_CONF_DIR=/opt/hadoop-conf:/opt/hadoop-conf" >> /opt/spark/conf/spark-env.sh
~~~


 

 
