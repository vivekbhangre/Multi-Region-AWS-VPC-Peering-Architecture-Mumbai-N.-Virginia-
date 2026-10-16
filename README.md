# Multi-Region-AWS-VPC-Peering-Architecture-Mumbai-N.-Virginia-
A fault-tolerant and secure multi-region AWS network topology featuring a VPC in Mumbai (ap-south-1) peered with a VPC in N. Virginia (us-east-1), allowing secure cross-region private communication and multi-hop jump-host administration.

## 🏗️ Architecture Overview

```
+-----------------------------------------------------------------------------------+
| AWS Cloud                                                                         |
|                                                                                   |
|   +---------------------------------------+       +---------------------------+   |
|   | Mumbai VPC (28.0.0.0/16)              |       | Virginia VPC              |   |
|   |                                       |       | (28.200.0.0/16)           |   |
|   |   +------------------+                |       |                           |   |
|   |   | Public Subnet    |                |       |   +-------------------+   |   |
|   |   | (28.0.1.0/24)    |                |       |   | Private Subnet    |   |   |
|   |   | [EC2 Jump Host]  |                |       |   | (28.200.1.0/24)   |   |   |
|   |   +--------+---------+                |       |   | [Private Server]  |   |   |
|   |            |                          |       |   +---------+---------+   |   |
|   |   +--------v---------+                |       |             ^             |   |
|   |   | Private Subnet   |                |       |             |             |   |
|   |   | (28.0.2.0/24)    |                |       |             |             |   |
|   |   | [Private Server] +<--- Peering ---+-------+-------------+             |   |
|   |   +------------------+    Connection  |                                   |   |
|   +---------------------------------------+                                   |
+-----------------------------------------------------------------------------------+
```

## 🌐 Network & IP Addressing Scheme

| Region | VPC Name | VPC CIDR | Subnet Name | Subnet CIDR | Purpose | 
| ----- | ----- | ----- | ----- | ----- | ----- | 
| **Mumbai** (`ap-south-1`) | `Mumbai-VPC` | `28.0.0.0/16` | `Public-Subnet` | `28.0.1.0/24` | Inbound jump host entry point via Internet Gateway | 
| **Mumbai** (`ap-south-1`) | `Mumbai-VPC` | `28.0.0.0/16` | `Private-Subnet-Mumbai` | `28.0.2.0/24` | Private server accessible via jump host | 
| **N. Virginia** (`us-east-1`) | `Virginia-VPC` | `28.200.0.0/16` | `Private-Subnet-Virginia` | `28.200.1.0/24` | Private cross-region application server | 

## 🖼️ Screenshots & Proof of Deployment

Below are the screenshots documenting key steps of the setup. Place your image files in an `images/` directory inside your repository to render them here.

### 1. VPC Peering & Network Setup
| Mumbai Route Table | N. Virginia Route Table |
| :---: | :---: |
| ![Mumbai Private Route Table](images/mumbai-route-table.png) | ![Virginia Private Route Table](images/virginia-route-table.png) |

### 2. Security Groups
| Mumbai Private SG | N. Virginia Private SG |
| :---: | :---: |
| ![Mumbai Private SG](images/mumbai-private-sg.png) | ![Virginia Private SG](images/virginia-private-sg.png) |

### 3. Verification & Connectivity Tests
| SSH Jump to Mumbai Private Server | Multi-Hop SSH Jump to Virginia Private Server |
| :---: | :---: |
| ![Jump to Mumbai Private](images/ssh-mumbai-private.png) | ![Jump to Virginia Private](images/ssh-virginia-private.png) |

---

## 🛠️ Step-by-Step Implementation Guide

### 1. Mumbai VPC Setup (`ap-south-1`)

1. Create VPC **`Mumbai-VPC`** with CIDR block `28.0.0.0/16`.

2. Create Subnets:
   * **Public Subnet:** `28.0.1.0/24` (Auto-assign public IPv4 enabled)
   * **Private Subnet:** `28.0.2.0/24`

3. Attach an Internet Gateway (`Mumbai-IGW`) to `Mumbai-VPC`.

4. Create **Public Route Table** (`Mumbai-Public-RT`):
   * Destination `0.0.0.0/0` $\rightarrow$ Target: `Mumbai-IGW`
   * Associate with `Public-Subnet`.

5. Create **Private Route Table** (`Mumbai-Private-RT`) and associate it explicitly with `Private-Subnet-Mumbai`.

### 2. N. Virginia VPC Setup (`us-east-1`)

1. Create VPC **`Virginia-VPC`** with CIDR block `28.200.0.0/16`.

2. Create Private Subnet **`Private-Subnet-Virginia`** with CIDR block `28.200.1.0/24`.

3. Create **Private Route Table** (`Virginia-Private-RT`) and explicitly associate it with `Private-Subnet-Virginia`.

### 3. Inter-Region VPC Peering

1. In `ap-south-1`, request a VPC Peering connection from `Mumbai-VPC` to `Virginia-VPC`.

2. Switch to `us-east-1` console and **Accept** the pending peering connection (`pcx-...`).

3. Update Route Tables:
   * **Mumbai Private Route Table (`Mumbai-Private-RT`):**
     * Add route: Destination `28.200.0.0/16` $\rightarrow$ Target: `pcx-...`
   * **Virginia Private Route Table (`Virginia-Private-RT`):**
     * Add route: Destination `28.0.0.0/16` $\rightarrow$ Target: `pcx-...`

### 4. Security Group Configurations

#### **Mumbai Public Server Security Group**
* **Inbound:** SSH (`Port 22`) from your local IP address (`My IP`).

#### **Mumbai Private Server Security Group (`Mumbai-Private-EC2-SG`)**
* **Inbound:** SSH (`Port 22`) from `28.0.1.0/24` (Mumbai Public Subnet).
* **Inbound:** All Traffic from `28.200.0.0/16` (Virginia VPC).

#### **Virginia Private Server Security Group (`Virginia-Private-EC2-SG`)**
* **Inbound:** All Traffic / SSH (`Port 22`) from `28.0.0.0/16` (Mumbai VPC).

---

## 🔐 Verification & Connection Workflow

### Step 1: Connect to Mumbai Public Jump Server
```bash
ssh -i "private-key-mumbai.pem" ec2-user@<MUMBAI_PUBLIC_IP>
```

### Step 2: Jump to Mumbai Private Server
Copy `private-key-mumbai.pem` onto the jump host, lock down permissions, and execute:
```bash
chmod 400 private-key-mumbai.pem
ssh -i "private-key-mumbai.pem" ec2-user@28.0.2.117
```

### Step 3: Jump Across VPC Peering to Virginia Private Server
Copy `private-key-virginia.pem` onto the Mumbai private server, set permissions, and execute:
```bash
chmod 400 private-key-virginia.pem
ssh -i "private-key-virginia.pem" ec2-user@28.200.1.103
```

