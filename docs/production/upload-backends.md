# File upload backends

Zulip in production supports a couple different backends for storing
files uploaded by users of the Zulip server (messages, profile
pictures, organization icons, custom emoji, etc.).

The default is the `LOCAL_UPLOADS_DIR` backend, which just stores
files on disk in the specified directory on the Zulip server.
Obviously, this backend doesn't work with multiple Zulip servers and
doesn't scale, but it's great for getting a Zulip server up and
running quickly.

We also support an `S3` backend, which uses the Python `boto` library
to upload files to Amazon S3 (and, with some work, it should be
possible to use any other storage provider compatible with `boto`).

## S3 backend configuration

Here, we document the process for configuring Zulip's S3 file upload
backend.  To enable this backend, you need to do the following:

1. Set `s3_key` and `s3_secret_key` in /etc/zulip/zulip-secrets.conf
to be the S3 access and secret keys that you want to use.

1. Set the `S3_AUTH_UPLOADS_BUCKET` and `S3_AVATAR_BUCKET` settings in
`/etc/zulip/settings.py` to be the S3 buckets you've created to store
user file uploads and user avatars, respectively.

1. Comment out the `LOCAL_UPLOADS_DIR` setting in
`/etc/zulip/settings.py` (add a `#` at the start of the line).

1. Edit /etc/nginx/sites-available/zulip-enterprise to comment out the
nginx configuration for `/user_avatars` and the `include
/etc/nginx/zulip-include/uploads.route` line and then reload the
`nginx` service (`service nginx reload`).  This will cause `nginx` to
direct requests to the Zulip server.

1. Finally, restart the Zulip server so that your settings changes
   take effect
   (`/home/zulip/deployments/current/scripts/restart-server`).

It's simplest to just do this configuration when setting up your Zulip
server for production usage.  Note that if you had any existing
uploading files, this process does not upload them to Amazon S3.  If
you have an existing server and are upgrading to the S3 backend, ask
in [#production help on chat.zulip.org][production-help] for advice on
how to migrate your data.

[production-help]: https://chat.zulip.org/#narrow/stream/31-production-help

## S3 bucket policy

The best way to do the S3 integration with Amazon is to create a new
IAM user just for your Zulip server with limited permissions.  For
each of the two buckets, you'll want to
[add an S3 bucket policy](https://awspolicygen.s3.amazonaws.com/policygen.html)
entry that looks something like this:

```
{
    "Version": "2012-10-17",
    "Id": "Policy1468991802321",
    "Statement": [
        {
            "Sid": "",
            "Effect": "Allow",
            "Principal": {
                "AWS": "ARN_PRINCIPAL_HERE"
            },
            "Action": [
                "s3:GetObject",
                "s3:DeleteObject",
                "s3:PutObject"
            ],
            "Resource": "arn:aws:s3:::BUCKET_NAME_HERE/*"
        },
        {
            "Sid": "Stmt1468991795389",
            "Effect": "Allow",
            "Principal": {
                "AWS": "ARN_PRINCIPAL_HERE"
            },
            "Action": "s3:ListBucket",
            "Resource": "arn:aws:s3:::BUCKET_NAME_HERE"
        }
    ]
}
```

The avatars bucket is intended to be world-readable, so you'll also
need a block like this:

```
{
    "Sid": "Stmt1468991795389",
    "Effect": "Allow",
    "Principal": {
        "AWS": "*"
    },
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::BUCKET_NAME_HERE/*"
}

```

The file-uploads bucket should not be world-readable.  See the
[documentation on the Zulip security model](security-model.html) for
details on the security model for uploaded files.
