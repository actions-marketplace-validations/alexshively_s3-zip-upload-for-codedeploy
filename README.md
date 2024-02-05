# S3 Zip Upload Action

A GitHub Action for easily creating a zip file from a folder and uploading it to S3.  
This action can also be used to upload a pre-made zip file or any other single file.  
It supports Linux, Windows, and any other `runs-on` that can run Node. It contains
metadata fields which can be used to automatically deploy to codedeploy via a lambda.

## Features

- Creates zip files and uploads them to S3
- Can upload pre-made zip files or any other single files
- Supports Linux, Windows, and other `runs-on` that can run Node
- Easy to configure and use

## Inputs

All inputs are environment variables. See the example below for usage.

## Required Parameters

| Parameter             | Description                  | Example                  |
| --------------------- | ---------------------------- | ------------------------ |
| `AWS_SECRET_ID`       | AWS access key ID            | `AKIAIOSFODNN7EXAMPLE`   |
| `AWS_SECRET_KEY`      | AWS secret access key        | `wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY` |
| `BUCKET_NAME`         | AWS bucket name              | `my-bucket`              |
| `SOURCE_PATH`         | Source file or directory     | `/path/to/source`        |
| `DEST_FILE`           | Output file name or path     | `destination-file.zip`   |
| `APPLICATION_NAME`    | Output file name or path     | `test`                   |
| `DEPLOYMENTGROUP_NAME`| Output file name or path     | `dev`                    |

## Optional Parameters

| Parameter       | Description                  | Default                  | Example             |
| --------------- | ---------------------------- | ------------------------ | ------------------- |
| `STORAGE_CLASS` | AWS storage class            | `STANDARD`               | `ONEZONE_IA`        |
| `AWS_REGION`    | AWS region                   | `eu-central-1`           | `us-west-2`         |
| `SOURCE_MODE`   | Source mode                  | `ZIP`                    | `FILE`              |
| `S3_ENDPOINT`   | S3 endpoint override         | None                     | `https://s3.example.com` |

## Example usage

```yaml
name: Upload to S3
on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Upload ZIP to S3
        uses: alexshively/s3-zip-upload-for-codedeploy@v1.4
        env:
          AWS_SECRET_ID: ${{ secrets.AWS_SECRET_ID }}
          AWS_SECRET_KEY: ${{ secrets.AWS_SECRET_KEY }}
          BUCKET_NAME: ${{ secrets.BUCKET_NAME }}
          AWS_REGION: eu-central-1
          SOURCE_MODE: ZIP
          SOURCE_PATH: ./debug-override
          DEST_FILE: debug-action.zip
          APPLICATION_NAME: test
          DEPLOYMENTGROUP_NAME: dev
```

## Automatic CodeDeploy via Lambda example
https://aws.amazon.com/blogs/devops/automatically-deploy-from-amazon-s3-using-aws-codedeploy/

```js
import { createRequire } from 'module';
const require = createRequire(import.meta.url);
const { S3Client, HeadObjectCommand } = require('@aws-sdk/client-s3');
const { CodeDeployClient, CreateDeploymentCommand } = require('@aws-sdk/client-codedeploy');

// Initialize the S3 and CodeDeploy clients
const s3Client = new S3Client({ apiVersion: '2006-03-01' });
const codedeployClient = new CodeDeployClient();

export const handler = async(event, context) => {
    console.log('Received event:');
    console.log(JSON.stringify(event, null, '  '));

    // Get the object from the event
    const bucket = event.Records[0].s3.bucket.name;
    const key = event.Records[0].s3.object.key;
    let artifact_type = key.split('.').pop();
    artifact_type = artifact_type === 'gz' ? 'tgz' : ['zip', 'tar', 'tgz'].includes(artifact_type) ? artifact_type : 'tar';

    try {
        const data = await s3Client.send(new HeadObjectCommand({
            Bucket: bucket,
            Key: key
        }));
        
        await createDeployment(data);
    } catch (err) {
        console.error('Error', 'Error getting s3 object or creating deployment: ' + err);
        return;
    }

    async function createDeployment(s3ObjectData) {
         if (!s3ObjectData.Metadata['application-name'] || !s3ObjectData.Metadata['deploymentgroup-name']) {
            throw new Error('application-name and deploymentgroup-name object metadata must be set.');
         }

        const params = {
            applicationName: s3ObjectData.Metadata['application-name'],
            deploymentGroupName: s3ObjectData.Metadata['deploymentgroup-name'],
            description: 'Lambda invoked codedeploy deployment',
            ignoreApplicationStopFailures: false,
            revision: {
                revisionType: 'S3',
                s3Location: {
                    bucket: bucket,
                    bundleType: artifact_type,
                    key: key
                }
            }
        };

        const createDeploymentCommand = new CreateDeploymentCommand(params);
        const deploymentResponse = await codedeployClient.send(createDeploymentCommand);
        console.log(deploymentResponse);
        console.log('Finished executing lambda function');
    }
};
```

## Manual action tests

Copy .env.example to .env and modify it.
Commands to run locally:

```bash
npm run debug # run using your .env configuration
npm run debug file # uploads a dummy file as debug-override/dog1.jpg
npm run debug zip # uploads a dummy folder as debug-override/debug-action.zip
```

## Releasing new version

1. Update the version in `package.json`
2. `npm run build`
3. Create a new tag and release on GitHub
