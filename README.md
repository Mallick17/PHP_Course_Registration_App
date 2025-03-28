# Documentation and Deployment of PHP Application
Below is the organized documentation of the steps to set up a PHP application (Online Course Registration App) on AWS, including creating a VPC, DB subnet groups, RDS, updating the RDS security group, launching an EC2 instance, and configuring the instance to work with the VPC and RDS. The steps are formatted as requested, with reasons provided after each major step. The steps align with your executed commands, handwritten notes (labeled as steps 1 through 10 in the images), and the additional context of manually creating a DB subnet group due to the missing modal in the RDS creation process.

---

### Documentation: Deploying a PHP Application with VPC, RDS, and EC2 on AWS

#### Step 1: Create a Custom VPC
1. **Navigate to VPC**
   - In the AWS Management Console, search for "VPC" and select it.

2. **Create VPC**
   - Click **Create VPC**.
   - **Name**: `php-mallow-vpc`.
   - **IPv4 CIDR block**: `10.0.0.0/16`.
   - **IPv6 CIDR block**: None.
   - **Tenancy**: Default.
   - **Enable DNS hostnames**: Yes.
   - Click **Create VPC**.

**Reason**:
- A custom VPC provides a logically isolated network environment for your resources, allowing you to define your own IP address range and subnet structure. The CIDR block `10.0.0.0/16` ensures enough IP addresses for multiple subnets. Enabling DNS hostnames allows instances in the VPC to resolve domain names (e.g., the RDS endpoint) without additional configuration.

---

#### Step 2: Create Subnets
1. **Navigate to Subnets**
   - In the VPC dashboard, select **Subnets** from the left sidebar.

2. **Create Public Subnets**
   - Click **Create subnet**.
   - **VPC**: `php-mallow-vpc`.
   - **Subnet 1**:
     - Name: `Public-1a`.
     - Availability Zone: `ap-south-1a`.
     - IPv4 CIDR block: `10.0.1.0/24`.
   - **Subnet 2**:
     - Name: `Public-1b`.
     - Availability Zone: `ap-south-1b`.
     - IPv4 CIDR block: `10.0.2.0/24`.
   - Click **Create subnet**.
   - Enable auto-assign public IP for both subnets:
     - Select `Public-1a`, click **Actions** > **Edit subnet settings**, enable **Auto-assign public IPv4 address**, and save.
     - Repeat for `Public-1b`.

3. **Create Private Subnets**
   - Click **Create subnet**.
   - **VPC**: `php-mallow-vpc`.
   - **Subnet 3**:
     - Name: `Private-1a`.
     - Availability Zone: `ap-south-1a`.
     - IPv4 CIDR block: `10.0.3.0/24`.
   - **Subnet 4**:
     - Name: `Private-1b`.
     - Availability Zone: `ap-south-1b`.
     - IPv4 CIDR block: `10.0.4.0/24`.
   - Click **Create subnet**.

**Reason**:
- Subnets divide the VPC’s IP range into smaller segments, allowing you to place resources in different Availability Zones (AZs) for high availability. Public subnets (`Public-1a`, `Public-1b`) are for resources that need internet access (e.g., the EC2 instance hosting the PHP app). Private subnets (`Private-1a`, `Private-1b`) are for resources that should not be publicly accessible (e.g., the RDS database). Enabling auto-assign public IP for public subnets ensures EC2 instances in these subnets can communicate with the internet.

---

#### Step 3: Create an Internet Gateway
1. **Navigate to Internet Gateways**
   - In the VPC dashboard, select **Internet Gateways** from the left sidebar.

2. **Create Internet Gateway**
   - Click **Create internet gateway**.
   - **Name**: `php-mallow-IGW`.
   - Click **Create**.

3. **Attach to VPC**
   - Select `php-mallow-IGW`, click **Actions** > **Attach to VPC**.
   - **VPC**: `php-mallow-vpc`.
   - Click **Attach**.

**Reason**:
- An Internet Gateway allows resources in the VPC (e.g., EC2 instances in public subnets) to connect to the internet and be accessible from the internet. This is necessary for the PHP application server to serve web traffic and for you to SSH into the EC2 instance.

---

#### Step 4: Create Route Tables
1. **Navigate to Route Tables**
   - In the VPC dashboard, select **Route Tables** from the left sidebar.

2. **Create Public Route Table**
   - Click **Create route table**.
   - **Name**: `php-mallow-public-RT`.
   - **VPC**: `php-mallow-vpc`.
   - Click **Create**.
   - Add route:
     - Select `php-mallow-public-RT`, go to **Routes** tab, click **Edit routes**.
     - Add: Destination `0.0.0.0/0`, Target `php-mallow-IGW`.
     - Click **Save routes**.
   - Associate subnets:
     - Go to **Subnet associations** tab, click **Edit subnet associations**.
     - Select `Public-1a` and `Public-1b`.
     - Click **Save associations**.

3. **Create Private Route Table**
   - Click **Create route table**.
   - **Name**: `php-mallow-private-RT`.
   - **VPC**: `php-mallow-vpc`.
   - Click **Create**.
   - Associate subnets:
     - Go to **Subnet associations** tab, click **Edit subnet associations**.
     - Select `Private-1a` and `Private-1b`.
     - Click **Save associations**.

**Reason**:
- Route tables control the traffic flow within the VPC. The public route table routes internet-bound traffic (`0.0.0.0/0`) to the Internet Gateway, allowing instances in public subnets to access the internet. The private route table does not have an internet route, ensuring instances in private subnets (e.g., RDS) are isolated from the internet. Associating subnets with the correct route tables ensures proper network behavior.

---

#### Step 5: Create a NAT Gateway
1. **Navigate to NAT Gateways**
   - In the VPC dashboard, select **NAT Gateways** from the left sidebar.

2. **Create NAT Gateway**
   - Click **Create NAT gateway**.
   - **Name**: `php-mallow-NATGW`.
   - **Subnet**: `Public-1a`.
   - **Elastic IP**: Allocate a new Elastic IP.
   - Click **Create NAT gateway**.

3. **Update Private Route Table**
   - Go to **Route Tables**, select `php-mallow-private-RT`.
   - Go to **Routes** tab, click **Edit routes**.
   - Add: Destination `0.0.0.0/0`, Target `php-mallow-NATGW`.
   - Click **Save routes**.

**Reason**:
- A NAT Gateway allows instances in private subnets (e.g., RDS) to access the internet for updates or backups without being publicly accessible. Placing the NAT Gateway in a public subnet ensures it can route traffic to the internet via the Internet Gateway. Updating the private route table ensures outbound internet traffic from private subnets goes through the NAT Gateway.

---

#### Step 6: Manually Create a DB Subnet Group
1. **Navigate to Subnet Groups**
   - In the AWS Management Console, go to the RDS dashboard.
   - Select **Subnet groups** from the left sidebar.

2. **Create a New DB Subnet Group**
   - Click **Create DB subnet group**.
   - **Name**: `php-mallow-db-subnet-group`.
   - **Description**: (Optional) "Subnet group for RDS in private subnets".
   - **VPC**: `php-mallow-vpc`.
   - **Add Subnets**:
     - Select `Private-1a` (`10.0.3.0/24`, `ap-south-1a`).
     - Select `Private-1b` (`10.0.4.0/24`, `ap-south-1b`).
     - Do not include public subnets (`Public-1a`, `Public-1b`).
   - Click **Create**.

**Reason**:
- The DB subnet group specifies which subnets the RDS instance can use, ensuring it’s deployed in private subnets (`Private-1a`, `Private-1b`) to isolate it from public access. Creating this manually is necessary because the modal to create a new DB subnet group during RDS creation did not appear, likely due to a UI glitch.

---

#### Step 7: Launch RDS in Private Subnet
1. **Navigate to RDS**
   - In the AWS Management Console, search for "RDS" and select it.

2. **Create Database**
   - Click **Create database** > **Standard create** > **MySQL**.
   - **Version**: MySQL 8.0.41.
   - **Templates**: Free Tier (Single-AZ).
   - **DB instance identifier**: `db-php-mallow`.
   - **Master username**: `admin`.
   - **Master password**: `MySecurePass123` (replace with your actual password).
   - **DB instance class**: `db.t3.micro`.
   - **Storage**: 20 GB.

3. **Configure VPC Settings**
   - **VPC**: `php-mallow-vpc`.
   - **Subnet group**: Select `php-mallow-db-subnet-group` (created in Step 6).
   - **Public access**: No (since it’s in a private subnet).
   - **VPC security group**: Create new (`db-php-mallow-sg`).
   - **Availability Zone**: No preference.

4. **Proceed to Additional Configuration**
   - **Database authentication**: "Password authentication" (default).
   - **Monitoring**: Leave defaults (no enhanced monitoring).
   - **Log exports**: Optional (e.g., enable "Error log" if desired).

5. **Create the Database**
   - Scroll to the bottom and click **Create database**.
   - Wait 5-10 minutes for the RDS instance (`db-php-mallow`) to become available.
   - Note the **Endpoint** (e.g., `db-php-mallow.xxxxx.ap-south-1.rds.amazonaws.com`) and **Port** (3306) from the RDS dashboard.

**Reason**:
- The RDS instance is placed in private subnets to ensure it’s not publicly accessible, enhancing security. The Free Tier template minimizes costs. Setting "Public access" to "No" ensures the RDS instance is only accessible within the VPC. The DB subnet group ensures deployment in private subnets. The new security group (`db-php-mallow-sg`) will be configured to allow access only from the PHP server.

---

#### Step 8: Update RDS Security Group
1. **Navigate to Security Groups**
   - In the VPC dashboard, select **Security Groups** from the left sidebar.

2. **Configure `db-php-mallow-sg`**
   - Select `db-php-mallow-sg`, go to **Inbound rules** > **Edit inbound rules**.
   - Delete the default rule:
     - Remove the rule allowing traffic from `122.165.71.106/32`.
   - Add a new rule:
     - **Type**: `MYSQL/Aurora`.
     - **Protocol**: `TCP`.
     - **Port range**: `3306`.
     - **Source**: `10.0.1.0/24` (CIDR of `Public-1a`).
     - **Description**: "Allow MySQL from PHP server".
   - Click **Save rules**.

**Reason**:
- The security group controls inbound traffic to the RDS instance. Allowing MySQL traffic (port 3306) from `10.0.1.0/24` ensures only the PHP server in `Public-1a` can connect to the RDS instance. Deleting the default rule removes unnecessary external access, aligning with the private subnet security model.

---

#### Step 9: Launch EC2 Instance in Public Subnet
1. **Navigate to EC2**
   - In the AWS Management Console, search for "EC2" and select it.

2. **Launch Instance**
   - Click **Launch instance**.
   - **Name**: `PHPServer`.
   - **AMI**: Amazon Linux 2 AMI.
   - **Instance type**: `t2.micro`.
   - **Key pair**: Create a new key pair (`mykey`).
   - **Network settings**:
     - VPC: `php-mallow-vpc`.
     - Subnet: `Public-1a`.
     - Auto-assign public IP: Enabled.
     - Security group: Create new (`web-sg`).
       - Inbound rules:
         - HTTP: Port 80, `0.0.0.0/0`.
         - SSH: Port 22, `0.0.0.0/0` (or your IP for security).
   - Click **Launch instance**.

**Reason**:
- The EC2 instance in `Public-1a` will host the PHP application and serve web traffic. `t2.micro` is Free Tier eligible, minimizing costs. Auto-assigning a public IP allows the instance to be accessible from the internet (for web access and SSH). The security group `web-sg` allows HTTP traffic (port 80) for the web application and SSH (port 22) for remote access.

---

#### Step 10: Configure EC2 Instance
1. **SSH into EC2**
   - ```bash
     ssh -i mykey.pem ec2-user@<public-ip>
     ```

2. **Update Instance**
   - ```bash
     sudo yum update -y
     ```

3. **Install Apache, PHP, and MySQL Client**
   - ```bash
     sudo amazon-linux-extras install -y php8.0
     sudo yum install -y httpd php php-mysqlnd mysql
     ```

4. **Start and Enable Apache**
   - ```bash
     sudo systemctl start httpd
     sudo systemctl enable httpd
     ```

5. **Clone the PHP Application**
   - ```bash
     sudo yum install git -y
     git clone https://github.com/Mallick17/PHP_Course_Registration_App.git
     mv PHP_Course_Registration_App phpapp
     mv phpapp/* /var/www/html/
     sudo chown -R apache:apache /var/www/html
     ```

**Reason**:
- Apache (`httpd`) serves the PHP application over HTTP. PHP 8.0 and `php-mysqlnd` are required to run the application and connect to MySQL. The MySQL client (`mysql`) is needed to interact with RDS (e.g., to import the SQL file). Starting and enabling Apache ensures the web server runs automatically on boot. Cloning the PHP application from GitHub provides the source code. Moving the files to `/var/www/html/` makes them accessible via Apache. Setting permissions (`apache:apache`) ensures Apache can read and write to the application files.

---

#### Step 11: Configure Database Connection and Set Up Database
1. **Locate and Edit `config.php`**
   - ```bash
     cd /var/www/html/includes
     sudo nano config.php
     ```
   - Update to:
     ```php
     <?php
     define('DB_SERVER', 'db-php-mallow.xxxxx.ap-south-1.rds.amazonaws.com');
     define('DB_USERNAME', 'admin');
     define('DB_PASSWORD', 'MySecurePass123');
     define('DB_NAME', 'onlinecourse');

     $conn = mysqli_connect(DB_SERVER, DB_USERNAME, DB_PASSWORD, DB_NAME);

     if ($conn == false) {
         die('Error: Cannot connect: ' . mysqli_connect_error());
     }
     ?>
     ```

2. **Set Permissions**
   - ```bash
     sudo chown apache:apache /var/www/html/includes/config.php
     sudo chmod 644 /var/www/html/includes/config.php
     ```

3. **Create Database on RDS**
   - ```bash
     mysql -h db-php-mallow.xxxxx.ap-south-1.rds.amazonaws.com -u admin -p
     ```
   - ```sql
     CREATE DATABASE onlinecourse;
     exit
     ```

4. **Import SQL File**
   - ```bash
     cd /var/www/html/sqlfile/
     mysql -h db-php-mallow.xxxxx.ap-south-1.rds.amazonaws.com -u admin -p onlinecourse < onlinecourse.sql
     ```

5. **Verify Database**
   - ```bash
     mysql -h db-php-mallow.xxxxx.ap-south-1.rds.amazonaws.com -u admin -p
     ```
   - ```sql
     USE onlinecourse;
     SHOW TABLES;
     SELECT * FROM admin;
     exit
     ```

6. **Restart Apache**
   - ```bash
     sudo systemctl restart httpd
     ```

7. **Test Application**
   - Access `http://<public-ip>` in a browser.
   - Login Details for Admin:
     - Username: `admin`.
     - Password: `Test@123`.
   - Login Details for Student:
     - Reg No.: `10806121`
     - Password: `Test@123`
     - Student Pincode for enroll Course Student: `822894`

**Reason**:
- Updating `config.php` with RDS credentials allows the PHP application to connect to the database. Setting permissions ensures Apache can read the configuration file. Creating the `onlinecourse` database on RDS prepares it for the application’s schema and data. Importing `onlinecourse.sql` sets up the necessary tables and initial data. Verifying the database ensures the import was successful. Restarting Apache applies the configuration changes. Testing the application confirms that the setup is complete and functional.

---

### Final Verification
- **Access the Application**: The PHP application should be accessible at `http://<public-ip>`, and you should be able to log in as an admin and perform actions like enrolling in a course.
- **Database Interaction**: Data added through the application should be stored in the RDS database, verifiable via:
  ```bash
  mysql -h db-php-mallow.xxxxx.ap-south-1.rds.amazonaws.com -u admin -p
  ```
  ```sql
  USE onlinecourse;
  SELECT * FROM students;
  ```

---

### Security and Best Practices
- **RDS Security**: The RDS instance is in a private subnet with "Public access" set to "No", and its security group only allows traffic from the PHP server’s subnet.
- **EC2 Security**: The EC2 instance’s security group allows HTTP (port 80) and SSH (port 22). For production, restrict SSH to your IP.
- **Application Security**: The PHP application uses MD5 for password hashing, which is insecure. For production, update to a secure hashing method like `password_hash()`.
- **HTTPS**: For production, enable HTTPS using an SSL certificate (e.g., via AWS Certificate Manager).

---

![collage](https://github.com/user-attachments/assets/3fc96d3e-fc95-4aec-b99f-0f3cf1edbfa4)

---

### Step-by-Step Guide to Delete Resources Created on March 27, 2025

#### Step 1: Identify Resources Created on March 27, 2025
1. **Use AWS CloudTrail to Confirm Creation Dates (Optional)**
   - Go to the AWS Management Console, search for "CloudTrail", and select it.
   - In the left sidebar, click **Event history**.
   - Filter events:
     - **Time range**: Set to March 27, 2025 (e.g., 00:00 to 23:59 UTC).
     - **Event name**: Look for events like `CreateVpc`, `RunInstances`, `CreateDBInstance`, `CreateSecurityGroup`, etc.
   - Note the resources created, such as:
     - VPC: `php-mallow-vpc`.
     - EC2 Instance: `PHPServer`.
     - RDS Instance: `db-php-mallow`.
     - Security Groups: `web-sg`, `db-php-mallow-sg`.
     - Subnets, Internet Gateway, Route Tables, etc.

   **Alternative**: If you’re certain all resources in your setup were created on March 27, 2025, you can skip CloudTrail and proceed based on the resource names from your documentation.

2. **List Resources in the Console**
   - We’ll manually check each service for the resources you created, using the names from your setup.

---

#### Step 2: Delete the RDS Instance (`db-php-mallow`)
1. **Navigate to RDS**
   - In the AWS Management Console, search for "RDS" and select it.

2. **Delete the RDS Instance**
   - Select **Databases** from the left sidebar.
   - Find `db-php-mallow` in the list.
   - Select `db-php-mallow`, click **Actions** > **Delete**.
   - **Settings**:
     - **Create final snapshot?**: Uncheck (since this is a test setup and you don’t need a backup).
     - **Acknowledge**: Check the box to confirm deletion.
   - Click **Delete**.
   - Wait for the RDS instance to be deleted (this may take a few minutes).

3. **Delete the DB Subnet Group**
   - In the RDS dashboard, select **Subnet groups** from the left sidebar.
   - Find `php-mallow-db-subnet-group`.
   - Select it, click **Delete**, and confirm.

**Reason**:
- The RDS instance (`db-php-mallow`) and its associated DB subnet group (`php-mallow-db-subnet-group`) were created on March 27, 2025, as part of your setup. Deleting them stops the RDS running cost (approximately $0.021 per hour for `db.t3.micro` in `ap-south-1`, or $15.12 per month) and removes associated resources.

---

#### Step 3: Terminate the EC2 Instance (`PHPServer`)
1. **Navigate to EC2**
   - In the AWS Management Console, search for "EC2" and select it.

2. **Terminate the EC2 Instance**
   - Select **Instances** from the left sidebar.
   - Find `PHPServer` in the list.
   - Select `PHPServer`, click **Instance state** > **Terminate instance**.
   - Confirm by clicking **Terminate**.
   - Wait for the instance to be terminated (status will change to **Terminated**).

**Reason**:
- The EC2 instance `PHPServer` was created on March 27, 2025, to host your PHP application. Terminating it stops the instance running cost (approximately $0.0116 per hour for `t2.micro` in `ap-south-1`, or $8.35 per month) and removes the instance from your account.

---

#### Step 4: Delete Security Groups (`web-sg` and `db-php-mallow-sg`)
1. **Navigate to Security Groups**
   - In the EC2 Dashboard, select **Security Groups** from the left sidebar (or go to the VPC dashboard and select **Security Groups**).

2. **Delete `web-sg`**
   - Find `web-sg` (created for the EC2 instance).
   - Select it, click **Actions** > **Delete security group**.
   - If it’s not deletable due to dependencies, ensure the EC2 instance `PHPServer` is terminated (Step 3). Then retry.
   - Confirm by clicking **Delete**.

3. **Delete `db-php-mallow-sg`**
   - Find `db-php-mallow-sg` (created for the RDS instance).
   - Select it, click **Actions** > **Delete security group**.
   - If it’s not deletable due to dependencies, ensure the RDS instance `db-php-mallow` is deleted (Step 2). Then retry.
   - Confirm by clicking **Delete**.

**Reason**:
- The security groups `web-sg` and `db-php-mallow-sg` were created on March 27, 2025, to control traffic to the EC2 instance and RDS instance, respectively. Deleting them removes unused resources and cleans up your account.

---

#### Step 5: Delete the VPC (`php-mallow-vpc`) and Associated Resources
To delete the VPC, you must first delete its dependencies (subnets, route tables, Internet Gateway, etc.).

1. **Delete Subnets**
   - In the VPC dashboard, select **Subnets** from the left sidebar.
   - Select the following subnets (created on March 27, 2025):
     - `Public-1a` (`10.0.1.0/24`).
     - `Public-1b` (`10.0.2.0/24`).
     - `Private-1a` (`10.0.3.0/24`).
     - `Private-1b` (`10.0.4.0/24`).
   - For each subnet, select it, click **Actions** > **Delete subnet**, and confirm.
   - If a subnet cannot be deleted due to dependencies (e.g., EC2 instance or RDS instance), ensure those resources are deleted (Steps 2 and 3).

2. **Delete Route Tables**
   - In the VPC dashboard, select **Route Tables** from the left sidebar.
   - Find `php-mallow-public-RT` and `php-mallow-private-RT`.
   - For each route table:
     - Select it, click **Actions** > **Delete route table**.
     - Confirm by clicking **Delete**.
   - Note: The default route table for the VPC cannot be deleted until the VPC is deleted.

3. **Detach and Delete the Internet Gateway**
   - In the VPC dashboard, select **Internet Gateways** from the left sidebar.
   - Find `php-mallow-IGW`.
   - Select it, click **Actions** > **Detach from VPC**, and confirm.
   - After detaching, select `php-mallow-IGW` again, click **Actions** > **Delete internet gateway**, and confirm.

4. **Delete the VPC**
   - In the VPC dashboard, select **Your VPCs** from the left sidebar.
   - Find `php-mallow-vpc`.
   - Select it, click **Actions** > **Delete VPC**.
   - Confirm by clicking **Delete**.
   - This will also delete the default route table and any remaining default resources associated with the VPC.

**Reason**:
- The VPC (`php-mallow-vpc`), subnets, route tables, and Internet Gateway were created on March 27, 2025, as part of your network setup. Deleting them removes all network resources and ensures no residual costs (e.g., from orphaned resources). You must delete dependencies (subnets, route tables, Internet Gateway) before deleting the VPC.

---

#### Step 6: Verify Deletion of NAT Gateway and Elastic IP
- In your previous step, you already deleted the NAT Gateway (`php-mallow-NATGW`) and released its associated Elastic IP. Let’s confirm they are removed:
  - **NAT Gateway**:
    - Go to **VPC** > **NAT Gateways**.
    - Ensure `php-mallow-NATGW` is not listed (or shows as **Deleted**).
  - **Elastic IP**:
    - Go to **EC2** > **Elastic IPs**.
    - Ensure the Elastic IP previously associated with `php-mallow-NATGW` is not listed.

**Reason**:
- The NAT Gateway and its Elastic IP were created on March 27, 2025, but you already deleted them to reduce costs. Verifying their deletion ensures no residual charges remain.

---

#### Step 7: Delete the Key Pair (`mykey`)
1. **Navigate to Key Pairs**
   - In the EC2 Dashboard, select **Key Pairs** from the left sidebar.

2. **Delete the Key Pair**
   - Find `mykey` (created for the EC2 instance `PHPServer`).
   - Select it, click **Actions** > **Delete**.
   - Confirm by clicking **Delete**.

**Reason**:
- The key pair `mykey` was created on March 27, 2025, to SSH into the EC2 instance. Since the instance is terminated, the key pair is no longer needed and can be deleted to clean up your account.

---

#### Step 8: Verify Deletion and Check for Residual Costs
1. **Verify Resource Deletion**
   - **RDS**: Ensure `db-php-mallow` and `php-mallow-db-subnet-group` are deleted.
   - **EC2**: Ensure `PHPServer` is terminated and `web-sg` is deleted.
   - **VPC**: Ensure `php-mallow-vpc`, subnets, route tables, and `php-mallow-IGW` are deleted.
   - **Security Groups**: Ensure `db-php-mallow-sg` is deleted.
   - **Key Pairs**: Ensure `mykey` is deleted.

2. **Check Billing Dashboard**
   - Go to the AWS Billing and Cost Management Dashboard.
   - Under **Bills**, check for charges related to EC2, RDS, VPC, or Elastic IPs.
   - Ensure no charges are accruing for the deleted resources.

**Reason**:
- Verifying deletion ensures all resources created on March 27, 2025, are removed, stopping all associated costs. Checking the billing dashboard confirms that no residual charges remain.

---

### Impact on Your Setup
- **PHP Application**: The PHP application hosted on `PHPServer` is no longer accessible because the EC2 instance is terminated.
- **RDS Database**: The RDS instance `db-php-mallow` and its data are deleted. If you did not create a final snapshot, the data cannot be recovered.
- **Network**: The VPC and all associated network resources (subnets, route tables, Internet Gateway) are deleted, leaving no network infrastructure in your account.
- **Cost Savings**:
  - **EC2 Instance (`t2.micro`)**: Saved $8.35 per month ($0.0116 per hour).
  - **RDS Instance (`db.t3.micro`)**: Saved $15.12 per month ($0.021 per hour).
  - **NAT Gateway and Elastic IP**: Already deleted, saving $36 per month (as calculated previously).
  - **Total Savings**: Approximately $59.47 per month (plus any minor charges for data transfer or storage).

---

### Additional Notes
- **CloudTrail for Precision**: If you have other resources in your account and are unsure about their creation dates, CloudTrail is the most reliable way to identify resources created on March 27, 2025. Alternatively, you can use the AWS CLI:
  ```bash
  aws ec2 describe-instances --query 'Reservations[*].Instances[*].[InstanceId, LaunchTime]' --output table
  aws rds describe-db-instances --query 'DBInstances[*].[DBInstanceIdentifier, DBInstanceStatus, DBCreationDateTime]' --output table
  ```
- **Snapshots or Backups**: If you need to recover the RDS data later, you should have created a final snapshot before deletion. Since this is a test setup, we skipped this step.
- **Other Regions**: Ensure you’re in the correct region (`ap-south-1`). If you created resources in other regions, repeat the process in those regions.

---

### Summary
- **Resources Deleted**: EC2 instance (`PHPServer`), RDS instance (`db-php-mallow`), DB subnet group (`php-mallow-db-subnet-group`), security groups (`web-sg`, `db-php-mallow-sg`), VPC (`php-mallow-vpc`), subnets, route tables, Internet Gateway (`php-mallow-IGW`), and key pair (`mykey`). The NAT Gateway and its Elastic IP were already deleted.
- **Cost Savings**: Approximately $59.47 per month by deleting all resources.
- **Impact**: Your entire setup (PHP application, RDS database, and network infrastructure) is removed, and no further charges will accrue.

All resources created on March 27, 2025, as part of your setup have been deleted. Let me know if you need assistance with recreating any resources or further cleanup!
