# Cloud Native WordPress on Alibaba Cloud
Quick start with Wordpress on Alibaba Cloud with cloud native features such as high availability, auto-scaling, etc.

You can access the tutorial artifact including deployment script (Terraform), related source code, sample data and instruction guidance from the github project:
[https://github.com/alibabacloud-howto/solution-cloud-native-web-hosting](https://github.com/alibabacloud-howto/solution-cloud-native-web-hosting)

More tutorial around Alibaba Cloud Database, please refer to:
[https://github.com/alibabacloud-howto/database](https://github.com/alibabacloud-howto/database)

---
### Project URL
[https://github.com/alibabacloud-howto/solution-cloud-native-web-hosting](https://github.com/alibabacloud-howto/solution-cloud-native-web-hosting)

---
### Architecture Overview
![image.png](https://github.com/alibabacloud-howto/solution-cloud-native-web-hosting/raw/main/images/archi.png)

---
### Index
- Deployment
  - [Terraform](https://github.com/alibabacloud-howto/solution-cloud-native-web-hosting#terraform)
- Run Demo
  - [Step 1: Install Apache HTTP Server and PHP on ECS](https://github.com/alibabacloud-howto/solution-cloud-native-web-hosting#step-1-install-apache-http-server-and-php-on-ecs)
  - [Step 2: Install and configure Wordpress on ECS](https://github.com/alibabacloud-howto/solution-cloud-native-web-hosting#step-2-install-and-configure-wordpress-on-ecs)
  - [Step 3: Configure Redis caching](https://github.com/alibabacloud-howto/solution-cloud-native-web-hosting#step-3-configure-redis-caching)
  - [Step 4 (Optional): Make custom ECS image for auto scaling](https://github.com/alibabacloud-howto/solution-cloud-native-web-hosting#step-4-optional-make-custom-ecs-image-for-auto-scaling)
  - [Step 5 (Optional): Setup Auto Scaling (ESS) for ECS auto scaling](https://github.com/alibabacloud-howto/solution-cloud-native-web-hosting#step-5-optional-setup-auto-scaling-ess-for-ecs-auto-scaling)
  - [Step 6 (Optional): Simulate fluctuating traffic to trigger auto scaling](https://github.com/alibabacloud-howto/solution-cloud-native-web-hosting#step-6-optional-simulate-fluctuating-traffic-to-trigger-auto-scaling)

---
### Deployment
#### Terraform
Use terraform to provision VPC, SLB, EIP, ESS, ECS, Redis and PolarDB instances that used in this solution against this .tf file:
[https://github.com/alibabacloud-howto/solution-cloud-native-web-hosting/blob/main/deployment/terraform/main.tf](https://github.com/alibabacloud-howto/solution-cloud-native-web-hosting/blob/main/deployment/terraform/main.tf)


For more information about how to use Terraform, please refer to this tutorial: [https://www.youtube.com/watch?v=zDDFQ9C9XP8](https://www.youtube.com/watch?v=zDDFQ9C9XP8)

---
### Run Demo

---
#### Step 1: Install Apache HTTP Server and PHP on ECS

- Logon to ECS via SSH, use the account root/N1cetest, the password has been predefined in Terraform script for this tutorial. If you changed the password, please use the correct password accordingly.

```bash
ssh root@<EIP_ECS>
```

There are 2 ways to get the EIP applied for the ECS in Terraform:
- Go to ECS web console:

![image.png](https://github.com/alibabacloud-howto/solution-cloud-native-web-hosting/raw/main/images/step1_0_1.png)

- In the command line, under the same folder of the terraform script file main.tf, open "terraform.tfstate":

![image.png](https://github.com/alibabacloud-howto/solution-cloud-native-web-hosting/raw/main/images/step1_0_2.png)

If you met this error when ssh to ECS, please go to **/Users/xxx/.ssh/known_hosts, VI to edit the file and remove the whole line with the EIP of the target ECS at the very beginning. After that, please SSH to log on again.**

![image.png](https://github.com/alibabacloud-howto/solution-cloud-native-web-hosting/raw/main/images/step1_1.png)


- Run the following command to install required utilities on the instance: 
```bash
yum install -y sysbench unzip zip dstat
```

   - Zip: a utility commonly used for compressing files and folders 
   - Unzip: a utility used for decompressing files and folders.  
   - Sysbench: a benchmarking tool used for system performance testing. 
   - Dstat: a monitoring tool that provides statistics about system performance.  



- Install Apache HTTP Server

Run the following command to install the Apache HTTP server:
```bash
yum -y install httpd
```


- Install PHP

List available versions of PHP:
```bash
dnf module list php
```
Most likely php 7.4 is included, so run the following commands to enable PHP 7.4 (Please make sure the PHP version is new to catch up the requirement of the Wordpress, otherwise Wordpress installation would fail possibly):
```bash
dnf module reset php
dnf module enable php:7.4 -y
dnf install -y php php-opcache php-gd php-curl php-mysqlnd
dnf install -y php-bcmath php-mbstring php-xmlwriter php-xmlreader php-cli php-ldap php-zip php-fileinfo
```
Then restart Apache HTTP Server:
```bash
service httpd restart
```
Configure to auto start httpd service when ECS restarting:
```bash
chmod +x /etc/rc.d/rc.local
vim /etc/rc.d/rc.local
```
Add the line at the end and save:
```bash
service httpd restart
```

![image.png](https://github.com/alibabacloud-howto/solution-cloud-native-web-hosting/raw/main/images/step1_1_1.png)


Create a PHP file to verify the PHP is working:
```bash
vim /var/www/html/info.php
```
Then input the following content in this info.php file, then save and exit.
```php
<?php
phpinfo();
?>
```
Then open the following URL in a Web browser (Note: Replace the <ECS_EIP> placeholder with the Elastic IP address of the ECS instance that you obtained previously): 
```php
http://<ECS_EIP>/info.php
```
If the following page appears, PHP is installed successfully.

![image.png](https://github.com/alibabacloud-howto/solution-cloud-native-web-hosting/raw/main/images/step1_2.png)

---
#### Step 2: Install and configure Wordpress on ECS
Create a folder, download the WordPress package to this folder, and extract the package. To do this, run the following commands in sequence:
```bash
mkdir -p /opt/WP  
cd /opt/WP 
wget https://wordpress.org/latest.tar.gz 
tar -xzvf latest.tar.gz 
```
Run the following commands in sequence to configure WordPress to access ApsaraDB for PolarDB:
```bash
cd /opt/WP/wordpress/ 
cp wp-config-sample.php wp-config.php 
vim wp-config.php 
```
Complete the database configurations as follows: 

| Setting | Value & description |
| --- | --- |
| DB_NAME | The name of the ApsaraDB for PolarDB database that you created.  In this tutorial, we use "wpdb", which is predefined in resource "alicloud_polardb_database" within Terraform script [https://github.com/alibabacloud-howto/solution-cloud-native-web-hosting/blob/main/deployment/terraform/main.tf](https://github.com/alibabacloud-howto/solution-cloud-native-web-hosting/blob/main/deployment/terraform/main.tf). |
| DB_USER | The user name of the database account you created. In this lab, we use "test_polardb" as predefined within Terraform script.  |
| DB_PASSWORD | The password of the database account you created within Terraform script. In this lab, we use "N1cetest" as predefined within Terraform script. |
| DB_HOST | The VPC-facing endpoint of the ApsaraDB for PolarDB cluster that you obtained previously. Do not include the port number.  Please use the Cluster endpoint of PolarDB. |

Endpoint on PolarDB web console:

![image.png](https://github.com/alibabacloud-howto/solution-cloud-native-web-hosting/raw/main/images/step2_1.png)

![image.png](https://github.com/alibabacloud-howto/solution-cloud-native-web-hosting/raw/main/images/step2_2.png)

Run the following commands in sequence to copy the wordpress folder to the /var/www/html/ path: 
```bash
cd /var/www/html 
cp -rf /opt/WP/wordpress/* /var/www/html/ 
```
Open the following URL in a Web browser to initialize WordPress: 
```bash
http://<ECS_EIP>
```
Note: Replace the <ECS_EIP> placeholder with the Elastic IP address of the ECS instance that you obtained previously. 


Then complete the settings and click "Install WordPress".

![image.png](https://github.com/alibabacloud-howto/solution-cloud-native-web-hosting/raw/main/images/step2_3.png)

Then the following page shows, which means the installation is success.

![image.png](https://github.com/alibabacloud-howto/solution-cloud-native-web-hosting/raw/main/images/step2_4.png)

---
#### Step 3: Configure Redis caching
Run the following commands in sequence to download the Redis object cache plugin and unzip the plugin package: 

```bash
cd /opt/WP
wget https://downloads.wordpress.org/plugin/redis-cache.2.0.18.zip 
unzip redis-cache.2.0.18.zip 
```

Run the following commands in sequence to copy the redis-cache folder to the ``/var/www/html/wp-content/plugins/`` path and configure WordPress to access ApsaraDB for Redis:
```bash
cp -rf redis-cache /var/www/html/wp-content/plugins/ 
vim /var/www/html/wp-config.php
```
Complete the settings as follows: 

| Setting | Value & Description |
| --- | --- |
| WP_REDIS_HOST | The internal endpoint of the ApsaraDB for Redis instance that you obtained previously. Such as r-xxxxx.redis.singapore.rds.aliyuncs.com |
| WP_REDIS_PORT | The port number.  |
| WP_REDIS_PASSWORD | The password for connecting to the instance.  |

```bash
// Redis settings
define( 'WP_REDIS_HOST', '<Redis URL>' );
define( 'WP_REDIS_CLIENT', 'predis' );
define( 'WP_REDIS_PORT', '6379' );
define( 'WP_REDIS_DATABASE', '0');
define( 'WP_REDIS_PASSWORD', 'test_redis:N1cetest' );
```

![image.png](https://github.com/alibabacloud-howto/solution-cloud-native-web-hosting/raw/main/images/step3_1.png)

Please MAKE SURE this Redis setting block is set at the first settings block of the wp-config.php file as shown in the image above.


Run the following command to copy the object-cache configuration file to the /var/www/html/wp-content/ path:
```bash
cp /var/www/html/wp-content/plugins/redis-cache/includes/object-cache.php /var/www/html/wp-content/ 
```
Log on to WordPress to enable Redis object cache. 

![image.png](https://github.com/alibabacloud-howto/solution-cloud-native-web-hosting/raw/main/images/step3_2.png)

In the left-side navigation pane, click Plugins. Find the Redis Object Cache plugin and click Activate.  
After the plugin is activated, click Settings. 

![image.png](https://github.com/alibabacloud-howto/solution-cloud-native-web-hosting/raw/main/images/step3_3.png)

Verify that the plugin status is Connected. Click Flush Cache to synchronize cache data to the ApsaraDB for Redis instance.

![image.png](https://github.com/alibabacloud-howto/solution-cloud-native-web-hosting/raw/main/images/step3_4.png)

Now, your cloud native Wordpress has been setup successfully. You can visit it via SLB EIP:
```php
http://<SLB_EIP>/
```

![image.png](https://github.com/alibabacloud-howto/solution-cloud-native-web-hosting/raw/main/images/step3_5.png)

---
#### Step 4 (Optional): Make custom ECS image for auto scaling
Follow these steps to create an image from the ECS instance: 

- Log on to the ECS console. Click "Images" in the left-side navigation pane, then Click "Create Now".

![image.png](https://github.com/alibabacloud-howto/solution-cloud-native-web-hosting/raw/main/images/step4_1.png)

- Select "Instance", select the target ECS that with WordPress installed previously, enter a name and a description for the image, and then click "Create". In this lab, the image name is wp_image. 

![image.png](https://github.com/alibabacloud-howto/solution-cloud-native-web-hosting/raw/main/images/step4_2.png)

- Wait until the image creation progress becomes 100%. 

![image.png](https://github.com/alibabacloud-howto/solution-cloud-native-web-hosting/raw/main/images/step4_3.png)

---
#### Step 5 (Optional): Setup Auto Scaling (ESS) for ECS auto scaling
Follow these steps to enable Alibaba Cloud Auto Scaling: 

- Log on to the Auto Scaling console. Click "Create Scaling Group".

![image.png](https://github.com/alibabacloud-howto/solution-cloud-native-web-hosting/raw/main/images/step5_1.png)

- Select "Create from Scratch" and click "Start Creation". 

![image.png](https://github.com/alibabacloud-howto/solution-cloud-native-web-hosting/raw/main/images/step5_2.png)

- In the "Create Scaling Group" section, complete the settings as follows: 

| Setting | Value & Description |
| --- | --- |
| Scaling Group Name   | wp_auto_scaling |
| Instance Configuration Source | Create from Scratch |
| Instance Removing Policy | For "Filter First", select "Earliest Instance Created Using Scaling Configuration". For "Then Remove from Results", select "Most Recent Created Instance". |
| Minimum Number of Instances  | 2 |
| Maximum Number of Instances  | 5 |
| Default Cooldown Time (Seconds)  | 300 |
| Network Type | VPC |
| Multi-zone Scaling Policy  | Balanced Distribution Policy  |
| Instance Reclaim Mode  | Release Mode |
| VPC | Select the VPC created before by Terraform |
| Select VSwitch | Select all the VSwitches created before by Terraform. There are 2 VSW. |
| Associate SLB Instance | Select the SLB created before by Terraform. |

![image.png](https://github.com/alibabacloud-howto/solution-cloud-native-web-hosting/raw/main/images/step5_3.png)

![image.png](https://github.com/alibabacloud-howto/solution-cloud-native-web-hosting/raw/main/images/step5_4.png)

![image.png](https://github.com/alibabacloud-howto/solution-cloud-native-web-hosting/raw/main/images/step5_5.png)

- Click "OK" to finish the creation of the scaling group.

![image.png](https://github.com/alibabacloud-howto/solution-cloud-native-web-hosting/raw/main/images/step5_6.png)

- Click "Add Scaling Configuration".

![image.png](https://github.com/alibabacloud-howto/solution-cloud-native-web-hosting/raw/main/images/step5_7.png)

- Complete the settings as following:

| Setting | Value & description |
| --- | --- |
| Billing Method | Pay-As-You-Go |
| Instance Type | ecs.g5.xlarge and ecs.c5.xlarge, or you can select other ECS class to the optional list for auto scaling |
| Image | Click "Custom Image" and select the "wp_image" image that you created previously. |
| Storage | Select "Ultra Disk" and "40 GiB" for the system disk. Click "Add Disk" and select "Ultra Disk" and "100 GiB" for the data disk. |
| Security Group | Select the security group that you created previously. |

![image.png](https://github.com/alibabacloud-howto/solution-cloud-native-web-hosting/raw/main/images/step5_8.png)

![image.png](https://github.com/alibabacloud-howto/solution-cloud-native-web-hosting/raw/main/images/step5_9.png)

- In the "Logon Credentials" section, select “Inherit Password from Image”. In the "Instance Name" field, enter a name for the instance. In this lab, we use WP. Then click "Preview".

![image.png](https://github.com/alibabacloud-howto/solution-cloud-native-web-hosting/raw/main/images/step5_10.png)

- In the "Scaling Configuration Name" field, enter a name for the scaling configuration. In this lab, we use wp_as_group. Then click "Create".

![image.png](https://github.com/alibabacloud-howto/solution-cloud-native-web-hosting/raw/main/images/step5_11.png)

- Wait until the success message appears. Click "Enable Configuration" and "OK" to enable the scaling configuration.

![image.png](https://github.com/alibabacloud-howto/solution-cloud-native-web-hosting/raw/main/images/step5_12.png)

![image.png](https://github.com/alibabacloud-howto/solution-cloud-native-web-hosting/raw/main/images/step5_13.png)

![image.png](https://github.com/alibabacloud-howto/solution-cloud-native-web-hosting/raw/main/images/step5_14.png)

- Click the "Instances", "Manually Added" tab and then click "Add Existing Instances".

![image.png](https://github.com/alibabacloud-howto/solution-cloud-native-web-hosting/raw/main/images/step5_15.png)

- Add the ECS that you installed WordPress previously.

![image.png](https://github.com/alibabacloud-howto/solution-cloud-native-web-hosting/raw/main/images/step5_16.png)

- Follow the following steps to create 2 scaling rules (ADD and DROP).

![image.png](https://github.com/alibabacloud-howto/solution-cloud-native-web-hosting/raw/main/images/step5_17.png)

| Setting | Value & description |
| --- | --- |
| Name | ADD1 |
| Rule Type | Simple Scaling Rule |
| Operation | Add 1 Instances |

![image.png](https://github.com/alibabacloud-howto/solution-cloud-native-web-hosting/raw/main/images/step5_18.png)

| Setting | Value & description |
| --- | --- |
| Name | DROP1 |
| Rule Type | Simple Scaling Rule |
| Operation | Remove 1 Instances |

![image.png](https://github.com/alibabacloud-howto/solution-cloud-native-web-hosting/raw/main/images/step5_19.png)

- Follow the following steps to create 2 event triggered taskes mapping to the scaling rules respectively (CPU busy to ADD, CPU idle to DROP).

![image.png](https://github.com/alibabacloud-howto/solution-cloud-native-web-hosting/raw/main/images/step5_20.png)

| Setting | Value & description |
| --- | --- |
| Task Name | cpu_busy |
| Resource Monitored | Select the "wp_auto_scaling" scaling group you created previously. |
| Monitoring Type | System Monitoring |
| Monitoring Metric | (ECS) CPU Utilization |
| Condition | Average >= Threshold 70% |
| Triggered Rule | Select the "ADD1" rule you created previously. |

![image.png](https://github.com/alibabacloud-howto/solution-cloud-native-web-hosting/raw/main/images/step5_21.png)

| Setting | Value & description |
| --- | --- |
| Task Name | cpu_idle |
| Resource Monitored | Select the "wp_auto_scaling" scaling group you created previously. |
| Monitoring Type | System Monitoring |
| Monitoring Metric | (ECS) CPU Utilization |
| Condition | Average <= Threshold 50% |
| Triggered Rule | Select the "DROP1" rule you created previously. |

![image.png](https://github.com/alibabacloud-howto/solution-cloud-native-web-hosting/raw/main/images/step5_22.png)

- Verify that the statuses of new scaling tasks are both **Normal**.

![image.png](https://github.com/alibabacloud-howto/solution-cloud-native-web-hosting/raw/main/images/step5_23.png)

- After for a while (since the default setting is "triggered after 3 times for each 1 minute reference period", so please be patient to wait at least 3 minutes), verify that the statuses of SLB and the backend servers are **Normal**. 1 new ECS server were scaled out by auto scaling service and attached to SLB instance for load balancing.

![image.png](https://github.com/alibabacloud-howto/solution-cloud-native-web-hosting/raw/main/images/step5_24.png)

---
#### Step 6 (Optional): Simulate fluctuating traffic to trigger auto scaling

- Modify the auto scaling event-triggered task "cpu_busy" with the condition to "Maximum >= 70%"

![image.png](https://github.com/alibabacloud-howto/solution-cloud-native-web-hosting/raw/main/images/step6_1.png)

![image.png](https://github.com/alibabacloud-howto/solution-cloud-native-web-hosting/raw/main/images/step6_2.png)

- Log on to the ECS with EIP bound before, and run the sysbench to simulate the high CPU workload

```bash
sysbench cpu --cpu-max-prime=4000000 --threads=4 --time=1000 run
```

- After for a while (since the default setting is "triggered after 3 times for each 1 minute reference period", so please be patient to wait at least 3 minutes), verify the auto scaling event triggered with a new instance added

![image.png](https://github.com/alibabacloud-howto/solution-cloud-native-web-hosting/raw/main/images/step6_3.png)

- After for a while, verify the SLB instance is attached with 3 ECS instances.

![image.png](https://github.com/alibabacloud-howto/solution-cloud-native-web-hosting/raw/main/images/step6_4.png)

- Then you can "ctrl + C" to stop the sysbench workload, and verify the ECS count backs to 2 following the "cpu_idle" auto scaling rule after the rule event is triggered.