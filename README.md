# 🔐 Secure Lambda function with AWS Secrets Manager & KMS

Securely access encrypted credentials and environment variables from within a Lambda function using AWS KMS and Secrets Manager.**.

---

## 🧰 STEP-BY-STEP GUIDE

### ✅ Task 1: Create a Customer-Managed KMS Key

Go to AWS KMS Console → Click Create key.
Choose Symmetric, Key Usage: Encrypt and Decrypt.
Give it an alias (e.g., alias/myLambdaKey).
Assign yourself and your Lambda role as key users.
Complete the wizard.

### ✅ Task 2: Encrypt Your Secret Value (API Key)

#### 1. In AWS CloudShell or local terminal, create a file:
```
echo -n "super-secret" > value.txt
```
#### 2. Encrypt it using the KMS key and an encryption context:

```
aws kms encrypt \
  --key-id alias/myLambdaKey \
  --plaintext fileb://value.txt \
  --encryption-context LambdaFunctionName=MySecureLambda \
  --output text \
  --query CiphertextBlob
```
#### Copy the output (long base64 string like AQICAHh7...==)

### ✅ Task 3: Store That Encrypted Value in Lambda
1. Go to Lambda Console → Select/Create your function.
2. Go to Configuration > Environment variables.
3. Add:
   - Key: ENC_API_KEY
   - Value: AQICAHh7...== (paste your encrypted blob)
#### This will be decrypted at runtime.

### ✅ Task 4: Store Additional Credentials in AWS Secrets Manager
1. Go to AWS Secrets Manager → Click Store a new secret.
2. Choose Other type of secret → Add key-value pairs:
```
{
  "username": "myUser",
  "password": "myPass123!"
}
```
3. Name the secret: `MyServiceCredentials`
4. Optionally use KMS encryption (you can select `myLambdaKey`).
5. Save.
   
### ✅ Task 5: Create the Lambda Function Code
Paste the following into your Lambda function editor:
```
import boto3
import os
import base64
import json

def decrypt_env_var(enc_value):
    kms = boto3.client('kms')
    decoded = base64.b64decode(enc_value)
    response = kms.decrypt(
        CiphertextBlob=decoded,
        EncryptionContext={
            "LambdaFunctionName": os.environ['AWS_LAMBDA_FUNCTION_NAME']
        }
    )
    return response['Plaintext'].decode('utf-8')

def get_secret(secret_name):
    client = boto3.client('secretsmanager')
    response = client.get_secret_value(SecretId=secret_name)
    return json.loads(response['SecretString'])

def lambda_handler(event, context):
    secret_data = get_secret("MyServiceCredentials")
    username = secret_data['username']
    password = secret_data['password']

    decrypted_api_key = decrypt_env_var(os.environ['ENC_API_KEY'])

    # Example use of secrets
    print(f"Connecting as {username} using decrypted key.")

    return {
        'message': 'Secrets retrieved and decrypted successfully.'
    }
```

### ✅ Task 6: Set IAM Permissions
Update the Lambda's execution role to include access to KMS and Secrets Manager:
```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "kms:Decrypt",
        "secretsmanager:GetSecretValue"
      ],
      "Resource": "*"
    }
  ]
}
```
👉 Use more specific ARNs in production for least-privilege access.

### ✅ Task 7: Test the Lambda
1. Click Test in the Lambda console.
2. View output in the logs:
    - Should say: Secrets retrieved and decrypted successfully.
    - Or see decrypted values in print() logs (CloudWatch).

✅ You now have a secure Lambda pattern using:
KMS for environment encryption
Secrets Manager for credential storage
Encryption context for scoped, secure access

