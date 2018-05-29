## EC2 Instance

EC2 Console  
click **Launch Instance**

_Step 1: Choose an Amazon Machine Image (AMI)_  
click **Select** button for **Ubuntu Server 16.04 LTS (HVM), SSD Volume Type** - ami-43a15f3e

_Step 2: Choose an Instance Type_  
choose type:  t2.large  
click **Next: Configure Instance Details**

_Step 3: Configure Instance Details_  
Number of instances:  1  
Purchasing option: [ ] Request spot instances  
Network:  vpc-2b2fef4a | default-vpc  
Subnet:  subnet-1e6bcd46 | dev-apps-subnet-2 | us-east-1b  
Auto-assign Public IP:  Enable  
IAM role:  None  
Shutdown behavior:  Stop  
Enable termination protection:  [ ] Protect against accidental termination  
Monitoring:  [ ] Enable CloudWatch detailed monitoring  
Tenancy:  Shared - Run a shared hardware instance  
T2 Unlimited:  [ ] Enable  
_Network interfaces_  
_Device_ / _Network Interface_ / _Subnet_ / _Primary IP_ / _Secondary IP addresses_ / _IPv6 IPs_  
eth0 / New network interface / subnet-1e6bcd46 / Auto-assign / - / -  
_Advanced Details_  
(none)  
click **Next: Add Storage**

_Step 4: Add Storage_  
_Volume Type_ / _Device_ / _Snapshot_ / _Size (GiB)_ / _Volume Type_ / _IOPS_ / _Throughput (MB/s)_ / _Delete on Termination_ / _Encrypted_  
Root / /dev/sda1 / snap-0eea1ed47e203f3b8 / 16 / General Purpose SSD (GP2) / 100/3000 / N/A / [x] / Not Encrypted  
click **Next: Add Tags**

_Step 5: Add Tags_  
Key / Value / Instances / Volumes  
Name / cibmtr-smart-dev / [x] / [x]  
Purpose / cibmtr:smart-sandbox / [x] / [x]  
Product / EC2 / [x] / [x]  
Project Name / AGNIS on FHIR / [x] / [x]  
Project Charge Number / BC-N3Y-2D11-00-AGNIS / [x] / [x]  
Environment / prod / [x] / [x]  
click **Next: Configure Security Group**

_Step 6: Configure Security Group_  
Assign a security group:  Select an existing security group  
select default-ssh-nmdp-aws and cibmtr-smart-ansible  
click **Review and Launch**

_Step 7: Review Instance Launch_  
click **Launch**  
_Select an existing key pair or create a new key pair_  
Choose an existing key pair  
jschneid_nmdp  
[x] I acknowledge that I have access to the selected private key file (jschneid_nmdp.pem), and
that without this file, I won't be able to log into my instance.  
click **Launch Instances**


## Install Sandbox (on EC2 Instance)

See also:  https://github.com/smart-on-fhir/installer

Log into the EC2 instance.

```
ssh jschneider@jump.b12x.org
ssh -i ~/aws/jschneid_nmdp.pem ubuntu@ec2-35-168-14-94.compute-1.amazonaws.com
```

Install extra Ubuntu packages.

```
sudo apt update
sudo apt -y upgrade
sudo apt -y install curl git python-pycurl python-pip python-yaml \
 python-paramiko python-jinja2 unzip libwww-perl libdatetime-perl
```

Upgrade PIP, and install the correct version of Ansible.

```
sudo pip install --upgrade pip
sudo pip install ansible==2.3.2.0
```

Clone the cibmtr-smart-ansible-public git repository (and verify the active
branch is cibmtr/develop/2.4.2).

```
git clone https://github.com/jschneid-nmdp/cibmtr-smart-ansible-public.git
cd cibmtr-smart-ansible-public
git branch
\* cibmtr/develop/2.4.2
```

(Optional) Add upstream and upstream-vendor git remotes.

```
git remote -v
git remote add upstream https://github.com/nmdp-bioinformatics/cibmtr-smart-ansible-public.git
git remote add upstream-vendor https://github.com/smart-on-fhir/installer.git
```

Switch to **aws-setup** branch.

```
git checkout aws-setup
```

Download the Ansible Galaxy roles.

```
ansible-galaxy install -r roles/requirements.yml -p ./roles/ --force
```

Tweak part of downloaded common-role.

```
mv roles/common-role/templates/nginx_gateway_template.j2 \
  roles/common-role/templates/nginx_gateway_template.j2.orig
sed "s/use_secure_http/use_secure_http and not using_aws_elb/" \
  roles/common-role/templates/nginx_gateway_template.j2.orig \
  > roles/common-role/templates/nginx_gateway_template.j2
rm roles/common-role/templates/nginx_gateway_template.j2.orig
```

Define ELB setup and corresponding DNS entries (see sections below).

Run the site.yml playbook for the aws-dev environment.

```
ansible-playbook site.yml -i "localhost," -c local \
  --extra-vars "env=aws-dev installer_user=ubuntu services_host=localhost"
```

## Security Groups

### cibmtr-smart-ansible-elb

Group name:  cibmtr-smart-ansible-elb  
Description:  load balancer for (Ansible-based) standalone SMART sandbox  
VPC:  vpc-2b2fef4a | default-vpc

_Inbound_  
_Type_ / _Protocol_ / _Port Range_ / _Source_ / _Description_  
HTTPS / TCP / 443 / Custom / 0.0.0.0/0 /

_Outbound_  
_Type_ / _Protocol_ / _Port Range_ / _Destination_ / _Description_  
Custom TCP Rule / TCP / 8060 / 0.0.0.0/0 / Auth Server  
Custom TCP Rule / TCP / 8070 - 8099 / 0.0.0.0/0 / Sandbox Manager & SMART Apps

_Tags_  
_Key_ / _Value_  
Name / cibmtr-smart-ansible-elb  
Purpose / cibmtr:smart-sandbox  
Product / EC2  
Project Name / AGNIS on FHIR  
Project Charge Number / BC-N3Y-2D11-00-AGNIS  
Environment / prod

Note:  The assigned ID of cibmtr-smart-ansible-elb security group is sg-1625505e.

### cibmtr-smart-ansible

Group name:  cibmtr-smart-ansible  
Description:  access to (Ansible-based) standalone SMART sandbox  
VPC:  vpc-2b2fef4a | default-vpc

_Inbound_  
_Type_ / _Protocol_ / _Port Range_ / _Source_ / _Description_  
SSH / TCP / 22 / 192.149.74.10/32 / SSH from NMDP  
SSH / TCP / 22 / 198.175.249.8/32 / SSH from NMDP  
Custom TCP Rule / TCP / 9060 / sg-1625505e / Auth Server  
Custom TCP Rule / TCP / 9070 - 9099 / sg-1625505e / Sandbox Manager & SMART Apps

_Outbound_  
_Type_ / _Protocol_ / _Port Range_ / _Destination_ / _Description_  
All traffic / All / 0 - 65535 / Custom / 0.0.0.0/0 /

_Tags_  
_Key_ / _Value_  
Name / cibmtr-smart-ansible  
Purpose / cibmtr:smart-sandbox  
Product / EC2  
Project Name / AGNIS on FHIR  
Project Charge Number / BC-N3Y-2D11-00-AGNIS  
Environment / prod


## DNS Names

Use AWS Route 53 to define CNAME for each ELB instance.

### ELB CNAME Records

cibmtr-smart-dev-auth.b12x.org.  
cibmtr-smart-dev-apps.b12x.org.  
cibmtr-smart-dev-dstu2.b12x.org.  
cibmtr-smart-dev-pwm.b12x.org.  
cibmtr-smart-dev-sandman.b12x.org.  
cibmtr-smart-dev-stu3.b12x.org.

### Example ELB CNAME Creation

Route 53  
click **Create Record Set**  
Name:  cibmtr-smart-dev-auth [.b12x.org.]  
Type:  CNAME - Canonical name  
Alias:  No
TTL (Seconds):  3600  
Value:  cibmtr-smart-dev-auth-560736173.us-east-1.elb.amazonaws.com  
Routing Policy:  Simple
click **Create**


## ELB Setup For Sandbox Servers

Use AWS EC2 console to define ELB instance for each exposed service.

### Auth Server

(DNS Name:  cibmtr-smart-dev-auth.b12x.org)  
ELB Name:  cibmtr-smart-dev-auth  
Protocol:  HTTP  
Port:  8060  

### DSTU2 API Server

(DNS Name:  cibmtr-smart-dev-dstu2.b12x.org)  
ELB Name:  cibmtr-smart-dev-dstu2  
Protocol:  HTTP  
Port:  8071  

### STU3 API Server

(DNS Name:  cibmtr-smart-dev-stu3.b12x.org)  
ELB Name:  cibmtr-smart-dev-stu3  
Protocol:  HTTP  
Port:  8074  

### Sandbox Manager

(DNS Name:  cibmtr-smart-dev-sandman.b12x.org)  
ELB Name:  cibmtr-smart-dev-sandman  
Protocol:  HTTP  
Port:  8080  

### SMART Apps

(DNS Name:  cibmtr-smart-dev-pwm.b12x.org)  
ELB Name:  cibmtr-smart-dev-pwm  
Protocol:  HTTP  
Port:  8092  

### SMART Apps

(DNS Name:  cibmtr-smart-dev-apps.b12x.org)  
ELB Name:  cibmtr-smart-dev-apps  
Protocol:  HTTP  
Port:  8093  

### ELB Setup Example

See also:  https://docs.aws.amazon.com/elasticloadbalancing/latest/application/application-load-balancer-getting-started.html

Open AWS console for EC2  
click **Load Balancers** link  
click **Create Load Balancers** button  
under _Application Load Balancer_, click **Create** button

_Step 1: Configure Load Balancer_  
Name:  cibmtr-smart-dev-auth  
Scheme:  internet-facing  
IP Address Type:  ipv4

_Listeners_  
Load Balancer Protocol / Load Balancer Port  
HTTPS / 443

_Availability Zones_  
VPC: vpc-2b2fef4a (10.223.0.0/16) | default-vpc  
_Availability Zone_ / _Subnet ID_ / _Subnet IPv4 CIDR_ / _Name_  
us-east-1a / subnet-31911e47 / 10.223.13.0/24 / dev-apps-subnet-1  
us-east-1b / subnet-1e6bcd46 / 10.223.14.0/24 / dev-apps-subnet-2

_Tags_  
_Key_ / _Value_  
Purpose / cibmtr:smart-sandbox  
Product / EC2  
Project Name / AGNIS on FHIR  
Project Charge Number / BC-N3Y-2D11-00-AGNIS  
Environment / prod

click **Next: Configure Security Settings** button

_Step 2: Configure Security Setting_  
_Select default certificate_  
Certificate type:  Choose a certificate from ACM (recommended)  
Certificate name:  \*.b12x.org (arn:aws:acm:us-east-1:682793961433:certificate/8e017c31-7fe0-4648-989d-393891f4298e)  
Security policy:  ELBSecurityPolicy-TLS-1-1-2017-01  
click **Next: Configure Security Groups**

_Step 3: Configure Security Groups_  
_Security_ / _Name_ / _Description_  
sg-1625505e / cibmtr-smart-ansible-elb / load balancer for (Ansible-based) standalone SMART sandbox services  
click **Next: Configure Routing**

_Step 4: Configure Routing_  
_Target group_  
Target group:  New target group  
Name:  cibmtr-smart-dev-auth  
Protocol:  HTTP  
Port:  8060  
Target type:  instance

_Health checks_  
Protocol:  HTTP  
Path:  /

_Advanced health check settings_  
Port:  traffic port  
Healthy threshold:  5  
Unhealthy threshold:  2  
Timeout:  5 seconds  
Interval:  30 seconds  
Success codes:  200  
click **Next: Register Targets**

_Step 5: Register Targets_  
_Registered targets_  
_Instance_ / _Name_ / _Port_ / _State_ / _Security groups_ / _Zone_  
i-0bb517ecbd4c32e62 / cibmtr-smart-dev / 8060 / running / default-ssh-nmdp-aws, cibmtr-smart-sandbox-... / us-east-1b  
(select instance and click **Add to registered**)  
click **Next: Review**

_Step 6: Review_  
click **Create**
