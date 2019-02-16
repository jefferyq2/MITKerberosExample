# MITKerberosExample

Terrific, let's test things out now that you have a Kerberos server and Kerberos user environment up and running.

In the real world, your administrators will have tighter controls around such things as linux file permissions and passwords for principals.

**Testing your principals and keytabs**

The steps here will include creating kerberos principals and keytabs (exporting principals into dedicated keytab files) and requesting and testing tickets.

On your kerberos server, let's create some user principals using using kadmin.local utility. Note, the "kadmin.local" can only be used on the kerberos host itself. You can use "kadmin" if you were doing this from a remote host.

**Create our keytab files**

Use randkey since we do not want any password for our principals.

```
sudo kadmin.local -q "add_principal -randkey reader@KAFKA.SECURE"
```

You should see something like this:
```
Principal "reader@KAFKA.SECURE" created.
```

And so on...let's create a couple more:

```
sudo kadmin.local -q "add_principal -randkey writer@KAFKA.SECURE"

sudo kadmin.local -q "add_principal -randkey admin@KAFKA.SECURE"
```

Now export those principals into keytab files.
```
sudo kadmin.local -q "xst -kt /tmp/reader.user.keytab reader@KAFKA.SECURE"
```

You should see something like this:
```
Entry for principal reader@KAFKA.SECURE with kvno 2, encryption type des-cbc-md5 added to keytab WRFILE:/tmp/reader.user.keytab.
```

And so on...let's create a couple more:

```
sudo kadmin.local -q "xst -kt /tmp/writer.user.keytab writer@KAFKA.SECURE"

sudo kadmin.local -q "xst -kt /tmp/admin.user.keytab admin@KAFKA.SECURE"

sudo chmod a+r /tmp/*.keytab
```

**Get a ticket**

Download these keytab files to your kerberos user server.

In this example, we will copy it to a local laptop from the kerberos server and then back up to the kerberos user server. In the real world again, you might want to put those somewhere other than /tmp so they survive a reboot.

```
Copy to local machine
scp -i ~<your pem file>.pem centos@<your kerberos server EC2 public DNS hostname>:/tmp/*.keytab /tmp/

Upload to kerberos user server
scp -i ~<your pem file>.pem *.keytab ubuntu@<your kerberos user EC2 public DNS hostname>:/tmp/
```

Now login to your kerberos user server.

Let's test by grabbing a ticket for the admin principal using the admin keytab.
```
kinit -kt /tmp/admin.user.keytab admin

klist
```

You should see something like:

```
Ticket cache: FILE:/tmp/krb5cc_1000
Default principal: admin@KAFKA.SECURE

Valid starting       Expires              Service principal
02/11/2019 01:28:00  02/12/2019 01:27:59  krbtgt/KAFKA.SECURE@KAFKA.SECURE
```

Let's look at the admin user keytab file.

Notice a lot of what look like duplicates? Those are really the same entries but for different encryption algorithms.

```
klist -kt /tmp/admin.user.keytab

Keytab name: FILE:/tmp/admin.user.keytab
KVNO Timestamp           Principal
---- ------------------- ------------------------------------------------------
   3 02/11/2019 00:45:53 admin@KAFKA.SECURE
   3 02/11/2019 00:45:53 admin@KAFKA.SECURE
   ...
```

That's it.

As a more concrete example, consider the following.

Let's say we have Confluent Kafka and krb5-user running on ubuntu and krb5-server running on a separate CentOS box. It's the same thing as above, but maybe you have a m4.xlarge or something running both krb5-user and Confluent Kafka.

On the ubuntu server, Confluent Kafka is running as the user ubuntu.

So on the Kerberos server machine, let's create the principal as such:

```
sudo kadmin.local -q "add_principal -randkey ubuntu/<your kafka server>@KAFKA.SECURE"

sudo kadmin.local -q "xst -kt /tmp/kafka.service.keytab ubuntu/<your kafka server>@KAFKA.SECURE"
```

Copy those keytabs to local laptop and up to the krb5-user/Kafka server.

```
scp -i ~<your pem file>.pem centos@<your Kerberos server>:/tmp/*.keytab /tmp/

scp -i ~<your pem file>.pem /tmp/*.keytab ubuntu@<your Kerberos user/Kafka server>:/tmp
```

Grab a ticket!

```
kinit -kt /tmp/kafka.service.keytab ubuntu/<your Kerberos user/Kafka server>
```

Verify it was ok with expiration.

```
klist
```

Start Kafka.
```
./confluent start zookeeper

KAFKA_OPTS="-Djava.security.auth.login.config=/home/ubuntu/confluent-5.1.1/etc/kafka/kafka_server_jaas.conf" ./confluent start kafka
```

Start the other available components:
  schema-registry
  kafka-rest
  connect
  ksql-server
  control-center

./confluent start <other components in the list>
