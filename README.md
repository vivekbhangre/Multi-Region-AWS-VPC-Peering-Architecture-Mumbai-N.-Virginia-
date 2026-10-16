# Multi-Region AWS VPC Peering Architecture & Implementation Guide (Mumbai & N. Virginia)

An end-to-end operational guide and architectural reference for setting up **Inter-Region VPC Peering** between AWS regions (**ap-south-1 / Mumbai** and **us-east-1 / N. Virginia**). This document covers high-availability VPC layout, route table configurations, Internet Gateway associations, security group definitions, and cross-region SSH verification.

---

## 🏛️ Architecture Overview

```text
+-----------------------------------------------------------------------------------+
| AWS Cloud                                                                         |
|                                                                                   |
|   +---------------------------------------+       +---------------------------+   |
|   | Mumbai VPC (28.0.0.0/16)              |       | Virginia VPC              |   |
|   |                                       |       | (28.200.0.0/16)           |   |
|   |   +------------------+                |       |                           |   |
|   |   | Internet Gateway |                |       |   +-------------------+   |   |
|   |   | (Mumbai-IGW)     |                |       |   | Private Subnet    |   |   |
|   |   +--------+---------+                |       |   | (28.200.1.0/24)   |   |   |
|   |            |                          |       |   | [Private Server]  |   |   |
|   |   +--------v---------+                |       |   +---------+---------+   |   |
|   |   | Public Subnet    |                |       |             ^             |   |
|   |   | (28.0.1.0/24)    |                |       |             |             |   |
|   |   | [EC2 Jump Host]  |                |       |             |             |   |
|   |   +--------+---------+                |       |             |             |   |
|   |            |                          |       |             |             |   |
|   |   +--------v---------+                |       |             |             |   |
|   |   | Private Subnet   |                |       |             |             |   |
|   |   | (28.0.2.0/24)    |                |       |             |             |   |
|   |   | [Private Server] +<--- Peering ---+-------+-------------+             |   |
|   |   +------------------+    Connection  |                                   |   |
|   +---------------------------------------+                                   |
+-----------------------------------------------------------------------------------+
```

---

## 📋 Addressing & Network Subnetting Scheme

| Region | Resource Name | CIDR Block / IP Range | Subnet Type | Auto-Assign Public IP |
| :--- | :--- | :--- | :--- | :--- |
| **ap-south-1** | `Mumbai-VPC` | `28.0.0.0/16` | VPC | N/A |
| **ap-south-1** | `Public-Subnet-Mumbai` | `28.0.1.0/24` | Public | **Enabled** |
| **ap-south-1** | `Private-Subnet-Mumbai` | `28.0.2.0/24` | Private | Disabled |
| **us-east-1** | `Virginia-VPC` | `28.200.0.0/16` | VPC | N/A |
| **us-east-1** | `Private-Subnet-Virginia` | `28.200.1.0/24` | Private | Disabled |

---

## 🛠️ Step-by-Step Implementation Guide

### Step 1: Mumbai VPC & Subnet Creation
* Create `Mumbai-VPC` with IPv4 CIDR `28.0.0.0/16`.
* Configure Public and Private subnets in `ap-south-1`.

| Configuration Step | Console Output / Screenshot |
| :--- | :--- |
| **1.1 Mumbai VPC Creation** | ![Mumbai VPC Creation](images/Mumbai-VPC-Creation.png) |
| **1.2 Public Subnet Configuration** | ![Public Subnet Mumbai](images/Public-Subnet-Mumbai.png) |
| **1.3 Private Subnet Configuration** | ![Private Subnet Mumbai](images/Private-Subnet-Mumbai.png) |
| **1.4 Subnets Verification** | ![Subnets Created](images/Pub-and-Pri-Subnet-Created.png) |

---

### Step 2: Internet Gateway & Route Tables (Mumbai)
* Create an Internet Gateway (`Mumbai-INGW`) and attach it to `Mumbai-VPC`.
* Create `Mumbai-Public-RT` (associated with `Public-Subnet-Mumbai`) and add default route `0.0.0.0/0` pointing to `Mumbai-INGW`.
* Create `Mumbai-Private-RT` (associated with `Private-Subnet-Mumbai`).

| Configuration Step | Console Output / Screenshot |
| :--- | :--- |
| **2.1 Internet Gateway Creation** | ![IGW Creation](images/Mumbai-INGW-Creation.png) |
| **2.2 Attach IGW to Mumbai VPC** | ![IGW Association](images/Mumbai-INGW-Association-Mumbai-VPC.png) |
| **2.3 Create Public Route Table** | ![Public RT Creation](images/Mumbai-Public-RT-Creation.png) |
| **2.4 Create Private Route Table** | ![Private RT Creation](images/Mumbai-Private-RT-Creation.png) |
| **2.5 Add Internet Route to Public RT** | ![Adding Route](images/Adding-Route-in-Mumbai-Public.png) |

---

### Step 3: N. Virginia VPC & Subnet Setup
* Switch to **us-east-1**.
* Create `Virginia-VPC` (`28.200.0.0/16`).
* Create `Private-Subnet-Virginia` (`28.200.1.0/24`) and associate it with `Virginia-Private-RT`.

| Configuration Step | Console Output / Screenshot |
| :--- | :--- |
| **3.1 Virginia VPC Creation** | ![Virginia VPC](images/Virginia-VPC-Creation.png) |
| **3.2 Subnet Details** | ![Private Subnet](images/Private-Subnet.png) |
| **3.3 Subnet Confirmation** | ![Private Subnet Creation](images/Private-Subnet-Creation.png) |
| **3.4 Virginia Private Route Table** | ![Virginia RT](images/Route-Virginia.png) |

---

### Step 4: Inter-Region VPC Peering Configuration
* Initiate a VPC Peering Connection request from **ap-south-1** targeting `Virginia-VPC` in **us-east-1**.
* Accept the pending peering request in **us-east-1**.
* Update route tables in both regions to route cross-region CIDR traffic over the peering connection.

| Configuration Step | Console Output / Screenshot |
| :--- | :--- |
| **4.1 Create Peering Request (Mumbai -> Virginia)** | ![Peering Request](images/Mumbai-to-Virginia-Peering.png) |
| **4.2 Peering Connection Pending State** | ![Peering Connection Created](images/Mumbai-to-Virginia-Peering-Creation.jpg) |
| **4.3 Accept Peering Request in Virginia** | ![Accept Peering](images/Accepting-Peering-Connection.jpg) |
| **4.4 Update Mumbai Route Table (`28.200.0.0/16`)** | ![Update Mumbai RT](images/Update-Route-Tables-for-Peering-Traffic-from-Mumbai-to-Virginia.png) |
| **4.5 Update Virginia Route Table (`28.0.0.0/16`)** | ![Update Virginia RT](images/Update-Route-Tables-for-Peering-Traffic-from-Virginia-to-Mumbai.png) |

---

### Step 5: Compute Instance Deployment & Launch Configurations
* Launch `Public-Server-Mumbai` in `Public-Subnet-Mumbai` with **Auto-assign Public IP** enabled.
* Launch `Private-Server-Mumbai` in `Private-Subnet-Mumbai` with SSH restricted to `28.0.1.0/24`.
* Launch `Private-Server-Virginia` in `Private-Subnet-Virginia` with security rules allowing inbound traffic from `28.0.0.0/16`.

| Deployment Step | Console Output / Screenshot |
| :--- | :--- |
| **5.1 Public Server Setup (Mumbai)** | ![Public Server Mumbai](images/Public-Server-Mumbai.png) |
| **5.2 Private Server Setup (Mumbai)** | ![Private Server Mumbai](images/Private-Server-Mumbai.png) |
| **5.3 Private Server Setup (Virginia)** | ![Private Server Virginia](images/Private-Server-Virginia.png) |

---

### Step 6: End-to-End Connectivity Verification & Multi-Hop SSH
Verify connectivity across the network by hopping sequentially from external internet to cross-region private subnets.

| Verification Step | Terminal Output / Screenshot |
| :--- | :--- |
| **Step 6.1: Connect to Public Jump Host & Ping Internet** | ![Terminal s1](images/s1.png) |
| **Step 6.2: Hop from Bastion to Mumbai Private Server** | ![Terminal s2](images/s2.png) |
| **Step 6.3: Cross-Region Hop to Virginia Private Server** | ![Terminal s3](images/s3.png) |
| **Step 6.4: Reverse Hop from Virginia back to Mumbai Private** | ![Terminal s4](images/s4.png) |

---

## 🔒 Execution Commands Summary

1. **Connect from Workstation to Mumbai Public Jump Host:**
   ```bash
   ssh -i "private-key-mumbai.pem" ec2-user@<Public-IP-Mumbai>
   ```

2. **Hop to Mumbai Private Server:**
   ```bash
   ssh -i "private-key-mumbai.pem" ec2-user@28.0.2.117
   ```

3. **Cross-Region Hop to Virginia Private Server:**
   ```bash
   ssh -i "private-key-virginia.pem" ec2-user@28.200.1.103
   ```

4. **Reverse Hop from Virginia back to Mumbai Private Server:**
   ```bash
   ssh -i "private-key-mumbai.pem" ec2-user@28.0.2.117
   ```
