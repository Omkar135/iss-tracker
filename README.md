# iss-tracker
# Live ISS Tracker – Cross-Account AWS STS Demo

A small weekend project where I used **AWS STS**, **Lambda**, **API Gateway**, and **S3 across two AWS accounts** to track the live position of the **International Space Station (ISS)** and expose it via a simple frontend hosted on **GitHub Pages**.

> Goal: show how to use **STS AssumeRole** for secure cross-account access, and turn it into a fun, visual ISS tracker.

---

## 🌍 High-Level Architecture

**Account A (caller)**  
- API Gateway (HTTP API)  
- Lambda function (`iss-tracker-lambda`)  
- IAM role for Lambda with permission to `sts:AssumeRole` on a role in Account B

**Account B (target)**  
- S3 bucket (stores ISS snapshots as JSON)  
- IAM role with:
  - **Trust policy**: _“Allow `sts:AssumeRole` from Account A’s Lambda role”_  
  - **Permissions policy**: `s3:PutObject` (and optionally `s3:GetObject`, `s3:ListBucket`) on that bucket

**Flow**

1. Frontend (GitHub Pages) calls API Gateway endpoint (`POST /uploads`).
2. API Gateway invokes Lambda in **Account A**.
3. Lambda:
   - Calls public ISS API `http://api.open-notify.org/iss-now.json`.
   - Enriches with distance from “home” (e.g., Boston, MA).
   - Uses **STS** to `AssumeRole` into **Account B**.
   - Writes an `iss/YYYY-MM-DD/iss-<timestamp>.json` object into S3.
   - Returns a JSON response with friendly fields like `pretty_lat`, `pretty_lon`, `distance_km_from_home`, `description`.
4. Frontend updates the card to show the latest ISS snapshot.

---

## 🧰 Tech Stack

- **AWS**
  - API Gateway (HTTP API)
  - Lambda (Python 3.14)
  - STS (`AssumeRole` for cross-account access)
  - S3 (JSON storage in Account B)
  - IAM (roles, trust & permission policies)
- **Backend**
  - Python (`boto3`, `urllib.request`, `math`, `datetime`)
- **Frontend**
  - Plain **HTML + CSS + vanilla JavaScript**
  - Hosted on **GitHub Pages**
- **External API**
  - [Open Notify ISS Now API](http://api.open-notify.org/iss-now.json)

---

## 🧱 Lambda Function Overview

The Lambda handler does:

1. **CORS & preflight**
   - Handles `OPTIONS` requests with:
     ```json
     Access-Control-Allow-Origin: "*"
     Access-Control-Allow-Methods: "POST, OPTIONS"
     Access-Control-Allow-Headers: "Content-Type"
     ```

2. **Fetch ISS position**
   - Calls `http://api.open-notify.org/iss-now.json` with a timeout.
   - Parses latitude, longitude, and timestamp.
   - Converts timestamp to ISO 8601 UTC.
   - Derives hemispheres (`N/S`, `E/W`) and builds `pretty_lat`, `pretty_lon`.

3. **Compute distance from “home”**
   - Uses the Haversine formula to compute great-circle distance between:
     - Home coords (configured by env vars)
     - ISS coords
   - Produces text like:
     > “At 2025-11-18T08:31:51+00:00, the ISS was at 15.80°N, 90.51°E, about 13278 km from Boston, MA.”

4. **Assume cross-account role & write to S3 (Account B)**
   - Uses `sts:AssumeRole` with `CROSS_ACCOUNT_ROLE_ARN`.
   - Creates an S3 client with the temporary credentials.
   - Writes a JSON object to:
     ```
     iss/YYYY-MM-DD/iss-<timestamp>.json
     ```
   - `ContentType = "application/json"`

5. **Return response (with CORS)**
   - On success:
     ```json
     {
       "message": "ISS position stored successfully",
       "bucket": "<bucket-name>",
       "key": "iss/2025-11-18/iss-2025-11-18T08-31-51+00-00.json",
       "iss": {
         "timestamp_utc": "...",
         "pretty_lat": "15.80°N",
         "pretty_lon": "90.51°E",
         "distance_km_from_home": 13278.0,
         "description": "..."
       }
     }
     ```
   - On error (including ISS API timeout): returns `500` with an `error` field **but still includes CORS headers**, so the frontend can show a friendly message.

---

## ⚙️ Environment Variables (Lambda)

In the Lambda configuration set:

- `CROSS_ACCOUNT_ROLE_ARN`  
  ARN of the IAM role in **Account B** that this Lambda is allowed to assume.

- `TARGET_BUCKET_NAME`  
  Name of the S3 bucket in **Account B**.

- `HOME_LAT` / `HOME_LON`  
  Your home coordinates (floats) to compute the distance (e.g., Boston).

- `HOME_LABEL`  
  Human label used in the UI text (e.g., `"Boston, MA"`).

---

## 🔐 IAM & STS Setup

### 1. Role in Account B (target)

**Trust policy** (simplified):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::<ACCOUNT_A_ID>:role/<LambdaExecutionRole>"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}

