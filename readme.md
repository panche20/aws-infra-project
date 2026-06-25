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

## Step 2 : Route 53 Hosted Zone (DNS delegation)

**Why: **

Route 53 needs to be the authoritative nameserver for your domain before anything (ACM validation, CloudFront alias) can use it. 
If you bought the domain on GoDaddy/Namecheap/etc., you must delegate DNS to Route 53.

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

**Why:** 

CloudFront is a global service but only accepts certs from us-east-1's ACM, regardless of where your other resources sit. 
This trips up almost everyone at least once.

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

## Step 5: CloudFront Distribution (OAC + custom domain)

**Why:** 

This is your CDN, TLS terminator, and the only entity allowed to read your S3 bucket. 
Edge caching means visitors in Singapore or São Paulo get your site from a nearby edge location, not from ap-south-1 directly.

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

Under the hood when a user visits <api.example.com>. Cloudfront receives the request, If the object is cached:

```
Edge Cache
    │
    ▼
Return index.html
```

If it isn't cached: 

```
CloudFront
     │
Signed SigV4 Request
     │
Private S3 Bucket
```

S3 verifies: "Is this request coming from my CloudFront distribution?" If yes: Return index.html.
If no: 403 Access Denied. This is exactly why we use Origin Access Control (OAC).

## Step 6 : Open CloudFront

- AWS Console - CloudFront - Create Distribution
- Enter Distribution name
- Select Distribution type : Single website or app - Next
- Origin Type - Amazon S3
- Origin - S3 origin - Browse S3 - Select S3 bucket
- Settings - Select Allow private S3 bucket access to CloudFront
- Keep origin settings as default - Next
- Enable Security - Select Do not enable security protections - Next
- Review & create - Create Distribution
- Select the create Distribution - General - Alternate domain names - Add Domain
- Edit - Custom SSL certificate - Select the created certificate
- Supported HTTP versions - HTTP/2
- Default root object - optional - index.html
- Connectivity - Price class - Use all edge locations (best performance)
- Select Use cache tags for cache invalidation - Save changes
- Origin - Select origin name
- Origin access - Origin access control settings (recommended)
- Origin access control - Copy policy - Go to S3 bucket permissions
- Paste the policy & save
- Enable origin shield - Yes - Save changes

### Interview Perspective

*Why use CloudFront with a private S3 bucket instead of enabling S3 Static Website Hosting?*

S3 Static Website Hosting requires a public bucket. By placing CloudFront in front of a private S3 bucket and using Origin Access Control (OAC), only CloudFront can access the bucket. 
This prevents users from bypassing the CDN, enforces HTTPS, improves caching and latency, and aligns with AWS security best practices.

## Step 7 : Connect Route 53 to CloudFront

**Why:** 

An Alias record (AWS-specific, free of charge, behaves like CNAME-at-the-apex) points your bare domain at CloudFront without the indirection/cost of a real CNAME, which isn't even allowed at a zone apex per DNS spec.

- AWS Console - Route 53 - Hosted Zones
- Select created hosted zone - Create record
- Configure record as below:
- Record Name - Blank
- Record Type - A - IPv4 address
- Alias - Yes
- Route traffic to - Alias to CloudFront distribution
- Choose Distribution - Select created distribution
- Routing Policy - Simple
- Evaluate Target Health - No
- Then click - Create record

## Step 8 — DynamoDB Table

**Why:** 

A visitor counter is a classic high-write, low-read-complexity workload — perfect for DynamoDB's single-digit-millisecond key-value access, vs. spinning up RDS for one integer.

**GUI:**

- DynamoDB console → Create table
- Table name: VisitorCounter, Partition key: counter_id (String)
- Table class: Standard, Capacity mode: On-demand (no traffic = no cost, vs provisioned which bills even at zero)
- Create table

**Gotcha (the real interview-grade one)**: 

Never implement a counter as GetItem → add 1 in code → PutItem. Two concurrent requests can both read count=5 and both write count=6 — you silently lose a visit. Use DynamoDB's atomic UpdateItem with ADD, which increments server-side in a single atomic operation.

## Step 9 — DynamoDB Table

**Why:** 

Lambda needs least-privilege permission to write to exactly this DynamoDB table, plus permission to write its own CloudWatch logs.

**GUI:**

- IAM console → Roles → Create role
- Trusted entity: AWS service → Lambda
- Attach policy: AWSLambdaBasicExecutionRole (logging)
- Create role: visitor-counter-lambda-role
- After creation, add an inline policy scoped to just UpdateItem on the table ARN

## Step 10 — Lambda Function

**Why:** 

Lambda gives you compute that scales to zero — you pay nothing when nobody's visiting your site, vs. an always-on EC2/container backend.
Code — lambda_function.py:

```
import json
import boto3
import os

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table(os.environ.get('TABLE_NAME', 'VisitorCounter'))

def lambda_handler(event, context):
    response = table.update_item(
        Key={'counter_id': 'homepage'},
        UpdateExpression='ADD visit_count :incr',
        ExpressionAttributeValues={':incr': 1},
        ReturnValues='UPDATED_NEW'
    )
    count = int(response['Attributes']['visit_count'])
    return {
        'statusCode': 200,
        'headers': {
            'Content-Type': 'application/json'
        },
        'body': json.dumps({'count': count})
    }
```

**GUI:**

- Lambda console → Create function
- Author from scratch, name: visitor-counter-fn, Runtime: Python 3.12
- Execution role: Use an existing role → visitor-counter-lambda-role
- Paste the code above into the inline editor → Deploy
- Configuration → Environment variables → add TABLE_NAME = VisitorCounter

## Step 11 — API Gateway (HTTP API + CORS)

**Why:** 

Lambda has no public URL by itself. API Gateway is the front door — it also handles CORS preflight (OPTIONS) so a browser on yourdomain.com is allowed to call an execute-api.amazonaws.com origin.

*HTTP API vs REST API (the trade-off you'll get asked about):*

<img width="917" height="317" alt="image" src="https://github.com/user-attachments/assets/32d255c8-24eb-4d58-b05f-cbb7e1f57aae" />

**GUI:**

- API Gateway console → Create API → HTTP API → Build
- Add integration: Lambda → visitor-counter-fn
- Route: GET /visitor-count
- Stage: $default, auto-deploy: on
- After creation → CORS tab → Access-Control-Allow-Origin: https://yourdomain.com (use * only while testing), Allow methods: GET
- Copy the Invoke URL

**Gotcha:** 

Forgetting the add-permission step is the #1 cause of "403/500 from API Gateway, works fine when I test Lambda directly" — proxy integrations need an explicit resource-based policy grant.

## Step 12 — Wire the Frontend & Redeploy

Update index.html's script tag with your real invoke URL:

```
<script>
  fetch('https://abc123xyz.execute-api.ap-south-1.amazonaws.com/visitor-count')
    .then(r => r.json())
    .then(data => {
      document.getElementById('visitor-count').textContent = data.count;
    })
    .catch(() => {
      document.getElementById('visitor-count').textContent = 'N/A';
    });
</script>
```

**Gotcha:** 

CloudFront caches aggressively by default. Every time you update index.html, you must invalidate (or version your filenames) — this is a very common "why isn't my change showing up" moment.

## Step 13 — Test End-to-End

```
curl -I https://yourdomain.com          # should return 200, served via CloudFront
curl https://abc123xyz.execute-api.ap-south-1.amazonaws.com/visitor-count
# {"count": 1}

Then load *https://yourdomain.com* in a browser a few times — the count should increment.
```

## Step 14 — Cost Control / Cleanup

Everything here is pay-per-use and near-zero at portfolio traffic, but if you ever want to tear it down:

```
aws cloudfront delete-distribution --id YOUR_DIST_ID --if-match ETAG   # must disable first, takes time
aws s3 rm s3://$BUCKET --recursive && aws s3api delete-bucket --bucket $BUCKET
aws lambda delete-function --function-name visitor-counter-fn
aws dynamodb delete-table --table-name VisitorCounter
aws apigatewayv2 delete-api --api-id YOUR_API_ID
aws route53 delete-hosted-zone --id YOUR_ZONE_ID   # only if you're done with the domain entirely
```

**Interview Cheat Sheet (the gotchas, in one place)**

- ACM for CloudFront must be us-east-1
- S3 website endpoint vs REST+OAC — know both, explain why OAC is preferred now
- Counter increments must be atomic (UpdateItem + ADD), never read-modify-write
- CloudFront caches — invalidate or version filenames after deploy
- API Gateway proxy integrations need explicit lambda:InvokeFunction permission
- DynamoDB on-demand vs provisioned — on-demand for spiky/unknown traffic, provisioned (+ autoscaling) for predictable steady load
- HTTP API vs REST API trade-off (table above)
