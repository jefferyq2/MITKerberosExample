# MITKerberosExample
This is a quick and dirty example of installing, configuring and testing MIT Kerberos using 2 different boxes (krb5 server and krb5 user).

Who might find this interesting is someone that has never used MIT Kerberos and wants to see what it is like to install, configure and test it. Someone that wants to run a few kerberos commands may find this interesting as well.

Naturally in the real world you will use password policies, adhere to your admin requirements for configuration files, lock down your keytabs and so on...but this should get you familiar with common kerberos commands as well as installing kerberos.

**Prereqs**

This assumes you already possess some know-how in AWS (SSH into boxes, create or use an appropriate VPC,create or use an appropriate security group) as well as run some basic linux commands. You probably would not be here if that was foreign.

**What we will do**

See the subdirectories for details.

Part 1 - KerbServer - create your kerberos server environment
<br/>
Part 2 - KerbUser - create your kerberos user environment
<br/>
Part 3 - KerbTest - successfully grab a ticket

That's it.
