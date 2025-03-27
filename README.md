Objectives
By the end of this project, you will be able to do the following:
•	Securely access critical application data items.
•	Store credentials in Secrets Manager and programmatically retrieve encrypted secret values at runtime.
•	Use AWS KMS to create a customer-managed symmetric key and encrypt environment variables.
•	Use encryption context when encrypting and decrypting data with AWS KMS programmatically.
•	Deploy the serverless application by using AWS Serverless Application Model (AWS SAM).
________________________________________
Prerequisites
This lab requires the following:
•	Access to a computer with Wi-Fi and Microsoft Windows, macOS, or Linux (Ubuntu, SUSE, or Red Hat)
•	An internet browser such as Chrome, Firefox, or Microsoft Edge ( Note: Previous versions of Internet Explorer are not supported.)
•	A text editor
Scenario
Your company’s IT team is making steady progress in implementing security best practices into the TranscriptionApp. The security team audited the application configuration and code. They flagged the exchange of critical application data items such as passwords and parameters in plain text format in the application code.
Your company assigned you as a lead security engineer to use Secrets Manager to store and refer to passwords securely, and use AWS KMS to encrypt the environment variables.







Project environment
This is the architecture of the application that you have been working on. The highlighted portion is what you will implement in this lab. It uses Secrets Manager in combination with AWS KMS to secure the username and password in your AWS Lambda function.
 
Task 1: Connecting to the AWS Cloud9 environment
In this task, you connect to your AWS Cloud9 development environment.
1.	On the AWS Management Console, in the search bar, search for and choose 
Cloud9
2.	Choose Open under Cloud9 IDE .
 Note: Before you start next step, wait for the git client to clone the api and web repositories on the first launch, which you can observe in the AWS Cloud9 terminal window. All additional bootstrapping configurations have been completed when you see the Lab-Is-Ready folder in the list of folders.


Task 2: Reviewing the application’s current state
In this task, you review how the environment variables are set for the loadtest Lambda function in plain text. Then you use the AWS Cloud9 terminal to invoke the loadtest Lambda function before the security best practices are implemented and observe the output.
1.	On the AWS Management Console, in the search bar, search for and choose 
Lambda
6.	Choose the loadtest function link.
7.	Choose the Configuration tab.
8.	In the left column, choose Environment variables, and what you see should look similar to the following:
Key	Value
API_URL	https://xxxxxxxxxx.execute-api.us-west-2.amazonaws.com/Prod/count
CLIENT_ID	xxxxxxxxxxxxxxxxxxxxxxxxx
USERPOOL_ID	us-west-2_xXxXXXxXx
USER_NAME	student
USER_PASSWORD	lab_password
 Note: Notice that the hardcoded values for USER_NAME and USER_PASSWORD are set in plaintext. This is not a secure method to store sensitive data items and is not considered a security best practice. You will implement security best practices by storing passwords secretly and encrypting key parameter values in upcoming tasks. For now, test the function as it is and review the results.
9.	Return to the AWS Cloud9 environment browser tab to test the loadtest Lambda function.
10.	 Command: Run the following command to invoke the loadtest Lambda function:
aws lambda invoke --function-name loadtest \
--payload '{"NumberOfCallsPerUser": "3"}' \
--cli-binary-format raw-in-base64-out out \
--log-type Tail \
--query 'LogResult' \
--output text | base64 -d
 Note:
•	Notice that the NumberOfCallsPerUser parameter is set to 3, which causes the function to send three requests to the Amazon API Gateway endpoint.
•	This command will invoke the Lambda function by passing payload in JSON format to extract the tail of run logs from Amazon CloudWatch.
 Expected output: The output should display the UserName and Password used to sign in to the TranscriptionApp and the total number of notes available, similar to this:
START RequestId: 87d8d205-7428-4be6-9f22-520697703369 Version: $LATEST
UserName provided:student
Password provided:lab_password
API response:
b"# of notes available for 'student':5"
b"# of notes available for 'student':5"
b"# of notes available for 'student':5"
END RequestId: 87d8d205-7428-4be6-9f22-520697703369
REPORT RequestId: 87d8d205-7428-4be6-9f22-520697703369  Duration: 2092.16 ms    Billed Duration: 2093 ms        Memory Size: 128 MB     Max Memory Used: 67 MB  Init Duration: 330.38 ms
 Note: The output logs show that the function is using the username and password passed as environment parameters to access the application APIs and retrieve the notes data.
The following are code snippets from the loadtest Lambda function. Review the following code snippets to see how the values for USER_NAME and USER_PASSWORD are retrieved from the environment variables section.
 Example code snippet:
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
 Note: The functions retrieve the username and password from the Lambda function’s environment variables with this code. In the next task, you will store and retrieve the PASSWORD value securely by using AWS Secrets Manager.
 Congratulations! You have successfully reviewed how the username and password environment variables are set, retrieved the values, and ran the loadtest Lambda function before security best practices have been implemented.
________________________________________
Task 3: Using Secrets Manager to store passwords
A highly recommended security best practice is to never store your secrets or passwords in plain text or hard code them as part of the function code. They should always be encrypted to secure them from attacks.
In this task, you use Secrets Manager because it helps you protect secrets required to access your applications, services, and IT resources. Secrets Manager helps you meet your security and compliance requirements by permitting you to rotate secrets safely without the need for code deployments. You can store and retrieve secrets by using the Secrets Manager console, AWS SDK, AWS Command Line Interface (AWS CLI), or AWS CloudFormation. To retrieve secrets, you replace plaintext secrets in your applications with code to pull in those secrets programmatically by using the Secrets Manager APIs.
In this set of steps, you learn how to use Secrets Manager to secure secrets and how to retrieve the secrets in your Lambda code. Lastly, you redeploy your application by using AWS SAM.
Task 3.1: Creating a new secret in Secrets Manager
In this task, you create a new secret by using the AWS Cloud9 terminal to encrypt the plain text password that is currently used in the Lambda function’s environment variables.
11.	 Command: Run the following AWS CLI command in the AWS Cloud9 terminal to create a new secret with name student and set the value of Password to lab_password:
aws secretsmanager create-secret \
--name student \
--description "Encrypted password for student user" \
--secret-string "{\"Password\":\"lab_password\"}"
 Expected output: You should see the Amazon Resource Name (ARN) for the secret, similar to the following:
{
    "ARN": "arn:aws:secretsmanager:us-west-2:********3876:secret:student-xxxxxx",
    "Name": "student",
    "VersionId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
Next, you run the secretsmanager command to ensure that the value set for the SecretString property has a key of Password and a value of lab_password.
12.	 Command: Run the following command to retrieve the value of the SecretString property:
aws secretsmanager get-secret-value --secret-id student
 Expected output:

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
 Note: The SecretString value should show that the Password value is set to lab_password:
Task 3.2: Changing the Lambda function configuration
In this task, you use the Lambda console to remove the USER_PASSWORD environment variable that currently sets the password using plain text. Then you use AWS Cloud9 to update the code to retrieve the secret that you just created from Secrets Manager to set the password.
13.	On the AWS Management Console, in the search bar, search for and choose 
Lambda
.
14.	Choose Functions from the left navigation pane.
15.	Choose the loadtest function link.
16.	Choose the Configuration tab.
17.	Choose Environment variables.
18.	Choose Edit.
19.	Find the key for USER_PASSWORD and choose the corresponding Remove button to remove this environment variable.
20.	Choose Save.
Task 3.3: Changing the Lambda function code
21.	Return to the browser tab opened to AWS Cloud9.
22.	In the folder navigation pane on the left, expand /home/ec2-user/environment > api > loadtest-function, and right-click the app.py file and choose Open.
23.	Replace the current getPassword(userName) function with the following code snippet:
def getPassword(userName):
    try:
        secret = boto3.client('secretsmanager')
        secretString = json.loads(secret.get_secret_value(SecretId=userName).get('SecretString'))
        if 'Password' in secretString:
            password = secretString['Password']
            print("Password retrieved from Secrets Manager:" + password)
        return password
    except Exception as e:
        print("ExceptionSecret exception for {} {}".format(userName, e))
24.	Save the app.py file to lock in your changes. Do this by pressing Ctrl+S on your keyboard, or navigate up to the File > Save option at the top of the console.
 Note: The code snippet opens a client connection with Secrets Manager. By using the get_secret_value() method with the SecretId as student, you will retrieve the SecretString. Then you retrieve the parsed value of Password. If all goes well with your code, you should receive the password and print its value as lab_password in the logs.
 Congratulations! You have successfully used Secrets Manager to create a secret, verified the value of the secret, and updated the app.py file to retrieve the password value from the secret.
 Hint: If you need assistance, you can refer to the SOLUTION FOLDER for TASK2/app.py.
Task 3.4: Using AWS SAM to redeploy the application
In this task, you use AWS SAM to deploy changes made to TranscriptionApp, the serverless application, instead of manually deploying changes made to individual components.
25.	 Command: Run the following command to set up the $apiBucket variable to store the transcriptionappapi bucket name that can be used when you deploy to AWS SAM in just a few moments:
apiBucket=$(aws s3api list-buckets --output text --query 'Buckets[?contains(Name, `transcriptionappapi`) == `true`].Name')
 Expected output:
None, unless there is an error.
26.	 Command: You can verify the value set to $apiBucket with the following command:
echo $apiBucket
 Expected output: The bucket name should contain transcriptionappapi.

xxxxxxxx-xxxxxxxx-xxxxxxxxxxxxx-transcriptionappapi-xxxxxxxxxxxx
27.	 Command: Run the following command to change directories into home/ec2-user/environment/api:
cd /home/ec2-user/environment/api
 Expected output:
None, unless there is an error.
28.	 Command: Run the following command to build the code artifacts:
sam build --use-container
 Expected output: This output has been truncated.

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
 Note: If you see the following message, choose Don’t show again.
•	 WARNING: Git
o	The git repository at ‘/home/ec2-user/environment/api’ has too many active changes, only a subset of Git features will be enabled.
29.	 Command: Now that the code artifacts are created, run the following command to deploy the updates to the serverless application:
sam deploy --stack-name transcription-app-api \
--s3-bucket $apiBucket \
--parameter-overrides apiBucket=$apiBucket
 Expected output: This output has been truncated.

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
Task 3.5: Verifying the application functionality
In this task, you test the functionality of the loadtest Lambda function to make sure that the password is now retrieved from Secrets Manager and successfully accesses the application APIs.
30.	 Command: Run the following command to invoke the Lambda function.
aws lambda invoke --function-name loadtest \
--payload '{"NumberOfCallsPerUser": "1"}' \
--cli-binary-format raw-in-base64-out out \
--log-type Tail \
--query 'LogResult' \
--output text | base64 -d
 Note: Notice the NumberOfCallsPerUser parameter is set to 1, which causes the function to send only one request to the API Gateway endpoint this time.
 Expected output:
START RequestId: a585096c-f5e4-4bf2-ad87-93272fa14447 Version: $LATEST
UserName provided:student
Password retrieved from Secrets Manager:lab_password
API response:
b"# of notes available for 'student':5"
END RequestId: a585096c-f5e4-4bf2-ad87-93272fa14447
REPORT RequestId: a585096c-f5e4-4bf2-ad87-93272fa14447  Duration: 1790.03 ms    Billed Duration: 1791 ms        Memory Size: 128 MB     Max Memory Used: 66 MB  Init Duration: 400.31 ms
 Note: The value for UserName is returned from the environment variable. The value for Password was retrieved from Secrets Manager and only one response was returned for # of available notes.
 Congratulations! You have successfully used Secrets Manager to create a new secret to store the value for the password used in your application code.
________________________________________
Task 4: Using AWS KMS to encrypt environment variables
In this set of tasks, you learn how to use AWS KMS keys to encrypt and decrypt data. The sensitive data related to your application shouldn’t be used in the plaintext readable format. You create a KMS key and encrypt one of the environment variables: UserName. Then you update the Lambda function code to decrypt the data programmatically and use AWS SAM to deploy the application changes.
Task 4.1: Creating a new KMS key
In this task, you use AWS KMS to create a new KMS key.
31.	On the AWS Management Console, in the search bar, search for and choose 
Key Management Service
.
32.	In the left navigation pane, choose Customer managed keys.
33.	Choose Create a key.
34.	On the Configure key page under Key type, leave the default option as  Symmetric.
35.	For the Key usage section, choose  Encrypt and decrypt.
36.	Choose Next.
37.	In the Add labels page, configure the following details for the Alias section:
•	Alias: 
UserName
•	Description: 
Encryption key for the TranscriptionApp.
38.	Choose Next.
39.	From the Define key administrative permissions page under the Key administrators section, select the user or role you are logged in with.
 Note: The user or role is displayed at the top right of your screen and should start with AWSLabsUser…
 Example:
AWSLabsUser-rS3fdGXvKKU5i6E81Lyqnr
40.	Choose Next.
41.	From the Define key usage permissions page under the This account section, select the user or role you are logged in with.
42.	Choose Next.
43.	On the Review page, review all the options that you selected.
44.	Review the policy in the Key policy section. Permissions were added based on the selections made in the previous sections.
45.	When you are finished reviewing the selections, choose Finish.
You should see a similar message:
 Success
Your AWS KMS key was created with alias UserName and key xxxxxxxx-xxxx-xxxx-xxxxxxxxxxxx
Task 4.2: Encrypting the UserName variable
In this task, you encrypt the plaintext value student by using the KMS key that you just created. Then you update the function’s configuration to use the cipher text instead of the plaintext value for the UserName environment variable.
46.	Return to the AWS Cloud9 browser tab.
47.	 Command: In the AWS Cloud9 terminal, run the following command to create a variable that exports the KMS KeyId you just created:
export KeyId=$(aws kms list-aliases --query 'Aliases[?starts_with(AliasName, `alias/UserName`)].TargetKeyId' \
--output text)
 Expected Output:
None, unless there is an error.
48.	 Command: Run the following command to encrypt the plain text value of student using the KMS key:
aws kms encrypt --key-id ${KeyId} \
--plaintext "student" \
--cli-binary-format raw-in-base64-out \
--encryption-context "LambdaFunctionName=loadtest"
49.	 Expected output: Notice the CiphertextBlob value in quotation marks.

{
"CiphertextBlob": "AQICAHiw5sXhs7BV9+DNLArsz4kNWvpviuuAv7Ap3jE2OCYeEgEz67leOZ0Yd93N844MLGbbAAAAZTBjBgkqhkiG9w0BBwagVjBUAgEAME8GCSqGSIb3DQEHATAeBglghkgBZQMEAS4wEQQMD4TN5ddlPgkIcPKpAgEQgCKyqGtT7lyjdEyf917DSuv79NhOvHrPJbRYT6vGMvIdbyIY",
    "KeyId": "arn:aws:kms:us-west-2:********3876:key/2c8dd0cb-0ad2-4695-afa2-81b620e4bee6",
    "EncryptionAlgorithm": "SYMMETRIC_DEFAULT"
}
 Note: The resulting encoded output for the CiphertextBlob is base64 encoded and is provided as an updated value for the USER_NAME environment variable in your loadtest Lambda function. The Lambda function will be updated to decrypt the data to get the plaintext in order to use it.
50.	 Copy text: Copy the base64 encoded output (CiphertextBlob value) from the console excluding the quotation marks and store it in a text editor. You will need to refer to it in the next step.
Task 4.3: Updating the Lambda function configuration
In this task, you update the Lambda function configuration to use the cipher text value for the UserName environment variable.
51.	Return to the browser tab opened to the Lambda console.
52.	Choose Functions.
53.	Choose the loadtest function link.
54.	Choose the Configuration tab.
55.	Choose Environment variables.
56.	Choose Edit.
57.	Replace the current value for USER_NAME, which should be student, and paste in the base64 encoded string that you copied earlier.
Key	Value
API_URL	https://xxxxxxxxxx.execute-api.us-west-2.amazonaws.com/Prod/count
CLIENT_ID	xxxxxxxxxxxxxxxxxxxxxxxxx
USERPOOL_ID	us-west-2_xXxXXXxXx
USER_NAME	AQICAHi1nLd1TZ4N+d838POc7xooc83QwoRqlu1Hjw8SjQDFDgFlvXwNnlPK6aTvQ2ncCHwqAAAAZTBjBgkqhkiG9w0BBwagVjBUAgEAME8GCSqGSIb3DQEHATAeBglghkgBZQMEAS4wEQQMaz0LMLGy0vQ6VnuSAgEQgCIkjyAZ9RlSX3ne2LGKde+pYq7SUWhF96x6UUsaIKsv5Or5
58.	Choose Save.
Task 4.4: Updating the Lambda function code
In this task, you update the getUserName method that is currently reading the plaintext of UserName, to decrypt the ciphertext using the same KMS encryption key context.
The base64 encoded string that was created in a previous step by using the KMS key to encrypt the string is also known as EncryptionContext. It is nonsecret data to support additional authenticated encryption, and is only applicable for symmetric encryptions. If you encrypt some data using a key-value pair of EncryptionContext, you must provide the exact EncryptionContext to decrypt the data.
Lambda passes the function name as EncryptionContext making the encrypt call to AWS KMS. So, while decrypting the data in your Lambda function, you need to specify EncryptionContext as shown below in order to decrypt. Otherwise, you will receive an InvalidCiphertextException error even though all your permissions are correct.
59.	Return to the browser tab opened to AWS Cloud9.
60.	 Copy/Paste: Replace the existing code for the getUserName() method within the app.py file with the following code block:
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
 Note: The code snippet opens a client connection with AWS KMS by using the decrypt() method with a CiphertextBlob value of UserName variable and EncryptionContext as the function’s name. Then, you decode the parsed value of plaintext. If all goes well with your code, then you should decrypt the ciphertext value of UserName to plaintext and print its value as student in the logs.
61.	Save the app.py file to lock in your changes.
Task 4.5: Redeploying the application by using AWS SAM
Now that the loadtest code has been updated to use encrypted parameters, in this task you build the code artifacts and then use AWS SAM to redeploy the serverless application.
62.	 Command: Run the following command to change into the api directory, if not already in that directory:
cd ~/environment/api
63.	 Expected Output:
None, unless there is an error.
64.	 Command: Run the following command to build the code artifacts:
sam build --use-container
65.	 Expected output: Output has been truncated.

Starting Build inside a container
Building codeuri: /home/ec2-user/environment/api/list-function runtime: python3.9 metadata: {} architecture: x86_64 functions: ['listFunction']


Build Succeeded

Built Artifacts  : .aws-sam/build
Built Template   : .aws-sam/build/template.yaml

Commands you can use next
=========================
[*] Invoke Function: sam local invoke
[*] Deploy: sam deploy --guided
    
Running PythonPipBuilder:ResolveDependencies
Running PythonPipBuilder:CopySource
66.	 Command: Run the following command to redeploy the serverless application:
sam deploy --stack-name transcription-app-api \
--s3-bucket $apiBucket \
--parameter-overrides apiBucket=$apiBucket
67.	 Expected output: Output has been truncated.

        Deploying with following values
        ===============================
        Stack name                   : transcription-app-api
        Region                       : us-west-2
        Confirm changeset            : False
        Deployment s3 bucket         : labstack-jorkdm-tpq86ms8hzgsmkd9s-transcriptionappapi-49nyw4z7sp2b
        Capabilities                 : null
        Parameter overrides          : {}
        Signing Profiles             : {}


Outputs                                                                                                                                                                                                                      
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Key                 ApiURL                                                                                                                                                                                                   
Description         API Gateway endpoint URL for Prod stage                                                                                                                                                                  
Value               https://bin6e6tm2l.execute-api.us-west-2.amazonaws.com/Prod                                                                                                                                              

Key                 UserPoolId                                                                                                                                                                                               
Description         -                                                                                                                                                                                                        
Value               us-west-2_zLhLYyjAZ                                                                                                                                                                                      

Key                 AppClientId                                                                                                                                                                                              
Description         -                                                                                                                                                                                                        
Value               14pe976gi57imvlevol1odcj1f                                                                                                                                                                               

Key                 CognitoPoolArn                                                                                                                                                                                           
Description         -                                                                                                                                                                                                        
Value               arn:aws:cognito-idp:us-west-2:********3876:userpool/us-west-2_zLhLYyjAZ                                                                                                                                  
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

Successfully created/updated stack - transcription-app-api in us-west-2
Task 4.6: Verifying the application functionality
With the serverless application redeployed, in this task you test and verify the Lambda function by invoking it.
68.	 Command: Run the following command to invoke the loadtest Lambda function and verify that it works as expected:
aws lambda invoke --function-name loadtest \
--payload '{"NumberOfCallsPerUser": "1"}' \
--cli-binary-format raw-in-base64-out out \
--log-type Tail \
--query 'LogResult' \
--output text | base64 -d
 Expected output:

START RequestId: c482b028-fb64-4588-800e-e1ad21b31204 Version: $LATEST
UserName provided:AQICAHjbcPU7CnOaZrndpE9hJ1IO1DP14gPTRTpLLGTIcP7xOwHW8s+OPb5GdxFnyWXKg3gdAAAAZTBjBgkqhkiG9w0BBwagVjBUAgEAME8GCSqGSIb3DQEHATAeBglghkgBZQMEAS4wEQQM1jMXD9DuPmWF0kREAgEQgCKE+za6byLAWdv/RVTKqpGKxIOVIKToPeZh/Ys7S9aW1GOi
UserName decrypted using KMS key:student
Password retrieved from Secrets Manager:lab_password
API response:
b"# of notes available for 'student':5"
END RequestId: c482b028-fb64-4588-800e-e1ad21b31204
REPORT RequestId: c482b028-fb64-4588-800e-e1ad21b31204  Duration: 993.91 ms     Billed Duration: 994 ms Memory Size: 128 MB     Max Memory Used: 68 MB
 Note: Review how the UserName and Password values were returned.
•	The function used the KMS key to decrypt the UserName value of student, which was the base64 encoded string.
•	The Password value of lab_password was returned from Secrets Manager.
 Congratulations! You have successfully created a new KMS key to encrypt the UserName value. It produced a base64 encoded string that you used for the UserName environment variable instead of a value in plain text. Then you updated the Lambda function code to read the KMS key, prepared to redeploy the application by using AWS SAM, and then invoked the loadtest function to ensure that the parameters returned as expected.
________________________________________
Conclusion
 Congratulations You have successfully done the following:
•	Understood how to access critical application data items securely.
•	Understood how to store credentials in AWS Secrets Manager and programmatically retrieve encrypted secret values at runtime.
•	Understood how to use AWS Key Management Service to create a customer-managed symmetric key and encrypt environment variables.
•	Understood how to use encryption context when encrypting and decrypting data using AWS KMS programmatically.
•	Understood how to deploy the serverless application using the AWS Service Application Model (AWS SAM).

