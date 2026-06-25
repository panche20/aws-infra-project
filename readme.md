# Complete Project: Personal Site on S3/CloudFront/Route53 + Serverless Visitor Counter

Build a personal website hosted on *Amazon S3 with cloudfront for distribution and Route 53 for DNS. 
It features a visitor counter using API gateway, lambda and Dynamodb*

## Naming convention used throughout (swap in your own where marked):

- Domain: yourdomain.com ← replace with your actual domain
- S3 bucket: chetan-portfolio-site-2026
- Compute/data region: ap-south-1
- ACM cert region: us-east-1 (CloudFront hard requirement — doesn't matter where everything else lives)
- DynamoDB table: VisitorCounter
- Lambda: visitor-counter-fn
- API Gateway: visitor-counter-api

**Phase 1: S3 Static Hosting (the storage layer)**
The 'Why' (The Problem)
Before this pattern existed, hosting a personal site meant running a server — EC2 instance, Nginx/Apache, OS patching, SSH hardening, and paying for compute 24/7 even when nobody's visiting. For static content (HTML/CSS/JS that doesn't change per-request), that's wasted spend and wasted ops burden. S3 solves this by decoupling "storage that can serve HTTP" from "compute" entirely — there's no server to patch, scale, or go down. You pay per-GB stored and per-request, not per-hour-running.
Deep-Dive Mechanics
There are actually two different S3 "website" mechanisms, and which one you pick is a real interview gotcha:

<img width="930" height="430" alt="image" src="https://github.com/user-attachments/assets/47f86739-0477-4712-a199-a01561c64f7b" />

We're using the REST endpoint + Origin Access Control (OAC) pattern — it's the AWS-recommended approach since late 2022 (OAC replaced the older OAI). The bucket stays 100% private; CloudFront authenticates to S3 using SigV4-signed requests on your behalf. Nobody can hit the S3 URL directly and bypass your CDN/cache/WAF.

<img width="941" height="452" alt="image" src="https://github.com/user-attachments/assets/ee83bf93-b6a8-49cb-a342-9f501729d564" />

**Interview POV & Edge Cases**

Classic question: "Design a globally-distributed static site hosting solution, cost-optimized." → S3+CloudFront+OAC is the canonical answer.
Gotcha #1: ACM certificates for CloudFront must be requested in us-east-1, regardless of where your other resources live. This trips people up constantly.
Gotcha #2: Bucket names are globally unique across all AWS accounts, not just yours.
Gotcha #3: "Why not just make the bucket public?" — because then CloudFront's caching/WAF/logging can be bypassed by hitting S3 directly, and you lose centralized access control.
Gotcha #4: S3 had eventual consistency historically — interviewers sometimes still ask about it. As of Dec 2020, S3 has strong read-after-write consistency for all operations.

**Pre-requisites:**
- custom domain from any registrar like godaddy,hostinger
- AWS account

**Step 1: Create minimal index.html**

```
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Chetan | DevOps & Cloud Engineer</title>
  <style>
    body { font-family: system-ui, sans-serif; max-width: 700px; margin: 60px auto; padding: 0 20px; }
    #visitor-count { font-weight: bold; }
  </style>
</head>
<body>
  <h1>Hi, I'm Chetan</h1>
  <p>DevOps / SRE / Platform Engineer in progress.</p>
  <p>Visitors so far: <span id="visitor-count">loading...</span></p>
  <script>
    // We'll wire this up to API Gateway in Phase 6-7
  </script>
</body>
</html>
```

**Overall Architecture**

By the end, we'll have:

```
                    +----------------+
                    |     Route53    |
                    +----------------+
                             |
                             |
                    +----------------+
                    |   CloudFront    |
                    +----------------+
                             |
                             |
                    +----------------+
                    | Private S3      |
                    | index.html      |
                    | style.css       |
                    | script.js       |
                    | resume.pdf      |
                    +----------------+

Browser
    |
    v
+-------------+
| API Gateway |
+-------------+
      |
      v
+-------------+
|   Lambda    |
+-------------+
      |
      v
+-------------+
| DynamoDB    |
+-------------+
```

## Step 1 : Create S3 Bucket

- AWS Console - S3 - Create Bucket
- Bucket Type - General purpose
- Bucket namespace - Global namespace
- Bucket name - <pick up unique name>
- Object ownership - ACLs disabled
- Block Public Access settings for this bucket - Select Block all public access
- Bucket Versioning - Enabled
- Encryption Type - Server-side encryption with Amazon S3 managed keys (SSE-S3)
- Bucket Key - Enable
- Now, select create bucket
- Upload Website Files - Upload index.html - Upload

**IMPORTANT** - Do not enable :
Properties
→ Static Website Hosting

## Step 2 : Open Route53

- AWS Console - Route53 - Hosted Zones - Create hosted zone
- Domain name - Enter your domain name (eg. api.example.com)
- Type - Public Hosted Zone
- Create Hosted Zone

**Verify Records**

Check NS records:

```
ns-111.awsdns-11.com
ns-222.awsdns-22.net
ns-333.awsdns-33.org
ns-444.awsdns-44.co.uk
```

## Step 3 : Update the NS records with your 3rd party registrar

- Go to your registrar
- DNS Management
- Name servers
- Replace the existing nameservers with the four AWS nameservers.

**Common Interview Question**

Q: Why create a Hosted Zone before requesting an ACM certificate?
**Answer:**

Because ACM uses DNS validation. If Route 53 already hosts the DNS zone, AWS can automatically create the required CNAME validation record. 
This makes certificate issuance automatic and avoids manually managing DNS records at another provider.

## Step 4 : Request an ACM Certificate
**Goal ** - Obtain an SSL/TLS certificate for your domain so visitors can securely access

**Why do we need ACM?**
When a user visits your website: <api.example.com> 
- the browser expects an SSL/TLS certificate to:
- Encrypt communication.
- Verify the website's identity.
- Display the secure padlock.

### Please note: CloudFront is a global service, and it only uses ACM certificates from: N. Virginia (us-east-1)

**Request Flow:**

```
Browser
      │
HTTPS Request
      │
      ▼
CloudFront
      │
Uses ACM Certificate
      │
      ▼
Secure TLS Connection
      │
      ▼
Private S3 Bucket
```

**Open ACM**
- AWS Console - Certificate Manager (ACM)
- Request Certificate
- Select certificate type - Public Certificate
- Domain name - Enter <api.example.com>
- Click Add another name - <*.api.example.com>
- Validation Method - DNS Validation
- Click Request
- Initially, the certificate status will show - Pending Validation

**Create DNS Validation Records**
- Open the certificate - You'll see two CNAME validation records (one for each domain name).
- Create records in Route53 - Click it
- AWS will automatically create the required DNS records.
- Wait for Validation - It will change to **Issued.**

## Step 5: Create CloudFront Distribution

**Objective**
By the end of this step:

Your website will be accessible using a CloudFront URL.
HTTPS will work.
Your S3 bucket will remain private.
Only CloudFront will be allowed to access your bucket.

**Why CloudFront?**
Without CloudFront:

```
Browser
   │
   ▼
Public S3 Bucket
```

Problems:

❌ Bucket must be public
❌ Anyone can bypass your CDN
❌ No custom domain HTTPS
❌ Less control over caching and security

**With CloudFront:**

```
Browser
   │
HTTPS
   │
Route53
   │
CloudFront
   │
OAC (Signed Request)
   │
Private S3 Bucket
```

**Benefits:**

✅ Private bucket
✅ HTTPS
✅ Global CDN
✅ Better security
✅ Lower latency

Under the hood when a user visits <api.example.com>
