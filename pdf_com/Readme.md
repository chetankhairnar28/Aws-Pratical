# AWS PDF Compression Pipeline

A serverless PDF compression workflow built with **Amazon S3**, **AWS Lambda**, and **PyMuPDF**. When a PDF is uploaded to the source S3 bucket, an S3 event triggers the Lambda function. The function attempts to optimize the PDF and saves the result in a separate destination bucket.

## Architecture

```text
PDF uploaded
    |
    v
Amazon S3 (Source Bucket)
    |
    | ObjectCreated event
    v
AWS Lambda (PDF-Compressor-Function)
    |
    | Download PDF and optimize with PyMuPDF
    | Keep optimized file only if it is smaller
    v
Amazon S3 (Destination Bucket)
```

## How It Works

1. A user uploads a PDF to the source S3 bucket.
2. An S3 `ObjectCreated` event invokes the Lambda function.
3. Lambda downloads the PDF and attempts structural optimization using PyMuPDF.
4. The function compares the original and optimized file sizes.
5. The smaller file is uploaded to the destination bucket. If optimization makes the file larger, the original is retained.
6. The execution can be monitored through Amazon CloudWatch Logs.

## AWS Resources

| Resource | Example name | Purpose |
|---|---|---|
| Source S3 bucket | `student-pdf-source-unique` | Receives original PDF files |
| Destination S3 bucket | `student-pdf-output-unique` | Stores processed PDFs |
| Lambda function | `PDF-Compressor-Function` | Runs the PDF optimization code |
| Lambda layer | `pymupdf-layer` | Provides the PyMuPDF dependency |
| Lambda execution role | `pdf-compressor-lambda-role` | Grants logging and required S3 permissions |

> Replace example resource names with your own globally unique S3 bucket names. Keep the source bucket, destination bucket, Lambda function, and layer in the same AWS Region for this setup.

## Prerequisites

- An AWS account with permission to create S3 buckets, IAM roles and policies, Lambda functions, and Lambda layers.
- The instructor-provided `lambda_function.py` file.
- The matching PyMuPDF Lambda Layer ZIP supplied with the project.
- A test PDF file.
- Both S3 bucket names.

**Important:** Use the supplied Lambda code and layer as a matching pair. The layer must be compatible with the Lambda Python runtime and CPU architecture. Do not replace it with an unrelated third-party layer.

## Setup

### 1. Create the S3 buckets

Create two private S3 buckets in the same AWS Region:

- **Source bucket:** receives uploaded PDFs.
- **Destination bucket:** stores processed PDFs.

Keep **Block all public access** enabled for both buckets.

### 2. Create the Lambda execution role

Create an IAM role trusted by AWS Lambda and attach the AWS managed policy:

- `AWSLambdaBasicExecutionRole` — allows the function to write logs to CloudWatch.

Add an inline policy granting only the required access to the two buckets. Replace the placeholders below with your actual bucket names:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadSourcePDFs",
      "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": "arn:aws:s3:::YOUR-SOURCE-BUCKET/*"
    },
    {
      "Sid": "WriteCompressedPDFs",
      "Effect": "Allow",
      "Action": ["s3:PutObject"],
      "Resource": "arn:aws:s3:::YOUR-DESTINATION-BUCKET/*"
    }
  ]
}
```

Save the policy, for example, as `PDFCompressionS3Access`. Avoid granting broad permissions such as `AmazonS3FullAccess` when they are not needed.

### 3. Create the Lambda function and attach the layer

1. Create a Lambda function named `PDF-Compressor-Function`.
2. Select the Python runtime and CPU architecture compatible with the supplied code and layer.
3. Attach the execution role created above.
4. Publish the supplied PyMuPDF ZIP as a Lambda layer named `pymupdf-layer`.
5. Attach the published layer version to the Lambda function.
6. Upload the instructor-provided `lambda_function.py` into the function and deploy it.

The layer ZIP should have a top-level `python/` directory containing the dependency. Lambda has a combined uncompressed deployment-size limit of 250 MiB for the function and its layers.

### 4. Configure Lambda settings

For an initial small-file test, the project guide suggests:

- **Memory:** `1024 MB`
- **Timeout:** `120 seconds`
- **Ephemeral storage:** `1024 MB`

Add these environment variables:

| Variable | Example value | Description |
|---|---|---|
| `DESTINATION_BUCKET` | `your-destination-bucket` | Exact name of the destination S3 bucket |
| `MAX_FILE_MB` | `50` | Maximum file size in MB, as used by the supplied code |

Review your function code to confirm how these environment variables are handled. Adjust resources only after testing and reviewing potential AWS charges.

### 5. Configure the S3 trigger

1. Open the Lambda function and choose **Add trigger**.
2. Select **S3** and choose the **source bucket**.
3. Select all object-created events (`ObjectCreated`).
4. Add the trigger and confirm it appears in the Lambda configuration.

**Important:** Attach the trigger to the source bucket only. Do not configure the same trigger on the destination bucket, as that could cause repeated invocations.

## Test the Pipeline

1. Upload a small test PDF (under 5 MB is recommended for the first test) to the source bucket.
2. Wait for the S3 event to invoke Lambda.
3. Open the Lambda function's **Monitor** tab and view the CloudWatch Logs.
4. Check that a processed PDF with the expected object key/name appears in the destination bucket.
5. Download the output PDF and verify that it opens correctly.
6. Compare the original and output file sizes.
7. Check important document content, including text, images, links, forms, and signatures where relevant.

## Troubleshooting

| Problem | What to check |
|---|---|
| `No module named 'fitz'` or `'pymupdf'` | Confirm the layer is attached, its ZIP contains a top-level `python/` directory, and the import name, Python runtime, and architecture match. |
| `403 Forbidden` / `HeadObject` permission issue | Check that the Lambda execution role has `s3:GetObject` for the source bucket. Review bucket policies, permissions boundaries, and KMS permissions if applicable. IAM does not define a separate `s3:HeadObject` action. |
| `AccessDenied` on `PutObject` | Confirm `s3:PutObject` is allowed for the destination bucket and review its bucket policy and encryption requirements. |
| Lambda times out | Test with a smaller PDF, inspect CloudWatch Logs, and increase timeout, memory, or temporary storage as needed. |
| Output file is missing | Check CloudWatch Logs, the `DESTINATION_BUCKET` value, S3 trigger configuration, and execution-role permissions. |
| Output PDF is larger than the original | The workflow is designed to keep the original when optimization does not reduce the file size. |

## Security Notes

- Keep both buckets private and block public access.
- Use least-privilege IAM permissions.
- Do not commit AWS access keys, secrets, or sensitive PDFs to source control.
- Review CloudWatch Logs before sharing them; logs may contain object names or other operational details.
- Test with non-sensitive documents first.

## Cost Considerations

Charges may apply for S3 storage and requests, Lambda invocations and compute duration, CloudWatch Logs, and any other AWS resources used. Check current AWS pricing and delete lab resources when you no longer need them.

## Limitations

- The actual compression behavior depends on the supplied `lambda_function.py` implementation and PyMuPDF options.
- Structural optimization may not substantially reduce every PDF.
- Validate output quality and document features before using the pipeline for important files.
- This README describes the workflow and configuration from the project guide; verify implementation-specific behavior against your supplied code.

## Cleanup

To avoid ongoing charges after testing:

1. Remove the S3 event notification/Lambda trigger.
2. Delete test objects from both buckets.
3. Delete the Lambda function and its layer if no longer needed.
4. Delete the IAM role if it is not used by other resources.
5. Delete the S3 buckets after emptying them.

## License

Add the license that applies to your own code and project dependencies. Confirm the license terms for the instructor-provided code and PyMuPDF before redistributing them.
