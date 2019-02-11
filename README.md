# MITKerberosExample
This is a quick and dirty example of installing, configuring and testing MIT Kerberos using 2 different boxes (krb5 server and krb5 user).

Who might find this interesting is someone that has never used MIT Kerberos and wants to see what it is like to install, configure and test it. Someone that wants to run a few kerberos commands may find this interesting as well.

Naturally in the real world you will use password policies, adhere to your admin requirements for configuration files, lock down your keytabs and so on...but this should get you familiar with common kerberos commands as well as installing kerberos.

**Prereqs**

This assumes you already possess some know-how in AWS (SSH into boxes, create or use an appropriate VPC,create or use an appropriate security group) as well as run some basic linux commands. You probably would not be here if that was foreign.

**Kerberos server**

You can use whatever boxes and O/S you want.

In this walkthrough, we will use a tiny CentOS box in AWS. Something like this would be fine:<br/>
t2.micro, AWS Marketplace (search for CentOS), CentOS 7 (x86_64) - with Updates HVM

**Kerberos user server**

Also, we will use a tiny Ubuntu box in AWS. Something like this would be fine:<br/>
t2.micro, AWS Marketplace (search for Ubuntu), Ubuntu 16.04 LTS - Xenial (HVM)

**What we will do**

Part 1 - create your kerberos server environment
<br/>
Part 2 - create your principals and keytabs
<br/>
Part 3 - test it out

<br/><br/>

**Part 1
Creating your Kerberos server environment**

Launch a CentOS AMI image from the AWS marketplace and setup your VPC, security group, etc. Note, kerberos KDC by default uses on port 88.

SSH to your instance
```
ssh -i ~<your pem file>.pem centos@<your kerberos EC2 public DNS hostname>
```

Install krb5-server service
```
sudo yum install -y krb5-server
```

In all of the following edits, just make sure your REALM is correct as well as things like the accessible public DNS of your kerberos instance.

Edit the kdc.conf file as appropriate. We will use the realm MYKAFKA.REALM.
```
sudo vi /var/kerberos/krb5kdc/kdc.conf

You should have something like this:
[kdcdefaults]
  kdc_ports = 88
  kdc_tcp_ports = 88
  default_realm=MYKAFKA.REALM
[realms]
  MYKAFKA.REALM = {
    acl_file = /var/kerberos/krb5kdc/kadm5.acl
    dict_file = /usr/share/dict/words
    admin_keytab = /var/kerberos/krb5kdc/kadm5.keytab
    supported_enctypes = aes256-cts:normal aes128-cts:normal des3-hmac-sha1:normal arcfour-hmac:normal camellia256-cts:normal camellia128-cts:normal des-hmac-sha1:normal des-cbc-md5:normal des-cbc-crc:normal
  }

```

Edit the kadm5.acl file
```
sudo cat /var/kerberos/krb5kdc/kadm5.acl

You should have something like this:
*/admin@MYKAFKA.REALM *
```

Edit the krb5.conf file
```
sudo vi /etc/krb5.conf

You should have something like this:
[logging]
  default = FILE:/var/log/krb5libs.log
  kdc = FILE:/var/log/krb5kdc.log
  admin_server = FILE:/var/log/kadmind.log

[libdefaults]
    default_realm = MYKAFKA.REALM
    kdc_timesync = 1
    ticket_lifetime = 24h

[realms]
    MYKAFKA.REALM = {
      admin_server = <your kerberos EC2 public DNS hostname>
      kdc  = <your kerberos EC2 public DNS hostname>
      }
```

Create your kerberos database
<br/>
The "s" flag specifies that this command should create a file where the master principal is stored.

```
sudo /usr/sbin/kdb5_util create -s -r MYKAFKA.REALM -P this-is-unsecure

You should see output that resembles something like this:
Loading random data
Initializing database '/var/kerberos/krb5kdc/principal' for realm 'MYKAFKA.REALM',
master key name 'K/M@MYKAFKA.REALM'
```

Create your admin principal
<br/>
The "q" flag means the query you are sending to the KDC.
<br/>
In the ACL file, we specified every principal with /admin is a dedicated admin user.
<br/>
The warning you can ignore for now because we do not specify a password policy for this principal.
```
sudo kadmin.local -q "add_principal -pw this-is-unsecure admin/admin"
```

Start your Services
```
sudo systemctl restart krb5kdc

sudo systemctl status krb5kdc

You should see something like this:
...
krb5kdc.service - Kerberos 5 KDC
Active: active (running)
Started Kerberos 5 KDC

sudo systemctl restart kadmin

sudo systemctl status kadmin

Similarly, you should see something like a SUCCESS active (running) completion
...
```

Congrats! You have a kerberos server up and running!

Now onto Part 2...

<br/><br/>

**Part 2
Create your principals and keytabs**

Steps will include creating kerberos principals for both users and services, keytabs (exporting principals into dedicated keytab files) and testing a ticket.

Let's create user principals using kadmin.local utility. Note, the "kadmin.local" can only be used on the kerberos host itself. You can use "kadmin" if you were doing this from a remote host.

Using randkey because we do not want any password for our principals.
```
sudo kadmin.local -q "add_principal -randkey reader@MYKAFKA.REALM"

You should see something like this:
Principal "reader@MYKAFKA.REALM" created.

And so on...

sudo kadmin.local -q "add_principal -randkey writer@MYKAFKA.REALM"

sudo kadmin.local -q "add_principal -randkey admin@MYKAFKA.REALM"
```

Now export those principals into keytab files.
```
sudo kadmin.local -q "xst -kt /tmp/reader.user.keytab reader@MYKAFKA.REALM"

You should see something like this:
...
Entry for principal reader@MYKAFKA.REALM with kvno 2, encryption type des-cbc-md5 added to keytab WRFILE:/tmp/reader.user.keytab.
...

sudo kadmin.local -q "xst -kt /tmp/writer.user.keytab writer@MYKAFKA.REALM"

sudo kadmin.local -q "xst -kt /tmp/admin.user.keytab admin@MYKAFKA.REALM"

sudo chmod a+r /tmp/*.keytab
```


<br/><br/>

**Part 3
Testing your principals and keytabs**

Download those to your kerberos user server. In this example, we will copy it to a local laptop from the kerberos server and then back up to the kerberos user server.
```
Copy to local machine
scp -i ~<your pem file>.pem centos@<your kerberos server EC2 public DNS hostname>:/tmp/*.keytab /tmp/

Upload to kerberos user server
scp -i ~<your pem file>.pem *.keytab ubuntu@<your kerberos user EC2 public DNS hostname>:/tmp/
```

Install krb5-user service on the kerberos user machine and point it to the kerberos server.
```
export DEBIAN_FRONTEND=noninteractive ; sudo apt-get install -y krb5-user

sudo vi /etc/krb5.conf

[logging]
  default = FILE:/var/log/krb5libs.log
  kdc = FILE:/var/log/krb5kdc.log
  admin_server = FILE:/var/log/kadmind.log

[libdefaults]
    default_realm = MYKAFKA.REALM
    kdc_timesync = 1
    ticket_lifetime = 24h

[realms]
    MYKAFKA.REALM = {
      admin_server = <your kerberos server EC2 public DNS hostname>
      kdc  = <your kerberos server EC2 public DNS hostname>
      }
```

Let's test by grabbing a ticket for the admin principal using the admin keytab.
```
kinit -kt /tmp/admin.user.keytab admin

klist

Ticket cache: FILE:/tmp/krb5cc_1000
Default principal: admin@MYKAFKA.REALM

Valid starting       Expires              Service principal
02/11/2019 01:28:00  02/12/2019 01:27:59  krbtgt/MYKAFKA.REALM@MYKAFKA.REALM
```

Let's look at the admin user keytab file. Notice a lot of what look like duplicates? Those are really the same entries but for different encryption algorithms.
```
klist -kt /tmp/admin.user.keytab

Keytab name: FILE:/tmp/admin.user.keytab
KVNO Timestamp           Principal
---- ------------------- ------------------------------------------------------
   3 02/11/2019 00:45:53 admin@MYKAFKA.REALM
   3 02/11/2019 00:45:53 admin@MYKAFKA.REALM
   ...
```
