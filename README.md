#How to enable proftpd postgresql authentication for registered Galaxy database users in CentOS 7 (together with SSL encryption -SFTP)

Because I have to collect information from various sites to make this work I wrote a small tutorial with exact instructions on how to install it!
Here are the full instructions with everything you need to set up!!!

## Part 1: Getting FTP user authentication working for your local galaxy users

Prequisites: 
* you need a running Galaxy installation which has to be configured into production mode (with a working Postgres database backend). 
  Now browse to your local Galaxy webserver with your browser and register a new account, we will use the name ```johndoe@example.com``` for our tests.
* proftpd is running as user ftp, group ftp (this is the standard on installation oft proftpd in CentOS 7)

First and foremost we need to define some important environment variables for later steps:
These values should reflect your galaxy production postgres server data. We will set up a new database user to query our user accounts for proftd 
with limited access capeabilites.
```bash
export GALAXY_DBSERVER=localhost
export GALAXY_DBNAME=galaxyprod
export GALAXY_PORT=5432
export GALAXY_USER=galaxyftp
export GALAXY_PW=xxxx
```bash

Also we need to define the exact system user, the galaxy process is running with (check Galaxy production documentation, normally this will be the user galaxy):
```
# we will need this for proper file upload permissions
export GALAXY_SYSUSER=galaxy
export GALAXY_SYSGROUP=galaxy
```

 Another THING you need to define is the galaxy ftp upload dir (please note that you also have to define the same value as ftp_upload_dir in your galaxy.ini):
 BE CAREFUL TO NOT add a leading '/' sign at the end of the path in the following definition
```bash
export FTP_UPLOAD_DIR=/srv/software/galaxy/galaxy-ftp-upload
```

now create the upload directory
```bash
mkdir -p $FTP_UPLOAD_DIR
chown $GALAXY_SYSUSER $FTP_UPLOAD_DIR
```
Now create the new database user on the database server with proper read access rights for our galaxy database
```bash
psql -U postgres -c $GALAXY_DBNAME "CREATE USER $galaxyftp WITH PASSWORD '$GALAXY_PW';"
psql -U postgres -c $GALAXY_DBNAME "GRANT READ PRIVILEGES ON DATABASE $GALAXY_DBNAME to $galaxyftp;"
```

Install Proftpd on CentOS 7 (do not start the proftpd service or enable it after installing it yet!):
```bash
yum install proftpd proftpd-postgresql
```

next open ftp ports in firewalld
```bash
firewall-cmd --permanent --add-service ftp
firewall-cmd --reload
```

first make a backup of the Proftpd file
```bash
cp /etc/proftpd.conf /etc/proftpd.conf.BAK
```

now comment out some stuff in the proftpd config file, we will overwrite later:
```bash
sed -i 's/AuthOrder(.*)/#AuthOrder\1/g' /etc/proftpd.conf
sed -i 's/ServerName(.*)/#ServerNaem\1/g' /etc/proftdp.conf
```

create a custom Welcome message for the FTP server when a galaxy user is logging in
change accordingly to fit your needs
```bash
mkdir -p /etc/opt/local/
echo "Welcome galaxy user. Please note that YOUR data will be stored on OUR server....BLABLABLA" > /etc/opt/local/proftpd_welcome.txt
```
now append the following content to the main proftpd config file (/etc/proftpd.conf) using the following command line:
```bash
echo "
# Galaxy postgres specific config ######################################
ServerName "Galaxy FTP File Upload"

# General database support (http://www.proftpd.org/docs/contrib/mod_sql.html)
LoadModule mod_sql.c
LoadModule mod_sql_postgres.c
LoadModule mod_sql_passwd.c

# set Authentication order
AuthOrder                       mod_sql.c

# This User & Group should be set to the actual user and group name which matche the UID & GID you will specify later in the SQLNamedQuery.
User                            ftp
Group                           ftp
DisplayConnect                  /etc/opt/local/proftpd_welcome.txt

# Cause every FTP user to be "jailed" (chrooted) into their home directory
DefaultRoot                     ~

# Automatically create home directory if it doesn't exist
CreateHome                      on dirmode 700

# Allow users to overwrite their files
AllowOverwrite                  on

# Allow users to resume interrupted uploads
AllowStoreRestart               on

# Bar use of SITE CHMOD
<Limit SITE_CHMOD>
    DenyAll
</Limit>

# Bar use of RETR (download) since this is not a public file drop
<Limit RETR>
    DenyAll
</Limit>

# Do not authenticate against real (system) users
AuthPAM                         off

# Set up mod_sql to authenticate against the Galaxy database
# Common SQL authentication options
SQLEngine                       on
SQLPasswordEngine               on
SQLBackend                      postgres
SQLConnectInfo                  ${GALAXY_DBNAME}@${GALAXY_DBSERVER}:${GALAXY_PORT} ${GALAXY_USER} ${GALAXY_PW}
SQLAuthTypes                    PBKDF2
SQLPasswordPBKDF2               SHA256 10000 24
SQLPasswordEncoding             base64
SQLAuthenticate                 users
# For PBKDF2 authentication
SQLPasswordUserSalt sql:/GetUserSalt
SQLUserInfo                     custom:/LookupGalaxyUser
SQLNamedQuery                   LookupGalaxyUser SELECT "email, (CASE WHEN substring(password from 1 for 6) = 'PBKDF2' THEN substring(password from 38 for 69) ELSE password END) AS password2,512,512,'${FTP_UPLOAD_DIR}/%U','/bin/bash' FROM galaxy_user WHERE email='%U'"
# Define custom query to fetch the password salt
SQLNamedQuery GetUserSalt SELECT "(CASE WHEN SUBSTRING (password from 1 for 6) = 'PBKDF2' THEN SUBSTRING (password from 21 for 16) END) AS salt FROM galaxy_user WHERE email='%U'"

#End of Galaxy specific config #########################################
" >> /etc/proftpd.conf
```

next test the configuration file
```bash
proftpd --config /etc/proftpd.conf -t
```
now run proftpd in full debug mode and leave it open
```bash
proftpd --config /etc/proftpd.conf -n -d 20
```
now test if you can successfully login (make sure the systemd proftp service is not running, if so stop it first) from another machine in the same network using your Galaxy user account
in our example the Galaxy user account is called johndoe@example.com and has been created using the Galaxy web application. The server running proftpd is called proftpserver:
```bash
ftp proftpserver
220-Welcome to Galaxy FTP
220 FTP Server ready.
Name (proftpserver:heino): johndoe@example.com
331 Password required for johndoe@example.com
Password:
```
After entering the correct password you should be able to connect
```bash
230 User johndoe@example.com logged in
Remote system type is UNIX.
Using binary mode to transfer files.
ftp>
```
test writing of files
```bash
ftp> mkdir testdir
257 "/testdir" - Directory successfully created
ftp> rmdir testdir
250 RMD command successful
```

now logout
```bash
ftp> exit
221 Goodbye.
```



## Part 2: after successfully setting up basic FTP server for our registered Galaxy users we tighten things up to be more secure using SSL encryption:

first lets generate a SSL certificate/public-private key pem file for use of SFTP on the commandline
when generating the .pem file, you will be asked some questions about the certificate we are about to create, you can enter any thing you like expect the
"Common Name" field, where you have to enter the base domain name of your FTP server e.g. "proftpserver":

```bash
cd /etc/pki/tls/certs
make proftp-server.pem
chmod 600 /etc/pki/tls/certs/proftp-server.pem
```

Now add the following code fragment to the end of the main proftpd config file (/etc/proftpd.conf):
```bash
# Add SSL/TLS support to the end of the file 
TLSEngine on
TLSLog /var/log/proftpd/tls.log
TLSProtocol TLSv1.2
TLSRenegotiate none
TLSCipherSuite ALL:!SSLv2:!SSLv3
TLSVerifyClient off
TLSRequired auth+data
TLSOptions NoSessionReuseRequired

TLSLog                    /var/log/proftpd/tls.log
TLSRSACertificateFile     /etc/pki/tls/certs/proftp-server.pem
TLSRSACertificateKeyFile  /etc/pki/tls/certs/proftp-server.pem

# Passive port range for the firewall - note that you must open these ports for this to work!
# Since the FTP traffic is now encrypted, your firewall can't peak to see that it is PASSV FTP
# and it will block it if you don't allow new connections one these ports.
PassivePorts                    30000 30100
# Cause every FTP user to be "jailed" (chrooted) into their home directory
DefaultRoot                     ~
AllowOverwrite                  on
# Allow users to resume interrupted uploads
AllowStoreRestart               on
# .. Other rules for directories, etc
SQLEngine                       on
# .. See above, the same SQL rules apply
```

now since proftpd is using passive ports from 30000 - 30100 
```bash
firewall-cmd --permanent --zone=public --add-port=30000-30100/tcp
firewall-cmd --reload
```

next test the configuration file
```bash
proftpd --config /etc/proftpd.conf -t
```
now run proftpd in full debug mode and leave it open so we can test client connections against its ftps connection:
```bash
proftpd --config /etc/proftpd.conf -n -d 20
```

Test if FTPS connection is working by using another client in the same network, if its a Red Hat based client install lftp:
```bash
yum -y install lftp
```
now config lftp on the fly
```bash
echo "
set ftp:ssl-auth TLS
set ftp:ssl-force true
set ftp:ssl-protect-list yes
set ftp:ssl-protect-data yes
set ftp:ssl-protect-fxp yes
set ssl:verify-certificate no
" > ~/.lftprc 
```
now finally test
```bash
lftp -u "johndoe@example.com" proftpserver -d
```
Now test if we have write access
```bash
mkdir test-dir
rmdir test-dir
exit
```

VICTORY !!!!  FATALITY


TODO: check if ftp is disabled for good now!

Sources:
https://wiki.galaxyproject.org/Admin/Config/UploadviaFTP
http://galacticengineer.blogspot.de/2015/02/ftp-upload-to-galaxy-using-proftpd-and.html
http://www.server-world.info/en/note?os=CentOS_6&p=ftp&f=6
various StackOverflow and other IT blogs


