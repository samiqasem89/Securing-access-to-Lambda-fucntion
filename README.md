# 🔐 Secure Serverless Application Lab - AWS Secrets Manager & KMS

This lab demonstrates how to securely access and manage critical application data by using **AWS Secrets Manager**, **AWS Key Management Service (KMS)**, and deploying with **AWS Serverless Application Model (SAM)**.

---

## 📌 Objectives

By the end of this lab, you will be able to:

- Securely access critical application data items.
- Store credentials in **AWS Secrets Manager** and retrieve them securely at runtime.
- Use **AWS KMS** to encrypt Lambda environment variables.
- Implement **encryption context** in KMS for additional security.
- Deploy a serverless application using **AWS SAM**.

---

## ⚙️ Prerequisites

- A computer with Wi-Fi (Windows/macOS/Linux)
- Chrome, Firefox, or Microsoft Edge
- A text editor (e.g. VS Code)

---

## 🧪 Scenario

You're tasked with improving the security of a serverless app (**TranscriptionApp**) by:

- Replacing hardcoded credentials in AWS Lambda with secure secrets
- Using **Secrets Manager** and **KMS** for encrypted credentials and secure environment variables

---

## 🧱 Lab Architecture

You will secure parts of the Lambda function using **Secrets Manager** and **AWS KMS**, including:

![image](https://github.com/user-attachments/assets/2d41381e-7b9b-475a-952f-3206a0124644)


- Password storage via Secrets Manager
- Username encryption via KMS

---

## 🧰 Task Overview

### ✅ Task 1: Connect to AWS Cloud9

AWS Console → Search: Cloud9 → Open Cloud9 IDE
  #### UPLAOD THE PROJECT FILE TO `Cloud9`

### ✅ Task 2: Review Plaintext Credentials
Go to Lambda → loadtest → Configuration → Environment Variables

### Observe:
```
USER_NAME = student
USER_PASSWORD = lab_password
```
Return to the AWS Cloud9 environment browser tab to test the `loadtest` Lambda function
#### Run the following command:

```
aws lambda invoke --function-name loadtest \
--payload '{"NumberOfCallsPerUser": "3"}' \
--cli-binary-format raw-in-base64-out out \
--log-type Tail \
--query 'LogResult' \
--output text | base64 -d
```
#### Note:
•	Notice that the `NumberOfCallsPerUser` parameter is set to `3`, which causes the function to send three requests to the Amazon API Gateway endpoint.
•	This command will invoke the Lambda function by passing payload in JSON format to extract the tail of run logs from Amazon CloudWatch.

#### Expected output: 
The output should display the `UserName` and `Password` used to sign in to the `TranscriptionApp` and the total number of notes available, similar to this:
Example:
```
START RequestId: 87d8d205-7428-4be6-9f22-520697703369 Version: $LATEST
UserName provided:student
Password provided:lab_password
API response:
b"# of notes available for 'student':5"
b"# of notes available for 'student':5"
b"# of notes available for 'student':5"
END RequestId: 87d8d205-7428-4be6-9f22-520697703369
REPORT RequestId: 87d8d205-7428-4be6-9f22-520697703369  Duration: 2092.16 ms    Billed Duration: 2093 ms        Memory Size: 128 MB     Max Memory Used: 67 MB  Init Duration: 330.38 ms
```
#### Note:
The output logs show that the function is using the `username` and `password` passed as environment parameters to access the application APIs and retrieve the notes data.

The following are code snippets from the `loadtest` Lambda function. Review the following code snippets to see how the values for USER_NAME and USER_PASSWORD are retrieved from the environment variables section.
#### Example code snippet:
```
**********************************
**** This is an EXAMPLE ONLY. ****
**********************************

def getUserName():
    try:
        userName =  os.environ['USER_NAME']
        print("UserName provided:" + userName)
        return userName
    except Exception as e:
        print("ExceptionSecret exception for {}".format(e))
        
def getPassword(userName):
    try:
        password =  os.environ['USER_PASSWORD']
        print("Password provided:" + password)

        return password
    except Exception as e:
        print("ExceptionSecret exception for {} {}".format(userName, e))
aws secretsmanager create-secret \
--name student \
--description "Encrypted password for student user" \
--secret-string "{\"Password\":\"lab_password\"}"
```
#### Note: 
The functions retrieve the username and password from the Lambda function’s environment variables with this code. In the next task, you will store and retrieve the PASSWORD value securely by using AWS Secrets Manager.

### ✅ Task 3: Use Secrets Manager for Password
A highly recommended security best practice is to never store your secrets or passwords in plain text or hard code them as part of the function code. They should always be encrypted to secure them from attacks.
#### 3.1 Create a secret:
Run the following AWS CLI command in the AWS Cloud9 terminal to create a new secret with name student and set the value of Password to lab_password:
```
aws secretsmanager create-secret \
--name student \
--description "Encrypted password for student user" \
--secret-string "{\"Password\":\"lab_password\"}"
```
#### Expected output: 
You should see the Amazon Resource Name (ARN) for the secret, similar to the following:
```
{
    "ARN": "arn:aws:secretsmanager:us-west-2:********3876:secret:student-xxxxxx",
    "Name": "student",
    "VersionId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```
#### 3.2 Retrieve the secret:
```
aws secretsmanager get-secret-value --secret-id student
```
#### Expected output:
```
{
    "ARN": "arn:aws:secretsmanager:us-west-2:********3876:secret:student-xxxxxx",
    "Name": "student",
    "VersionId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
    "SecretString": "{\"Password\":\"lab_password\"}",
    "VersionStages": [
        "AWSCURRENT"
    ],
    "CreatedDate": "2022-05-11T20:03:12.081000+00:00"
}
```
#### 3.3 Update Lambda
Remove `USER_PASSWORD` from environment variables in the Lambda console.
#### 3.4 Update Lambda code:
Return to the browser tab opened to AWS Cloud9.
In the folder navigation pane on the left, expand /home/ec2-user/environment →  api →  loadtest-function →  app.py
```
def getPassword(userName):
    try:
        secret = boto3.client('secretsmanager')
        secretString = json.loads(secret.get_secret_value(SecretId=userName).get('SecretString'))
        password = secretString['Password']
        print("Password retrieved from Secrets Manager:" + password)
        return password
    except Exception as e:
        print(f"Exception while retrieving password for {userName}: {e}")
```
Save the `app.py` file to lock in your changes. Do this by pressing Ctrl+S on your keyboard, or navigate up to the File → Save option at the top of the console.

### Congratulations! 
#### You have successfully used Secrets Manager to create a secret, verified the value of the secret, and updated the app.py file to retrieve the password value from the secret.

### ✅ Task 4: Using AWS SAM to redeploy the application
In this task, you use AWS SAM to deploy changes made to TranscriptionApp, the serverless application, instead of manually deploying changes made to individual components.
#### Command: Run the following command to set up the $apiBucket variable to store the transcriptionappapi bucket name that can be used when you deploy to AWS SAM in just a few moments:
```
echo $apiBucket
```
### Expected output: 
The bucket name should contain transcriptionappapi.
```
xxxxxxxx-xxxxxxxx-xxxxxxxxxxxxx-transcriptionappapi-xxxxxxxxxxxx
```
#### Command: Run the following command to change directories into home/ec2-user/environment/api:
```
cd /home/ec2-user/environment/api
```
#### Expected output:
None, unless there is an error.
#### Command: Run the following command to build the code artifacts:
```
sam build --use-container
```
#### Expected output: 
This output has been truncated.
```
Starting Build inside a container
Building codeuri: /home/ec2-user/environment/api/list-function runtime: python3.9 metadata: {} architecture: x86_64 functions: ['listFunction']

Fetching public.ecr.aws/sam/build-python3.9:latest-x86_64 Docker container image....................................................

Build Succeeded

Built Artifacts  : .aws-sam/build
Built Template   : .aws-sam/build/template.yaml

Commands you can use next
=========================
[*] Invoke Function: sam local invoke
[*] Deploy: sam deploy --guided
    
Running PythonPipBuilder:ResolveDependencies
Running PythonPipBuilder:CopySource
```
#### Command: Now that the code artifacts are created, run the following command to deploy the updates to the serverless application:

```
sam deploy --stack-name transcription-app-api \
--s3-bucket $apiBucket \
--parameter-overrides apiBucket=$apiBucket
```

#### Expected output: 
This output has been truncated.
```
Deploying with following values
        ===============================
        Stack name                   : transcription-app-api
        Region                       : us-west-2
        Confirm changeset            : False
        Deployment s3 bucket         : labstack-jorkdm-6osvvoxyma7hr6nmb-transcriptionappapi-dtbshmj20v14
        Capabilities                 : null
        Parameter overrides          : {"apiBucket": "labstack-jorkdm-6osvvoxyma7hr6nmb-transcriptionappapi-dtbshmj20v14"}
        Signing Profiles             : {}



Successfully created/updated stack - transcription-app-api in us-west-2
```

### Verifying the application functionality
In this task, you test the functionality of the loadtest Lambda function to make sure that the password is now retrieved from Secrets Manager and successfully accesses the application APIs.

#### Command: Run the following command to invoke the Lambda function.
```
aws lambda invoke --function-name loadtest \
--payload '{"NumberOfCallsPerUser": "1"}' \
--cli-binary-format raw-in-base64-out out \
--log-type Tail \
--query 'LogResult' \
--output text | base64 -d
```
#### Note: 
Notice the NumberOfCallsPerUser parameter is set to 1, which causes the function to send only one request to the API Gateway endpoint this time.

### Expected output:
```
START RequestId: a585096c-f5e4-4bf2-ad87-93272fa14447 Version: $LATEST
UserName provided:student
Password retrieved from Secrets Manager:lab_password
API response:
b"# of notes available for 'student':5"
END RequestId: a585096c-f5e4-4bf2-ad87-93272fa14447
REPORT RequestId: a585096c-f5e4-4bf2-ad87-93272fa14447  Duration: 1790.03 ms    Billed Duration: 1791 ms        Memory Size: 128 MB     Max Memory Used: 66 MB  Init Duration: 400.31 ms
```
#### Note: The value for UserName is returned from the environment variable. The value for Password was retrieved from Secrets Manager and only one response was returned for # of available notes.
### Congratulations! You have successfully used Secrets Manager to create a new secret to store the value for the password used in your application code.

### ✅ Task 5: Encrypt Username with KMS
#### Create KMS key: 
Use the AWS Console → KMS → `Create Key`

1. On the Configure key page under Key type, leave the default option as  Symmetric
2. For the Key usage section, choose  Encrypt and decrypt.
3. Choose `Next`
4. In the Add labels page, configure the following details for the Alias section:
•	Alias: 
UserName
•	Description: 
Encryption key for the TranscriptionApp.
5. Choose `Next`
6. From the Define key administrative permissions page under the Key administrators section, select the user or role you are logged in with.
7. Choose `Next`
8. When you are finished reviewing the selections, choose `Finish`

#### You should see a similar message:
Success
Your AWS KMS key was created with alias UserName and key xxxxxxxx-xxxx-xxxx-xxxxxxxxxxxx

#### Encrypt username using KMS:
```
export KeyId=$(aws kms list-aliases --query 'Aliases[?AliasName==`alias/UserName`].TargetKeyId' --output text)

aws kms encrypt --key-id ${KeyId} \
--plaintext "student" \
--cli-binary-format raw-in-base64-out \
--encryption-context "LambdaFunctionName=loadtest"
```
### Encrypt username using KMS:
Return to the AWS Cloud9 browser tab.
#### Command: In the AWS Cloud9 terminal, run the following command to create a variable that exports the KMS KeyId you just created:
```
export KeyId=$(aws kms list-aliases --query 'Aliases[?starts_with(AliasName, `alias/UserName`)].TargetKeyId' \
--output text)
```
#### Expected Output:
None, unless there is an error.

#### Command: Run the following command to encrypt the plain text value of student using the KMS key
```
aws kms encrypt --key-id ${KeyId} \
--plaintext "student" \
--cli-binary-format raw-in-base64-out \
--encryption-context "LambdaFunctionName=loadtest"
```
#### Expected output: 
Notice the `CiphertextBlob` value in quotation marks.
```
{
"CiphertextBlob": "AQICAHiw5sXhs7BV9+DNLArsz4kNWvpviuuAv7Ap3jE2OCYeEgEz67leOZ0Yd93N844MLGbbAAAAZTBjBgkqhkiG9w0BBwagVjBUAgEAME8GCSqGSIb3DQEHATAeBglghkgBZQMEAS4wEQQMD4TN5ddlPgkIcPKpAgEQgCKyqGtT7lyjdEyf917DSuv79NhOvHrPJbRYT6vGMvIdbyIY",
    "KeyId": "arn:aws:kms:us-west-2:********3876:key/2c8dd0cb-0ad2-4695-afa2-81b620e4bee6",
    "EncryptionAlgorithm": "SYMMETRIC_DEFAULT"
}
```
#### Note: 
The resulting encoded output for the `CiphertextBlob` is `base64 encoded` and is provided as an updated value for the `USER_NAME` environment variable in your `loadtest` Lambda function. The Lambda function will be updated to decrypt the data to get the plaintext in order to use it.
   #### Copy text: 
   Copy the `base64 encoded` output (CiphertextBlob value) from the console excluding the quotation marks and store it in a text editor. You will need to refer to it in the next step.

### Updating the Lambda function configuration
In this task, you update the Lambda function configuration to use the cipher text value for the UserName environment variable.
Functions → loadtest → Configuration → Environment variables → choose `Edit`
#### Replace the current value for `USER_NAME`, which should be `student`, and paste in the `base64 encoded string` that you copied earlier.
Choose `Save`

### ✅ Task 6: Update Lambda code to decrypt

In this task, you update the getUserName method that is currently reading the plaintext of UserName, to decrypt the ciphertext using the same KMS encryption key context.
The base64 encoded string that was created in a previous step by using the KMS key to encrypt the string is also known as EncryptionContext. It is nonsecret data to support additional authenticated encryption, and is only applicable for symmetric encryptions. If you encrypt some data using a key-value pair of EncryptionContext, you must provide the exact EncryptionContext to decrypt the data.
Lambda passes the function name as EncryptionContext making the encrypt call to AWS KMS. So, while decrypting the data in your Lambda function, you need to specify EncryptionContext as shown below in order to decrypt. Otherwise, you will receive an InvalidCiphertextException error even though all your permissions are correct.

#### Return to the browser tab opened to AWS Cloud9
#### Copy/Paste: 
Replace the existing code for the `getUserName()` method within the app.py file with the following code block:
```
def getUserName():
    userName = os.environ['USER_NAME']
    try:
        print("UserName provided:" + userName)
        decrypted_User = boto3.client('kms').decrypt(
            CiphertextBlob=b64decode(userName),
            EncryptionContext={'LambdaFunctionName':os.environ['AWS_LAMBDA_FUNCTION_NAME']})['Plaintext'].decode('utf-8')
        print("UserName decrypted using KMS key:" + str(decrypted_User))
        return decrypted_User
    except Exception as e:
        print("ExceptionSecret exception for {} {}".format(userName, e))
```
#### Note: 
The code snippet opens a client connection with AWS KMS by using the `decrypt()` method with a CiphertextBlob value of `UserName` variable and `EncryptionContext` as the function’s name. Then, you decode the parsed value of plaintext. If all goes well with your code, then you should decrypt the ciphertext value of UserName to plaintext and print its value as `student` in the logs.
Save the `app.py` file to lock in your changes

### ✅ Task 7: Redeploy with AWS SAM
Now that the loadtest code has been updated to use encrypted parameters, in this task you build the code artifacts and then use AWS SAM to redeploy the serverless application.
#### Command: Run the following command to change into the api directory, if not already in that directory:
```
# Navigate to API directory
cd ~/environment/api

# Build with SAM
sam build --use-container

# Deploy
sam deploy --stack-name transcription-app-api \
--s3-bucket $apiBucket \
--parameter-overrides apiBucket=$apiBucket
```

### ✅ Task 8: Verifying the application functionality

#### Command: Run the following command to invoke the loadtest Lambda function and verify that it works as expected:
```
aws lambda invoke --function-name loadtest \
--payload '{"NumberOfCallsPerUser": "1"}' \
--cli-binary-format raw-in-base64-out out \
--log-type Tail \
--query 'LogResult' \
--output text | base64 -d
```
#### Expected output:
```
START RequestId: c482b028-fb64-4588-800e-e1ad21b31204 Version: $LATEST
UserName provided:AQICAHjbcPU7CnOaZrndpE9hJ1IO1DP14gPTRTpLLGTIcP7xOwHW8s+OPb5GdxFnyWXKg3gdAAAAZTBjBgkqhkiG9w0BBwagVjBUAgEAME8GCSqGSIb3DQEHATAeBglghkgBZQMEAS4wEQQM1jMXD9DuPmWF0kREAgEQgCKE+za6byLAWdv/RVTKqpGKxIOVIKToPeZh/Ys7S9aW1GOi
UserName decrypted using KMS key:student
Password retrieved from Secrets Manager:lab_password
API response:
b"# of notes available for 'student':5"
END RequestId: c482b028-fb64-4588-800e-e1ad21b31204
REPORT RequestId: c482b028-fb64-4588-800e-e1ad21b31204  Duration: 993.91 ms     Billed Duration: 994 ms Memory Size: 128 MB     Max Memory Used: 68 MB
```
#### Note: Review how the UserName and Password values were returned.
      •	The function used the KMS key to decrypt the UserName value of student, which was the base64 encoded string.
      •	The Password value of lab_password was returned from Secrets Manager.
 ### Congratulations! 
 You have successfully created a new KMS key to encrypt the `UserName` value. It produced a `base64 encoded string` that you used for the `UserName` environment variable instead of a value in plain text. Then you updated the Lambda function code to read the KMS key, prepared to redeploy the application by using AWS SAM, and then invoked the `loadtest` function to ensure that the parameters returned as expected.

### ✅ Summary
- ✔ Stored passwords securely with Secrets Manager
- ✔ Encrypted environment variables with KMS
- ✔ Decrypted secrets in Lambda at runtime
- ✔ Used SAM for deployment

