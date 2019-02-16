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

See the subdirectories for details.

Part 1 - KerbServer - create your kerberos server environment
<br/>
Part 2 - KerbUser - create your principals and keytabs
<br/>
Part 3 - KerbTest - test it out

That's it.
