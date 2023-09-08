# serverless-image-processing
It is an event driven architecture in which I use an s3 bucket to add the file , which then invokes the lambda function. The function creates a thumbnail of the file and saves it in another bucket .

<img width="262" alt="image2" src="https://github.com/japn3et/serverless-image-processing/assets/140784801/1b0b6ad0-c94b-4c5d-9c06-f4694df12ff6">

**STEP 1: Make 2 S3 buckets (source and destination)**

I have kept names as 
source-bucket29 
source-bucket29-resized

Keep all other options default 

<img width="647" alt="image9" src="https://github.com/japn3et/serverless-image-processing/assets/140784801/9e4e3bd3-eda0-4ed7-9210-769b738c8fba">

Now upload any jpg or png file to source bucket 

<img width="627" alt="image14" src="https://github.com/japn3et/serverless-image-processing/assets/140784801/4327e418-c558-44c6-8be0-ff7867bae1ef">

**Step 2 : create a policy that allows lambda function to access s3**
Paste this json code into policy editor 

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "logs:PutLogEvents",
                "logs:CreateLogGroup",
                "logs:CreateLogStream"
            ],
            "Resource": "arn:aws:logs:*:*:*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject"
            ],
            "Resource": "arn:aws:s3:::*/*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:PutObject"
            ],
            "Resource": "arn:aws:s3:::*/*"
        }
    ]
}
```
<img width="787" alt="image15" src="https://github.com/japn3et/serverless-image-processing/assets/140784801/9fa881e7-e5a9-442f-8392-4883d3b2b92b">

**Step 3 : Create an IAM role that allows Lambda function to access resources**

Select the policy from the previous step for lambda

<img width="785" alt="image11" src="https://github.com/japn3et/serverless-image-processing/assets/140784801/d5bb8c7e-20b9-49aa-a370-5f5ed31ca1c8">

**Step 4 : Make function deployment package**
This will have function code and dependencies for image resizing 

Save this python code as lambda_function.py 

```
import boto3
import os
import sys
import uuid
from urllib.parse import unquote_plus
from PIL import Image
import PIL.Image
            
s3_client = boto3.client('s3')
            
def resize_image(image_path, resized_path):
  with Image.open(image_path) as image:
    image.thumbnail(tuple(x / 2 for x in image.size))
    image.save(resized_path)
            
def lambda_handler(event, context):
  for record in event['Records']:
    bucket = record['s3']['bucket']['name']
    key = unquote_plus(record['s3']['object']['key'])
    tmpkey = key.replace('/', '')
    download_path = '/tmp/{}{}'.format(uuid.uuid4(), tmpkey)
    upload_path = '/tmp/resized-{}'.format(tmpkey)
    s3_client.download_file(bucket, key, download_path)
    resize_image(download_path, upload_path)
    s3_client.upload_file(upload_path, '{}-resized'.format(bucket), 'resized-{}'.format(key))

```

Now in the same directory 
Make a new directory called package which has Pillow library and boto 3 ( aws python sdk) with pre compiled binaries for amazon linux 2

```
mkdir package
pip install \
--platform manylinux2014_x86_64 \
--target=package \
--implementation cp \
--python-version 3.9 \
--only-binary=:all: --upgrade \
pillow boto3

```
Make a zip file with the python folders and dependency folders at root of the zip file 
```
cd package
zip -r ../lambda_function.zip .
cd ..
zip lambda_function.zip lambda_function.py
```

<img width="449" alt="image4" src="https://github.com/japn3et/serverless-image-processing/assets/140784801/c7e01269-9498-4505-bbaa-7b334d3332d2">

**Sep 5 : Make lambda function** 

![image13](https://github.com/japn3et/serverless-image-processing/assets/140784801/a093076a-17e7-48a0-af60-a4e0f08fe676)

Create function and in code source , upload zip file 

<img width="606" alt="image1" src="https://github.com/japn3et/serverless-image-processing/assets/140784801/4036d04d-67dd-4ffb-9ef5-081bec3e8b9a">

**Step 6 : Now create trigger with the following options** 

<img width="618" alt="image8" src="https://github.com/japn3et/serverless-image-processing/assets/140784801/8fd9c2b9-562e-4ab9-a643-b89372a61d9c">

**STep 7 : Test using dummy event** 
Make new test event 

![image7](https://github.com/japn3et/serverless-image-processing/assets/140784801/ea24b277-034f-4c30-af69-aad8975a9fb9)

<img width="418" alt="image6" src="https://github.com/japn3et/serverless-image-processing/assets/140784801/c9783999-ecb2-439b-9a2d-6abbe03f0db1">

After we get a success message and we can check the resized file in destination bucket 

<img width="1000" alt="image5" src="https://github.com/japn3et/serverless-image-processing/assets/140784801/34a7b212-2226-413b-be76-09f1fbd58eed">

**Step 8 : Test with trigger** 

Now that we know that it is working 
We will directly upload to s3 bucket , lambda will be invoked and we can see resized image in destination bucket

![image12](https://github.com/japn3et/serverless-image-processing/assets/140784801/6c8022b9-2fdf-4209-b065-1078e6f939b3)

![image10](https://github.com/japn3et/serverless-image-processing/assets/140784801/9f3332fb-fdb1-41cc-a1fc-7f34b58e0c7c)

**Cleanup** 

Delete lambda function , role , policy and s3 buckets 


