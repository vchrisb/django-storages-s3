# django-storages-s3

S3 storage backend for Django.

This project is based on [django-storages](https://github.com/jschneier/django-storages) but focuses exclusively on Amazon S3 and S3-compatible storage services (e.g. RustFS, Cloudflare R2, DigitalOcean Spaces). By removing support for other storage backends, it aims to be a leaner, more focused package for projects that only need S3.

## Requirements

- Python >= 3.12
- Django >= 5.2
- boto3 >= 1.42

## Installation

```bash
pip install django-storages-s3
```

## Usage

To save media files to S3:

```python
STORAGES = {
    "default": {
        "BACKEND": "storages.s3.S3Storage",
        "OPTIONS": {
            "bucket_name": "my-media-bucket",
            "region_name": "us-east-1",
        },
    },
}
```

To put static files on S3 via `collectstatic`:

```python
STORAGES = {
    "staticfiles": {
        "BACKEND": "storages.s3.S3StaticStorage",
        "OPTIONS": {
            "bucket_name": "my-static-bucket",
        },
    },
}
```

Settings can be provided per-backend via `OPTIONS` or globally via Django settings (e.g. `AWS_STORAGE_BUCKET_NAME`). Per-backend options take precedence.

### Storage backends

- `storages.s3.S3Storage` - General file storage
- `storages.s3.S3StaticStorage` - Static files (querystring auth disabled)
- `storages.s3.S3ManifestStaticStorage` - Static files with Django's `ManifestFilesMixin`

## Authentication

There are several methods for specifying AWS credentials. `S3Storage` searches for them in this order:

1. `session_profile` / `AWS_S3_SESSION_PROFILE`
2. `access_key` / `AWS_S3_ACCESS_KEY_ID` or `AWS_ACCESS_KEY_ID`
3. `secret_key` / `AWS_S3_SECRET_ACCESS_KEY` or `AWS_SECRET_ACCESS_KEY`
4. `security_token` / `AWS_SESSION_TOKEN` or `AWS_SECURITY_TOKEN`
5. Environment variables `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`
6. Boto3's default session (instance profile, config files, etc.)

## Settings

All settings can be provided as `OPTIONS` keys (for per-backend configuration) or as global Django settings.

### Bucket and storage

| Option | Django setting | Default | Description |
| --- | --- | --- | --- |
| `bucket_name` | `AWS_STORAGE_BUCKET_NAME` | **Required** | The name of the S3 bucket that will host the files. |
| `location` | `AWS_LOCATION` | `""` | A path prefix that will be prepended to all uploads. |
| `file_overwrite` | `AWS_S3_FILE_OVERWRITE` | `True` | Whether files with the same name overwrite each other. Set to `False` to have extra characters appended. |
| `object_parameters` | `AWS_S3_OBJECT_PARAMETERS` | `{}` | Parameters set on all objects. See the [Boto3 docs for put_object](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/s3.html#S3.Client.put_object) for options like `CacheControl`, `StorageClass`, `Tagging`, and `Metadata`. |
| `default_acl` | `AWS_DEFAULT_ACL` | `None` | ACL for uploaded files (e.g. `public-read`, `private`). If `None`, files are `private` per Amazon's default. Ignored if `ACL` is set in `object_parameters`. See [canned ACLs](https://docs.aws.amazon.com/AmazonS3/latest/dev/acl-overview.html#canned-acl). |

### URLs

| Option | Django setting | Default | Description |
| --- | --- | --- | --- |
| `querystring_auth` | `AWS_QUERYSTRING_AUTH` | `True` | Whether generated URLs include query parameter authentication. Set to `False` for public buckets. |
| `querystring_expire` | `AWS_QUERYSTRING_EXPIRE` | `3600` | The number of seconds a generated URL is valid for. |
| `url_protocol` | `AWS_S3_URL_PROTOCOL` | `"https:"` | The protocol for constructed URLs. Must end in `:`. Only effective when `custom_domain` is set. |
| `custom_domain` | `AWS_S3_CUSTOM_DOMAIN` | `None` | Custom domain for constructed URLs. Must not end in `/`. |

### Connection

| Option | Django setting | Default | Description |
| --- | --- | --- | --- |
| `endpoint_url` | `AWS_S3_ENDPOINT_URL` | `None` | Custom S3 URL including scheme. Overrides `region_name` and `use_ssl`. Set `region_name` too to avoid `AuthorizationQueryParametersError`. |
| `region_name` | `AWS_S3_REGION_NAME` | `None` | Name of the AWS S3 region (e.g. `eu-west-1`). |
| `use_ssl` | `AWS_S3_USE_SSL` | `True` | Whether to use SSL when connecting to S3. |
| `verify` | `AWS_S3_VERIFY` | `None` | Whether to verify the connection to S3. Can be `False` or a path to a CA cert bundle. |
| `addressing_style` | `AWS_S3_ADDRESSING_STYLE` | `None` | `"virtual"` or `"path"`. |
| `signature_version` | `AWS_S3_SIGNATURE_VERSION` | `None` | Default is `s3v4`. Set to `s3` for legacy v2 signing (only supported in [certain regions](https://docs.aws.amazon.com/general/latest/gr/s3.html#s3_region)). |
| `proxies` | `AWS_S3_PROXIES` | `None` | Dictionary of proxy servers, e.g. `{"http": "foo.bar:3128"}`. |
| `client_config` | `AWS_S3_CLIENT_CONFIG` | `None` | A `botocore.config.Config` instance for advanced configuration. **Note:** overrides `addressing_style`, `signature_version`, and `proxies`. See [botocore docs](https://botocore.amazonaws.com/v1/documentation/api/latest/reference/config.html#botocore.config.Config). |
| `transfer_config` | `AWS_S3_TRANSFER_CONFIG` | `None` | A `boto3.s3.transfer.TransferConfig` instance to customize transfer options. See [Boto3 docs](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/customizations/s3.html#boto3.s3.transfer.TransferConfig). |

### Compression

| Option | Django setting | Default | Description |
| --- | --- | --- | --- |
| `gzip` | `AWS_IS_GZIPPED` | `False` | Whether to enable gzip compression for content types specified by `gzip_content_types`. |
| `gzip_content_types` | `GZIP_CONTENT_TYPES` | `text/css`, `text/javascript`, `application/javascript`, `application/x-javascript`, `image/svg+xml` | Content types to gzip when `gzip` is `True`. |

### Other

| Option | Django setting | Default | Description |
| --- | --- | --- | --- |
| `max_memory_size` | `AWS_S3_MAX_MEMORY_SIZE` | `0` | Maximum bytes a file can take before being rolled over to a temporary file on disk. `0` means no rollover. |

## IAM Policy

Minimum IAM policy for common usage:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:GetObjectAcl",
                "s3:GetObject",
                "s3:ListBucket",
                "s3:DeleteObject",
                "s3:PutObjectAcl"
            ],
            "Resource": [
                "arn:aws:s3:::example-bucket-name/*",
                "arn:aws:s3:::example-bucket-name"
            ]
        }
    ]
}
```

For more information about Principal, see [AWS JSON Policy Elements](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_principal.html).

## License

BSD 3-Clause. See [LICENSE](LICENSE) for details.
