---
title: "Using Localstack Emulate AWS S3 and SQS With Node"
date: 2022-07-04T11:25:04+05:30
draft: false
---

Some time back, I was teaching Node.js and we were covering AWS technologies like s3 and sqs. But the thing is most students didn’t have access to AWS and to try it out.

So, here’s how we can try out AWS services like: S3 and SQS.

# Installing Localstack on your system

Localstack repo: https://github.com/localstack/localstack

We’ll need python3, pip3 and docker to install localstack on our system.

- python3: [https://www.python.org/downloads/](https://www.python.org/downloads/)
- pip3: [https://pip.pypa.io/en/stable/installation/](https://pip.pypa.io/en/stable/installation/)
- docker: [https://docs.docker.com/engine/install/](https://docs.docker.com/engine/install/)

Once, we have all the above requirements, installed. We can install localstack by :

```
pip3 install localstack
```

Note: My localstack installation failed the first time, I had to upgrade pip. I ran this command :

```
python3 -m pip install --upgrade pip
```

To start localstack run the following command :
```
localstack start -d
```

Now, let’s install `awslocal`, which is a thin wrapper around the aws cli for use the localstack.

We can easily install it by :
```
pip3 install "awscli-local[ver1]"
```

# Using Localstack S3 and uploading images

Creating a new bucket named: hyperlearn-bucket
```
$> awslocal s3api create-bucket --bucket hyperlearn-bucket
{
    "Location": "/hyperlearn-bucket"
}
```

And to see a list of all the buckets in our localstack s3:
```
$> awslocal s3api list-buckets
{
    "Buckets": [
        {
            "Name": "hyperlearn-bucket",
            "CreationDate": "2022-06-29T10:58:03.000Z"
        }
    ],
    "Owner": {
        "DisplayName": "webfile",
        "ID": "bcaf1ffd86f41161ca5fb16fd081034f"
    }
}
```

Uploading an object to localstack s3:

```
$> awslocal s3api put-object --bucket hyperlearn-bucket --key screenshot --body  Screenshot\ 2022-06-29\ at\ 10.31.54\ AM.png
{
    "ETag": "\"22b742d501654e7e9f781280cc805d0e\""
}
```
This command will upload an screenshot file to hyperlearn-bucket


Getting all objects in a bucket:
```
$> awslocal s3api list-objects --bucket hyperlearn-bucket --query 'Contents[].{Key: Key, Size: Size}'
[
    {
        "Key": "screenshot",
        "Size": 673908
    }
]
```

# Uploading to Localstack s3 using Node.js

```
const AWS = require('aws-sdk');
const fs = require('fs');

AWS_ACCESS_KEY_ID='test';
AWS_REGION='ap-south-1'  // does not matter
AWS_SECRET_ACCESS_KEY='test';
AWS_BUCKET='hyperlearn-bucket'

const s3 = new AWS.S3({
  endpoint: 'http://localhost:4566',  // required for localstack
  accessKeyId: AWS_ACCESS_KEY_ID,
  secretAccessKey: AWS_SECRET_ACCESS_KEY,
  s3ForcePathStyle: true,  // required for localstack
})

const file = './demo-text';
const fileName = 'demo-text';

const uploadFile = () => {
  fs.readFile(file, (err, data) => {
    if (err) throw err;

    const params = {
      Bucket: AWS_BUCKET,
      Key: fileName,
      Body: data
    }

    s3.upload(params, function(s3err, data) {
      if (s3err) throw s3err;
      console.log('File uploaded', data);
    })
  })
}

uploadFile()
```


So, now when you run the above program it will upload the file `demo-text` to localstack s3.

Things to keep in mind:

- AWS_ACCESS_KEY_ID would be test.
- AWS_SECRET_ACCESS_KEY would be test.
- AWS_BUCKET would be the name of the bucket you created.
- AWS_REGION can be any region.
- endpoint will point to localstack: `http://localhost:4566` by default.
- s3ForcePathStyle will be true.

At last, on running we get:
```
File uploaded {
  ETag: '"a3930527807725b75c1ba92927d885c9"',
  Location: 'http://localhost:4566/hyperlearn-bucket/demo-text',
  key: 'demo-text',
  Key: 'demo-text',
  Bucket: 'hyperlearn-bucket'
}
```

You can even point the browser to the location (http://localhost:4566/hyperlearn-bucket/demo-text in this case) to download the file.

# Using Localstack SQS to publish and  consume messages

Creating a new queue in cli:
```
$> awslocal sqs create-queue --queue-name hyperlearn-queue
{
    "QueueUrl": "http://localhost:4566/000000000000/hyperlearn-queue"
}
```

Listing all the queue in localstack sqs:
```
$> awslocal sqs list-queues
{
    "QueueUrls": [
        "http://localhost:4566/000000000000/sample-queue"
    ]
}
```

Publishing messages to a queue:
```
$> awslocal sqs send-message --queue-url http://localhost:4566/000000000000/hyperlearn-queue --message-body test
{
    "MD5OfMessageBody": "098f6bcd4621d373cade4e832627b4f6",
    "MessageId": "b7d3f1f0-5e10-431e-9359-94a188eb95c8"
}
```

## Writing a sqs consumer in node

```
const { Consumer } = require('sqs-consumer');
const AWS = require('aws-sdk');

const awsAccessKeyId ='test';
const awsRegion='us-east-1'  // other regions don't seem to work
const awsSecretAccessKey='test';

const sqs = new AWS.SQS({
  accessKeyId:  awsAccessKeyId,
  secretAccessKey:  awsSecretAccessKey,
  region: awsRegion,
  apiVersion: '2012-11-05',
  endpoint: 'http://localhost:4566', // localstack
});

const queueUrl = "http://localhost:4566/000000000000/hyperlearn-queue";

const consumer = Consumer.create({
  queueUrl: queueUrl,
  handleMessage: async (message) => {
    console.log(message)
  },
  sqs: sqs
});

consumer.on('error', (err) => {
  console.error(err.message);
});

consumer.on('processing_error', (err) => {
  console.error(err.message);
});

consumer.start();
```

On running the above script and publishing messages to queue, the consumer will pick up messages and print them out.

Finally, publishing a message to queue:

```
$> awslocal sqs send-message --queue-url http://localhost:4566/000000000000/hyperlearn-queue --message-body Hello
{
    "MD5OfMessageBody": "8b1a9953c4611296a827abf8c47804d7",
    "MessageId": "9ce33e55-dd81-43fe-8c65-ceba05ee98ff"
}
```

Here’s a github link of the code used:  [https://github.com/hyperlearn/localstack-demo](https://github.com/hyperlearn/localstack-demo)

Hope, this was helpful for you. Also let me know, if I have missed something.Thanks!