Install Pentaho-cerver CE 9.0 + PostgreSQL 10 + CentOS 8

INSTALL POSTGRES
https://wiki.postgresql.org/wiki/YUM_Installation

1) find 
 /etc/yum.repos.d/CentOS-Base.repo
 vi /etc/yum.repos.d/CentOS-Base.repo
 remove 
 add 
 exclude=postgresql*

2) add rpm  
 yum install https://download.postgresql.org/pub/repos/yum/9.6/redhat/rhel-7-x86_64/pgdg-centos96-9.6-3.noarch.rpm

 after u can use 
 yum list postgresql*
 yum install postgresql96-server

3) 

/usr/pgsql-9.6/bin/postgresql96-setup initdb

 systemctl enable postgresql-9.6.service
 systemctl start postgresql-9.6.service

4) add to user (root) path
   add to PATH .bash_profile: ":/usr/pgsql-9.6/bin"
   PATH=$PATH:$HOME/bin:/usr/pgsql-9.6/bin 


4.1) 
   configure pg_hba to make easy connect
   vi /var/lib/pgsql/9.6/data/pg_hba.conf

    # "local" is for Unix domain socket connections only
	local   all             all                                     peer
	# IPv4 local connections:
	host    all             all             127.0.0.1/32            ident
	# IPv6 local connections:
	host    all             all             ::1/128                 ident

    to change :
	to 
	# "local" is for Unix domain socket connections only
	local   all             all                                     trust
	# IPv4 local connections:
	host    all             all             127.0.0.1/32            trust
	# IPv6 local connections:
	host    all             all             ::1/128                 trust

5) for root access u can add user root to db 
su - postgres -c psql 
or
su postgres
psql  

add role root;
alter role root with SUPERUSER;
alter role root with login;


6) download last version of pentaho ba (if no wget install via yum: yam install wget)
  wget https://downloads.sourceforge.net/project/pentaho/Business%20Intelligence%20Server/7.0/pentaho-server-ce-7.0.0.0-25.zip?r=http%3A%2F%2Fcommunity.pentaho.com%2F&ts=1490957327&use_mirror=nchc

7) move to opt and unzip

   mkdir /opt/pentaho
   cp pentaho-server-ce-7.0.0.0-25.zip /opt/pentaho
   unzip pentaho-server-ce-7.0.0.0-25.zip (install unzip if no via yam yam install unzip)

8) create repository for pentaho in posgres
   psql -U postgres -a -f create_repository_postgresql.sql 
   psql -U postgres -a -f create_quartz_postgresql.sql
   psql -U postgres -a -f create_jcr_postgresql.sql

9) configuring BA
   https://help.pentaho.com/Documentation/5.3/0F0/0K0/040/0A0   
   http://blog.endpoint.com/2013/11/install-pentaho-bi-server-48-community.html

   1. Set Up Quartz on PostgreSQL BA Repository Database
   vi /opt/pentaho/pentaho-server/pentaho-solutions/system/quartz/quartz.properties

   check block #_replace_jobstore_properties:  org.quartz.jobStore.driverDelegateClass = org.quartz.impl.jdbcjobstore.PostgreSQLDelegate

   2. Set Hibernate Settings for PostgreSQL
   vi /opt/pentaho/pentaho-server/pentaho-solutions/system/hibernate/hibernate-settings.xml

   CHANGE TO 
   <config-file>system/hibernate/postgresql.hibernate.cfg.xml</config-file>

   3. Modify Jackrabbit Repository Information for PostgreSQL
   vi /opt/pentaho/pentaho-server/pentaho-solutions/system/jackrabbit/repository.xml

   change FileSystem
   <FileSystem class="org.apache.jackrabbit.core.fs.db.DbFileSystem">
    <param name="driver" value="com.mysql.jdbc.Driver"/>
    <param name="url" value="jdbc:mysql://localhost:3306/jackrabbit"/>
    <param name="user" value="jcr_user"/>
    <param name="password" value="password"/>
    <param name="schema" value="mysql"/>
    <param name="schemaObjectPrefix" value="fs_repos_"/>
  </FileSystem>
  TO
  <param name="driver" value="com.mysql.jdbc.Driver"/>
    <param name="url" value="jdbc:mysql://localhost:3306/jackrabbit"/>

  SAME with DataStore
  <DataStore class="org.apache.jackrabbit.core.data.db.DbDataStore">
    <param name="url" value="jdbc:postgresql://localhost:5432/jackrabbit"/>
    <param name="driver" value="com.postgresql.jdbc.Driver"/>
  </DataStore>

  SAME with Workspaces
    <FileSystem class="org.apache.jackrabbit.core.fs.db.DbFileSystem">
      <param name="driver" value="com.postgresql.jdbc.Driver"/>
      <param name="url" value="jdbc:postgresql://localhost:5432/jackrabbit"/>

  PersistenceManager 	
	nothing change

  SAME with Versioning 
       <FileSystem class="org.apache.jackrabbit.core.fs.db.DbFileSystem">
      <param name="driver" value="com.postgresql.jdbc.Driver"/>
      <param name="url" value="jdbc:postgresql://localhost:5432/jackrabbit"/>

10) Configurenig BA 2
	find 
	biserver-ce/tomcat/webapps/pentaho/META-INF/context.xml

   1. vi /opt/pentaho/pentaho-server/tomcat/webapps/pentaho/META-INF/context.xml

   change to 

	<?xml version="1.0" encoding="UTF-8"?>
	<Context path="/pentaho" docbase="webapps/pentaho/">
			<Resource name="jdbc/Hibernate" auth="Container" type="javax.sql.DataSource"
					factory="org.apache.commons.dbcp.BasicDataSourceFactory" maxTotal="20" maxIdle="5"
					maxWaitMillis="10000" username="hibuser" password="password"
					driverClassName="org.postgresql.jdbcDriver" url="jdbc:postgresql://localhost:5432/hibernate"
					validationQuery="select 1" />

			<Resource name="jdbc/Quartz" auth="Container" type="javax.sql.DataSource"
					factory="org.apache.commons.dbcp.BasicDataSourceFactory" maxTotal="20" maxIdle="5"
					maxWaitMillis="10000" username="pentaho_user" password="password"
					driverClassName="org.postgresql.jdbcDriver" url="jdbc:postgresql://localhost:5432/quartz"
					validationQuery="select 1"/>

	</Context>

	2. vi /opt/pentaho/pentaho-server/pentaho-solutions/system/applicationContext-spring-security-hibernate.properties

	change to PG

	jdbc.driver=org.postgresql.jdbcDriver
	jdbc.url=jdbc:postgresql://localhost:5432/hibernate
	jdbc.username=hibuser
	jdbc.password=password
	hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect

	3. vi /opt/pentaho/pentaho-server/pentaho-solutions/system/simple-jndi/jdbc.properties 

	change provider and port

	SampleData/url=jdbc:postgresql://localhost:5432/sampledata
	SampleData/user=pentaho_user
	SampleData/password=password
	Hibernate/type=javax.sql.DataSource
	Hibernate/driver=org.postgresql.jdbcDriver
	Hibernate/url=jdbc:postgresql://localhost:5432/hibernate
	Hibernate/user=hibuser
	Hibernate/password=password
	Quartz/type=javax.sql.DataSource
	Quartz/driver=org.postgresql.jdbcDriver
	Quartz/url=jdbc:postgresql://localhost:5432/quartz
	Quartz/user=pentaho_user
	Quartz/password=password

	4. configuring tomcat port and solution
	vi /opt/pentaho/pentaho-server/tomcat/webapps/pentaho/WEB-INF/web.xml

	set  solution-path

	  <context-param>
		<param-name>solution-path</param-name>
		<param-value>/opt/pentaho/pentaho-server/pentaho-solutions</param-value>
	  </context-param>

11)
	start stop pentaho 
	biserver-ce$ ./start-pentaho.sh 
	biserver-ce$ ./stop-pentaho.sh 

	Trouble shooting 

	biserver-ce$ tail -f tomcat/logs/catalina.out
	biserver-ce$ tail -f tomcat/logs/pentaho.out	  

12) disable firewall to connect from net

	systemctl stop firewalld
	systemctl mask firewalld 
