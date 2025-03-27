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
   - Log in as admin:
     - Username: `admin`.
     - Password: `Test@123`.

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

This documentation organizes the steps in the requested format, with reasons provided after each major step, aligning with your executed commands and handwritten notes. Let me know if you need further adjustments!
