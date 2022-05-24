# oracle-odbc-qt
oracle-odbc-qt



# Подключение к БД Oracle через ODBC + код на Qt

1) Install pre-requisites (if they aren't already installed).  
*sudo apt-get install build-essential libaio1*  //optional...  


2) Install ODBC Driver Manager (unixODBC).  

### Install packages  
*sudo apt-get install unixodbc unixodbc-dev*   //sudo yum install unixODBC   

### Verify unixODBC installation  
*/usr/bin/odbcinst -j*   

# Expected output:  
unixODBC 2.3.4  
DRIVERS............: /etc/odbcinst.ini  
SYSTEM DATA SOURCES: /etc/odbc.ini   
FILE DATA SOURCES..: /etc/ODBCDataSources  
USER DATA SOURCES..: /home/<logged-in-user>/.odbc.  
SQLULEN Size.......: 8  
SQLLEN Size........: 8  
SQLSETPOSIROW Size.: 8  

3) Install Oracle ODBC driver.  

### Download files below from 
### https://www.oracle.com/technetwork/topics/linuxx86-64soft-092277.html  
instantclient-basic-linux.x64-12.2.0.1.0.zip   //11 версия  
instantclient-odbc-linux.x64-12.2.0.1.0-2.zip  //11   

### Unzip files to /opt/oracle  
*sudo unzip instantclient-basic-linux.x64-12.2.0.1.0.zip -d /opt/oracle*  
*sudo unzip instantclient-odbc-linux.x64-12.2.0.1.0-2.zip -d /opt/oracle*  
4) Create tnsnames.ora file and add your database connection to it.  
  
### File: /opt/oracle/instantclient_12_2/network/admin/tnsnames.ora //здесь указываем данные для входа наши 
oradbconn =  
(  
  DESCRIPTION =  
  (  
    ADDRESS_LIST =  
      (ADDRESS =  
        (PROTOCOL = TCP)  
        (HOST = oradbserver.acme.com)  
        (PORT = 1521)  
       )  
  )  
  (  
    CONNECT_DATA = (SERVICE_NAME = oradb.acme.com)  
  )  
)  

5) Run odbc_update_ini.sh, which creates/updates the unixODBC configuration needed to register the Oracle ODBC driver with unixODBC and partially configure an Oracle ODBC data source.  
  
*cd /opt/oracle/instantclient_12_2*  
*sudo ./odbc_update_ini.sh /*  


# This error can be ignored:  
# *** ODBCINI environment variable not set,defaulting it to HOME directory!  
Expected contents of unixODBC config files after running odbc_update_ini.sh:  

### /etc/odbcinst.ini (Tells unixODBC where to find Oracle ODBC driver)//11 верс  
[Oracle 12c ODBC driver]  
Description     = Oracle ODBC driver for Oracle 12c  
Driver          = /opt/oracle/instantclient_12_2/libsqora.so.12.1  
Setup           =  
FileUsage       =  
CPTimeout       =  
CPReuse         =   

### ~/.odbc.ini (Partially-configured Oracle ODBC Data Source)  
[OracleODBC-12c]  
Application Attributes = T  
Attributes = W  
BatchAutocommitMode = IfAllSuccessful  
BindAsFLOAT = F  
.  
.  
.  
6) "Chown" ~/.odbc.ini to the uid/gid of the currently-logged in user. This file is initially created as root:root. If the ownership is not changed, database connections through the ODBC driver may fail.  
  
*sudo chown $(id -u):$(id -g) ~/.odbc.ini*  
7) Complete the data source configuration by adding/updating the ~/odbc.ini parameters shown below.  
  
### ~/.odbc.ini  
ServerName = oradbconn    ### Should reference the connection in the tnsnames.ora file //из tnsnames.ora  
UserID = oradb_user       ### User name for your Oracle database connection     //логин  
Password = oradb_password ### Password for username above						//пароль  
9) Update .bash_profile with Oracle environment variables and source the file.  
  
### ~/.bash_profile  
export TNS_ADMIN=/opt/oracle/instantclient_12_2/network/admin  
export LD_LIBRARY_PATH=/opt/oracle/instantclient_12_2  
// нужно еще добавить export ORACLE_HOME=/opt/oracle/instantclient_12_2  
  
### Source the file  
*. ~/.bash_profile*  
10) Verify connection to Oracle ODBC data source.  
  
*isql -v OracleODBC-12c* //здесь соответственно имя из odbc.ini  
//если будет ошибка: не хватает бибилиотек, нужно смотреть через ldd все и указывать.  
Expected output:  
 ``` 
+---------------------------------------+  
| Connected!                            |  
|                                       |  
| sql-statement                         |  
| help [tablename]                      |  
| quit                                  |  
|                                       |  
+---------------------------------------+  
SQL> 
 ```
  
11) Create program to test ODBC connectivity to Oracle.  
  
//Qt-ый проект 
все в main, сборка:  
*make cleen  
qmake-qt5  
make   
make install*  
Запуск:   
*./testconsole  
через gdb:  
gdb ./testconsole  
run*  
  

Исходники ниже из инструкции.   
  
```  
your-project.pro:   
.  
.  
QT += sql  ### Add this to make SQL libraries available   
main.cpp:  
 
#include <iostream>  
#include <QCoreApplication>  
#include <QDebug>  
#include <QSqlDatabase>  
#include <QSqlQuery>  
#include <QSqlError>  
  
int main( int argc, char *argv[] )  
{  
    QCoreApplication a(argc, argv);  
  
    // "OracleODBC-12c" is the data source configured in ~/.odbc.ini  
    QSqlDatabase db = QSqlDatabase::addDatabase( "QODBC3", "OracleODBC-12c" );  
    if(db.open())  
        qDebug() << "Opened db connection!";  
    else  
        qDebug() << db.lastError().text();  
  
    QSqlQuery query(db);  
  
    // Example query selects a few table names from the system catalog  
    query.exec("SELECT table_name FROM all_tables WHERE owner = 'SYS' and ROWNUM <= 3");  
  
    while (query.next()) {  
      QString table_name = query.value(0).toString();  
      qDebug() << table_name;  
    }  
  
    return a.exec();  
}``` 
      
      
Expected output (table names may vary):  
  
Opened db connection!  
"DUAL"  
"SYSTEM_PRIVILEGE_MAP" 
"TABLE_PRIVILEGE_MAP"  
Above steps were verified on OS / Qt version below:  


 
