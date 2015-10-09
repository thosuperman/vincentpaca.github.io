---
layout: post
title: Openfire + Hazelcast on EC2
---

I have been searching high and low for something, somwhere, to guide me on this beast that is clustering. After shedding a few liters of blood sweat and tears, I finally was able to get my cluster up.

This post should prove useful for my future self. You’re welcome, future self.

The end result: One DB server, a cluster of Openfire servers and a load balancer to distribute traffic. Sweet!

### Launch an Instance
Launch an EC2 Instance, in my case I used a micro instance for testing. Just follow the Request Instance Wizard and you’ll be fine.

Take note of your key-pair. Download it, keept it and savor it’s essence.

When setting up your firewall, free dem ports:

- 22 (SSH)
- 3478
- 3479
- 5222
- 5701
- 7070
- 7443
- 7777
- 9090
- 9091
-
SSH to your server by doing `ssh -i your_pem_file.pem ubuntu@your-instance-public-dns`

### Launch a DB Server on RDS
I’m using MySQL, so I’m just gonna go ahead and choose that.
Name your database appropriately.

Remember your Master Username and Password. You’re gonna need it.
Update your RDS security groups. Head on to the RDS console and select DB Security Groups. Add your EC2 instance to the list as an EC2 Security Group.

### Setup Openfire

1. Install Java with:

```
sudo add-apt-repository ppa:webupd8team/java
sudo apt-get update
sudo apt-get remove —purge openjdk*
sudo apt-get install oracle-java-7-installer
java -version
```
2. Install Openfire with:

```
wget -O openfire.tar.gz http://www.igniterealtime.org/downloads/download-landing.jsp?file=openfire/openfire_3_8_1.tar.gz
```

Change the version to the latest stable available. Then untar and move like so:

```
tar -xzvf openfire.tar.gz
mv openfire/ /opt
```
3. Edit your hosts file in /etc/hosts and add a line to let the server know what’s its name is. Hosts file should look like:

```
127.0.0.1 localhost
127.0.1.1 ubuntu
127.0.0.1 chat.yourdomain.com
```
4. Run Openfire by going to `/your_openfire_directory/bin` and running `./openfire start`
5. Visit http://your-instance-public-dns:9090 and it should give you the first step for the Openfire setup wizard. Plow through the wizard like a champ. Just remember the following important points:
- The host should be the one you set in your /etc/hosts file a while ago.
- The database host should be the host on the RDS DB server from the setup above.
- The database username and password is your MASTER username and password from RDS.
6. Install Hazelcast! On your Openfire Admin Panel, Go to Plugins > Available Plugins > Install. If you don’t see any plugins on the Available Plugins page, you can find an update link on the page.
7. Go back to the terminal and we’ll change a few settings for the cluster to work. Edit the file in `/your_openfire_directory/plugins/hazelcast/classes/hazelcast-cache-config.xml`. Find the <network> tag and we should configure it like so:

```
…
<port auto-increment=“true”>5701</port>
<join>
  <multicast enabled=“false” />
  <tcp-ip enabled=“true”>
    <hostname>private-ip-address-of-this-machine:5701</hostname>
  </tcp-ip>
  <aws enabled=“false”/>
</join>
<interfaces enabled=“true”>
  <interface>private-ip-address-of-this-machine</interface>
</interfaces>
…
```

The private IP address of your machine can be found on your Amazon EC2 Console on the instance details under Private IPs.
8. Restart your Openfire server by going to `/your_openfire_dir/bin` and doing `./openfire stop` and `./openfire start`. Visit your Openfire Admin Console and enable clustering. You should be able to join a cluster with one node running. Neat.

### Growing your Openfire army

1. On the EC2 Console, create an AMI of the Openfire instance. Launch a new instance using the AMI.
2. On the wizard, use the same settings as we had with the other instance.
3. SSH to the new server and edit `/your_openfire_directory/plugins/hazelcast/classes/hazelcast-cache-config.xml`. The config should now look like:

```
…
<port auto-increment=“true”>5701</port>
<join>
  <multicast enabled=“false” />
  <tcp-ip enabled=“true”>
    <hostname>private-ip-of-the-other-machine:5701</hostname>
    <hostname>private-ip-address-of-this-machine:5701</hostname>
  </tcp-ip>
  <aws enabled=“false”/>
</join>
<interfaces enabled=“true”>
  <interface>private-ip-address-of-this-machine</interface>
</interfaces>
…
```

Make sure the IP addresses for the machine is correct.

4. Restart Openfire.
5. Go to the Openfire Admin Console and enable clustering.
6. SSH to the server that we first set up and edit the hazelcast config file. Add a new <hostname> line on the file with the IP address of the new instance we just created.
7. Restart Openfire for that
8. Just repeat the process should you want to add more instances to the cluster.

### Setup Load Balancing
1. On the EC2 Console under Load Balancers, create a new load balancer.
2. Open up these ports under Load Balancers > Listeners:

- HTTP 80
- TCP 3478
- TCP 3479
- TCP 5222
- TCP 5262
- HTTP 9090
- HTTP 7070

If you need SSL, you need to upload them and attach them to a listener.

### Final Steps
On your DNS manager, make a CNAME record that points to the load balancer, and that’s it!
