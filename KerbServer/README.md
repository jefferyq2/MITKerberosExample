# MITKerberosExample

**Kerberos server**

You can use whatever boxes and O/S you want.

In this walkthrough, we will use a tiny CentOS box in AWS. Something like this would be fine:

t2.micro, AWS Marketplace (search for CentOS), CentOS 7 (x86_64) - with Updates HVM

<br/>
**Creating your Kerberos server environment**

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

Edit the kdc.conf file as appropriate. We will use the realm KAFKA.SECURE.

Take note of the port 88 so you can open up firewall rules or security group access points as appropriate to/from your krb5-user server.

```
sudo vi /var/kerberos/krb5kdc/kdc.conf

You should have something like this:
[kdcdefaults]
  kdc_ports = 88
  kdc_tcp_ports = 88
  default_realm=KAFKA.SECURE
[realms]
  KAFKA.SECURE = {
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
*/admin@KAFKA.SECURE *
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
    default_realm = KAFKA.SECURE
    kdc_timesync = 1
    ticket_lifetime = 24h

[realms]
    KAFKA.SECURE = {
      admin_server = <your kerberos server>
      kdc  = <your kerberos server>
      }
```

Create your kerberos database.

The "s" flag specifies that this command should create a file where the master principal is stored.

```
sudo /usr/sbin/kdb5_util create -s -r KAFKA.SECURE -P this-is-unsecure

You should see output that resembles something like this:
Loading random data
Initializing database '/var/kerberos/krb5kdc/principal' for realm 'KAFKA.SECURE',
master key name 'K/M@KAFKA.SECURE'
```

Create your admin principal.

The "q" flag means the query you are sending to the KDC.

In the ACL file, we specified every principal with /admin is a dedicated admin user.

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
