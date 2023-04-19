# Introduction
This repository contains the documentation for installing and configuring a system with [BOCA Online Contest Administrator](https://github.com/cassiopc/boca), database replication for PostgreSQL, and Animeitor Configuration.

# Requirements
The operating system that has been tested in Ubuntu 20.04. Currently, Ubuntu 22.04 is supported too.

# What was used?
At least two Machines/VMs running Ubuntu. One is for the main BOCA and the second one works for the replication of the PostgreSQL database.

# BOCA Installation
The BOCA package must be installed on both machines. The installation process has four commands that must be executed on the bash terminal.

```
sudo apt-get update # Updating packages
sudo add-apt-repository ppa:icpc-latam/maratona-linux # Adding the official package of BOCA
sudo apt-get update # Updating packages, again
sudo apt-get install boca -y # Installing BOCA
```
In this process, you should be prompted to locate the database location. By default, The database is placed on the same machine as the BOCA web. After this, you should be prompted for database password creation. Don't forget this password because you'll need it for future steps. After this, you should be prompted again to put the database password. The installation asks you to create a fake initial contest if this database password is correct.

After this, the installation will be completed and BOCA can be accessed at `http://localhost/boca`.

### Removing `/boca` on the URL
To remove the `/boca` on the URL, you need to configure a file from Apache, that redirects to the BOCA system.
The file is located on `/etc/apache2/sites-enabled/000-boca.conf`. Edit this file with your preferred text editor. Inside this file, you must locate a line that contains `DocumentRoot /var/www/boca/`. Add `src` at the end of this line. The final result should be like this
```
DocumentRoot /var/www/boca/src
systemctl stop apache2
systemctl start apache2
```

## Database configuration
If you want, you can install BOCA on other machines and use them as autojudges. For this, you need to configure a PostgreSQL file, that is located at `/etc/postgresql/12/main/postgresql.conf`. Search on this file for `listen_addresses`. On this line, you must set this `listen_addresses` to allow connection from all locations. The final result should be this: `listen_addresses = '*'`. Save this file and close.

## Allowing multi-login for the BOCA users
On BOCA, there has a bug that denies access from multiple IP addresses, for the users. To fix this bug, you need to replace the `flog.php` file located at `/var/www/boca/src/`. Overwrite this file by the file placed on this repository, named [`flog.php`](./flog.php).

## Installing Auto Judges
There are two possibilities to run an auto-judge system. One of them is the [local installation](#local-auto-judges), on the same machine that runs BOCA's database. Another one is to [install an auto-judge system on another machine](#external-auto-judges).

### Local Auto Judges
For automatically judging exercises, you'll need to install the autojudge. This tool is a robot that will connect to the database and waits for newer runs, that have been sent by the competitors.
To install this, you'll need to run the following commands:
```
boca-createjail # This installation process can be time-consuming
apt-cache show maratona-linguagens
apt-get install maratona-linguagens
```
After this, you can run autojudge by typing `sudo boca-autojudge`. If you prefer, this tool can be run in the background. To make this, nohup can be used. To use this, you can run `sudo nohup boca-autojudge &` on the terminal and hit the enter key.

### External Auto Judges
With this type of setup, you could run a self-judging system using another machine, which could be dedicated to judging contest exercises.

To run it, you will install BOCA on this machine ([Follow previous steps](#boca-installation)). After the installation is completed, you will have to configure a single file, which points the self-trial connection to the main machine. The machine
running the dedicated self-trial need not be the second machine in the replication steps.

Firstly, you will need to run the following commands as a superuser.

1. Locate the `conf.php` file. By default, it can be found at `/var/www/boca/src/private`.
      ```
      sudo su
      cd /var/www/boca/src/private
      ```
2. Open the file using your preferred text editor.
      ```
      nano conf.php
      ```
3. Locate the line containing `$conf["dbhost"]=`. Replace the contents of this line (under `localhost`) with the IP address of the machine running the BOCA database. Insert this IP address under double quotes 
      ```
      $conf["dbhost"]="BOCADatabaseIPAddress";
      ```
4. Verify that the port the database is on, on the main machine, is the default (port 5432). Otherwise, overwrite this information in the file with the current port.

5. Insert the credentials that will be used to connect to this database. Locate the line that contains `$conf["dbpass"]=` and insert the password of the primary database, to connect to him. Insert this password under double quotes.
    ```
    $conf["dbpass"]="SecretPassword";
    ```
6. Insert the credentials that will be used to connect to this database, with privileged permissions. Locate the line that contains `$conf["dbsuperpass"]=` and insert the password of the primary database, to connect to him. Insert this password under double quotes.
    ```
    $conf["dbsuperpass"]="SecretPassword";
    ```

7. By default, BOCA creates a user named `bocauser` and the password for him is the password that you set on the BOCA installation process. If you don't want to use this user, change this on `conf.php` (Locate the line that contains `$conf["dbhost"]="bocauser"` and the line that contains `$conf["dbsuperuser"]="bocauser"` and change this user to the user that you define).

8. After these previous steps, the file should have the pieces of information that you've inserted, and look like this:
    ```
    $conf["dbhost"]="BOCADatabaseIPAddress";
    $conf["dbport"]="5432";

    $conf["dbname"]="bocadb"; // name of the boca database

    $conf["dbuser"]="bocauser"; // unprivileged boca user
    $conf["dbpass"]="SecretPassword";

    $conf["dbsuperuser"]="bocauser"; // privileged boca user
    $conf["dbsuperpass"]="SecretPassword";
    ```
9. Save this file. After this, the connection of auto-judge will be on the main BOCA database.

### Killing autojudge
To kill autojudge that is running in the background, you can get the PID value and terminate this process.
```
ps -aux | grep autoj # Get the information from the process
kill -9 PID_VALUE
```

# Database replication using PostgreSQL
To replicate the database, you must have two machines. The first one will be the primary and the second will be the secondary. The configuration files are located at `/etc/postgresql/12/main/`

## Configuration of the primary machine
### Creating database replication user
First of all, you need to connect to the database, using psql. To this, run the following commands on the terminal:
```
sudo su postgres
psql
```
After this, you'll need to create a database replication user and a database for this user. In our tests, we create a superuser for this. Run the following commands on psql:
```
CREATE USER <replicationUser> WITH LOGIN SUPERUSER PASSWORD '<passwordUser>'; --Remember this password
CREATE DATABASE <replicationUser>;
```
### Editing PostgreSQL configuration files
To this step, you'll need to change two files from PostgreSQL.

#### `pg_hba.conf` file
This file works like a host-based authentication. With this, we can allow connection from other machines, based on users, IP addresses, and the database. For the replication process, you'll need to add the following line at the end of the `pg_hba.conf` file.
```
host replication  <replicationUser>  <IPSecondaryMachine>/32  md5
```
`host` tells that a connection using TCP/IP will be allowed. `replication` tells that a user can be connected for replication. `replicationUser` will be the user that has been created earlier. `IPSecondaryMachine` is the IP address from the secondary machine. Finally, `md5` perform an md5 authentication to verify the user's password. More settings can be accessed on [PostgreSQL documentation](https://www.postgresql.org/docs/current/auth-pg-hba-conf.html).

#### `postgresql.conf` file
This is the main configuration file and the primary source of configuration parameter settings. In this file, you'll set some parameters. Search for these parameters and overwrite the lines with these configurations.

```
wal_level = replica
```
```
wal_keep_segments = 64
```
```
max_wal_senders = 10
```
### Applying changes
After these configuration steps, you'll need to restart the database process. To this, run the following commands:
```
systemctl stop postgresql
systemctl start postgresql
```

## Configuration of the secondary machine
After the configuration of the primary machine, you'll need to configure the secondary machine. After this configuration process, BOCA will still work, but the database will be in read-only mode. In future steps, we show how to revert this.

### Editing PostgreSQL configuration files
To this step, you'll need to change two files from PostgreSQL.

#### `postgresql.conf` file
First of all, you'll need to change the `postgresql.conf` file. Search for the following parameters and overwrite the lines that contain them.
```
wal_keep_segments = 64
```
```
wal_level = replica
```
```
hot_standby = on
```
```
max_wal_senders = 10
```

#### `pg_hba.conf` file
Configure the `pg_hba.conf` file to allow the connection from the primary machine
```
host replication  <replicationUser>  <IPPrimaryMachine>/32  md5
```

### Starting replication process
To start the replication process, the database, located on the primary machine should be copied to the secondary. To this, the database located on the secondary must be erased or reallocated. The following commands should be typed on the terminal, as superuser:

```
systemctl stop postgresql
```
Stops the database process.
```
rm -rfv /var/lib/postgresql/12/main/*
```
Removes the database content. To reallocate this database, run `mv /var/lib/postgresql/12/main/ /var/lib/postgresql/12/main-backup/`, or something like this, on the terminal.
```
pg_basebackup -h <ipMaster> -D /var/lib/postgresql/12/main/ -P -U usuarioReplicacao -R 
```
Copy the primary database. Should be prompted for the primary database connection password.
```
chown postgres:postgres -R /var/lib/postgresql/12/main 
```
Changes the owner of the new database content. If this command is not executed, the database initialization process may generate errors
```
systemctl start postgresql
```
Starts the database process.

### Showing initialization logs
To show the init logs, you must type the following command on bash:
```
/usr/lib/postgresql/12/bin/postgres -d 3 -D /var/lib/postgresql/12/main -c config_file=/etc/postgresql/12/main/postgresql.conf
```

## Checking replication status
If the previous steps are followed correctly, the replication process should be started. To show if is working as well, run this on the terminal, on the primary machine:

```
sudo su postgres
psql
SELECT client_addr, state FROM pg_stat_replication;
```
After this, the IP of the secondary machine should be shown.

# Database Restore
One of the most important processes is the restoration of the replicated database, in case of failure on the primary machine.
To execute this, you should connect to the secondary machine and follow these steps:

1. Connect to the database.
    ```
    sudo su postgres
    psql
    ```

2. Check if the database is in recovery mode (read-only).
    ```
    SELECT current_setting('transaction_read_only');
    ```
    If true was returned, this means that the database is in read-only mode.

3. Promote the database to be the new primary database and stops replication. Run this on the `psql`:
    ```
    SELECT pg_promote();
    ```

4. Check again if the database is in recovery mode. If returns false, this means that the database is in write mode and the restoring database worked as well.
    ```
    SELECT current_setting('transaction_read_only');
    ```