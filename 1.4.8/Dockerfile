FROM ubuntu:14.04

# ProxySQL stuff
RUN apt-get update && apt-get install -y lsb-release wget \
    && wget -O - 'http://repo.proxysql.com/ProxySQL/repo_pub_key' | apt-key add - \
    && echo deb http://repo.proxysql.com/ProxySQL/proxysql-1.4.x/$(lsb_release -sc)/ ./ | tee /etc/apt/sources.list.d/proxysql.list \
    && apt-get update && apt-get install proxysql=1.4.8

# Provide mysql client interface do use as Admin interface of ProxySQL
RUN apt-get update && apt-get install -y mysql-client

CMD ["proxysql", "-f"]