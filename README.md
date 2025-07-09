# Session-15-Serverless

## Lab 1: Serverless Image Upload and Compression System

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
   - `AWSLambdaBasicExecutionRole'
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
OUTPUT_BUCKET = os.environ['OUTPUT_BUCKET']
TARGET_QUALITY = 40

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

            if img.mode in ("RGBA", "P"):
                img = img.convert("RGB")

            width, height = img.size
            new_size = (int(width * 0.3), int(height * 0.3))
            img = img.resize(new_size, Image.Resampling.LANCZOS)
            logger.info("Resized to: %s", new_size)

            img.info.pop("exif", None)
            img.info.pop("icc_profile", None)

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

Add the following Lambda environment variable:

- Key: `OUTPUT_BUCKET`  
- Value: `image-output-bucket-<yourname>`  

Then Deploy the function.

---

### Step 4: Add Pillow Layer to Lambda

Pillow is not included in the Lambda runtime by default.  
Use this public Layer ARN for Python 3.13 in us-east-1:

```
arn:aws:lambda:us-east-1:770693421928:layer:Klayers-p313-Pillow:1
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

Upload a PNG file to your upload bucket:

```bash
aws s3 cp test.png s3://image-upload-bucket-<yourname>/uploads/test.png
```

Check the output bucket:

```bash
aws s3 ls s3://image-output-bucket-<yourname>/uploads/
```

You should see: `test.jpg`

---

### Optional Enhancements

- Resize to a max width/height while maintaining aspect ratio  
- Store processed images in subfolders  
- Add support for BMP, GIF, etc.  
- Tag files with metadata in S3  
