# Session-15-Serverless Lab 1: Serverless Image Upload and Compression System

## Scenario: Automating Image Optimization in a Serverless Architecture

### Background

You are part of a DevOps team responsible for managing infrastructure and automation for a content-heavy application. The product team reports a sharp increase in high-resolution image uploads from users, which is driving up S3 storage costs and slowing down content delivery. Engineering wants a solution that:

- Requires no EC2 or container maintenance  
- Scales automatically with user uploads  
- Is cost-effective and observable  
- Supports future enhancements like tagging, foldering, and analytics  

---

### Your Task (DevOps Perspective)

Design and implement a **serverless image upload and optimization pipeline** using AWS-managed services:

| Requirement           | DevOps Objective                                          |
|------------------------|------------------------------------------------------------|
| Accept image uploads   | Configure and secure an input S3 bucket                   |
| Trigger processing     | Set up event-driven Lambda using S3 trigger              |
| Image optimization     | Use Lambda (Python + PIL) to convert/resize images       |
| Store output           | Route optimized images to a separate S3 bucket           |
| Observe system behavior| Enable CloudWatch logging and monitoring                 |
| Ensure access control  | Create IAM roles with least-privilege permissions        |
| Improve file handling  | Add logging, format validation, and error handling       |
| Optimize cost          | Compress to reduce average image size significantly      |

---

### Why This Matters (DevOps Skill Development)

This lab teaches:

- Event-driven architectures (S3 to Lambda)  
- Managing Lambda layers and external dependencies (Pillow via public or custom layer)  
- Role-based access control using IAM  
- Logging and observability using CloudWatch Logs  
- S3-based automation without persistent compute resources  
- Error handling, logging, and fail-safe design  
- Reusable serverless design patterns for media pipelines  

---

### Real-World Use Case Applications

- Image ingestion for content moderation pipelines  
- Pre-processing user uploads for blog, portfolio, or gallery sites  
- Cost optimization for image-heavy S3 workflows  
- CI/CD-style image pipelines before pushing to production  
- Lightweight resizing and conversion prior to ML model input  



### Objective

Automatically convert user-uploaded images (e.g., PNG to JPEG) using:

- Amazon S3 for uploads and output storage  
- AWS Lambda for image processing using the Pillow (PIL) library  
- S3 event-based triggers to invoke Lambda  

This lab simulates workflows common in photo apps, social media, e-commerce, image archives, and moderation systems.

---

### Architecture Overview

<img width="468" alt="image" src="https://github.com/user-attachments/assets/4e695e44-884f-473b-8393-650aa7d7772d" />


---

### Prerequisites

- AWS account with permissions for S3 and Lambda  
- IAM role for Lambda with S3 access and basic Lambda execution  
- CloudWatch logging enabled  
- AWS Console or CLI access  

---

## Step-by-Step Setup

### Step 1: Create Two S3 Buckets

1. Open the S3 Console  
2. Create a bucket:
   - Name: `image-upload-bucket-<yourname>`  
   - Disable public access  
3. Create another bucket:
   - Name: `image-output-bucket-<yourname>`  

---

### Step 2: Create a Lambda Execution Role

1. Open the IAM Console → Roles  
2. Click Create role  
3. Choose Lambda as the trusted entity  
4. Attach the following policies:  
   - `AmazonS3FullAccess` (for testing; use fine-grained access in production)  
   - `AWSLambdaFullAccess'
   - `CloudwatchFullAccess'  
5. Name the role: `lambda-s3-image-role`  

Note: `AWSLambdaBasicExecutionRole` provides required permissions for CloudWatch logging.

---

### Step 3: Create the Lambda Function

1. Open the Lambda Console  
2. Click Create function  
   - Name: `ImageConverter`  
   - Runtime: `Python 3.13`  
   - Role: Choose existing → `lambda-s3-image-role`  
3. Replace the default code with the following:

```python
import boto3
import os
import logging
from PIL import Image, UnidentifiedImageError
from io import BytesIO

logger = logging.getLogger()
logger.setLevel(logging.INFO)

s3 = boto3.client('s3')
OUTPUT_BUCKET = 'image-output-bucket-dan'  # Change to your actual output bucket
TARGET_QUALITY = 40  # Low value for aggressive compression

def lambda_handler(event, context):
    logger.info("Event: %s", event)

    try:
        src_bucket = event['Records'][0]['s3']['bucket']['name']
        src_key = event['Records'][0]['s3']['object']['key']
        logger.info("Image: %s from %s", src_key, src_bucket)

        obj = s3.get_object(Bucket=src_bucket, Key=src_key)
        image_data = obj['Body'].read()

        with Image.open(BytesIO(image_data)) as img:
            logger.info("Original size: %s, mode: %s, format: %s", img.size, img.mode, img.format)

            # Convert to RGB if needed
            if img.mode in ("RGBA", "P"):
                img = img.convert("RGB")

            # Resize to 30% of original dimensions
            width, height = img.size
            new_size = (int(width * 0.3), int(height * 0.3))
            img = img.resize(new_size, Image.Resampling.LANCZOS)
            logger.info("Resized to: %s", new_size)

            # Strip metadata
            img.info.pop("exif", None)
            img.info.pop("icc_profile", None)

            # Save compressed JPEG
            buffer = BytesIO()
            output_key = os.path.splitext(src_key)[0] + ".jpg"

            img.save(buffer, format="JPEG", quality=TARGET_QUALITY, optimize=True, progressive=True)
            buffer.seek(0)

            size_mb = len(buffer.getvalue()) / (1024 * 1024)
            logger.info("Final image size: %.2f MB", size_mb)

            s3.put_object(
                Bucket=OUTPUT_BUCKET,
                Key=output_key,
                Body=buffer,
                ContentType='image/jpeg'
            )

        return {
            'statusCode': 200,
            'body': f"Image resized and compressed to {OUTPUT_BUCKET}/{output_key} (Size: {size_mb:.2f}MB)"
        }

    except UnidentifiedImageError:
        logger.error("Unsupported or corrupted image format.")
        return {
            'statusCode': 400,
            'body': "Unsupported or corrupted image format."
        }

    except Exception as e:
        logger.error("Error: %s", str(e))
        return {
            'statusCode': 500,
            'body': "Internal server error during image processing."
        }

```
optional
Add the following Lambda environment variable:

- Key: `OUTPUT_BUCKET`  
- Value: `image-output-bucket-<yourname>`  

Then Deploy the function.

---

### Step 4: Add Pillow Layer to Lambda

Pillow is not included in the Lambda runtime by default.  
Use this public Layer ARN for Python 3.13 in us-east-1:

```
arn:aws:lambda:us-east-1:225989350860:layer:pillow-layer:1
```

To add:

1. Go to your Lambda function  
2. Click Layers → Add a layer  
3. Choose Specify an ARN and paste the ARN  
4. Click Add  

For other regions, refer to the Klayers GitHub repository.

---

### Step 5: Configure S3 Event Trigger

1. Open your image-upload bucket  
2. Navigate to Properties → Event notifications  
3. Add a new event:
   - Name: trigger-lambda-on-upload  
   - Event type: PUT  
   - Prefix: uploads/  
   - Suffix: .png  
   - Destination: Lambda → select ImageConverter  

---

### Test the System

** Option 1 **
Upload a PNG file to your upload bucket:

```bash
aws s3 cp test.png s3://image-upload-bucket-<yourname>/uploads/test.png
```

Check the output bucket:

```bash
aws s3 ls s3://image-output-bucket-<yourname>/uploads/
```

You should see: `test.jpg`

** Option 2 **
On the console

---

### Troubleshooting Tips
* Check CloudWatch Logs
Use the Lambda → Monitor → Logs tab (or go directly to CloudWatch) to view real-time errors, event data, and print/debug logs.

* Start with Common Format (.JPG)
For early testing, use .JPG files to ensure baseline functionality. Once the flow between the upload and output buckets is stable, begin testing with other formats as supported in your application logic.

* Expect up to 70% file size reduction
If resizing and compression are applied, especially on large or high-quality images, you can see significant size savings.

* Update the Bucket Name Placeholder
Remember to replace the OUTPUT_BUCKET variable in your Lambda function with your actual S3 output bucket name.

* Layer Setup Is Critical
Ensure you add the PIL/Pillow Lambda Layer via the AWS Console by selecting "Layers → Add layer → Specify ARN". If this step is skipped or misconfigured, your function will fail with No module named 'PIL'.

* No Lambda Destinations Needed
If your Lambda role includes AmazonS3FullAccess, you do not need to configure a Lambda destination.

* S3 Trigger Is Sufficient
The only trigger your function needs is the S3 Event Notification set to invoke the Lambda when a new object is uploaded (PUT event).

* Add CloudWatch Logging Permissions
To ensure that Lambda logs are captured, attach the CloudWatchFullAccess policy to the Lambda execution role.
  Minimum required:

AWSLambdaBasicExecutionRole

CloudWatchFullAccess (or fine-tuned logs permission)

### Optional Enhancements
Smart Resizing
Add logic to resize images to a maximum width or height (e.g., 1920px), while preserving aspect ratio.

Output Folder Structure
Save converted images under a specific subdirectory such as /compressed/ or /processed/.

Multi-Format Support
Extend processing to include formats like BMP, GIF, and TIFF using Pillow's available modules.

Tag Output Files
Use s3.put_object_tagging() to attach metadata (e.g., source=converted, quality=low) to help track image flow.

Two-Layer Compression (3-Bucket Architecture)
Implement a multi-stage pipeline:

Bucket A: Upload input

Bucket B: Convert to JPEG and downscale

Bucket C: Further compress to ultra-light versions or Web-optimized
