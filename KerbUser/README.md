# MITKerberosExample

**Kerberos user server**

We will use a tiny Ubuntu box in AWS. Something like this would be fine:

t2.micro, AWS Marketplace (search for Ubuntu), Ubuntu 16.04 LTS - Xenial (HVM)

<br/>

**Create your kerberos user server**

Install krb5-user service on the kerberos user machine and point it to the kerberos server.

Again, make sure you have the proper firewall ports open or security group access rules as appropriate to talk to your kerberos server over port 88 (TCP is fine for simple command line testing here, but you would want to open UDP for service based communication).

```
export DEBIAN_FRONTEND=noninteractive ; sudo apt-get install -y krb5-user

sudo vi /etc/krb5.conf

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

That's it. You have what you need and can move onto testing.
