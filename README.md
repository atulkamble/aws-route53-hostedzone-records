Let's create a **Full AWS Route 53 Project** where we'll set up:

## 🗂️ **Project Structure**

```
aws-route53-project/
├── terraform/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   └── provider.tf
├── scripts/
│   └── update-record.sh
├── README.md
```

---

## 📝 **Project Overview**

We'll automate:

1. Creating a Route 53 Hosted Zone (Public).
2. Adding A, CNAME records.
3. Optional — Automate record updates (script).

---

## 🛠️ **Step-by-Step Guide**

### 1️⃣ **Terraform Code for Route 53 Hosted Zone & Records**

### `provider.tf`

```hcl
provider "aws" {
  region = "us-east-1"
}
```

### `variables.tf`

```hcl
variable "domain_name" {
  description = "The domain name to create hosted zone"
  type        = string
}

variable "a_record_name" {
  description = "A record subdomain (like www)"
  type        = string
}

variable "a_record_ip" {
  description = "IP Address for A record"
  type        = string
}
```

### `main.tf`

```hcl
resource "aws_route53_zone" "main" {
  name = var.domain_name
}

resource "aws_route53_record" "a_record" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "${var.a_record_name}.${var.domain_name}"
  type    = "A"
  ttl     = 300
  records = [var.a_record_ip]
}

resource "aws_route53_record" "cname_record" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "app.${var.domain_name}"
  type    = "CNAME"
  ttl     = 300
  records = ["${var.a_record_name}.${var.domain_name}"]
}
```

### `outputs.tf`

```hcl
output "hosted_zone_id" {
  value = aws_route53_zone.main.zone_id
}

output "a_record_fqdn" {
  value = aws_route53_record.a_record.fqdn
}

output "cname_record_fqdn" {
  value = aws_route53_record.cname_record.fqdn
}
```

---

### 2️⃣ **Terraform Deployment Steps**

```bash
cd terraform
terraform init
terraform plan -var="domain_name=example.com" -var="a_record_name=www" -var="a_record_ip=1.2.3.4"
terraform apply -auto-approve -var="domain_name=example.com" -var="a_record_name=www" -var="a_record_ip=1.2.3.4"
```

---

### 3️⃣ **Optional: Script to Update DNS Record Dynamically**

`scripts/update-record.sh`

```bash
#!/bin/bash

HOSTED_ZONE_ID="ZXXXXXXXXXXXX"
RECORD_NAME="www.example.com."
RECORD_TYPE="A"
TTL=300
NEW_IP="5.6.7.8"

cat > change-batch.json << EOF
{
  "Comment": "Updating A record",
  "Changes": [{
    "Action": "UPSERT",
    "ResourceRecordSet": {
      "Name": "$RECORD_NAME",
      "Type": "$RECORD_TYPE",
      "TTL": $TTL,
      "ResourceRecords": [{"Value": "$NEW_IP"}]
    }
  }]
}
EOF

aws route53 change-resource-record-sets --hosted-zone-id $HOSTED_ZONE_ID --change-batch file://change-batch.json
```

Run:

```bash
chmod +x update-record.sh
./update-record.sh
```

---

## 📂 **Repo Name Suggestion**

`aws-route53-hostedzone-records`

---

## 📄 **README.md Overview**

* What is Route 53 Hosted Zone?
* How to deploy with Terraform?
* How to update DNS dynamically with CLI?
* Example use-case: Setting up A & CNAME records for a static website.

---
Let's enhance your **AWS Route 53 Project** with **Advanced Tasks** — including domain registration, MX/TXT records, health checks, failover DNS, and CI/CD automation via GitHub Actions.

---

## ✅ **Advanced Route 53 Tasks — Full Guide**

---

## 1️⃣ **Automate Domain Registration via Route 53 (Terraform)**

### `domain-registration.tf`

```hcl
resource "aws_route53domains_registered_domain" "my_domain" {
  domain_name = var.domain_name
  admin_contact {
    name          = "Your Name"
    organization  = "Your Organization"
    address_line_1 = "Your Address"
    city          = "Your City"
    state         = "Your State"
    country_code  = "US"
    zip_code      = "12345"
    phone_number  = "+1.1234567890"
    email         = "you@example.com"
  }

  registrant_contact {
    # Same as admin_contact
  }

  tech_contact {
    # Same as admin_contact
  }

  name_server {
    name = "ns-123.awsdns-45.com"
  }

  auto_renew = true
}
```

> **Important:** Domain registration requires manual approval of domain ownership (via email). Terraform will provision it, but confirmation will be manual for security reasons.

---

## 2️⃣ **Add MX and TXT Records (for Email/DKIM)**

### Extend `main.tf`

```hcl
resource "aws_route53_record" "mx_record" {
  zone_id = aws_route53_zone.main.zone_id
  name    = var.domain_name
  type    = "MX"
  ttl     = 300
  records = ["10 mail.example.com."]
}

resource "aws_route53_record" "txt_record" {
  zone_id = aws_route53_zone.main.zone_id
  name    = var.domain_name
  type    = "TXT"
  ttl     = 300
  records = ["v=spf1 include:amazonses.com ~all"]
}
```

> For DKIM verification (SES, Google, etc.), add the TXT records they provide in the same format.

---

## 3️⃣ **Setup Route53 Health Checks & Failover DNS**

### `healthcheck-failover.tf`

```hcl
resource "aws_route53_health_check" "primary_health_check" {
  fqdn              = "primary.example.com"
  port              = 80
  type              = "HTTP"
  resource_path     = "/health"
  failure_threshold = 3
  request_interval  = 30
}

resource "aws_route53_record" "primary_record" {
  zone_id = aws_route53_zone.main.zone_id
  name    = var.domain_name
  type    = "A"
  ttl     = 60
  set_identifier = "Primary"
  health_check_id = aws_route53_health_check.primary_health_check.id
  records = ["1.2.3.4"]
}

resource "aws_route53_record" "secondary_record" {
  zone_id = aws_route53_zone.main.zone_id
  name    = var.domain_name
  type    = "A"
  ttl     = 60
  set_identifier = "Secondary"
  records = ["5.6.7.8"]
  failover_routing_policy {
    type = "SECONDARY"
  }
}
```

---

## 4️⃣ **GitHub Actions CI/CD for Route53 Terraform Deployment**

### `.github/workflows/terraform-route53.yml`

```yaml
name: Terraform Route53 Deploy

on:
  push:
    branches: [ main ]
  pull_request:

jobs:
  terraform:
    name: 'Terraform Apply'
    runs-on: ubuntu-latest

    steps:
    - name: Checkout Code
      uses: actions/checkout@v3

    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v3
      with:
        terraform_version: 1.6.0

    - name: Configure AWS Credentials
      uses: aws-actions/configure-aws-credentials@v2
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: us-east-1

    - name: Terraform Init
      run: terraform init

    - name: Terraform Plan
      run: terraform plan

    - name: Terraform Apply
      run: terraform apply -auto-approve
```

---

## 📂 **Secrets to Configure in GitHub Repo**

* `AWS_ACCESS_KEY_ID`
* `AWS_SECRET_ACCESS_KEY`

---

## 🚀 **Usage Flow Summary**

1. Push changes to GitHub.
2. CI/CD pipeline triggers.
3. Route53 DNS records, health checks, and domain registration are deployed automatically.
4. Failover DNS ensures high availability.

---
