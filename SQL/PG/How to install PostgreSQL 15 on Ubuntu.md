If you try to run the command `sudo apt install postgresql` in Ubuntu it will install Postgresql version 14 instead of version 15. As of writing the Ubuntu repository does not include version 15.

Follow the tutorial below for a detailed guide to successfully installing PostgreSQL 15.

---

## Step-by-Step Instructions on Configuring Postgresql 15 in Ubuntu

The following steps were tested in Ubuntu 22.04 LTS. If there are new versions of Ubuntu, I will test the steps again and update this post.

# 1. Installation

### 1.1 Add the Postgresql Package Repository to Ubuntu

Run the commands below to add the official Postgresql package repository to Ubuntu.

```bash
# Create the file repository configuration
$ sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'

# Import the repository signing key
$ wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo tee /etc/apt/trusted.gpg.d/pgdg.asc &>/dev/null

# Update list of available packages
$ sudo apt update
```

Bash

COPY

---

### 1.2 Install Postgresql 15 Server and Client

Run the command below to install PostgreSQL 15 Server and Client.

```bash
$ sudo apt install postgresql-15 postgresql-client-15 -y
```

Bash

COPY

---

### 1.3 Verify installed PostgreSQL version

Run the command below to check if PostgreSQL has successfully been installed and if the PostgreSQL version 15 has been installed.

```bash
$ psql --version
```

Bash

COPY

Output

![](https://www.radishlogic.com/wp-content/uploads/2023/02/image-4.png)

Installed PostgreSQL version

---

### 1.4 Verify if PostgreSQL is running

Run the command below to check the status of PostgreSQL.

```bash
$ sudo systemctl status postgresql
```

Bash

COPY

The output would look like this if PostgreSQL is running.

![](https://www.radishlogic.com/wp-content/uploads/2023/02/image-1024x212.png)

PostgreSQL is running (active)

Ff the PostgreSQL is not running, the output will be like below.

![](https://www.radishlogic.com/wp-content/uploads/2023/02/image-1-1024x264.png)

PostgreSQL is not running (inactive)

If PostgreSQL is not running, then running the command below will start it.

```bash
$ sudo systemctl start postgresql
```

Bash

COPY

The command above does not have an output, so it will be better to check the status by running the command `sudo systemctl status postgresql`.

---

### 1.5 Making PostgreSQL automatically run after reboot

To make sure that PostgreSQL will run after a restart or when you turn on your Ubuntu server/computer, then run the command below.

```bash
$ sudo systemctl enable postgresql
```

Bash

COPY

Output.

![](https://www.radishlogic.com/wp-content/uploads/2023/02/image-3-1024x63.png)

To check whether PostgreSQL will run at restart run the following command.

```bash
$ sudo systemctl is-enabled postgresql
```

Bash

COPY

Output.

![](https://www.radishlogic.com/wp-content/uploads/2023/02/image-2.png)

If the output is `enabled` then PostgreSQL database will run at reboot.

If `disabled`, then it will not automatically run. You should run the command `sudo systemctl enable postgresl` to automatically start PostgreSQL upon boot.

**Note:** By experience, PostgreSQL is already running and will start upon starting of Ubuntu once I have finished installing it.

---

# 2. Configure PostgreSQL 15

## 2.1 Change Ubuntu User to PostgreSQL admin

```bash
sudo -u postgres psql
```

Bash

COPY

## 2.2 Update PostgreSQL Admin User Password

```sql
ALTER USER postgres WITH ENCRYPTED PASSWORD '[adminPassword]';
```

SQL

COPY

## 2.3 Create a new user in PostgreSQL

This user will be used by your application to access the database.

```sql
CREATE USER [Username] WITH ENCRYPTED PASSWORD '[Password]';
```

SQL

COPY

## 2.4 Create a new database

```sql
CREATE DATABASE [DatabaseName];
```

SQL

COPY

## 2.5 Grant user privileges to the database

```sql
GRANT ALL PRIVILEGES ON DATABASE [DatabaseName] TO [Username];
```

SQL

COPY

## 2.5 Connect to the database

```sql
\connect [DatabaseName];
```

SQL

COPY

## 2.6 Create a schema and authorize the user to use it.

```sql
CREATE SCHEMA [SchemaName] AUTHORIZATION [Username];
```

SQL

COPY

**Note:** Before PostgreSQL 15, the `public` schema is accessible to any user by default. But starting PostgreSQL 15, they disallow default access to the `public` schema due to security risks.

This step to create a schema and authorize the user to use it is necessary since we are using PostgreSQL 15.

## 2.7 Exit psql

```none
\q
```

Plain text

COPY

## 2.8 Test user login to PostgreSQL

```bash
psql -U [Username]-d [DatbaseName] -h localhost
```


This will ask you for your password.

If you were able to login without errors, you have successfully created your PostgreSQL database and user.

## 3. Allowing access to PostgreSQL from outside Ubuntu machine

If you need remote access to your PostgreSQL from outside of your Ubuntu machine, then you will have to follow the steps below.

By default, PostgreSQL only allows connection from localhost only.

## 3.1 Allow PostgreSQL to listen to connections from anywhere

Open the PostgreSQL configuration file in an editor.

Below I will be using vim, but you can use what you are comfortable with like nano or something else.

```bash
sudo vi /etc/postgresql/15/main/postgresql.conf
```


Uncomment the line under Connection Settings that says `listen_address` by removing the number sign (`#`) sign.

![](https://www.radishlogic.com/wp-content/uploads/2023/02/image-5-1024x395.png)

Save and exit by pressing `:wq` then `enter`.

## 3.2 Allow login to PostgreSQL via IPv4 remote connection from anywhere

Open the PostgreSQL Host-Based Authentication (HBA) file.

```bash
sudo vi /etc/postgresql/15/main/pg_hba.conf
```

Bash

COPY

Go to the line under **IPv4 local connections** and change `127.0.0.1/32` to `0.0.0.0/0`.

![](https://www.radishlogic.com/wp-content/uploads/2023/02/image-6.png)

Save and exit by pressing `:wq` then `enter`.

**Note:** Allowing access from anywhere (0.0.0.0/0) is not really recommended security-wise. If you want to limit access from a specific subnet or CIDR address range the address range, like 192.168.0.0/24, instead of 0.0.0.0/0.

I only placed 0.0.0.0/0 since this is only my test server and I’m lazy to check which IP address range my Ubuntu machine is.

## 3.3 Allow TCP 5432 port in Ubuntu Firewall

If Ubuntu firewall is enabled you will need to allow the default PostgreSQL 5432 port to allow TCP connection.

To do that, run the command below.

```none
sudo ufw allow 5432/tcp
```

Plain text

COPY

---

## 3.4 Connect to PostgreSQL remotely to check

On a different machine in your network, connect to your PostgreSQL server.

Here’s the command via psql.

```bash
psql -U [Username]-d [DatbaseName] -h localhost
```

Bash

COPY

You can also try connecting using pgAdmin4 or other database clients if you want.

---

We hope this helps you install PostgreSQL 15 in your Ubuntu.

If you have any issues or comments installing, let us know in the comments below.