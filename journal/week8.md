# Week 8 — Serverless Image Processing

This week the team will be talking about CDK

# CDK
This section will illustrate the steps to create a CDK project.





Next Define our bucket `

Before launching the CDK, you need to boostrap

Bootstrapping is the process of provisioning resources for the AWS CDK before you can deploy AWS CDK apps into an AWS environment.

If you launch in several regions, you need to bootstrap for each region that you need.

Follow you find the command to boostrapping in a specific region

```sh
cdk bootstrap "aws://ACCOUNTNUMBER/REGION"
```

Example:
```sh
for a single region
cdk bootstrap "aws://123456789012/us-east-1"

for a multiple region
cdk bootstrap 123456789012/us-east-1 123456789012/us-west-1
```

Create a file named `thumbing-serverless-cdk` and run 

```

npm install aws-cdk -g
cdk init app --language typescript
```
These will generate several files in the in the folder, note the life blood of these project will be in the `lib` folder generated. All the infrastructure as code will live in the `lib/thumbing-serverless-cdk-stack.ts` folder

First you want to do is define our s3 bucket, this is where our Avater images are going to live for image processing, we will have to import some of these objests, `'import { Construct } from 'constructs';`)

From gitpod.yml add these lines of code. This automatically reinstalls cdk every time you launch a new workspace in gitpod. plus it copies the file .env.example into .env (the file .env.example will be created later)

```sh
 - name: cdk
    before: |
      cd thumbing-serverless-cdk
      cp .env.example .env
      npm i
      npm install aws-cdk -g
      cdk --version
```

To initialise the project type `cdk init app --language typescript` (instead of typescript, you can choose another language supported by cdk such JavaScript, TypeScript, Python, Java, C#)

To work with the cdkfile, go to the file inside the `lib/thumbing-serverless-cdk-stack.ts`

To define the s3 buckeet do the following:

import the library for s3 

```sh
import * as s3 from 'aws-cdk-lib/aws-s3';
```

########

set the variable (Since this project is in typescript which is strongly typed. it means all must have  a specific data type)
```sh
const uploadsBucketName: string = process.env.UPLOADS_BUCKET_NAME as string;
const uploadsBucket = this.createBucket(uploadsBucketName);
```

with the following code, cdk creates the s3 bucket

```sh
 createBucket(bucketName: string): s3.IBucket{
    const bucket = new s3.Bucket(this, 'UploadsBucket', {
      bucketName: bucketName,
      removalPolicy: cdk.RemovalPolicy.DESTROY
    });
    return bucket;
  }
```

To launch the AWS CloudFormation template based on the AWS CDK, type:
```sh
cdk synth
```
**Note** 
- if you run again the code, the folder cdk.out will be updated.
- this is a good way to troubleshoot and see what resources have been created before deploying.




**Note**
If you can deploy your stack and add new resources on top, you will face some issues s with Dynamodb when you start renaming it and there is some data on it. this could delete the entire dynamodb resource.


The next step is to add lambda to our stack.

You need to import the lambda from the cdk library and the dotenv library
```sh
import * as lambda from 'aws-cdk-lib/aws-lambda'
import * as dotenv from 'dotenv';
```

from the folder, run the following command to install the dotenv dependency to import the file .env
```sh
 npm i dotenv
```

set the variables for the lambda
```sh
const uploadsBucketName: string = process.env.UPLOADS_BUCKET_NAME as string;
const assetsBucketName: string = process.env.ASSETS_BUCKET_NAME as string;
const functionPath: string = process.env.THUMBING_FUNCTION_PATH as string;
const folderInput: string = process.env.THUMBING_S3_FOLDER_INPUT as string;
const folderOutput: string = process.env.THUMBING_S3_FOLDER_OUTPUT as string;
const webhookUrl: string = process.env.THUMBING_WEBHOOK_URL as string;
const topicName: string = process.env.THUMBING_TOPIC_NAME as string;
console.log('uploadsBucketName',uploadsBucketName)
console.log('assetsBucketName',assetsBucketName)
console.log('folderInput',folderInput)
console.log('folderOutput',folderOutput)
console.log('webhookUrl',webhookUrl)
console.log('topicName',topicName)
console.log('functionPath',functionPath)

const lambda = this.createLambda(functionPath, uploadsBucketName, assetsBucketName, folderInput, folderOutput)

```

and then we create the lambda function

```sh
  createLambda(functionPath: string, uploadsBucketName: string, assetsBucketName:string, folderInput: string, folderOutput: string): lambda.IFunction{
    const lambdaFunction = new lambda.Function(this, 'thumbLambda', {
      runtime: lambda.Runtime.NODEJS_18_X,
      handler: 'index.handler',
      code: lambda.Code.fromAsset(functionPath),
      environment: {
        DEST_BUCKET_NAME: assetsBucketName,
        FOLDER_INPUT: folderInput,
        FOLDER_OUTPUT: folderOutput,
        PROCESS_WIDTH: '512',
        PROCESS_HEIGHT: '512'
      }
    });
    return lambdaFunction;
  }
  ```
**Note**
Lambda function needs at least 3 parameters Runtime (language of the code), handler and code (which is the source where is located our code)

create the .env.example inside of our cdk project with the following info
```sh
UPLOADS_BUCKET_NAME="johnbuen-uploaded-avatars"
ASSETS_BUCKET_NAME="assets.yourdomanin.com"
THUMBING_FUNCTION_PATH="/workspace/aws-bootcamp-cruddur-2023/aws/lambdas/process-images"
THUMBING_S3_FOLDER_INPUT="avatars/original"
THUMBING_S3_FOLDER_OUTPUT="avatars/processed"
THUMBING_WEBHOOK_URL="https://api.yourdomain.com/webhooks/avatar"
THUMBING_TOPIC_NAME="crudduer-assets"

```
**Note** 
- It is a good practice to create a folder for the lambda codes for each project so it is to refer to which project belongs the code.

- The UPLOADS_BUCKET_NAME and ASSETS_BUCKET_NAME must be unique as this will refer to the s3 bucket. change the name of the bucket with your domain (for example assets.example.com)

- This file will be copied with the extension ".env" and will be necessery for thumbing-serverless-cdk-stack file.

if you launch the **cdk synth** the result will be something similar to this:



Create the folder for the lambda called **process-images** under aws/lambdas

One file called **index.js**

```sh
const process = require('process');
const {getClient, getOriginalImage, processImage, uploadProcessedImage} = require('./s3-image-processing.js')
const path = require('path');

const bucketName = process.env.DEST_BUCKET_NAME
const folderInput = process.env.FOLDER_INPUT
const folderOutput = process.env.FOLDER_OUTPUT
const width = parseInt(process.env.PROCESS_WIDTH)
const height = parseInt(process.env.PROCESS_HEIGHT)

client = getClient();

exports.handler = async (event) => {
  console.log('event',event)

  const srcBucket = event.Records[0].s3.bucket.name;
  const srcKey = decodeURIComponent(event.Records[0].s3.object.key.replace(/\+/g, ' '));
  console.log('srcBucket',srcBucket)
  console.log('srcKey',srcKey)

  const dstBucket = bucketName;
  filename = path.parse(srcKey).name
  const dstKey = `${folderOutput}/${filename}.jpg`
  console.log('dstBucket',dstBucket)
  console.log('dstKey',dstKey)

  const originalImage = await getOriginalImage(client,srcBucket,srcKey)
  const processedImage = await processImage(originalImage,width,height)
  await uploadProcessedImage(client,dstBucket,dstKey,processedImage)
};
```

one file called **test.js** which has less code and hardcoded some env vars:
```sh
const {getClient, getOriginalImage, processImage, uploadProcessedImage} = require('./s3-image-processing.js')

async function main(){
  client = getClient()
  const srcBucket = 'cruddur-thumbs'
  const srcKey = 'avatar/original/data.jpg'
  const dstBucket = 'cruddur-thumbs'
  const dstKey = 'avatar/processed/data.png'
  const width = 256
  const height = 256

  const originalImage = await getOriginalImage(client,srcBucket,srcKey)
  console.log(originalImage)
  const processedImage = await processImage(originalImage,width,height)
  await uploadProcessedImage(client,dstBucket,dstKey,processedImage)
}

main()
```

Another file  called **s3-image-processing.js**

```sh
const sharp = require('sharp');
const { S3Client, PutObjectCommand, GetObjectCommand } = require("@aws-sdk/client-s3");

function getClient(){
  const client = new S3Client();
  return client;
}

async function getOriginalImage(client,srcBucket,srcKey){
  console.log('get==')
  const params = {
    Bucket: srcBucket,
    Key: srcKey
  };
  console.log('params',params)
  const command = new GetObjectCommand(params);
  const response = await client.send(command);

  const chunks = [];
  for await (const chunk of response.Body) {
    chunks.push(chunk);
  }
  const buffer = Buffer.concat(chunks);
  return buffer;
}

async function processImage(image,width,height){
  const processedImage = await sharp(image)
    .resize(width, height)
    .jpeg()
    .toBuffer();
  return processedImage;
}

async function uploadProcessedImage(client,dstBucket,dstKey,image){
  console.log('upload==')
  const params = {
    Bucket: dstBucket,
    Key: dstKey,
    Body: image,
    ContentType: 'image/jpeg'
  };
  console.log('params',params)
  const command = new PutObjectCommand(params);
  const response = await client.send(command);
  console.log('repsonse',response);
  return response;
}

module.exports = {
  getClient: getClient,
  getOriginalImage: getOriginalImage,
  processImage: processImage,
  uploadProcessedImage: uploadProcessedImage
}
```

create the file called **example.json**

This file is just for reference.
```sh
{
    "Records": [
        {
            "eventVersion": "2.1",
            "eventSource": "aws:s3",
            "awsRegion": "eu-west-2",
            "eventTime": "2023-04-04T12:34:56.000Z",
            "eventName": "ObjectCreated:Put",
            "userIdentity": {
                "principalId": "EXAMPLE"
            },
            "requestParameters": {
                "sourceIPAddress": "127.0.0.1"
            },
            "responseElements": {
                "x-amz-request-id": "EXAMPLE123456789",
                "x-amz-id-2": "EXAMPLE123/abcdefghijklmno/123456789"
            },
            "s3": {
                "s3SchemaVersion": "1.0",
                "configurationId": "EXAMPLEConfig",
                "bucket": {
                    "name": "assets.johnbuen.co.uk",
                    "ownerIdentity": {
                        "principalId": "EXAMPLE"
                    },
                    "arn": "arn:aws:s3:::assets.johnbuen.co.uk"
                },
                "object": {
                    "key": "avatars/original/data.jpg",
                    "size": 1024,
                    "eTag": "EXAMPLEETAG",
                    "sequencer": "EXAMPLESEQUENCER"
                }
            }
        }
    ]
  }
```

**Note**
- On the "awsRegion", insert your region
- On "name" under bucket and arn, replace with your bucket name


from the terminal, move to aws\lambdas\process-image\ and launch the following command
```
npm init -y
```
**Note** This create a new init file 

launch these command to install the libraries

```
npm i sharp
npm i @aws-sdk/client-s3

```
**Note** Make sure to check on the internet the library before installing it as there are some packages with similar names

To deploy the CDK project launch the following code. then check it on cloudformation

```sh
cdk deploy
```
**Note**
- Before deploying, CDK launch a synth to check if the code is correct.

- If you rename a bucket and deploy the entire stack, this wont affect the changes. you need to destroy the entire stack and relaunch using the following command:
```
cdk destroy
```

Launch the following command to install Sharp form the folder of CDK
```sh
npm install
rm -rf node_modules/sharp
SHARP_IGNORE_GLOBAL_LIBVIPS=1 npm install --arch=x64 --platform=linux --libc=glibc sharp
```

with the same code, create a bash script under .bin/avatar called build
```sh
#! /usr/bin/bash

ABS_PATH=$(readlink -f "$0")
SERVERLESS_PATH=$(dirname $ABS_PATH)
BIN_PATH=$(dirname $SERVERLESS_PATH)
PROJECT_PATH=$(dirname $BIN_PATH)
SERVERLESS_PROJECT_PATH="$PROJECT_PATH/thumbing-serverless-cdk"

cd $SERVERLESS_PROJECT_PATH
npm install
rm -rf node_modules/sharp
SHARP_IGNORE_GLOBAL_LIBVIPS=1 npm install --arch=x64 --platform=linux --libc=glibc sharp
```

under the same folder create bash script called **clear**
```sh
#! /usr/bin/bash

ABS_PATH=$(readlink -f "$0")
SERVERLESS_PATH=$(dirname $ABS_PATH)
DATA_FILE_PATH="$SERVERLESS_PATH/files/data.jpg"

aws s3 cp "$DATA_FILE_PATH" "s3://johnbuen-uploaded-avatars/data.jpg"

```



under the same folder create bash script called **upload**
```sh
#! /usr/bin/bash

ABS_PATH=$(readlink -f "$0")
SERVERLESS_PATH=$(dirname $ABS_PATH)
DATA_FILE_PATH="$SERVERLESS_PATH/files/data.jpg"

aws s3 cp "$DATA_FILE_PATH" "s3://assets.$DOMAIN_NAME/avatars/original/data.jpg"
```

and also a script for the list called **ls**
```sh
#! /usr/bin/bash

aws s3 ls s3://$THUMBING_BUCKET_NAME
```

Create the env var to reference your domain name locally and on gitpod (these 2 line will be done just one time)
```sh
export DOMAIN_NAME="yourdomain.com"
gp env DOMAIN_NAME="yourdomain.com"
```
**Note**: if you are using codespace, make sure to add this env var as well there.

and a folder called **files** where you load the image that will be load to s3.


The next step is to hook up the lambda with s3 Event Notification

It is some additional code to our **thumbing-serverless-cdk-stack.ts**

from the section of  variable add the following command
```sh
this.createS3NotifyToLambda(folderInput,lambda,uploadsBucket)
this.createS3NotifyToSns(folderOutput,snsTopic,assetsBucket)
```

Add also this portion of code to sets up an event notification in an Amazon S3 bucket to trigger a specific AWS Lambda function when an object is created with a PUT operation and a key that matches a specific prefix.

```sh

createS3NotifyToLambda(prefix: string, lambda: lambda.IFunction, bucket: s3.IBucket): void {
  const destination = new s3n.LambdaDestination(lambda);
  bucket.addEventNotification(
    s3.EventType.OBJECT_CREATED_PUT,
    destination//,
    //{prefix: prefix}
  )
}
```

Also it needs the library for s3 notification.

```sh
import * as s3n from 'aws-cdk-lib/aws-s3-notifications';
```

We need to import an existing bucket

```sh
  importBucket(bucketName: string): s3.IBucket{
    const bucket = s3.Bucket.fromBucketName(this,"AssetsBucket", bucketName);
    return bucket;
  }
```


and add the following under the previous one
```sh
const assetsBucket = this.importBucket(assetsBucketName);
```

We will need to create an s3 bucket called **assets.yourdomain.com** manually just for this time.

To add the permission to write to the s3 buckets, add the following code to the **thumbing-serverless-cdk-stack**

```
const s3UploadsReadWritePolicy = this.createPolicyBucketAccess(uploadsBucket.bucketArn)
const s3AssetsReadWritePolicy = this.createPolicyBucketAccess(assetsBucket.bucketArn)


```

add the new function

```sh
createPolicyBucketAccess(bucketArn: string){
    const s3ReadWritePolicy = new iam.PolicyStatement({
      actions: [
        's3:GetObject',
        's3:PutObject',
      ],
      resources: [
        `${bucketArn}/*`,
      ]
    });
    return s3ReadWritePolicy;
  }
```

import the library for iam

```sh
import * as iam from 'aws-cdk-lib/aws-iam';
```

and with these lines of code, lambda will have the policy created under the const **s3ReadWritePolicy**
```sh
lambda.addToRolePolicy(s3UploadsReadWritePolicy);
lambda.addToRolePolicy(s3AssetsReadWritePolicy);
```

The last part of the implementation  notification

from **thumbing-serverless-cdk-stack** add the following 

import the libraries for sns and sns subscription

```sh
import * as sns from 'aws-cdk-lib/aws-sns';
import * as subscriptions from 'aws-cdk-lib/aws-sns-subscriptions';
```

and add the creation of  SNStopic, SNS subscritption, SNS policy and attach to the lambda
```sh
const snsTopic = this.createSnsTopic(topicName)
this.createSnsSubscription(snsTopic,webhookUrl)
//const snsPublishPolicy = this.createPolicySnSPublish(snsTopic.topicArn)
//lambda.addToRolePolicy(snsPublishPolicy);
```

and the add the function to create the sns subscription

```sh
createSnsSubscription(snsTopic: sns.ITopic, webhookUrl: string): sns.Subscription {
  const snsSubscription = snsTopic.addSubscription(
    new subscriptions.UrlSubscription(webhookUrl)
  )
  return snsSubscription;
}
```

add the function to create the snstopic
```sh
createSnsTopic(topicName: string): sns.ITopic{
  const logicalName = "ThumbingTopic";
  const snsTopic = new sns.Topic(this, logicalName, {
    topicName: topicName
  });
  return snsTopic;
}
```

add the function to create the event notification
```sh
createS3NotifyToSns(prefix: string, snsTopic: sns.ITopic, bucket: s3.IBucket): void {
  const destination = new s3n.SnsDestination(snsTopic)
  bucket.addEventNotification(
    s3.EventType.OBJECT_CREATED_PUT, 
    destination,
    {prefix: prefix}
  );
}
```

add the function to create the policy to sns publish

```sh
  /*
  createPolicySnSPublish(topicArn: string){
    const snsPublishPolicy = new iam.PolicyStatement({
      actions: [
        'sns:Publish',
      ],
      resources: [
        topicArn
      ]
    });
    return snsPublishPolicy;
  }
```

# Cloudfront Network Distribution

Next is to create a CDN. We will distribute our image assets using this service. 

- Navigate to cloud front
- Click "Create distribution"
- Click the "Origin domain" field.
- Select the bucket in this case `assets.rietta.online.s3.us-east-1.amazonaws.com`
- Click "Origin access" and select "Origin Access Control Settings"
- Click "Create control setting"
- Click "Create"
- Click "Redirect HTTP to HTTPS"
- Click "Select origin policy"
- Click "CORS-CustomOrigin"
- Click "Select response headers"
- Click "SimpleCORS"
- Click "Do not enable security protections" on WAF section
- Click "Click "Add item"" on Settings
- Type "assets.rietta.online"
- Click "Choose certificate" you can get an ssl certificate  through ACM
- Click "rietta.online  (5a4320fa-dc41-4d4d-8113-c8535a2d65b8)".
- Click the "Description - optional" and add a description. As a description you can use "Serve Assets for Cruddur"
- Click "Create distribution"
- You need to apply a bucket policy so the bucket will communicate exclusively with the CDN
- Go to Route 53 and create an A record for Cloudfront and add the cloudfront distribution created
- Copy the policy on your cloudfront and paste it on your s3 bucket permissions
- Make sure to configure the bucket policy.

Go to the tab "Origins", select the origin created before and click to edit. Go to "You must allow access to CloudFront using this policy statement. Learn more about giving CloudFront permission to access the S3 bucket ." click copy policy and click to "Go to S3 bucket permissions " so you will be redirected to the bucket in the section "Bucket policy". Edit and paste the policy


In this case I have created the ACM on us-east-1 region before creating the CDN

[Guide CDN step by step](https://scribehow.com/shared/Create_Cloudfront_distribution__7zZ6diCvTz2YN-h753zqBw)

# Implementation User Profile Page

# Errors 

Your account must be verified before you can add new CloudFront resources. To verify your account, please contact AWS Support (https://console.aws.amazon.com/support/home#/) and include this error message.