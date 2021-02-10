# Configuring Heterogeneous Data Replication Between MySQL and ADW using GoldenGate 

### Peter Song - Cloud Engineer

## Introduction

This post will go over how to set up a unidirectional pipeline using Oracle GoldenGate Classic Architecture on the source and Microservices Architecture on the target. Goldengate is a software package for real-time data integration and replication, and is used for solutions that require high availabiltiy, real-time data integration, transaction change data capture, data replication, and transformations. You can use GoldenGate to keep your database online during migrations.

## Environment Details

- The source database is MySQL Client Version 8.0.18
- The Target Database is Autonomous Data Warehouse 
- The source GoldenGate is GG Classic Edition for MySQL 19.1.0.0.201013 (Marketplace)
- The target GoldenGate is GG Microservices Edition for Oracle 19.1.0.0.201013 (Marketplace)

This blog will cover end to end, starting from the creation of the ADW Instance

1. #### Create the ADW Instance

   1. To create an ADW instance you need an OCI account. 

   2. Click the hamburger icon and go to Autonomous Data Warehouse

   3. Select Data Warehouse and Shared infrastructure.

   ![](https://objectstorage.us-ashburn-1.oraclecloud.com/n/orasenatdoracledigital01/b/bucket-20201209-1145/o/create_ADWScreen%20Shot%202021-01-21%20at%205.02.58%20PM.png)

2. #### Create mySQL Instance on OCI Using OCI Marketplace

   1. Create the MySQL Instance using OCI Marketplace ![](https://objectstorage.us-ashburn-1.oraclecloud.com/n/orasenatdoracledigital01/b/bucket-20201209-1145/o/mySQL%20marketplaceScreen%20Shot%202021-02-10%20at%202.14.30%20PM.png)
   2. Select the Default version. In this case it is 8.0.23. On the next screen, select an existing VCN & subnet or create one yourself. You may upload your own SSH key or let Oracle generate one for you.

   1. Create 2 GoldenGate Instances

      1. Choose GoldenGate for Non-Oracle (classic) ![](https://objectstorage.us-ashburn-1.oraclecloud.com/n/orasenatdoracledigital01/b/bucket-20201209-1145/o/GoldenGate%20for%20non%20OracleScreen%20Shot%202021-01-25%20at%2010.11.24%20AM.png)
      2. choose the correct version (in environment details above)
      3. Choose Goldengate for Oracle (microservices) ![](https://objectstorage.us-ashburn-1.oraclecloud.com/n/orasenatdoracledigital01/b/bucket-20201209-1145/o/GoldenGate%20for%20OracleScreen%20Shot%202021-01-25%20at%2010.18.40%20AM.png)
      4. Choose the correct version (in environment details above)

   2. Connect mySQL [SOURCE] to non-Oracle GG

   3. [Connect](https://docs.oracle.com/en-us/iaas/mysql-database/doc/connecting-db-system.html#GUID-591AFFED-1181-4DBC-837C-55BD3E726E6A) to MySQL DB

      1. Create a compute instance
      2. Add port 3306/TCP to ingress rule for VCN
      3. Connect to the compute instance remotely

      ```
      ssh -i <private-key-filename> opc@<gg_public-id-address>
      ```

      4. Run the commands to [download](https://docs.oracle.com/en-us/iaas/mysql-database/doc/connecting-db-system.html#GUID-591AFFED-1181-4DBC-837C-55BD3E726E6A) mySQL and connect to it

      5. Create GoldenGate User

         1. login as root and enter password when prompted. If you didn't create a password during the creation of the database, you can get the temporary password with the following comand

            ```
            sudo grep ‘temporary password’ /var/log/mysqld.log
            ```

             Then log in with the command below using the temporary password

            ```
            mysql -u root -p
            ALTER USER 'root'@'localhost' IDENTIFIED BY '<MY_NEW_PASS>'
            ```

         2. Create the database and table to use to pull data from

         ``` 
         create database GGTEST;
         use GGTEST;
         <create table here>
         ```

         3. Create the User

         ``` 
         create user 'ggsrc'@'<GG_IP>' identified by '<PASSWORD'
         ```

         4. Grant privileges

         ```
         grant all privileges on *.* to 'ggsrc'@'<GG_IP>';
         ```

      6. Login to goldengate (non-oracle) instance.
         1. ```
            ssh -i <private-key> opc@<goldengateIP>
            ```

         2. Start the Extract

         ``` 
         ./ggsci
         ```

         3. Edit the Extract param file

         ``` 
         EXTRACT extmysql
         sourcedb GGTEST@<mySQLIPADDRESS>, USERID ggsrc, password <PASSWORD>
         TRANLOGOPTIONS ALTLOGDEST REMOTE
         EXTTRAIL ./dirdat/et
         TABLE GGTEST.<TABLENAME>;
         ```

         4. Add the extract

         ```
         ADD extract extmysql, TRANLOG, BEGIN NOW
         ```

         5. Add the extract trail

         ```
         ADD EXTTRAIL ./dirdat/et, EXTRACT extmysql
         ```

         6. Login to the mySQL database

         ```
         dblogin sourcedb GGTEST@<db_ip_address>:3306, userid ggsrc, password <password>
         ```

         7. Start manager

         ```
         start mgr
         ```

         8.

         ```
         START EXTRACT extmysql
         ```

         ### NOTE:

         If you are having trouble connecting to the root user, you can get a temporary password from within the mySQL instance:

         ```
         sudo grep 'temporary password' /var/log/mysqld.log
         *** OR MAKE A NEW PASSWORD WITH***
         mysql -u root -p
         ALTER USER 'root'@'localhost' IDENTIFIED BY 'MyNewPass4!';
         ```

      7. Create the Pump Extract. This is what will send the extract file to the  target GoldenGate manager

         1. Create the pump extract parameter

         ```
         edit param PMPSQL
         **START OF PMPSQL FILE NOW ***
         EXTRACT PMPSQL
         RMTHOST <GoldenGate for Oracle IP addres>, PORT <receiver_port>
         RMTTRAIL aa
         PASSTHRU
         TABLE GGTEST.<TABLE_NAME>;
         ```

         2. Then add the extract 

         ```
         add extract <pump_extract_name>, exttrailsource ./dirdat/et
         ```

         3. Add the rmttrail

         ```
         Add rmttrail aa, extract PMPSQL
         ```

      ### POSSIBLE ERRORS

      Error 1: process abends, 'tcp/ip' error no route to host.

      This error is caused usually by firewall issues with the target side

      1. Check which ports are open

         1. go into your target goldengate instance and check which ports are open

         ```
         sudo firewall-cmd --list-ports
         ```

         2. If you see that the receiver port you specified isn't there add it with these commands:

         ```
         sudo firewall-cmd --permanent --add-port=<receiver port number>/tcp
         sudo firewall-cmd --reload
         ```

      Error 2: OGG-01224 command 'start server' is not supported by OGG admin Server. This error is becuase you did not list the receiver server as the port and the admin server. This is a microservice architecture, so you must specify the receiver server port number. Edit the param file and specify the RECEIVER PORT.

      

   ## Configuring the Microservices Target Side

   1. Logging into the target ADW 

      1. Download the wallet file from the OCI console if you have not already done so. Click DB Connection & click download wallet.

      ![](https://objectstorage.us-ashburn-1.oraclecloud.com/n/orasenatdoracledigital01/b/bucket-20201209-1145/o/ADW_CONNECITONScreen%20Shot%202021-01-25%20at%202.32.57%20PM.png)

      2. Open SQL Developer and connect to the Database with the wallet file and database admin credentials. [you can get this info here too](https://docs.oracle.com/en/middleware/goldengate/core/19.1/oggmp/oracle-goldengate-microservices-oracle.html#GUID-50B52EC1-6633-432F-BE90-7F3DD5A8C758)

      3. Then issue the following commands to select the ggadmin user. When using goldengate with ADW a goldengate user is created for you.

         1. ```
            SQL> select * from dba_users order by username;
            SQL> alter user ggadmin identified by <password> account unlock;
            ```

         2. Check whether the parameter enable_goldengate_replication is set to true. If not, then modify the parameter

            ```
            SQL> select * from v$parameter where name = 'enable_goldengate_replication';
            SQL> alter system set enable_goldengate_replication = true scope = both;
            ```

   2. Create the target schema

      1. Create the new application user/schema. This should be different from the goldengate user. It is best practice to keep these users separate. This user/schema stores the target objects for replication.

      ```
      SQL> create user appadmin identified by <*****>
      SQL> grant create session, resource, create view, create table to appadmin;
      SQL> alter user appadmin quota unlimited on data;
      ```

      2. Connect to the ADW as user/schema and create your application table with the same properties as the source table you want to replicate

   3. Move Client Credentials to Oracle GoldenGate compute node

      1. Connect to the Goldengate node using ssh and opc user credentials

      ```
      ssh -i <private_key> opc@<public_ip_address>
      ```

      2. Create a staging directory and grant the essential permissions and then exit the session.

      ```
      $ mkdir stage
      $ exit
      ```

      3. Copy the credentials zip file to the compute node (outside the goldengate node)

      ```
      $ scp -i <private_key_file> ./<credential_file> opc@<public_ip_address>:~/stage
      ```

      4. ssh back into the goldengate node

      ```
      ssh -i <private_key> opc@<public_ip_address>
      ```

      5. Verify whether the credentials zip file is in the stage location

      ```
      $ cd ~/stage
      $ ls -ltr
      ```

   4. Configure the Goldengate compute node with Autonomous Client Credentials

      1. In the compute node, unzip the client credentials file

      ```
      unzip ./<credential_file> -d ./client_credentials
      ```

      2. Copy the sqlnet.ora and tnsnames.ora files to the location of your TNS_ADMIN

      ```
      $ cd ~/stage/client_credentials
      $ cp ./sqlnet.ora /u02/deployments/<deployment>/etc
      $ cp ./tnsnames.ora /u02/deployments/<deployment>/etc
      ```

      3. Edit the sqlnet.ora file and replace the directory parameter with the locaiton of the information pointing to the location where the client credentials were unzipped

      ```
      $ cd /u02/deployments/<deployment>/etc
      $ vi ./sqlnet.ora
      ```

      4. Within the sqlnet.ora file do the following

      ```
      $ export ORACLE_HOME=/u01/app/client/<oracle version>
      $ export TNS_ADMIN=/u02/deployments/<deployment>/etc
      ```

      5. Test the connection to the ADW by connnecting to one of the entries 

      ```
      $ cd $ORACLE_HOME/bin
      $ ./sqlplus appadmin/**********@orcladw_low
      ```

   5. Configure the Microservice side for replication
      1. Log in to the Service Manager using password for oggadmin (in the ogg-credentials.json file in the goldengate node). There are known issues with Google Chrome. It works with Mozilla. You can access the service manager with **https://<public_ip_address>**

      2. From the service manager main page, select the hyperlink for the port number associated with the administration service

      3. Open the context menu in the top left corner of the Overview page.

      4. From the context menu, select Configuration.

      5. From the Database tab, click the plus ( + ) icon, to add a new credential.

      6. Provide the following information and click Submit

         Credential Domain: [Defaults to OracleGoldenGate]
         Credential Alias: [Name of the Alias]
         User ID: ggadmin@<adw_tnsnames_reference>
         Pasword: [Password for ggadmin]
         Verify Password: [Password for ggadmin]

      7. Test the connection to the Autonomous Data Warehouse by clicking the Log in database icon after the credential has been added.

   6. Create the checkpoint table on the target side

      1. First, log in to the database from the target microservices side

      ![](https://objectstorage.us-ashburn-1.oraclecloud.com/n/orasenatdoracledigital01/b/bucket-20201209-1145/o/db_connnection_to_microserviceScreen%20Shot%202021-01-25%20at%204.31.52%20PM.png)

      2. Add the checkpoint table ![](https://objectstorage.us-ashburn-1.oraclecloud.com/n/orasenatdoracledigital01/b/bucket-20201209-1145/o/add_checkpoint_tableScreen%20Shot%202021-01-25%20at%204.37.17%20PM.png)
      3. Add the transaction information in the form of schemaname.table_name![](https://objectstorage.us-ashburn-1.oraclecloud.com/n/orasenatdoracledigital01/b/bucket-20201209-1145/o/add_transaction_tableScreen%20Shot%202021-01-25%20at%204.41.01%20PM.png)
      4. Then go back to the service manager and to the administration server and click the + button in the replicat page ![](https://objectstorage.us-ashburn-1.oraclecloud.com/n/orasenatdoracledigital01/b/bucket-20201209-1145/o/add%20replicatScreen%20Shot%202021-02-05%20at%2010.54.54%20AM.png)
      
      5. Once you click the replicat page, choose non-integrated replicat. Click Next. Fill it out like so. Make sure the trail name is the same trail you specify from the pump extract on the source side.![ ](https://objectstorage.us-ashburn-1.oraclecloud.com/n/orasenatdoracledigital01/b/bucket-20201209-1145/o/RepScreen%20Shot%202021-02-05%20at%2011.49.51%20AM.png)
      
      6. Next create the parameter file.
      
      ```
      replicat rep2
      useridalias ggadmin domain OracleGoldenGate
      MAP <DB_NAME>.<TABLE_NAME>, TARGET <DB_NAME>.<TABLE_NAME>;
      ```
      
      7. You are done setting up the target side. The only thing to do now is make sure it works! Insert something on the source mySQL database and see if it shows up on the target ADW. Another easy way to check is to go to statistics and see if the DML operation shows up. 
      
      
      
      
      
      
      
      
      
      

