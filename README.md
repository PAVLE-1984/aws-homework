import os
import logging
from dotenv import load_dotenv
import boto3
import click
import magic

# Load environment variables from a .env file
load_dotenv()

# Set up logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# Initialize S3 client
def init_client():
    access_key_id = os.getenv('AWS_ACCESS_KEY_ID')
    secret_access_key = os.getenv('AWS_SECRET_ACCESS_KEY')
    region_name = os.getenv('AWS_REGION')
    if not (access_key_id and secret_access_key and region_name):
        logger.error("AWS credentials or region not found in environment variables.")
        exit(1)
    return boto3.client('s3', aws_access_key_id=access_key_id, aws_secret_access_key=secret_access_key, region_name=region_name)

# List all buckets
def list_buckets(client):
    response = client.list_buckets()
    buckets = [bucket['Name'] for bucket in response['Buckets']]
    logger.info(f"Available buckets: {buckets}")

# Create a new bucket
def create_bucket(client, bucket_name):
    if not bucket_exists(client, bucket_name):
        client.create_bucket(Bucket=bucket_name)
        logger.info(f"Bucket '{bucket_name}' created successfully.")
    else:
        logger.error(f"Bucket '{bucket_name}' already exists.")

# Delete a bucket
def delete_bucket(client, bucket_name):
    if bucket_exists(client, bucket_name):
        client.delete_bucket(Bucket=bucket_name)
        logger.info(f"Bucket '{bucket_name}' deleted successfully.")
    else:
        logger.error(f"Bucket '{bucket_name}' does not exist.")

# Check if a bucket exists
def bucket_exists(client, bucket_name):
    try:
        client.head_bucket(Bucket=bucket_name)
        return True
    except:
        return False

# Validate file extension
def validate_file_extension(filename):
    valid_extensions = ['.bmp', '.jpg', '.jpeg', '.png', '.webp', '.mp4']
    file_extension = os.path.splitext(filename)[1].lower()
    if file_extension not in valid_extensions:
        raise ValueError(f"Invalid file format. Supported formats are: {', '.join(valid_extensions)}")
    return filename

# Download a file and upload it to S3
def download_file_and_upload_to_s3(client, bucket_name, local_filename, s3_key):
    try:
        validate_file_extension(local_filename)
        with open(local_filename, 'rb') as file:
            client.put_object(Bucket=bucket_name, Key=s3_key, Body=file)
            logger.info(f"File '{local_filename}' uploaded to S3 bucket '{bucket_name}' with key '{s3_key}'.")
    except Exception as e:
        logger.error(f"An error occurred: {e}")

# Set object access policy
def set_object_access_policy(client, bucket_name, s3_key, access_policy):
    client.put_object_acl(Bucket=bucket_name, Key=s3_key, AccessControlPolicy=access_policy)
    logger.info(f"Access policy set for object '{s3_key}' in bucket '{bucket_name}'.")

# Generate public read policy
def generate_public_read_policy(bucket_name, s3_key):
    return {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Sid": "PublicReadGetObject",
                "Effect": "Allow",
                "Principal": "*",
                "Action": "s3:GetObject",
                "Resource": f"arn:aws:s3:::{bucket_name}/{s3_key}"
            }
        ]
    }

# Create bucket policy
def create_bucket_policy(client, bucket_name, policy):
    client.put_bucket_policy(Bucket=bucket_name, Policy=policy)
    logger.info(f"Policy created for bucket '{bucket_name}'.")

# Read bucket policy
def read_bucket_policy(client, bucket_name):
    response = client.get_bucket_policy(Bucket=bucket_name)
    policy = response['Policy']
    logger.info(f"Policy for bucket '{bucket_name}': {policy}")

@click.group()
def cli():
    pass

@cli.command()
def init():
    """
    Initialize the S3 client.
    """
    init_client()

@cli.command()
def list():
    """
    List all S3 buckets.
    """
    client = init_client()
    list_buckets(client)

@cli.command()
@click.argument('bucket_name')
def create(bucket_name):
    """
    Create a new S3 bucket.
    """
    client = init_client()
    create_bucket(client, bucket_name)

@cli.command()
@click.argument('bucket_name')
def delete(bucket_name):
    """
    Delete an S3 bucket.
    """
    client = init_client()
    delete_bucket(client, bucket_name)

@cli.command()
@click.argument('bucket_name')
def exists(bucket_name):
    """
    Check if an S3 bucket exists.
    """
    client = init_client()
    if bucket_exists(client, bucket_name):
        logger.info(f"Bucket '{bucket_name}' exists.")
    else:
        logger.info(f"Bucket '{bucket_name}' does not exist.")

@cli.command()
@click.argument('bucket_name')
@click.argument('local_filename')
@click.argument('s3_key')
def upload(bucket_name, local_filename, s3_key):
    """
    Upload a file to an S3 bucket.
    """
    client = init_client()
    download_file_and_upload_to_s3(client, bucket_name, local_filename, s3_key)

@cli.command()
@click.argument('bucket_name')
@click.argument('s3_key')
def read_policy(bucket_name, s3_key):
    """
    Read the policy for an object in an S3 bucket.
    """
    client = init_client()
    read_bucket_policy(client, bucket_name)

@cli.command()
@click.argument('bucket_name')
@click.argument('s3_key')
def public_policy(bucket_name, s3_key):
    """
    Generate and apply a public read policy to an object in an S3 bucket.
    """
    client = init_client()
    policy = generate_public_read_policy(bucket_name, s3_key)
    create_bucket_policy(client, bucket_name, policy)

if __name__ == '__main__':
    cli()
