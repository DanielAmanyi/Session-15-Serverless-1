
# 🖼️ Serverless Image Converter – Session 15

This project demonstrates a simple AWS serverless image processing pipeline using **S3**, **Lambda (Python 3.12)**, and the **Pillow (PIL)** library via a public Lambda layer.

## 🔧 What It Does

When a user uploads an image (e.g., PNG) to an input S3 bucket, the Lambda function:
1. Downloads the image
2. Converts it to JPEG
3. Uploads it to an output S3 bucket

## 🗂️ Project Structure

```
├── lambda_function.py         # Lambda code to convert and upload images
├── pillow-layer-info.txt      # Reference to the public Klayers ARN for Pillow
└── sample-image.png           # (Optional) Test image to trigger the flow
```

## 🚀 Setup Guide

### 1. Create the Required Buckets
Create two S3 buckets:
- **Input bucket**: `image-input-bucket-<yourname>`
- **Output bucket**: `image-output-bucket-<yourname>`

Update `OUTPUT_BUCKET` in `lambda_function.py` accordingly.

---

### 2. Create the IAM Role
Attach the following policy to a role named: `lambda-s3-image-role`

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::image-*/*"
    }
  ]
}
```

---

### 3. Create the Lambda Function

- **Name**: `ImageConverter`
- **Runtime**: Python 3.12
- **Role**: `lambda-s3-image-role`

Replace the default code with `lambda_function.py`.

---

### 4. Add the Pillow Layer

Use the **Klayers** public ARN for Pillow (Python 3.12):

```
arn:aws:lambda:us-east-1:770693421928:layer:Klayers-p312-Pillow:1
```

Steps:
- Go to **Layers** → **Add a layer**
- Choose **“Specify an ARN”**
- Paste the ARN above and add

---

### 5. Add S3 Trigger

Configure the Lambda trigger:
- Source bucket: `image-input-bucket-<yourname>`
- Event type: **PUT**

---

### 6. Test the Flow

Upload a PNG to the input bucket:
```bash
aws s3 cp sample-image.png s3://image-input-bucket-<yourname>/
```

Check the output bucket for a converted `.jpg` image.

---

## ✅ Output Example

Uploaded:
```
sample-image.png → image-input-bucket
```

Result:
```
sample-image.jpg → image-output-bucket
```

---

## 🧠 Notes

- The Pillow layer is not bundled in the repo; it's pulled via ARN.
- If you change runtime to Python 3.13, you'll need a custom layer build.

---

## 📎 Author

Amanyi Daniel — [GitHub Profile](https://github.com/DanielAmanyi)
