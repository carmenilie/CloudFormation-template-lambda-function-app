# CloudFormation-template-Lambda-function-app
This repository contains the steps I took in creating a CloudFormation template for a Lambda function application. The function would put/get objects into/from an S3 bucket. 

**Step 1** - Create a bucket by entering into the AWS CLI the command below. Choose an appropriate name, as well as your preferred region. In my example, I named the bucket *carmenilie-lambda-bucket-function* and I chose the region *eu-west-1*.

```sh
aws s3 mb s3://carmenilie-lambda-function-bucket --region eu-west-1
```

If everything went well, an output similar to the following one should appear:

```sh
make_bucket: carmenilie-lambda-function-bucket
```

**Step 2** - Write an AWS CloudFormation template which includes, at a minimum, the Resources we desire to use. For my Lambda function, I will include a role and two functions: upload and download. Using a text editor, I created a template file in YAML format and then named it *stack.yml*.

Firstly, in order for the Lambda to function, I created a role, giving the function its necessary privileges. Afterwards, I granted that role to the Lambda function.

```yaml
LambdaRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: LambdaRole
      AssumeRolePolicyDocument:
        Statement:
          - Effect: Allow
            Principal:
              Service: lambda.amazonaws.com
            Action: sts:AssumeRole
```

Secondly, I added the code corresponding to the upload action of the function.

```python
  LambdaUploadFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: LambdaUploadFunction
      Role: !GetAtt LambdaRole.Arn
      Runtime: python3.7
      Handler: index.upload_file
      Code:
        ZipFile: |
            import logging
            import boto3
            from botocore.exceptions import ClientError
            
            def upload_file(file_name, bucket, object_name=None):
            
                # If S3 object_name was not specified, use file_name
                if object_name is None:
                    object_name = file_name
                    
                # Upload the file
                s3_client = boto3.client('s3')
                try:
                    response = s3_client.upload_file(file_name, bucket, object_name)
                except ClientError as e:
                    logging.error(e)
                    return False
                return True
```

Lastly, I added the code which performs the download action.

```python
  LambdaDownloadFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: LambdaDownloadFunction
      Role: !GetAtt LambdaRole.Arn
      Runtime: python3.7
      Handler: index.download_file
      Code:
        ZipFile: |
            import boto3
            
            def download_file(bucket, object_name=None, file_name):
            
                # If S3 object_name was not specified, use file_name
                if object_name is None:
                    object_name = file_name
                    
                # Download the file
                s3 = boto3.client('s3')
                try:
                    response = s3.download_file(bucket, object_name, file_name)
                except ClientError as e:
                    logging.error(e)
                    return False
                return True
```

**Step 3** - Create a stack by running the CloudFormation template which was just created. This can be done by typing into the CLI the next command:

```sh
aws cloudformation create-stack --stack-name lambda-stack --template-body file://stack.yml --capabilities CAPABILITY_NAMED_IAM
```

Check the status of the stack with the command:

```sh
aws cloudformation list-stacks --query 'StackSummaries[*].[StackName, StackStatus]' --stack-status-filter CREATE_IN_PROGRESS CREATE_COMPLETE --output table
```

Make sure the stack's creation is complete before moving to the next steps.

**Step 4** - Invoke the Lambda function. To test the function I created by manually invoking it, I attempted to simulate a couple of events.

For the upload action:

```sh
aws lambda invoke --invocation-type RequestResponse --function-name LambdaUploadFunction --log-type Tail outputfile.txt;  more outputfile.txt
```

Output obtained:

```sh
{
    "StatusCode": 200,
    "FunctionError": "Unhandled",
    "LogResult": "[...]",
    "ExecutedVersion": "$LATEST"
}

{"errorMessage": "Filename must be a string", "errorType": "ValueError", "stackTrace": ["  File \"/var/task/index.py\", line 14, in upload_file\n    response = s3_client.upload_file(file_name, bucket, object_name)\n", "  File \"/var/runtime/boto3/s3/inject.py\", line 131, in upload_file\n    extra_args=ExtraArgs, callback=Callback)\n", "  File \"/var/runtime/boto3/s3/transfer.py\", line 273, in upload_file\n    raise ValueError('Filename must be a string')\n"]}
```

For the download action:

```sh
aws lambda invoke --invocation-type RequestResponse --function-name LambdaDownloadFunction --log-type Tail outputfile.txt;  more outputfile.txt
```

Output obtained:

```sh
{
    "StatusCode": 200,
    "FunctionError": "Unhandled",
    "LogResult": "[...]",
    "ExecutedVersion": "$LATEST"
}

{"errorMessage": "Syntax error in module 'index': non-default argument follows default argument (index.py, line 3)", "errorType": "Runtime.UserCodeSyntaxError", "stackTrace": ["  File \"/var/task/index.py\" Line 3\n    def download_file(bucket, object_name=None, file_name):\n"]}
```

Through further reading, I managed to find out that a StatusCode equal to 200 means the request was successful, even though the function returned an error. Undoubtedly, the error must be caused by the fact that I never provided either of my functions with the necessary arguments. The next improvement I must bring to the project is to find the correct syntax I can use to pass the functions their corresponding arguments.

** Bibliography

https://leaherb.com/aws-lambda-tutorial-101/
https://boto3.amazonaws.com/v1/documentation/api/latest/guide/s3-uploading-files.html
https://boto3.amazonaws.com/v1/documentation/api/latest/guide/s3-example-download-file.html
