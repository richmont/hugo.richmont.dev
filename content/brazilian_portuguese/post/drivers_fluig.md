+++
author = "Richelmy Monteiro"
title = "Instalação de drivers do MariaDB e PostgreSQL no FLUIG"
date = "2026-05-03"
description = "Configuração de datasources alternativos para integração"

+++
FLUIG utiliza WildFly como servidor de aplicação JBoss, é nele que é instalado o datasource.  
FLUIG já possui por padrão o suporte a MySQL, mas a conexão para bancos MariaDB é recomendado utilizar o driver JDBC próprio.  
Adicionar novos datasources permite integrar a plataforma da TOTVS a aplicações e serviços adicionais, expandindo as possibilidades do que você pode construir com FLUIG.  
As instruções abaixo não contam com suporte oficial da TOTVS e devem ser realizadas por sua conta em risco, em caso de dúvidas entre em contato com um consultor FLUIG de sua confiança.  

# Requisitos
Instalação baseada no princípio que você utiliza o FLUIG 1.8 em um servidor Linux, pode ser adaptado facilmente para instalações do FLUIG 2.0 e em servidores Windows, mas não será abordado neste contexto.  
Você deve saber utilizar linha de comandos, mas procedimento pode ser realizado via interface gráfica também.  

# Baixando o driver e instalando o módulo
Entre no diretório de módulos do Wildfly, como o FLUIG está instalado em *```/opt/fluig```* ( também conhecido como a variável de ambiente ```WILDFLY_HOME```), o caminho
completo é este:  
```bash
/opt/fluig/
```
O diretório de módulos, onde ficam localizados os pacotes de biblioteca java, será este aqui:  
```bash
/opt/fluig/appserver/modules/
```
Nele você encontrará estruturas de pastas referentes ao endereço de cada biblioteca.
## Módulo do MariaDB
Crie a estrutura de pastas a seguir:
```bash
/opt/fluig/appserver/modules/org/mariadb/jdbc/main
```
Precisa baixar o conector e criar o arquivo modules.xml.  

O driver está disponível para download no site do próprio MariaDB:  
https://mariadb.com/docs/connectors/mariadb-connector-j/about-mariadb-connector-j  
A versão mais atual hoje é a 3.5.8  
Prefiro pegar a URL direta do pacote e baixar no sistema do que copiar depois, então:  
```bash
wget https://dlm.mariadb.com/4639714/Connectors/java/connector-java-3.5.8/mariadb-java-client-3.5.8.jar -O /opt/fluig/appserver/modules/org/mariadb/jdbc/main
```
Crie o arquivo de módulo:  
```bash
vi /opt/fluig/appserver/modules/org/mariadb/jdbc/main/module.xml
```
Conteúdo do arquivo:  
```xml
<module xmlns="urn:jboss:module:1.3" name="org.mariadb">

    <resources>
        <resource-root path="mariadb-java-client-3.5.8.jar"/>
    </resources>
    <dependencies>
        <module name="javax.api"/>
        <module name="javax.transaction.api"/>
    </dependencies>
</module>
```
O que o arquivo diz?  
Que o módulo chamado ```org.mariadb``` chamará o recurso ```mariadb-java-client-3.5.8.jar``` que possui dependência nos módulos ```javax.api``` e ```javax.transaction.api```.  


# Módulo do PostgreSQL
Caminho do módulo, crie os diretórios nesta estrutura: 
```bash
/opt/fluig/appserver/modules/org/postgresql/main
```

Driver disponível no site:  
https://jdbc.postgresql.org/

A versão mais atual e compatível com Java 11 e superiores é a 42.7.11 no momento.
```bash
wget https://jdbc.postgresql.org/download/postgresql-42.7.11.jar -O /opt/fluig/appserver/modules/org/postgresql/main
```
Crie o arquivo de módulo:  
```bash
vi /opt/fluig/appserver/modules/org/postgresql/main/module.xml
```

Conteúdo:  
```xml
<module xmlns="urn:jboss:module:1.3" name="org.postgresql">
    <resources>
        <resource-root path="postgresql-42.7.11.jar"/>
    </resources>
    <dependencies>
        <module name="javax.api"/>
        <module name="javax.transaction.api"/>
    </dependencies>
</module>
```
# Criando datasources
Se você utiliza o FLUIG versão 1.8, precisará editar o arquivo ```domain.xml```, caso use 2.0, será o ```standalone.xml```.  
Procure pelas tags ```<datasources>```, dentro delas inclua as tags a seguir, conforme sua conexão com o banco de dados:  

## MariaDB
Datasource simples, adequado para foco em performance e quando se acessa apenas um recurso. É o modo de conexão padrão. Sim, a classe que valida a conexão é a mesma do MySQL, não tem problema.  
```xml
    <datasource enabled="true" jndi-name="java:/jdbc/MariadbDS" jta="true" pool-name="MariadbDS" statistics-enabled="true" use-java-context="true">
        <connection-url>jdbc:mariadb://127.0.0.1:3306/banco</connection-url>
        <driver>mariadbDriver</driver>
        <pool>
            <min-pool-size>5</min-pool-size>
            <max-pool-size>65</max-pool-size>
        </pool>
        <security>
            <user-name>usuario</user-name>
            <password>senha</password>
        </security>
        <validation>
            <valid-connection-checker class-name="org.jboss.jca.adapters.jdbc.extensions.mysql.MySQLValidConnectionChecker" />
            <validate-on-match>true</validate-on-match>
            <background-validation>false</background-validation>
        </validation>
        <timeout>
            <blocking-timeout-millis>30000</blocking-timeout-millis>
        </timeout>
        <statement>
            <share-prepared-statements>false</share-prepared-statements>
        </statement>
    </datasource>
```
Datasource XA, utiliza recursos de integridade do JDBC, apropriado quando se acessam múltiplos recursos (bancos de dados mensagens JMS, etc) e é preciso que cada transação nesses recursos seja atômica, caso um falhe, todos falham. Não vou entrar em detalhes sobre a distinção, mas é bom saber como configurar as duas formas.  
```xml
    <xa-datasource jndi-name="java:/jdbc/MariaDBXADS" pool-name="MariaDBXADS" tracking="true" statistics-enabled="true">
        <xa-datasource-property name="user">usuario</xa-datasource-property>
        <xa-datasource-property name="password">senha</xa-datasource-property>
        <xa-datasource-property name="url">jdbc:mariadb://127.0.0.1:3306/banco</xa-datasource-property>
        <driver>mariadbDriver</driver>
        <validation>
            <valid-connection-checker class-name="org.jboss.jca.adapters.jdbc.extensions.mysql.MySQLValidConnectionChecker" />
            <check-valid-connection-sql>select 1;</check-valid-connection-sql>
            <validate-on-match>true</validate-on-match>
            <background-validation>true</background-validation>
        </validation>
        <statement>
            <share-prepared-statements>true</share-prepared-statements>
        </statement>
    </xa-datasource>
```
É possível configurar o driver dentro de cada datasource, mas é mais fácil usar um nodo próprio que é usado por ambos. Ainda dentro da tag ```<datasources>``` insira a de driver.
```xml
    <driver name="mariadbDriver" module="org.mariadb">
    <driver-class>org.mariadb.jdbc.Driver</driver-class>
    <xa-datasource-class>org.mariadb.jdbc.MariaDbPoolDataSource</xa-datasource-class>
    <!--<datasource-class>org.mariadb.jdbc.MariaDbPoolDataSource</datasource-class>-->
    </driver>
```
A tag com datasource-class está comentada devido a eu ter tido dificuldades de conectar o datasource simples a ela, já a xa-datasource-class é necessária para conexão do XA.  
O nome do driver deve ser o mesmo do datasource, nesse caso, mariadbDriver.

## PostgreSQL

Datasource simples
```xml
    <datasource jndi-name="java:/jdbc/PostgreSQLDS" pool-name="PostgreSQLDS" statistics-enabled="true" jta="true" use-java-context="true">
        <connection-property name="serverName">127.0.0.1</connection-property>
        <connection-property name="databaseName">banco</connection-property>
        <connection-property name="portNumber">5432</connection-property>
        <connection-property name="user">usuario</connection-property>
        <connection-property name="password">senha</connection-property>
        <driver>postgresqlDriver</driver>
        <validation>
        <valid-connection-checker class-name="org.jboss.jca.adapters.jdbc.extensions.postgres.PostgreSQLValidConnectionChecker"/>
        <check-valid-connection-sql>select 1</check-valid-connection-sql>
        <validate-on-match>true</validate-on-match>
        <background-validation>true</background-validation>
        </validation>
    </datasource>
```

Datasource XA
```xml
    <xa-datasource jndi-name="java:/jdbc/PostgreSQLXADS" pool-name="PostgreSQLXADS"
    statistics-enabled="true">
        <xa-datasource-property name="serverName">127.0.0.1</xa-datasource-property>
        <xa-datasource-property name="databaseName">banco</xa-datasource-property>
        <xa-datasource-property name="portNumber">5432</xa-datasource-property>
        <xa-datasource-property name="user">usuario</xa-datasource-property>
        <xa-datasource-property name="password">senha</xa-datasource-property>
        <driver>postgresqlDriver</driver>
        <validation>
            <valid-connection-checker class-name="org.jboss.jca.adapters.jdbc.extensions.postgres.PostgreSQLValidConnectionChecker"/>
            <check-valid-connection-sql>select 1</check-valid-connection-sql>
            <validate-on-match>true</validate-on-match>
            <background-validation>true</background-validation>
        </validation>
    </xa-datasource>
```
E a configuração do driver:  
```xml
<driver name="postgresqlDriver" module="org.postgresql">
    <driver-class>org.postgresql.Driver</driver-class>
    <xa-datasource-class>org.postgresql.xa.PGXADataSource</xa-datasource-class>
    <datasource-class>org.postgresql.ds.PGPoolingDataSource</datasource-class>
</driver>
```

Reinicie o serviço do FLUIG e teste com um dataset como este:  
```javascript
function createDataset(fields, constraints, sortFields) { 
	var newDataset = DatasetBuilder.newDataset();
	/*
	if(fields == null){
		newDataset.addColumn("INFO");
		newDataset.addRow(new Array("Informe uma query!"));
		return newDataset;
	}
	var minhaQuery = fields[0];
	*/
	var minhaQuery = "select * from usuarios limit 10;"
	log.info("****** start - [ds_sql_mariadb] QUERY: " + minhaQuery);
	var dataSource = "java:/jdbc/MariadbDS";

	var conn = null;
	var stmt = null;
	var rs = null;
	var ic = new javax.naming.InitialContext();
	var ds = ic.lookup(dataSource);
	var created = false;
	try{
		conn = ds.getConnection();
		stmt = conn.createStatement();
		
		// Sets the number of seconds the driver will wait for
		// a statement object to execute to the given number of
		// seconds. If the limit is exceeded, an SQLException
		// is thrown.
		stmt.setQueryTimeout(1800); //30 minutos = 1800 segundos
		rs = stmt.executeQuery(minhaQuery);
		var columnCount = rs.getMetaData().getColumnCount();
		while(rs.next()) {
			if(!created) {
				for(var i=1;i<=columnCount; i++) {
					newDataset.addColumn(rs.getMetaData().getColumnName(i));
				}
				created = true;
			}
			var Arr = new Array();
			for(var i=1;i<=columnCount; i++) {
				var obj = rs.getObject(rs.getMetaData().getColumnName(i));
				if(null!=obj){
					Arr[i-1] = rs.getObject(rs.getMetaData().getColumnName(i)).toString();
				}
				else {
					Arr[i-1] = "null";
				}
			}
			newDataset.addRow(Arr);
		};
	}catch(e){
		newDataset.addRow(new Array(e.message));
		log.error("ERRO==============> " + e.message);
	}finally{
		try{
			if(rs != null) rs.close();
			if(stmt != null) stmt.close();
			if(conn != null) conn.close();
		}catch(er){
			newDataset.addRow(new Array(er));
			log.error("Erro ao fechar as conexões: " + er);
		};
	};
	return newDataset;
};
```