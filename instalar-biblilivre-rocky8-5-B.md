# <u>Instalar Biblivre5 no Rocky Linux 8.5 e 8.6</u>

A instalação será baseada no GitHub de [Cleydyr Bezerra de Albuquerque](https://github.com/cleydyr) que fez um script para CentOS 7 para a instalação do [Biblivre5](https://gist.github.com/cleydyr/95db7654ca2d915ddf3d8fe2e2c04fbe) com algumas pequenas modificações para funcionamento correto no Rocky Linux 8.5 

OBS: todos os comandos no Terminal, sendo executados como usuário root (#)

Após a instalação do Rocky Linux que não será abordado aqui, vamos iniciar com a atualização do sistema:

```
yum update -y && yum upgrade -y
```

Instalar os seguintes pacotes: 

```
yum install -y wget tar postgresql-server postgresql-contrib java-1.8.0-openjdk java-1.8.0-openjdk-devel
```

Iniciar o serviço do postgresql;

```
postgresql-setup --initdb
que deve dar essa saída:
* Initializing database in '/var/lib/pgsql/data'
* Initialized, logs are in /var/lib/pgsql/initdb_postgresql.log
```

Habilitar o serviço do postgresql no boot do sistema;

```
systemctl enable postgresql
```

Reinicar o serviço do postgresql;

```
systemctl restart postgresql
```

Verificar se o serviço do postgresql está rodando com o comando:

```
systemctl status postgresql
```

Permite a autenticação com senha no Postgres com o comando:

```
sed -i -e 's/ident$/md5/g' /var/lib/pgsql/data/pg_hba.conf
```

Restart o serviço do postgresql:

```
systemctl start postgresql
```

Criar uma senha padrão para o PostgreSQL (abracadabra):

```
sudo su - postgres -c "psql -o /dev/null -U postgres -c "'"'"ALTER USER postgres WITH PASSWORD 'abracadabra'"'"'"";
```

Criar usuário biblivre e o banco de dados básico biblivre4:

```
sudo su - postgres -c "wget --quiet -O - https://raw.githubusercontent.com/cleydyr/Biblivre-5/5.1.0/sql/createdatabase.sql | psql -o /dev/null -U postgres"
```

Criar o esquema básico do Biblivre (isso pode demorar um pouco):

```
sudo su - postgres -c "wget --quiet -O - https://raw.githubusercontent.com/cleydyr/Biblivre-5/5.1.0/sql/biblivre4.sql | psql -o /dev/null -U postgres -d biblivre4"
```

# Instalar o Tomcat 7.0.42 (obrigatório esta versão somente)

Criar um usuário e diretório para tomcat:

```
useradd -m -U -d /opt/tomcat -s /bin/false tomcat
```

Mudar para o diretório criado:

```
cd /opt/tomcat
```

Baixar o tomcat  versão 7.0.42 com o comando:

```
wget https://archive.apache.org/dist/tomcat/tomcat-7/v7.0.42/bin/apache-tomcat-7.0.42.tar.gz
```

Extrair o arquivo do tomcat baixado com o comando:

```
tar -xf apache-tomcat-7.0.42.tar.gz 
```

Declarar uma variável e Criar um link simbólico como ponto de última instalação:

```
VERSION=7.0.42
ln -s /opt/tomcat/apache-tomcat-${VERSION} /opt/tomcat/latest
```

Mudar o proprietáro do diretório do Tomcat para o usuário tomcat criado antes recursivamente:

```
chown -R tomcat:tomcat /opt/tomcat
```

Criar um arquivo Systemd Unit (serviço) para o tomcat. Mude para o diretório:

```
cd /etc/systemd/system/ 
```

Crie um arquivo chamado tomcat.service (será usado o editor nano):

```
nano tomcat.service
```

Copie e cole o conteúdo abaixo para seu arquivo criado:

```
[Unit]
Description=Apache Tomcat 7.0.42 Servlet container
Wants=network.target
After=network.target

[Service]
Type=forking

User=tomcat
Group=tomcat

Environment="JAVA_HOME=/usr/lib/jvm/jre"
Environment="JAVA_OPTS=-Djava.security.egd=file:///dev/urandom"

Environment="CATALINA_BASE=/opt/tomcat/latest"
Environment="CATALINA_HOME=/opt/tomcat/latest"
Environment="CATALINA_PID=/opt/tomcat/latest/temp/tomcat.pid"
Environment="CATALINA_OPTS=-Xms512M -Xmx1024M -server -XX:+UseParallelGC"

ExecStart=/opt/tomcat/latest/bin/startup.sh
ExecStop=/opt/tomcat/latest/bin/shutdown.sh
Restart=always

[Install]
WantedBy=multi-user.target
```

Faça **control+o** para salvar o arquivo e **control+x** para sair do editor **nano**:

Habilitar e iniciar o tomcat 

```
 systemctl daemon-reload
 systemctl enable tomcat --now
```

Checar se o serviço do tomcat está rodando

```
systemctl status tomcat
```

Criar um arquivo de configurações padrão para o Tomcat rodar o Biblivre mas antes criar o diretório:

```
mkdir -p /etc/tomcat/conf.d
```

Então criar o arquivo de configuração para o tomcat rodar o Biblivre

```
sudo sh -c "echo 'JAVA_OPTS="'"'"-Djava.security.egd=file:/dev/./urandom -Djava.awt.headless=true -Xmx512m -XX:MaxPermSize=256m -XX:+UseConcMarkSweepGC"'"'"' >> /etc/tomcat/conf.d/biblivre.conf"
```

Criar um diretório **Biblivre4** (NÃO MUDAR O NOME) dentro do sub-diretório **webapps** do tomcat. Como o tomcat foi instalado no diretório **/opt/tomcat** é necessário mudar para esse diretório:

```
cd /opt/tomcat/apache-tomcat-7.0.42/webapps
```

Criar então um diretório **Biblivre4** com o comando:

```
mkdir Biblivre4
```

Baixar o **Biblivre 5** atualizado do repositório de **Cleydyr**:

```
echo 'https://github.com/cleydyr/biblivre/releases/download'`wget --spider -SO-  https://github.com/cleydyr/biblivre/releases/latest 2>&1 >/dev/null | grep "Location:" | egrep -o "/v.*$"`"/Biblivre4.war" | tr -d "\r" | xargs wget -O /tmp/Biblivre4.war
```

Descompactar o arquivo baixado para o diretório Biblivre4 criado anteriormente com o comando:

```
unzip -q /tmp/Biblivre4.war -d /opt/tomcat/apache-tomcat-7.0.42/webapps/Biblivre4
```

Liberar a porta padrão do Tomcat (8080) no firewall do Rocky Linux e recarregar o firewall com os comandos:

```
firewall-cmd --zone=public --permanent --add-port=8080/tcp
firewall-cmd --reload
```

Permita o Tomcat rodar mais permissivamente, sem restrições da política SELinux com o comando:

```
sudo sed -i 's/^SELINUX=.*/SELINUX=permissive/g' /etc/selinux/config
```

Reestart o serviço do Tomcat e verifique o status do serviço do Tomcat

```
systemctl restart tomcat
systemctl status tomcat
```

Pode ser necessário dar um reboot no servidor:

```
reboot
```

Após dar o reboot no servidor, tente acessar o **Biblivre5**:  Abra seu navegador de internet e faça:

```
http://IP_seu_server_Biblivre:8080/Biblivre4
```

Obs: **sim** o nome ao final necessita ser **Biblivre4** . Se tudo deu certo até este momento, é provável que tenha carregado o serviço do Biblivre no seu navegador. 
