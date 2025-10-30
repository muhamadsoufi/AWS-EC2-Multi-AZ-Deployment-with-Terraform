# 🚀 AWS EC2 Multi-AZ Auto Deployment with Terraform

This Terraform project dynamically deploys **Amazon EC2 instances** across all **available Availability Zones (AZs)** within a chosen AWS region — but only in those AZs that actually **support the selected instance type**.  
Each instance automatically installs and configures a **web server (Apache)** and serves a small demo app.

---

## 🧩 Features

- 🌍 **Dynamic Multi-AZ Deployment**  
  Using `for_each`, Terraform automatically determines which **Availability Zones support the chosen EC2 instance type**, and only creates instances in those zones.

- ⚙️ **Dynamic AMI Discovery**  
  Automatically fetches the **latest Amazon Linux 2 AMI** using AWS data sources (`data "aws_ami"`).

- 🧠 **User Data Bootstrap Script**  
  The included `app1-install.sh` installs Apache, sets up an HTML demo page, and retrieves **instance metadata** from the EC2 Metadata Service (IMDSv2).

- 🔒 **Security Groups**  
  - SSH access (port 22)  
  - Web access (ports 80 and 443)

- 📡 **Rich Outputs**  
  Outputs are exposed as `toset` and `tomap` — showing public IPs, public DNS names, and AZ-to-DNS mappings.

---

## 🧱 Project Structure
.
├── c1-version.tf
├── c2-variable.tf
├── c3-ec2securitygroups.tf
├── c4-ami-datasource.tf
├── c5-ec2instance.tf
├── c6-outputs.tf
└── app1-install.sh


## 📜 Key Components

### `app1-install.sh`
The user-data script executed at instance boot:
- Installs and enables Apache (`httpd`)
- Creates a simple demo webpage at `/app1/`
- Fetches instance metadata with IMDSv2 token for security
- Outputs metadata to `/var/www/html/app1/metadata.html`

---

## ⚙️ Input Variables

| Variable | Type | Default | Description |
|-----------|-------|----------|-------------|
| `aws_region` | `string` | `"us-east-1"` | AWS region for resource creation |
| `instance_type` | `string` | `"t3.micro"` | EC2 instance type |
| `instance_keypair` | `string` | `"terraform-key"` | SSH key pair name for EC2 instances |

---

## 🧾 Outputs

| Output | Description |
|---------|-------------|
| `instance_publicip` | Set of EC2 public IPs |
| `instance_publicdns` | Set of EC2 public DNS names |
| `instance_publicdns2` | Map of AZ → EC2 public DNS |

---

## ⚙️ Dynamic Logic Explained

This project uses **advanced Terraform functions** to create a **fully dynamic infrastructure**:

### 🧮 Instance Type Availability
```hcl
data "aws_ec2_instance_type_offerings" "my_ins_type" {
  for_each = toset(["us-east-1a", "us-east-1b", "us-east-1c", "us-east-1d", "us-east-1e", "us-east-1f"])
  filter {
    name   = "instance-type"
    values = ["t2.micro"]
  }
  filter {
    name   = "location"
    values = [each.key]
  }
}


🧠 Dynamic EC2 Creation
for_each = toset(keys({
  for az, details in data.aws_ec2_instance_type_offerings.my_ins_type :
  az => details.instance_types if length(details.instance_types) != 0
}))


Terraform loops only over valid AZs.
Each instance is created only where the instance type is available.

🧩 Dynamic AMI Discovery
data "aws_ami" "amzlinux2" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-gp2"]
  }
  filter {
    name   = "root-device-type"
    values = ["ebs"]
  }
  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
  filter {
    name   = "architecture"
    values = ["x86_64"]
  }
}
Dynamically retrieves the latest Amazon Linux 2 image ID.
Ensures only official and up-to-date AMIs are used.


🧰 Prerequisites
🧑‍💻 Terraform ≥ 1.6
☁️ AWS CLI configured (~/.aws/credentials)
🔑 A valid AWS key pair matching instance_keypair


🚀 How to Deploy
1️⃣ Initialize Terraform
terraform init

2️⃣ Validate and Plan
terraform plan

3️⃣ Apply and Build Infrastructure
terraform apply -auto-approve

4️⃣ Get Outputs
terraform output

5️⃣ Test Application

Open in your browser:
http://<instance-public-ip>/app1/
Or using curl:
curl http://<instance-public-dns>/app1/

🧹 Cleanup
To destroy all created resources:
terraform destroy -auto-approve


🧠 Notes & Best Practices
Instances are created only in AZs that support the given instance type.
Uses IMDSv2 tokens for secure metadata access.
Outputs use toset and tomap since for_each returns maps instead of lists.
Great for learning dynamic resource creation, loops, and data sources in Terraform.