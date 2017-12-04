# Installing Quay Enterprise in GovCloud


## Create a VPC

In order to creat the instances used for Quay Enterprise a VPC must be created first.

## Create a VPN Connection to the VPC

In order to interact with instances inside the new VPC a VPN connection will need to be created. To do so follow these steps:
* In AWS navigate to your VPN and select "Virtual Private Gateway" and press the "Create Virtual Private Gateway" button.
  * Enter a name for the new virtual private gateway and the type of ASN required and press the "Create Virtual Private Gateway" button.
  * Back in the "Virtual Private Gateways" page select the VPG just created and press the "Actions" button and select "Attach to VPC"
  * In the "Attach to VPC" page select the VPC created above in the "VPC" dropdown and press the "Yes, Attach" button.
* Navigate away from "Virtual Private Gateways", select "VPN Connections", and press the "Create VPN Connection" button.
  * In the "Create VPN Connection" page enter:
    * A name for the new VPN connection
    * Select the Virtual Private Gateway created above
    * Create a new Customer Gateway and enter the PUBLIC IP address you will be accessing the VPN from.

## Create Quay Instances

The easiest way to create an EC2 instance for Quay is to base it off of the latest release of CoreOS Container Linux.

To spin up a Container Linux EC2 instance:
* In AWS GovCloud navigate to EC2 
* In EC2 select the "AMIs" menu option
* Search "Public Images" for "190570271432". This is CoreOS' Owner ID. Find the latest STABLE AMI whose Virtualization type is "HVM". Right click this AMI and select "Launch Instance".
* In the "Launch instance" page select "t2.medium" as the instance type and press the "Next: Configure Instance Details" button.
* In the "Configure Instance Details" page:
  *  Change the Number of Instances to 3
  *  Select the appropriate subnet for the instances and press the "Launch Instances" button.

## Create a MYSQL RDS instance

Quay Enterprise will need a database to act as its backend. To configure an RDS instance for Quay do the following:

* Navigate to the RDS service in AWS, select the "Instances" menu item, and press the "Launch DB Instance".
* Select "MySql" from the list of options and press the "Select" button.
* Select the "Production" MySql option and press "Next Step"
* Select "db.m3.medium" for the DB Instance Class.
* Select "Yes" for Multi-AZ deployment.
* Specify "100 GB" for Allocated Storage
* Specify "Quay" for DB Instance Identifier
* Specify a username and password for the Database.
* Press the "Next Step" button.
* In the "Configure Advanced Settings" page:
  * Select the VPC created above in the VPC dropdown menu.
  * Select the "default (VPC)" Security Group in the "VPC Security Group(s)" field.
  * In the "Database Name" text field enter in "quay"
  * Select any other option that may be required or useful for this deployment and press the "Launch DB Instance" button.

## Create an S3 bucket to temporarily store necessarily images

In order to transfer images from public image repositories to the internal Quay repository there must be a location to store the images temporarily. A standard S3 bucket is preferred, there should be no additional configuration needed.

## Copy images to the S3 bucket.

* On an internet facing machine run the following commands:
  * `docker pull quay.io/quay/redis`
  * `docker pull quay.io/coreos/quay:v2.6.1`
  * `docker save quay.io/quay/redis -o /tmp/quay-redis.tar`
  * `docker save quay.io/coreos/quay:v2.6.1 -o /tmp/quay-enterprise.tar`
* Upload the images to the S3 bucket created above via the AWS S3 Web UI


## Run Quay Enterprise

On the Quay instances created above:
* Login to Quay.io and get your Quay Pull Secret
  * Copy the Pull secret file "config.json" to the following directories:
    * /home/core/.docker/config.json
    * /root/.docker/config.json
* Run the following commands:
  * mkdir /config
  * mkdir /storage
* Download the images above from the S3 bucket and run the following commands:
  * `docker load -i /tmp/quay-redis.tar`
  * `docker load -i /tmp/quay-enterprise.tar`
	
* Copy the following into `/etc/systemd/system/quay-enterprise.service`:
```
[Unit]
Description=Quay Enterprise
After=multi-user.target

[Service]
Type=oneshot
RemainAfterExit=true

User=root
Group=root

ExecStartPre=/usr/bin/docker run --restart=always --name quay-redis -d -p 6379:6379 quay.io/quay/redis
ExecStart=/usr/bin/docker run --restart=always --name quay-enterprise -p 443:443 -p 80:80 -p 8443:8443 --privileged=true -v /config:/conf/stack -v /storage:/datastorage -d quay.io/coreos/quay:v2.6.1

ExecStopPost=/usr/bin/docker stop quay-redis
ExecStopPost=/usr/bin/docker rm quay-redis

[Install]
WantedBy=multi-user.target
```

* Run the following commands:
  * `systemctl daemon-reload`
  * `systemctl start quay-enterprise`

* Login to one of the Quay Enterprise instances using whatever IP address is available (public IP or private ip address using a VPN) in a browser. Follow the Quay Enterprise configuration steps.

* Create a Load Balancer for Quay using the following doc: https://coreos.com/quay-enterprise/docs/latest/setup-elb.html
