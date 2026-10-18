# Terraform & AWS VPC: Advanced MCQs

> **Practice for [Lecture 17 – Terraform (IaC)](17-Terraform-IaC.md) and [Lecture 18 – AWS Core Services & VPC](18-AWS-VPC-and-Core-Services.md).** Each answer is shown right below its question. Cover it if you want to test yourself first.

| Part | Lecture | Questions |
|------|---------|-----------|
| [A](#part-a-lecture-17--terraform) | 17 – Terraform | Q1–Q8 |
| [B](#part-b-lecture-18--aws--vpc) | 18 – AWS & VPC | Q9–Q16 |

---

## Part A: Lecture 17 – Terraform

### Q1. Applying twice
You run `terraform apply` and it creates an S3 bucket. You run `terraform apply` again **without changing any code**. What happens?
- A) A second bucket is created
- B) `No changes. Your infrastructure matches the configuration.`
- C) The bucket is destroyed and re-created
- D) It fails with `BucketAlreadyExists`

> ✅ **Answer: B.** Terraform is **declarative and idempotent**. It compares code ↔ state ↔ reality, sees nothing to do, and changes nothing. A shell script with `aws s3 mb` would behave differently. · 📖 [L17 §1](17-Terraform-IaC.md#1-the-big-picture-what-is-iac)

### Q2. `fmt` in CI
In a CI pipeline, you want the build to **fail** if any `.tf` file is badly formatted, **without** changing files on the runner. Which command?
- A) `terraform fmt`
- B) `terraform fmt -check`
- C) `terraform validate`
- D) `terraform plan -format`

> ✅ **Answer: B.** Plain `fmt` **rewrites** files. `fmt -check` only reports, and exits non-zero if anything needs formatting. `validate` checks syntax and logic, not style. · 📖 [L17 §11](17-Terraform-IaC.md#11-the-core-workflow-init--fmt--validate--plan--apply--destroy)

### Q3. Which value wins?
`aws_region` has `default = "us-east-1"`, `terraform.tfvars` sets `"ap-south-1"`, the env var `TF_VAR_aws_region=eu-west-1` is exported, and you run `terraform apply -var="aws_region=us-west-2"`. Which region is used?
- A) `us-east-1`
- B) `ap-south-1`
- C) `eu-west-1`
- D) `us-west-2`

> ✅ **Answer: D.** Precedence, lowest to highest: default → `TF_VAR_` env var → `terraform.tfvars` → `*.auto.tfvars` → `-var` / `-var-file` on the command line. Watch out: `terraform.tfvars` beats the env var, so without `-var` the answer would be `ap-south-1`, not `eu-west-1`. · 📖 [L17 §7](17-Terraform-IaC.md#7-variables)

### Q4. Renaming a bucket
You change `bucket = "aarham-demo-a1"` to `bucket = "aarham-demo-b2"` on an existing `aws_s3_bucket`. What does `terraform plan` show, and why should you be careful?
- A) `~ update in-place`: the bucket is renamed safely
- B) `-/+ must be replaced`: the old bucket is destroyed and a new one created, so its data could be lost
- C) `No changes`: names can't change
- D) `+ create` only: both buckets will exist

> ✅ **Answer: B.** An S3 bucket can't be renamed, so Terraform has to **replace** it. Always look for `-/+` in a plan before typing `yes`. · 📖 [L17 §11](17-Terraform-IaC.md#11-the-core-workflow-init--fmt--validate--plan--apply--destroy)

### Q5. Lost state file
Your bucket exists in AWS, but someone deleted the local `terraform.tfstate`. You run `terraform plan`. What does it show?
- A) `No changes`, because Terraform asks AWS what exists
- B) `+ create` for everything, because without state Terraform doesn't know the bucket is its own. Applying would fail or create duplicates.
- C) `- destroy` for the bucket
- D) An error that state can't be rebuilt

> ✅ **Answer: B.** State is how Terraform links your code to real resources. Without it, every resource looks new. That's why teams keep state in a **remote backend** (S3 with locking). ➕ Existing resources can be linked again with `terraform import`. · 📖 [L17 §10](17-Terraform-IaC.md#10--state-how-terraform-remembers)

### Q6. Version constraint
`required_providers` has `aws = { source = "hashicorp/aws", version = "~> 6.0" }`. Which provider versions can `terraform init` choose?
- A) Only 6.0.0
- B) Any 6.x (e.g. 6.9.1), but not 7.0
- C) 6.0 or anything newer, including 7.x
- D) Only 6.0.x patches

> ✅ **Answer: B.** `~> 6.0` allows the last number shown (the minor version) to go up, but not the major. `~> 6.2.0` would allow only 6.2.x. · 📖 [L17 §5](17-Terraform-IaC.md#5-the-terraform-block--providers)

### Q7. A change made in the console
A teammate adds a tag `Owner=temp` to the bucket **in the AWS console**. Your code doesn't have that tag. What does your next `terraform plan` show?
- A) Nothing, because Terraform only tracks resources it created
- B) `~ update in-place` that removes `Owner=temp`, so the bucket goes back to what the code says
- C) `-/+` replacement of the bucket
- D) An error about drift that must be fixed in the console

> ✅ **Answer: B.** This is **drift**. Terraform refreshes the real state, sees the difference, and plans to restore the coded state. To keep the tag, add it to the code. · 📖 [L17 §10](17-Terraform-IaC.md#10--state-how-terraform-remembers)

### Q8. Destroy fails on a versioned bucket
The lab bucket has versioning enabled. You ran `aws s3 rm s3://bucket --recursive`, then `terraform destroy`, which fails with `BucketNotEmpty`. Why?
- A) `destroy` can't delete S3 buckets
- B) Versioning keeps **old object versions** (and delete markers) after `rm`. Empty those too, or set `force_destroy = true` (labs only).
- C) The state file is locked
- D) The provider version is too old

> ✅ **Answer: B.** `aws s3 rm` only hides the current versions. AWS won't delete a bucket that still has any versions. `force_destroy` lets Terraform empty it first, so never use it on real data. · 📖 [L17 §12](17-Terraform-IaC.md#12-hands-on-create-an-s3-bucket-on-aws)

---

## Part B: Lecture 18 – AWS & VPC

### Q9. A "public" subnet that isn't public
Your Terraform creates a VPC, a subnet with `map_public_ip_on_launch = true`, and an Internet Gateway attached to the VPC. An EC2 instance in that subnet has a public IP and its security group allows port 80, but `curl` from the internet times out. What's missing?
- A) A NAT Gateway
- B) A route table with `0.0.0.0/0 → IGW`, associated with the subnet
- C) An Elastic IP on the instance
- D) A NACL deny rule

> ✅ **Answer: B.** A subnet is public **only** if its route table sends `0.0.0.0/0` to an Internet Gateway. The class snippet had the IGW but no route table, which is exactly this bug. · 📖 [L18 §7](18-AWS-VPC-and-Core-Services.md#7-public-vs-private-subnets)

### Q10. Where the NAT Gateway goes
To give private instances outbound internet access, you create a NAT Gateway **in the private subnet** and point the private route table at it. `yum update` still times out. Why?
- A) NAT Gateways only support IPv6
- B) The NAT Gateway must sit in a **public** subnet (whose route table reaches the IGW). In the private subnet, it has no path out itself.
- C) Private subnets can never reach the internet
- D) The security group must allow inbound port 443

> ✅ **Answer: B.** The path is private instance → NAT (public subnet) → IGW → internet. A NAT inside the private subnet has no route to the IGW. · 📖 [L18 §9](18-AWS-VPC-and-Core-Services.md#9-internet-gateway-vs-nat-gateway)

### Q11. No outbound rules
A security group allows inbound TCP 80 from `0.0.0.0/0` and has **no outbound rules at all**. Can browsers still load the website?
- A) No, because the responses are blocked on the way out
- B) Yes, because security groups are **stateful**: replies to allowed inbound traffic are allowed out automatically
- C) Only over HTTPS
- D) Only if a NACL allows it inbound

> ✅ **Answer: B.** Stateful means the security group remembers the connection. Outbound rules only matter for connections the instance **starts** itself (e.g. `yum update`, calling APIs). · 📖 [L18 §10](18-AWS-VPC-and-Core-Services.md#10-security-groups-vs-nacls)

### Q12. A stateless NACL
A custom NACL allows inbound TCP 80 and allows outbound **only** TCP 80. The security group allows 80 inbound. Web requests reach the server, but browsers never get a reply. Why?
- A) The security group blocks the replies
- B) NACLs are **stateless**: replies go to the client's ephemeral port (1024–65535), which the outbound NACL rules don't allow
- C) Port 80 is reserved by AWS
- D) The route table has no `local` route

> ✅ **Answer: B.** Unlike security groups, NACLs check each direction separately. Add an outbound rule for TCP 1024–65535. · 📖 [L18 §10](18-AWS-VPC-and-Core-Services.md#10-security-groups-vs-nacls)

### Q13. Blocking one IP range
Attack traffic is coming from `203.0.113.0/24`. You want to **block that range** for every instance in a subnet. What do you use?
- A) A security group rule with `Deny`
- B) A NACL rule with `DENY`, numbered lower than the allow rules
- C) A route table entry sending the range to `local`
- D) An IAM policy

> ✅ **Answer: B.** Security groups have **allow rules only**, so they can't deny. NACLs support explicit `DENY`, checked in number order with the first match winning. Route tables don't filter traffic. · 📖 [L18 §10](18-AWS-VPC-and-Core-Services.md#10-security-groups-vs-nacls), [§8](18-AWS-VPC-and-Core-Services.md#8-route-tables)

### Q14. Usable addresses
How many IP addresses can you assign to instances in a `10.0.1.0/24` subnet in AWS?
- A) 256
- B) 254
- C) 251
- D) 250

> ✅ **Answer: C.** 256 − 5. AWS reserves 5 in every subnet: network address, VPC router, DNS, future use and broadcast. In ordinary networking it would be 254 (−2). · 📖 [L18 §7](18-AWS-VPC-and-Core-Services.md#7-public-vs-private-subnets), [L3 §4](03-Networking.md#4-subnetting--cidr)

### Q15. Giving an EC2 app access to S3
A Python app on EC2 needs to read from one S3 bucket. What's the most secure way to give it access?
- A) Put an IAM user's access keys in the app's config file
- B) Attach an **IAM role** (instance profile) to the instance, with a policy allowing `s3:GetObject` on that bucket only
- C) Make the bucket public
- D) Use the root account's keys, stored in an environment variable

> ✅ **Answer: B.** A role gives the instance **temporary, automatically rotated** credentials with nothing stored on disk, and the policy follows **least privilege**. Stored keys can leak, and a public bucket exposes the data to everyone. · 📖 [L18 §4](18-AWS-VPC-and-Core-Services.md#4-core-aws-services)

### Q16. Choosing an S3 class
You must keep compliance logs for 7 years. They're almost never read, and a retrieval time of **up to ~12 hours** is fine. Which S3 storage class is cheapest?
- A) S3 Standard
- B) S3 Standard-IA
- C) S3 Glacier Deep Archive
- D) S3 "Premium"

> ✅ **Answer: C.** Deep Archive is the lowest-cost class, for data read rarely with retrieval in hours. Standard-IA is for rare but **fast** access. There is **no "Premium"** class (that was a mistake in the class notes). · 📖 [L18 §4](18-AWS-VPC-and-Core-Services.md#4-core-aws-services)
