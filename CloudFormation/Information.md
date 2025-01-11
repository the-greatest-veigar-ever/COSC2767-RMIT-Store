# 📄 Instruction

## 📚 **AWS CloudFormation Template Explanation**

### 🔍 **Overview**

This AWS CloudFormation template is designed to set up a **production environment** comprising a **custom Virtual Private Cloud (VPC)**, an **Application Load Balancer (ALB)**, **Front-End and Back-End EC2 instances**, and **Auto Scaling Groups (ASGs)** to ensure scalability and high availability. The template includes configurations for networking, security, load balancing, instance provisioning, and auto-scaling policies based on CPU utilization.

---

## 1. **Parameters**

Parameters allow you to input custom values to your CloudFormation template each time you create or update a stack. They make the template reusable and flexible.

| **Parameter**          | **Type**                     | **Default Value** | **Description**                                          |
|------------------------|------------------------------|--------------------|----------------------------------------------------------|
| `KeyName`              | `AWS::EC2::KeyPair::KeyName` | N/A                | Name of an existing EC2 KeyPair for SSH access.         |
| `ImageIdFrontEnd`      | `AWS::EC2::Image::Id`         | N/A                | AMI ID for the Front-End EC2 instances.                  |
| `ImageIdBackEnd`       | `AWS::EC2::Image::Id`         | N/A                | AMI ID for the Back-End EC2 instances.                   |
| `VpcCidr`              | `String`                      | `10.0.0.0/16`       | CIDR block for the custom VPC.                           |
| `PublicSubnet1Cidr`    | `String`                      | `10.0.1.0/24`       | CIDR block for Public Subnet 1.                          |
| `PublicSubnet2Cidr`    | `String`                      | `10.0.2.0/24`       | CIDR block for Public Subnet 2.                          |
| `TrustedSSHCIDR`       | `String`                      | `0.0.0.0/0`         | CIDR block for SSH access to EC2 instances. **⚠️ Important:** Replace with your specific IP/CIDR in production for enhanced security. |

---

## 2. **Resources**

### 2.1. **VPC and Networking**

#### a. **VPC (`ProdVPC`)**

- **Type:** `AWS::EC2::VPC`
- **Purpose:** Creates a Virtual Private Cloud to logically isolate your AWS resources.
- **Properties:**
  - **`CidrBlock`:** `10.0.0.0/16` — Defines the IP address range for the VPC.
  - **`EnableDnsSupport`:** `true` — Enables DNS resolution within the VPC.
  - **`EnableDnsHostnames`:** `true` — Assigns DNS hostnames to instances launched in the VPC.
- **Tags:** Name set to `ProdVPC`.

#### b. **Internet Gateway (`InternetGateway`)**

- **Type:** `AWS::EC2::InternetGateway`
- **Purpose:** Provides a gateway for the VPC to access the internet.
- **Tags:** Name set to `ProdInternetGateway`.

#### c. **VPC Gateway Attachment (`VPCGatewayAttachment`)**

- **Type:** `AWS::EC2::VPCGatewayAttachment`
- **Purpose:** Attaches the Internet Gateway to the VPC, enabling internet access.
- **Properties:**
  - **`VpcId`:** Reference to `ProdVPC`.
  - **`InternetGatewayId`:** Reference to `InternetGateway`.

#### d. **Public Route Table (`PublicRouteTable`)**

- **Type:** `AWS::EC2::RouteTable`
- **Purpose:** Manages routing for the VPC's public subnets.
- **Properties:**
  - **`VpcId`:** Reference to `ProdVPC`.
- **Tags:** Name set to `PublicRouteTable`.

#### e. **Public Route (`PublicRoute`)**

- **Type:** `AWS::EC2::Route`
- **Purpose:** Adds a default route (`0.0.0.0/0`) via the Internet Gateway for internet-bound traffic.
- **Properties:**
  - **`RouteTableId`:** Reference to `PublicRouteTable`.
  - **`DestinationCidrBlock`:** `0.0.0.0/0` — Represents all IPv4 addresses.
  - **`GatewayId`:** Reference to `InternetGateway`.
- **DependsOn:** `VPCGatewayAttachment` — Ensures the Internet Gateway is attached before creating the route.

#### f. **Public Subnets (`PublicSubnet1` & `PublicSubnet2`)**

- **Type:** `AWS::EC2::Subnet`
- **Purpose:** Defines two public subnets within the VPC, each in different Availability Zones for high availability.
- **Properties:**
  - **`VpcId`:** Reference to `ProdVPC`.
  - **`CidrBlock`:** `10.0.1.0/24` for Subnet 1 and `10.0.2.0/24` for Subnet 2.
  - **`MapPublicIpOnLaunch`:** `true` — Assigns public IPs to instances launched in these subnets.
  - **`AvailabilityZone`:** Automatically selects AZs using `!Select` and `!GetAZs`.
- **Tags:** Names set to `PublicSubnet1` and `PublicSubnet2`.

#### g. **Subnet Route Table Associations (`PublicSubnet1RouteTableAssociation` & `PublicSubnet2RouteTableAssociation`)**

- **Type:** `AWS::EC2::SubnetRouteTableAssociation`
- **Purpose:** Associates each public subnet with the `PublicRouteTable` to enable routing via the Internet Gateway.
- **Properties:**
  - **`SubnetId`:** Reference to the respective public subnet.
  - **`RouteTableId`:** Reference to `PublicRouteTable`.

---

### 2.2. **Security Groups**

Security Groups act as virtual firewalls controlling inbound and outbound traffic to your AWS resources.

#### a. **ALB Security Group (`ALBSecurityGroup`)**

- **Type:** `AWS::EC2::SecurityGroup`
- **Purpose:** Controls inbound traffic to the Application Load Balancer.
- **Properties:**
  - **`GroupDescription`:** "Allow inbound HTTP on port 8080 for the ALB"
  - **`VpcId`:** Reference to `ProdVPC`.
  - **`SecurityGroupIngress`:** 
    - **Port `8080`:** Allows TCP traffic from any IP (`0.0.0.0/0`).
  - **`SecurityGroupEgress`:**
    - **All Traffic:** Allows all outbound traffic.
- **Tags:** Name set to `ALBSG`.

#### b. **Front-End EC2 Security Group (`FrontEndSecurityGroup`)**

- **Type:** `AWS::EC2::SecurityGroup`
- **Purpose:** Controls inbound traffic to Front-End EC2 instances.
- **Properties:**
  - **`GroupDescription`:** "Security Group for Front-End EC2 Instance"
  - **`VpcId`:** Reference to `ProdVPC`.
  - **`SecurityGroupIngress`:**
    - **Port `8080`:** Allows TCP traffic from the ALB Security Group only.
    - **Port `22`:** Allows SSH access from the specified `TrustedSSHCIDR`.
  - **`SecurityGroupEgress`:**
    - **All Traffic:** Allows all outbound traffic.
- **Tags:** Name set to `FrontEndSG`.

#### c. **Back-End EC2 Security Group (`BackEndSecurityGroup`)**

- **Type:** `AWS::EC2::SecurityGroup`
- **Purpose:** Controls inbound traffic to Back-End EC2 instances.
- **Properties:**
  - **`GroupDescription`:** "Security Group for Back-End EC2 Instance"
  - **`VpcId`:** Reference to `ProdVPC`.
  - **`SecurityGroupIngress`:**
    - **Port `3000`:** Allows TCP traffic from the Front-End Security Group only.
    - **Port `27017`:** Allows TCP traffic from the Front-End Security Group only (MongoDB).
    - **Port `22`:** Allows SSH access from the specified `TrustedSSHCIDR`.
  - **`SecurityGroupEgress`:**
    - **All Traffic:** Allows all outbound traffic.
- **Tags:** Name set to `BackEndSG`.

---

### 2.3. **Load Balancer and Target Group**

#### a. **Application Load Balancer (`AppLoadBalancer`)**

- **Type:** `AWS::ElasticLoadBalancingV2::LoadBalancer`
- **Purpose:** Distributes incoming application traffic across multiple Front-End EC2 instances to ensure high availability and fault tolerance.
- **Properties:**
  - **`Name`:** "ProdALB" — Name of the ALB.
  - **`Subnets`:** References to `PublicSubnet1` and `PublicSubnet2` for deploying the ALB in multiple Availability Zones.
  - **`SecurityGroups`:** Attached to `ALBSecurityGroup` to control inbound traffic.
  - **`Scheme`:** `internet-facing` — The ALB is accessible from the internet.
  - **`Type`:** `application` — Specifies that this is an Application Load Balancer.
  - **`IpAddressType`:** `ipv4` — Specifies the IP address type.
- **Tags:** Name set to `ProdALB`.

#### b. **Target Group (`AppTargetGroup`)**

- **Type:** `AWS::ElasticLoadBalancingV2::TargetGroup`
- **Purpose:** Defines the targets (Front-End EC2 instances) to which the ALB will route incoming traffic.
- **Properties:**
  - **`Name`:** "ProdTG" — Name of the Target Group.
  - **`Port`:** `8080` — The port on which Front-End EC2 instances are listening.
  - **`Protocol`:** `HTTP` — Protocol used for routing.
  - **`VpcId`:** Reference to `ProdVPC`.
  - **`TargetType`:** `instance` — Specifies that targets are EC2 instances.
  - **`HealthCheckProtocol`:** `HTTP` — Protocol used for health checks.
  - **`HealthCheckPort`:** `'8080'` — Port used for health checks.
  - **`HealthCheckPath`:** `/` — Path for health check requests.
  - **`Matcher`:**
    - **`HttpCode`:** `'200'` — Expected HTTP response code for healthy targets.
  - **`HealthCheckIntervalSeconds`:** `30` — Time between health checks.
  - **`HealthCheckTimeoutSeconds`:** `10` — Timeout for health check responses.
  - **`HealthyThresholdCount`:** `2` — Number of consecutive successful health checks before considering a target healthy.
  - **`UnhealthyThresholdCount`:** `5` — Number of consecutive failed health checks before considering a target unhealthy.
- **Tags:** Name set to `ProdTG`.

#### c. **ALB Listener (`ALBListener`)**

- **Type:** `AWS::ElasticLoadBalancingV2::Listener`
- **Purpose:** Listens for incoming traffic on the ALB and forwards it to the specified Target Group.
- **Properties:**
  - **`LoadBalancerArn`:** Reference to `AppLoadBalancer`.
  - **`Protocol`:** `HTTP` — Protocol used for listening.
  - **`Port`:** `8080` — Port on which the ALB listens for incoming traffic.
  - **`DefaultActions`:**
    - **Type:** `forward` — Specifies that incoming traffic is forwarded.
    - **`TargetGroupArn`:** Reference to `AppTargetGroup` — The target group to which traffic is forwarded.

---

### 2.4. **Launch Templates**

Launch Templates define the configuration for EC2 instances, making it easier to manage and update instance settings.

#### a. **Front-End Launch Template (`FrontEndLaunchTemplate`)**

- **Type:** `AWS::EC2::LaunchTemplate`
- **Purpose:** Defines the configuration for Front-End EC2 instances launched by the Auto Scaling Group.
- **Properties:**
  - **`LaunchTemplateName`:** "FrontEndLaunchTemplate" — Name of the Launch Template.
  - **`LaunchTemplateData`:** Configuration data for the instances.
    - **`InstanceType`:** `t2.micro` — EC2 instance type (adjustable based on requirements).
    - **`ImageId`:** Reference to `ImageIdFrontEnd` — AMI ID for Front-End instances.
    - **`KeyName`:** Reference to `KeyName` — EC2 KeyPair for SSH access.
    - **`SecurityGroupIds`:**
      - Reference to `FrontEndSecurityGroup` — Associates the Security Group with the instances.
    - **`UserData`:** Base64-encoded script that runs on instance launch to set up the environment.
      - **Purpose of User Data Script:**
        - **Updates the system packages.**
        - **Installs necessary dependencies (e.g., Node.js, Git).**
        - **Clones the React.js application repository.**
        - **Installs Node.js packages and starts the application on port `8080`.**
      
      ```bash
      #!/bin/bash
      yum update -y
      # Install Front-End dependencies, e.g., Node.js, Git
      curl -sL https://rpm.nodesource.com/setup_14.x | bash -
      yum install -y nodejs git
      # Clone your repository
      git clone https://github.com/your-repo/react-app.git /home/ec2-user/react-app
      cd /home/ec2-user/react-app
      npm install
      npm start -- --port 8080
      # Ensure the application runs on port 8080
      ```

#### b. **Back-End Launch Template (`BackEndLaunchTemplate`)**

- **Type:** `AWS::EC2::LaunchTemplate`
- **Purpose:** Defines the configuration for Back-End EC2 instances launched by the Auto Scaling Group.
- **Properties:**
  - **`LaunchTemplateName`:** "BackEndLaunchTemplate" — Name of the Launch Template.
  - **`LaunchTemplateData`:** Configuration data for the instances.
    - **`InstanceType`:** `t2.micro` — EC2 instance type (adjustable based on requirements).
    - **`ImageId`:** Reference to `ImageIdBackEnd` — AMI ID for Back-End instances.
    - **`KeyName`:** Reference to `KeyName` — EC2 KeyPair for SSH access.
    - **`SecurityGroupIds`:**
      - Reference to `BackEndSecurityGroup` — Associates the Security Group with the instances.
    - **`UserData`:** Base64-encoded script that runs on instance launch to set up the environment.
      - **Purpose of User Data Script:**
        - **Updates the system packages.**
        - **Installs necessary dependencies (e.g., Node.js, Git, MongoDB).**
        - **Clones the Node.js application repository.**
        - **Installs Node.js packages and starts the application on port `3000`.**
      
      ```bash
      #!/bin/bash
      yum update -y
      # Install Back-End dependencies, e.g., Node.js, Git, MongoDB
      curl -sL https://rpm.nodesource.com/setup_14.x | bash -
      yum install -y nodejs git
      # Install MongoDB
      cat <<EOF | tee /etc/yum.repos.d/mongodb-org-4.4.repo
      [mongodb-org-4.4]
      name=MongoDB Repository
      baseurl=https://repo.mongodb.org/yum/amazon/2/mongodb-org/4.4/x86_64/
      gpgcheck=1
      enabled=1
      gpgkey=https://www.mongodb.org/static/pgp/server-4.4.asc
      EOF
      yum install -y mongodb-org
      systemctl start mongod
      systemctl enable mongod
      # Clone your repository
      git clone https://github.com/your-repo/node-backend.git /home/ec2-user/node-backend
      cd /home/ec2-user/node-backend
      npm install
      npm start -- --port 3000
      # Ensure the application runs on port 3000
      ```

---

### 2.5. **Auto Scaling Groups (ASGs)**

Auto Scaling Groups manage the scaling of EC2 instances based on defined policies to handle varying loads.

#### a. **Front-End Auto Scaling Group (`FrontEndAutoScalingGroup`)**

- **Type:** `AWS::AutoScaling::AutoScalingGroup`
- **Purpose:** Manages the scaling of Front-End EC2 instances.
- **Properties:**
  - **`AutoScalingGroupName`:** "FrontEndASG" — Name of the ASG.
  - **`LaunchTemplate`:**
    - **`LaunchTemplateId`:** Reference to `FrontEndLaunchTemplate`.
    - **`Version`:** Latest version of the Launch Template.
  - **`MinSize`:** `'1'` — Minimum number of instances.
  - **`DesiredCapacity`:** `'1'` — Initial number of instances.
  - **`MaxSize`:** `'3'` — Maximum number of instances.
  - **`VPCZoneIdentifier`:**
    - References to `PublicSubnet1` and `PublicSubnet2` — Specifies subnets for instance placement.
  - **`TargetGroupARNs`:**
    - Reference to `AppTargetGroup` — Associates the ASG with the Target Group for load balancing.
  - **`Tags`:**
    - **`Name`:** "FrontEndASG" — Tag name for identification.
    - **`PropagateAtLaunch`:** `true` — Ensures tags are applied to instances launched by the ASG.

#### b. **Back-End Auto Scaling Group (`BackEndAutoScalingGroup`)**

- **Type:** `AWS::AutoScaling::AutoScalingGroup`
- **Purpose:** Manages the scaling of Back-End EC2 instances.
- **Properties:**
  - **`AutoScalingGroupName`:** "BackEndASG" — Name of the ASG.
  - **`LaunchTemplate`:**
    - **`LaunchTemplateId`:** Reference to `BackEndLaunchTemplate`.
    - **`Version`:** Latest version of the Launch Template.
  - **`MinSize`:** `'1'` — Minimum number of instances.
  - **`DesiredCapacity`:** `'1'` — Initial number of instances.
  - **`MaxSize`:** `'3'` — Maximum number of instances.
  - **`VPCZoneIdentifier`:**
    - References to `PublicSubnet1` and `PublicSubnet2` — Specifies subnets for instance placement.
  - **`Tags`:**
    - **`Name`:** "BackEndASG" — Tag name for identification.
    - **`PropagateAtLaunch`:** `true` — Ensures tags are applied to instances launched by the ASG.

---

### 2.6. **Scaling Policies**

Scaling policies define the conditions under which the ASGs scale out (add instances) or scale in (remove instances) based on specified metrics.

#### a. **Front-End Scaling Policies**

- **Scale Out Policy (`FrontEndScaleOutPolicy`):**
  - **Type:** `AWS::AutoScaling::ScalingPolicy`
  - **Purpose:** Increases the number of Front-End instances when average CPU utilization exceeds 30%.
  - **Properties:**
    - **`PolicyName`:** "FrontEndScaleOut"
    - **`AutoScalingGroupName`:** Reference to `FrontEndAutoScalingGroup`.
    - **`PolicyType`:** "TargetTrackingScaling"
    - **`TargetTrackingConfiguration`:**
      - **`PredefinedMetricSpecification`:**
        - **`PredefinedMetricType`:** "ASGAverageCPUUtilization"
      - **`TargetValue`:** `30.0` — The target CPU utilization percentage.

- **Scale In Policy (`FrontEndScaleInPolicy`):**
  - **Type:** `AWS::AutoScaling::ScalingPolicy`
  - **Purpose:** Decreases the number of Front-End instances when average CPU utilization drops below 30%.
  - **Properties:**
    - **`PolicyName`:** "FrontEndScaleIn"
    - **`AutoScalingGroupName`:** Reference to `FrontEndAutoScalingGroup`.
    - **`PolicyType`:** "TargetTrackingScaling"
    - **`TargetTrackingConfiguration`:**
      - **`PredefinedMetricSpecification`:**
        - **`PredefinedMetricType`:** "ASGAverageCPUUtilization"
      - **`TargetValue`:** `30.0` — The target CPU utilization percentage.

#### b. **Back-End Scaling Policies**

- **Scale Out Policy (`BackEndScaleOutPolicy`):**
  - **Type:** `AWS::AutoScaling::ScalingPolicy`
  - **Purpose:** Increases the number of Back-End instances when average CPU utilization exceeds 30%.
  - **Properties:**
    - **`PolicyName`:** "BackEndScaleOut"
    - **`AutoScalingGroupName`:** Reference to `BackEndAutoScalingGroup`.
    - **`PolicyType`:** "TargetTrackingScaling"
    - **`TargetTrackingConfiguration`:**
      - **`PredefinedMetricSpecification`:**
        - **`PredefinedMetricType`:** "ASGAverageCPUUtilization"
      - **`TargetValue`:** `30.0` — The target CPU utilization percentage.

- **Scale In Policy (`BackEndScaleInPolicy`):**
  - **Type:** `AWS::AutoScaling::ScalingPolicy`
  - **Purpose:** Decreases the number of Back-End instances when average CPU utilization drops below 30%.
  - **Properties:**
    - **`PolicyName`:** "BackEndScaleIn"
    - **`AutoScalingGroupName`:** Reference to `BackEndAutoScalingGroup`.
    - **`PolicyType`:** "TargetTrackingScaling"
    - **`TargetTrackingConfiguration`:**
      - **`PredefinedMetricSpecification`:**
        - **`PredefinedMetricType`:** "ASGAverageCPUUtilization"
      - **`TargetValue`:** `30.0` — The target CPU utilization percentage.

**💡 *Note:***

- **Target Tracking Scaling:** This policy type automatically adjusts the ASG capacity to maintain the specified metric at the target value. In this case, when CPU utilization exceeds 30%, it adds instances, and when it drops below 30%, it removes instances.

---

## 3. **EC2 Instances**

#### a. **Front-End EC2 Instance (`FrontEndEC2`)**

- **Type:** `AWS::EC2::Instance`
- **Purpose:** Represents the Front-End component of your application, typically serving the React.js application.
- **Properties:**
  - **`InstanceType`:** `t2.micro` — Specifies the EC2 instance type.
  - **`ImageId`:** Reference to `ImageIdFrontEnd` — AMI ID for Front-End instances.
  - **`KeyName`:** Reference to `KeyName` — EC2 KeyPair for SSH access.
  - **`NetworkInterfaces`:**
    - **`AssociatePublicIpAddress`:** `true` — Assigns a public IP to the instance.
    - **`DeviceIndex`:** `0` — Primary network interface.
    - **`SubnetId`:** Reference to `PublicSubnet1` — Deploys the instance in the first public subnet.
    - **`GroupSet`:**
      - Reference to `FrontEndSecurityGroup` — Associates the Security Group with the instance.
  - **`Tags`:**
    - **`Name`:** "FrontEndEC2" — Tag name for identification.
  - **`UserData`:** *(Optional)* Script to configure the instance upon launch (already defined in the Launch Template for ASG).

#### b. **Back-End EC2 Instance (`BackEndEC2`)**

- **Type:** `AWS::EC2::Instance`
- **Purpose:** Represents the Back-End component of your application, typically running the Node.js server and MongoDB database.
- **Properties:**
  - **`InstanceType`:** `t2.micro` — Specifies the EC2 instance type.
  - **`ImageId`:** Reference to `ImageIdBackEnd` — AMI ID for Back-End instances.
  - **`KeyName`:** Reference to `KeyName` — EC2 KeyPair for SSH access.
  - **`NetworkInterfaces`:**
    - **`AssociatePublicIpAddress`:** `true` — Assigns a public IP to the instance.
    - **`DeviceIndex`:** `0` — Primary network interface.
    - **`SubnetId`:** Reference to `PublicSubnet2` — Deploys the instance in the second public subnet.
    - **`GroupSet`:**
      - Reference to `BackEndSecurityGroup` — Associates the Security Group with the instance.
  - **`Tags`:**
    - **`Name`:** "BackEndEC2" — Tag name for identification.
  - **`UserData`:** *(Optional)* Script to configure the instance upon launch (already defined in the Launch Template for ASG).

**💡 *Note:***

- **Launch Templates vs. Individual EC2 Instances:** In this template, individual EC2 instances (`FrontEndEC2` and `BackEndEC2`) are defined alongside Launch Templates and ASGs. **⚠️ Important:** Typically, when using Auto Scaling Groups, individual EC2 instances are **not** necessary unless they serve a specific purpose outside the ASG-managed instances. Consider removing these individual instances if they are redundant to avoid conflicts and unnecessary resource usage.

---

## 4. **Outputs**

Outputs provide key information about the resources created by the CloudFormation stack. They can be accessed easily after stack creation and used for integration with other services or for reference purposes.

| **Output**                      | **Description**                                         | **Value**                                     |
|---------------------------------|---------------------------------------------------------|-----------------------------------------------|
| `VPCId`                         | The ID of the newly created VPC.                       | Reference to `ProdVPC`.                       |
| `PublicSubnet1Id`               | ID of the first public subnet.                         | Reference to `PublicSubnet1`.                 |
| `PublicSubnet2Id`               | ID of the second public subnet.                        | Reference to `PublicSubnet2`.                 |
| `LoadBalancerDNS`               | DNS name of the Application Load Balancer.             | Attribute `DNSName` of `AppLoadBalancer`.     |
| `FrontEndAutoScalingGroupName`   | Name of the Front-End Auto Scaling Group.              | Reference to `FrontEndAutoScalingGroup`.      |
| `BackEndAutoScalingGroupName`    | Name of the Back-End Auto Scaling Group.               | Reference to `BackEndAutoScalingGroup`.       |

**💡 *Usage:***

- **LoadBalancerDNS:** Use this DNS name to access your application via the browser.
- **Auto Scaling Group Names:** Useful for monitoring, management, or integration with other services.

---

## 5. **Auto Scaling Configuration**

Auto Scaling ensures that your application can handle varying levels of traffic by automatically adjusting the number of EC2 instances in response to demand.

### 5.1. **Auto Scaling Groups (ASGs)**

- **Front-EndAutoScalingGroup (`FrontEndAutoScalingGroup`):**
  - **Purpose:** Manages the Front-End EC2 instances, ensuring optimal performance and availability.
  - **Configuration:**
    - **`MinSize`:** `1` — Ensures at least one instance is always running.
    - **`DesiredCapacity`:** `1` — Starts with one instance.
    - **`MaxSize`:** `3` — Scales out up to three instances based on demand.
    - **`VPCZoneIdentifier`:** Deployed across both public subnets.
    - **`TargetGroupARNs`:** Attached to `AppTargetGroup` to receive traffic from the ALB.
    - **`Tags`:** Propagates the `Name` tag to all launched instances for identification.

- **Back-EndAutoScalingGroup (`BackEndAutoScalingGroup`):**
  - **Purpose:** Manages the Back-End EC2 instances, ensuring optimal performance and availability.
  - **Configuration:**
    - **`MinSize`:** `1` — Ensures at least one instance is always running.
    - **`DesiredCapacity`:** `1` — Starts with one instance.
    - **`MaxSize`:** `3` — Scales out up to three instances based on demand.
    - **`VPCZoneIdentifier`:** Deployed across both public subnets.
    - **`Tags`:** Propagates the `Name` tag to all launched instances for identification.

### 5.2. **Scaling Policies**

Scaling policies determine when and how the ASGs should scale in or out based on the average CPU utilization.

- **Front-End Scaling Policies:**
  - **Scale Out Policy (`FrontEndScaleOutPolicy`):** Scales out by adding an instance when average CPU > 30%.
  - **Scale In Policy (`FrontEndScaleInPolicy`):** Scales in by removing an instance when average CPU < 30%.

- **Back-End Scaling Policies:**
  - **Scale Out Policy (`BackEndScaleOutPolicy`):** Scales out by adding an instance when average CPU > 30%.
  - **Scale In Policy (`BackEndScaleInPolicy`):** Scales in by removing an instance when average CPU < 30%.

**💡 *Note:***

- **Target Tracking Scaling:** These policies use target tracking, which automatically adjusts the ASG size to maintain the average CPU utilization at the specified target (30% in this case).

---

## 6. **Best Practices and Recommendations**

### 6.1. **Security Enhancements**

1. **Restrict SSH Access:**
   - **Action:** Update the `TrustedSSHCIDR` parameter to your specific IP address or a secure CIDR block.
   - **Example:** If your office IP is `203.0.113.5`, set `TrustedSSHCIDR` to `203.0.113.5/32`.
   - **Reason:** Prevents unauthorized SSH access to your EC2 instances, enhancing security.

2. **Use Private Subnets for Sensitive Components:**
   - **Benefit:** Isolates Back-End and Database instances from direct internet access, reducing attack surfaces.
   - **Action:** Modify the template to create private subnets and place Back-End EC2 instances and MongoDB within them. Implement NAT Gateways for outbound internet access if required.

3. **Implement HTTPS:**
   - **Purpose:** Secures data transmission between users and your application.
   - **Action:**
     - **Obtain SSL/TLS Certificate:** Use AWS Certificate Manager (ACM) to request a certificate.
     - **Configure ALB for HTTPS:** Add an HTTPS listener on port `443` to the ALB, associating it with the ACM certificate.
     - **Redirect HTTP to HTTPS:** Modify the HTTP listener to redirect traffic to HTTPS for secure communication.
     - **Update Security Groups:** Allow inbound traffic on port `443` in `ALBSecurityGroup`.

### 6.2. **Monitoring and Logging**

1. **Enable VPC Flow Logs:**
   - **Purpose:** Captures information about the IP traffic going to and from network interfaces in your VPC.
   - **Action:** Add a Flow Log resource to the template, specifying the VPC, traffic type, and destination (e.g., CloudWatch Logs or S3).

2. **Enable ALB Access Logs:**
   - **Purpose:** Records detailed information about requests sent to the ALB.
   - **Action:** Configure the ALB to store access logs in an S3 bucket by adding the necessary properties to the `AppLoadBalancer` resource.

3. **Utilize Amazon CloudWatch:**
   - **Purpose:** Monitors metrics, logs, and sets up alarms for your resources.
   - **Action:** 
     - **Metrics:** Monitor CPU utilization, network traffic, request counts, etc.
     - **Alarms:** Set up alarms to notify you of unusual activity or performance issues.
     - **Dashboards:** Create CloudWatch dashboards for a consolidated view of your application's health.

### 6.3. **Continuous Integration/Continuous Deployment (CI/CD)**

1. **Automate AMI Updates:**
   - **Purpose:** Ensures EC2 instances use the latest AMIs with up-to-date security patches.
   - **Action:** Integrate AWS Systems Manager (SSM) Parameter Store in your Jenkins pipeline to dynamically fetch and use the latest AMI IDs.

2. **Integrate with Jenkins Pipeline:**
   - **Purpose:** Automates the deployment process, reducing manual intervention and potential errors.
   - **Action:** Ensure your Jenkinsfile correctly references and passes all necessary parameters to the CloudFormation template.

### 6.4. **Infrastructure as Code Best Practices**

1. **Use AWS CloudFormation StackSets:**
   - **Benefit:** Deploys CloudFormation stacks across multiple AWS accounts and regions with a single template.
   - **Action:** Implement StackSets for multi-account or multi-region deployments if required.

2. **Implement Validation Tools:**
   - **Tools:** Use `cfn-lint` and other validation tools to ensure template correctness and adherence to best practices.
   - **Action:** Integrate these tools into your CI/CD pipeline to catch errors before deployment.

3. **Use Change Sets:**
   - **Purpose:** Allows you to preview changes to your stack before executing them.
   - **Action:** Utilize CloudFormation Change Sets to review and approve changes, ensuring controlled deployments.

---

## 7. **Summary of the Architecture**

### **1. Virtual Private Cloud (VPC):**
   - **ProdVPC:** The foundational networking layer isolating your AWS resources.
   - **Internet Gateway:** Enables internet access for the VPC.
   - **Public Subnets:** Two subnets across different Availability Zones to host load balancers and scalable EC2 instances.

### **2. Security:**
   - **Security Groups:**
     - **ALBSecurityGroup:** Controls inbound traffic to the ALB on port `8080`.
     - **FrontEndSecurityGroup:** Controls inbound traffic to Front-End EC2 instances on port `8080` and SSH access.
     - **BackEndSecurityGroup:** Controls inbound traffic to Back-End EC2 instances on ports `3000` and `27017`, and SSH access.

### **3. Load Balancing:**
   - **Application Load Balancer (ALB):** Distributes incoming HTTP traffic on port `8080` to Front-End EC2 instances.
   - **Target Group:** Defines the Front-End EC2 instances as targets for the ALB, performing health checks on port `8080`.

### **4. EC2 Instances and Launch Templates:**
   - **Launch Templates:** Define the configuration for Front-End and Back-End EC2 instances, including instance type, AMI ID, security groups, and startup scripts.
   - **User Data Scripts:** Automate the installation and configuration of application dependencies and services on instance launch.

### **5. Auto Scaling:**
   - **Auto Scaling Groups (ASGs):** Manage the number of Front-End and Back-End EC2 instances based on CPU utilization, ensuring scalability and high availability.
   - **Scaling Policies:** Automatically scale in or out the number of instances when CPU utilization crosses the 30% threshold.

### **6. Outputs:**
   - Provides essential information such as VPC ID, Subnet IDs, Load Balancer DNS, and Auto Scaling Group names for easy access and integration.

---

## 8. **Next Steps**

1. **Secure SSH Access:**
   - Update the `TrustedSSHCIDR` parameter with your specific IP/CIDR range to restrict SSH access.

2. **Implement HTTPS:**
   - Obtain an SSL/TLS certificate using AWS Certificate Manager.
   - Configure the ALB to handle HTTPS traffic.
   - Redirect HTTP traffic to HTTPS for secure communication.

3. **Enhance Monitoring and Logging:**
   - Enable VPC Flow Logs and ALB Access Logs.
   - Set up CloudWatch dashboards and alarms for real-time monitoring and alerts.

4. **Automate AMI Updates:**
   - Use AWS Systems Manager (SSM) Parameter Store to dynamically fetch the latest AMI IDs.
   - Integrate AMI updates into your Jenkins pipeline for seamless deployments.

5. **Optimize Auto Scaling Policies:**
   - Adjust scaling thresholds and policies based on application performance and usage patterns.
   - Implement additional metrics (e.g., request count) for more granular scaling control.

6. **Remove Redundant EC2 Instances:**
   - If using Auto Scaling Groups, consider removing the standalone `FrontEndEC2` and `BackEndEC2` instances to prevent unnecessary resource consumption and potential conflicts.

7. **Test the Deployment:**
   - Deploy the CloudFormation stack and verify that all resources are created as expected.
   - Test the auto-scaling functionality by simulating CPU load and observing scaling actions.

---

## 9. **Troubleshooting Tips**

1. **Stack Deployment Failures:**
   - **Check Events:** Review the CloudFormation stack events in the AWS Management Console to identify the point of failure.
   - **Resource Limits:** Ensure that your AWS account has sufficient limits for the resources being created (e.g., EC2 instances, Load Balancers).

2. **Auto Scaling Not Triggering:**
   - **Verify CloudWatch Metrics:** Ensure that CPU utilization metrics are being correctly reported.
   - **Check Scaling Policies:** Confirm that the scaling policies are correctly attached to the ASGs and that the target values are appropriate.

3. **Instances Not Receiving Traffic:**
   - **Security Groups:** Verify that the Security Groups allow the necessary inbound and outbound traffic.
   - **Target Group Health Checks:** Ensure that the health checks are passing, indicating healthy instances.
   - **Load Balancer Configuration:** Confirm that the ALB is correctly associated with the Target Group and that listeners are properly configured.

4. **SSH Access Issues:**
   - **Key Pair:** Ensure that the correct Key Pair is specified and that you have access to the corresponding private key.
   - **IP Restrictions:** Verify that your current IP address falls within the specified `TrustedSSHCIDR` range.

---

## 10. **Additional Resources**

- **AWS CloudFormation Documentation:** [https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html)
- **AWS Auto Scaling Documentation:** [https://docs.aws.amazon.com/autoscaling/ec2/userguide/what-is-amazon-ec2-auto-scaling.html](https://docs.aws.amazon.com/autoscaling/ec2/userguide/what-is-amazon-ec2-auto-scaling.html)
- **AWS Application Load Balancer Documentation:** [https://docs.aws.amazon.com/elasticloadbalancing/latest/application/introduction.html](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/introduction.html)
- **AWS EC2 Launch Templates:** [https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-launch-templates.html](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-launch-templates.html)
- **AWS Systems Manager Parameter Store:** [https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-parameter-store.html](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-parameter-store.html)
- **AWS Certificate Manager:** [https://docs.aws.amazon.com/acm/latest/userguide/acm-overview.html](https://docs.aws.amazon.com/acm/latest/userguide/acm-overview.html)
- **Jenkins AWS Pipeline Integration:** [https://www.jenkins.io/doc/book/pipeline/aws/](https://www.jenkins.io/doc/book/pipeline/aws/)

---

# 📝 **Conclusion**

This AWS CloudFormation template sets up a robust, scalable, and secure production environment tailored to your application's specific port requirements. By integrating Auto Scaling Groups with targeted scaling policies, your infrastructure can dynamically adjust to handle varying loads, ensuring both performance and cost-efficiency.

**Key Takeaways:**

- **Scalability:** Auto Scaling Groups automatically manage the number of EC2 instances based on CPU utilization, maintaining optimal performance.
- **High Availability:** Deploying resources across multiple Availability Zones and using a Load Balancer ensures that your application remains available even if one AZ faces issues.
- **Security:** Properly configured Security Groups and restricted SSH access enhance the security posture of your environment.
- **Automation:** Utilizing Launch Templates and Auto Scaling automates the provisioning and scaling of resources, reducing manual intervention and potential errors.
- **Monitoring:** Implementing comprehensive monitoring and logging ensures that you can maintain visibility into your infrastructure's health and performance.

**🚀 Ready for Deployment:** With this detailed understanding of your CloudFormation script, you are well-equipped to deploy, manage, and scale your production environment effectively. Always ensure to follow best practices and continuously monitor and update your infrastructure to adapt to evolving requirements and security standards.

If you have any further questions or need additional assistance with your AWS infrastructure or Jenkins pipeline configurations, feel free to reach out!
