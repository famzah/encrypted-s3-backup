# encrypted-s3-backup
Encrypted secure remote backup over Internet on Linux. The data is stored on [Amazon S3](https://aws.amazon.com/s3).

# Motivation
This backup system leaves control in your hands. Additionally, the backup scripts are very small and you can easily audit them. You can review this [blog page](https://blog.famzah.net/2016/10/23/goodbye-acronis-cloud-hello-encrypted-s3-backup/) for further information regarding projected S3 costs and performance.

# Encryption
It's important to mention that the data is transferred securely to S3 (noone else can see it) but encrypted server-side by S3 (therefore AWS S3 sees your data unencrypted for a moment). This means you have to trust Amazon S3 and their mechanism for [Server-Side Encryption with Customer-Provided Keys (SSE-C)](http://docs.aws.amazon.com/AmazonS3/latest/dev/ServerSideEncryptionCustomerKeys.html). They claim and I believe them that:
- When you upload an object, Amazon S3 uses the encryption key you provide to apply AES-256 encryption to your data and removes the encryption key from memory.
- Amazon S3 does not store the encryption key you provide. Instead, we store a randomly salted HMAC value of the encryption key in order to validate future requests. The salted HMAC value cannot be used to derive the value of the encryption key or to decrypt the contents of the encrypted object. That means, if you lose the encryption key, you lose the object.

# Confidentiality
Note that while your S3 data is encrypted and nobody can read its content, anyone with access to the S3 bucket can list the filenames in it. If the filenames without the data in them reveal any sensitive information, you must not give access to your S3 bucket to anyone else. Which is a good idea anyway.

# Security
The customer-provided encryption key "--sse-c-key" is part of the command-line arguments for "aws s3 sync" which can be insecure. Many Linux distros allow anyone to see the whole "ps" output which would reveal the key to a third-party user on the same Linux machine. While this shouldn't be a big problem for a single-account Desktop system, it could be a security concern for multi-user systems which do not hide the "ps" entries of processes not owned by the current user.

# Prerequisites
You have to [install](http://docs.aws.amazon.com/cli/latest/userguide/installing.html) and [configure](http://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html) the AWS Command Line Interface (CLI).

Additionally, you have to GIT clone this repository on your Linux machine and put the script "encrypted-s3-sync" in your PATH executables. The most easy way to do this on Ubuntu, for example, is to make a symlink in your private ~/bin directory to "encrypted-s3-sync":
```
ln -s ~/repos/encrypted-s3-backup/encrypted-s3-sync ~/bin/
```

# Setup and usage
Copy the "config.template" file and populate the settings in it. Let's assume that you saved the config in `~/s3-backup.config` and used the following example settings:
```bash
#!/bin/bash

AWS_PROFILE='default'
AWS_REGION='us-east-1'
AWS_BUCKET='example-bucket-f9kgixb'
AWS_RETENTION_DAYS=30

LOCAL_DIR='/home/user32/documents'
LOCAL_LOCK=~/.example-s3-backup.lock
REMOTE_DIR_PREFIX='workstation1/linux/user32/documents'
LOCAL_LIST=''

ENCRYPTION_KEY='2TilZUVi0XN99QjBSpdRCKC8xDtLoUsy'
```

Then do the first backup. Depending on the size of your local data, this may take a lot of time:
```bash
$ encrypted-s3-sync ~/s3-backup.config
encrypted-s3-sync[9199]: WARNING: You have not backed up your encryption key, yet.
```

Note that the backup script produces no debug or progress output on the console. Only errors are displayed, if any. The debug info goes to Syslog and you can usually see it in the "/var/log/syslog" file.

Now let's pay attention to the warning. You **must** store the encryption key online by protecting it with an easy-to-remember password. You will not be able to restore from your backup if you don't store the original key or if you forget the easy-to-remember password. Please do not skip this step:
```bash
$ ./encrypted-s3-upload-key
```

You can verify if you can retrieve your backup encryption key by executing the following command, which you will also need in case your hard disk dies:
```bash
$ ./encrypted-s3-download-key default eu-central-1 example-bucket-f9kgixb
```

If you run the backup again, it won't produce any warnings:
```bash
$ encrypted-s3-sync ~/s3-backup.config
```

# Scheduling your backup
You are free to schedule the backup as often as you need, and using any suitable Linux mechanism. You would usually create an "/etc/cron.daily" task. Make sure that you receive and read any email messages sent from "crond". Otherwise you may miss any error messages by the backup script.

# Restoring a single file
You can download a single file using the following command:
```bash
$ ./encrypted-s3-download-file default eu-central-1 s3://example-bucket-f9kgixb/workstation1/linux/user32/documents/notes2.txt /tmp/test.txt
```

This will download the latest version of the file.

Downloading an older version or whole directories recursively is possible but not yet wrapped with scripts. Refer to the AWS CLI documentation about the "aws s3 cp" and "aws s3 sync" commands.

# Additional checks for backup freshness
If you are paranoid whether the backup was executed on schedule and without errors, you should do the following:
- make sure that any output by "encrypted-s3-sync" is emailed to you
- wrap "encrypted-s3-sync" and after its successful execution, newly create and upload a file with known name to your S3 bucket, like "backup-completed-marker"
- check for the oldness of "backup-completed-marker" by an external service which notifies you if the last backup was completed too long time ago
