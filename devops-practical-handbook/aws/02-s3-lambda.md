# AWS — S3, Lambda, and DynamoDB Labs

Practical steps and code from Virtualization, Cloud, and AWS slides 20–49.

## 1. S3 static website hosting

In the AWS Console:

1. Open **S3 → Create bucket**. Enter a globally unique bucket name.
2. For this lab only, uncheck **Block all public access**, acknowledge the warning, and create the bucket.
3. Upload `index.html` and `error.html` from the source repository: https://github.com/newdelthis/demo
4. Open **Permissions → Bucket Policy**.
5. Paste the `bucket_policy.json` from the repository and replace its bucket name with your bucket name:
   - https://github.com/newdelthis/demo/blob/main/bucket_policy.json
6. Open **Properties → Static website hosting → Edit → Enable**.
7. Set the index document to `index.html` and error document to `error.html`.
8. Save and open the Website endpoint shown in the S3 properties page.

> This exercise deliberately makes the website public. Do not place private or sensitive files in a public website bucket. The uploaded slides link to the bucket policy file but do not include its JSON body.

Cleanup:

1. Empty the S3 bucket.
2. Delete the bucket.

## 2. Lambda — return a custom PRN message

In the AWS Console:

1. Open **Lambda → Create function → Author from scratch**.
2. Function name: `Lambda_<YOUR_PRN>`.
3. Runtime: Python 3.12 (or another available Python runtime).
4. Architecture: `x86_64`.
5. Create a new execution role with basic Lambda permissions.
6. Create the function.

Replace `lambda_function.py` with:

```python
def lambda_handler(event, context):
    message = "Hello from <YOUR_PRN>"
    print(message)
    return {
        "statusCode": 200,
        "body": message
    }
```

Click **Deploy**.

Create a test event:

- Event name: `MyTestEvent`
- Event JSON:

```json
{}
```

Save the event and click **Test**. The response should contain the custom message.

To inspect logs, open **Monitor → CloudWatch Logs** and open the latest log stream. The explicit `print()` writes the message to CloudWatch; Lambda does not automatically print the returned body.

## 3. Trigger the Lambda function when a file is uploaded to S3

1. Create a bucket with a globally unique name, for example `lambda-prn-<YOUR_PRN>-<UNIQUE_SUFFIX>`.
2. Keep the S3 bucket in the same Region as the Lambda function.
3. Open the Lambda function and choose **Add trigger → S3**.
4. Select the bucket.
5. Select **All object create events**.
6. Leave Prefix and Suffix blank for the simple hello-event exercise.
7. Acknowledge the permission warning and add the trigger.
8. Upload `<YOUR_PRN>.txt` to the bucket.
9. Open the Lambda function's **Monitor → CloudWatch Logs** and inspect the invocation.

Replace the Lambda code with this event-inspection version and deploy:

```python
def lambda_handler(event, context):
    print("Lambda function invoked!")

    for record in event["Records"]:
        bucket = record["s3"]["bucket"]["name"]
        filename = record["s3"]["object"]["key"]
        print("Bucket:", bucket)
        print("Uploaded file:", filename)

    return {
        "statusCode": 200,
        "body": "Hello from <YOUR_PRN>"
    }
```

Cleanup:

1. Empty and delete the S3 bucket.
2. Delete the Lambda function.

## 4. Lambda — calculate total sales from an uploaded CSV

Create a new S3 bucket with a globally unique name, for example `<YOUR_NAME>-sales-lambda-demo-<UNIQUE_SUFFIX>`.

Create a Lambda function:

- Function name: `sales_total_function`
- Runtime: Python 3.x
- Author from scratch.

Give its execution role access to read and write the lab bucket. The slides use `AmazonS3FullAccess`; that is broad account-wide access. Prefer a bucket-scoped policy for a real deployment.

Create `sales.csv`:

```csv
product,quantity,price
Laptop,2,60000
Mouse,5,500
Keyboard,3,1500
```

Replace the Lambda code with:

```python
import boto3
import csv
import io

s3 = boto3.client("s3")


def lambda_handler(event, context):
    bucket = event["Records"][0]["s3"]["bucket"]["name"]
    key = event["Records"][0]["s3"]["object"]["key"]

    # Avoid invoking the function recursively on its own output file.
    if key == "total_sales.csv":
        return

    response = s3.get_object(Bucket=bucket, Key=key)
    lines = response["Body"].read().decode("utf-8").splitlines()
    reader = csv.DictReader(lines)

    total_sales = 0
    for row in reader:
        total_sales += int(row["quantity"]) * int(row["price"])

    print("Total Sales:", total_sales)

    output = io.StringIO()
    writer = csv.writer(output)
    writer.writerow(["Total_Sales"])
    writer.writerow([total_sales])

    s3.put_object(
        Bucket=bucket,
        Key="total_sales.csv",
        Body=output.getvalue()
    )

    return {
        "statusCode": 200,
        "total_sales": total_sales
    }
```

Click **Deploy**.

Configure the S3 trigger:

1. Open the bucket's **Properties → Event notifications**.
2. Create event notification `sales-upload-trigger`.
3. Select **All object create events**.
4. Choose the Lambda function as destination and save.
5. Upload `sales.csv` to the bucket.
6. Verify `total_sales.csv` appears in the bucket.
7. If it does not, inspect **Lambda → Monitor → CloudWatch Logs**.

The sample rows should produce a total sales value of `127000`.

Cleanup:

1. Delete the Lambda function.
2. Empty and delete the S3 bucket.

## 5. Exercise — per-student total and average marks

Create `marks.csv`:

```csv
Alice,Python,75
Alice,Linux,60
Alice,Cloud,68
Bob,Python,55
Bob,Linux,70
Bob,Cloud,82
Charlie,Python,49
Charlie,Linux,69
Charlie,Cloud,71
```

Upload `marks.csv` to an S3 bucket and create a Lambda function that calculates total and average marks per student and writes `result.csv` back to the bucket.

The slide deck gives the exercise requirement but does **not** include the Lambda function code. No code is invented here.

## 6. Lambda — copy an uploaded object from a source bucket to a target bucket

Create two S3 buckets with globally unique names:

- `<YOUR_PRN>-source-<UNIQUE_SUFFIX>`
- `<YOUR_PRN>-destination-<UNIQUE_SUFFIX>`

Create a Lambda function:

- Function name: `my-lambda-s3-function`
- Runtime: Python 3.12
- Execution role: new role with basic Lambda permissions.

Replace the function code with this corrected version of the slide snippet:

```python
import boto3
import urllib.parse

s3 = boto3.client("s3")
SOURCE_BUCKET_NAME = "<YOUR_SOURCE_BUCKET_NAME>"
TARGET_BUCKET_NAME = "<YOUR_TARGET_BUCKET_NAME>"


def lambda_handler(event, context):
    for record in event["Records"]:
        source_bucket_name = record["s3"]["bucket"]["name"]
        key = urllib.parse.unquote_plus(
            record["s3"]["object"]["key"],
            encoding="utf-8"
        )

        if source_bucket_name != SOURCE_BUCKET_NAME:
            raise ValueError(f"Unexpected source bucket: {source_bucket_name}")

        copy_source = {"Bucket": source_bucket_name, "Key": key}
        s3.copy(copy_source, TARGET_BUCKET_NAME, key)
        print(f"Copied {key} from {source_bucket_name} to {TARGET_BUCKET_NAME}")

    return {"statusCode": 200}
```

Click **Deploy**. Replace both bucket placeholders with the actual names.

Add permissions to the Lambda execution role. The source slide points to `lambda_s3_copy_policy.json`; its policy body is also shown in slide 46. Use this bucket-scoped policy after replacing both bucket names:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": "arn:aws:s3:::<YOUR_SOURCE_BUCKET_NAME>/*"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:PutObject"],
      "Resource": "arn:aws:s3:::<YOUR_TARGET_BUCKET_NAME>/*"
    }
  ]
}
```

Also attach the basic Lambda logging permissions to the execution role (the slide names `AWSLambdaBasicExecutionRole`).

Configure the S3 trigger on the **source bucket**:

1. Open **Properties → Event notifications → Create event notification**.
2. Name it `lambda-trigger`.
3. Select all object-create events.
4. Choose the Lambda function as the destination.
5. Save.
6. Upload two or three small files to the source bucket.
7. Verify that they appear in the destination bucket and inspect CloudWatch logs.

Cleanup:

1. Empty both buckets.
2. Delete both buckets.
3. Delete the Lambda function.
4. Delete the lab-created execution role/policy if it is no longer needed.

## 7. CSV to JSON conversion in S3

The exercise target layout is:

```text
input/
└── sales.csv
output/
└── sales.json
```

The slide's function writes an output object to the same bucket that triggers the Lambda function. Configure the S3 event notification to filter on prefix `input/` and suffix `.csv`; otherwise, the output upload can invoke the function again. The slide's original output-key expression also produces an extra `input/` path segment for `input/sales.csv`. The corrected implementation below writes exactly `output/sales.json`.

```python
import boto3
import csv
import io
import json
import os

s3 = boto3.client("s3")


def lambda_handler(event, context):
    record = event["Records"][0]
    bucket_name = record["s3"]["bucket"]["name"]
    file_key = record["s3"]["object"]["key"]

    # Process only incoming CSV files in input/; ignore output objects.
    if not file_key.startswith("input/") or not file_key.lower().endswith(".csv"):
        print("Skipping non-input CSV object:", file_key)
        return {"statusCode": 200, "body": "Skipped"}

    response = s3.get_object(Bucket=bucket_name, Key=file_key)
    content = response["Body"].read().decode("utf-8").splitlines()
    rows = list(csv.DictReader(content))

    base_name = os.path.basename(file_key)
    output_key = "output/" + os.path.splitext(base_name)[0] + ".json"

    s3.put_object(
        Bucket=bucket_name,
        Key=output_key,
        Body=json.dumps(rows, indent=2).encode("utf-8"),
        ContentType="application/json"
    )

    return {
        "statusCode": 200,
        "body": f"Converted {file_key} to {output_key}"
    }
```

Configure the S3 notification for `ObjectCreate` events with prefix `input/` and suffix `.csv`, deploy the function, upload `input/sales.csv`, and verify `output/sales.json`.

The Lambda execution role needs `s3:GetObject` on the input objects and `s3:PutObject` on the output objects in the bucket.

## 8. DynamoDB exercise reference

The slides describe creating a DynamoDB table with:

- Table name: `StudentResults`
- Partition key: `student`
- Key type: String

They direct you to replace the Lambda code with `lambda_student_marks_total_dynamodb.py` from:

- https://github.com/newdelthis/demo/blob/main/lambda_student_marks_total_dynamodb.py

They also instruct attaching `AmazonDynamoDBFullAccess`, re-uploading `marks.csv`, checking **Explore table items**, and deleting the Lambda function, S3 bucket, and DynamoDB table during cleanup.

The actual DynamoDB Lambda code is not included in the uploaded slides, so it is not recreated here.

Source reference: Virtualization, Cloud, and AWS slides 20–49.
