FROM ubuntu:16.04

MAINTAINER PaaS-Team
LABEL name="selfcontained_cloudstack"
LABEL version="1.0.0"

ENV allpaas 'changeme'

USER root

#4 Dev Use Cases
RUN apt-get update && \
    apt-get install openntpd wget dnsutils curl nfs-kernel-server openntpd openssh-client -y && \
    apt-get clean

#See http://www.shapeblue.com/packages/
#Install CSM
RUN apt-key adv --keyserver keys.gnupg.net --recv-keys 584DF93F && \
    echo "deb http://packages.shapeblue.com/cloudstack/upstream/debian/4.9 /" > /etc/apt/sources.list.d/cloudstack.list && \
    apt-get update && \ 
    apt-get install cloudstack-management -y && \
    apt-get clean

#See http://docs.cloudstack.apache.org/projects/cloudstack-installation/en/4.9/management-server/index.html#install-mgt
#Install vhd-util 4 XenServer
RUN mkdir -p /usr/share/cloudstack-common/scripts/vm/hypervisor/xenserver && \
    wget -P /usr/share/cloudstack-common/scripts/vm/hypervisor/xenserver http://download.cloud.com.s3.amazonaws.com/tools/vhd-util && \
    chmod 0755 /usr/share/cloudstack-common/scripts/vm/hypervisor/xenserver/vhd-util


#Install Database
RUN apt-get update && \
    export DEBIAN_FRONTEND="noninteractive" && \
    echo "mysql-server mysql-server/root_password password $allpaas" | debconf-set-selections && \
    echo "mysql-server mysql-server/root_password_again password $allpaas" | debconf-set-selections && \
    apt-get install mysql-server -y && \
    apt-get clean all

RUN sed -i -e 's/\(user.*\)/user\t\t= root/g' /etc/mysql/my.cnf && \
    sed -i -e 's/\(\# \* InnoDB.*\)/\# \* InnoDB\ninnodb_rollback_on_timeout=1\ninnodb_lock_wait_timeout=600\n/g' /etc/mysql/my.cnf && \
    sed -i -e 's/\(bind-address.*\)/bind-address\t\t= 0.0.0.0/g' /etc/mysql/my.cnf && \
    sed -i -e 's/\(\#max_connections.*\)/max_connections\t\t=350/g' /etc/mysql/my.cnf && \
    sed -i -e 's/\(\#log_bin.*\)/log_bin\t\t\t= \/var\/log\/mysql\/mysql-bin.log/g' /etc/mysql/my.cnf && \
    sed -i -e 's/\(\max_binlog_size.*\)/max_binlog_size\t= 100M\nbinlog-format = '\''ROW'\''/g' /etc/mysql/my.cnf

RUN service mysql restart

RUN cloudstack-setup-databases cloud:password@mysql \
    --deploy-as=root:$allpaas

RUN mkdir -p /etc/sysconfig && \
    /bin/bash -c "echo -e \"LOCKD_TCPPORT=32803\n\
MOUNTD_PORT=892\n\
RQUOTAD_PORT=875\n\
STATD_PORT=662\n\
TATD_OUTGOING_PORT=2020\" >> /etc/sysconfig/nfs"

#From ansible-cs project
RUN mkdir -p /export/primary && \
    mkdir -p /export/secondary && \
    chmod -R 0777 /export && \
    echo "/export  *(rw,async,no_root_squash,no_subtree_check)" >> /etc/exports && \
    exportfs -a && \
    /usr/share/cloudstack-common/scripts/storage/secondary/cloud-install-sys-tmplt \
    -m /export/secondary -u http://packages.shapeblue.com.s3-eu-west-1.amazonaws.com/systemvmtemplate/4.6/new/systemvm64template-4.6-xen.vhd.bz2 -h xenserver && \
    cloudstack-setup-management && \
    mkdir -p /var/lib/cloudstack/management/.ssh && \
    ssh-keygen -f /var/lib/cloudstack/management/.ssh/id_rsa -N '' && \
    chown -R cloud:cloud /var/lib/cloudstack/management

EXPOSE 8080 8250

CMD ["/bin/bash"]
