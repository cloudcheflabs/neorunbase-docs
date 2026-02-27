# Getting Started

This shows how to install NeorunBase on local to experience it quickly.

## Prerequisites

Because NeorunBase is written in Java, Java 17 needs to be installed on local.

## Install NeorunBase on Local

NeorunBase distribution can be downloaded like this.

```agsl
curl -L -O https://github.com/cloudcheflabs/neorunbase-pack/releases/download/neorunbase-archive/neorunbase-1.0.0.tar.gz
```

And untar the downloaded package.
```agsl
tar zxvf neorunbase-1.0.0.tar.gz

cd neorunbase-1.0.0;
```

Run the example servers which are 1 API server and 3 Data nodes with Zookeeper on local.

```agsl
export NEORUNBASE_MASTER_KEY="NeorunBaseMasterKey1200303003abc"
bin/start-example-servers.sh;
```

> Environment variable `NEORUNBASE_MASTER_KEY` that must be at least 32 characters needs to be exported when running NeorunBase servers.


After a few seconds, visit admin page of NeorunBase.

```agsl
http://localhost:8888/admin
```

First initial admin user and password is `admin` / `admin`, after that you need to change the initial password.

<img width="1200" src="../../images/getting-started/dashboard.png"/>


## Create Bucket and Credential

In order to upload files to NeorunBase, a bucket and a S3 credential need to be created in Admin UI.

In the menu of `Identity Control`, click `New Keypair` to create a S3 credential(access key / secret key).
After that, click the download icon in the created keypair to download access key and secret key.
<img width="1200" src="../../images/getting-started/credential.png"/>

And you also need to create a bucket, go to the menu of `Object Store`, click `New Bucket`
to create a bucket with the name of `test-bucket`.


## Upload files to NeorunBase

Now, you can connect NeorunBase to upload files. AWS CLI will be used for this example.
Configure a S3 profile with the S3 credential(access key and secret key) from the downloaded credential file before.

```agsl
aws configure --profile=custom-s3 set default.s3.signature_version s3v4;
aws configure --profile=custom-s3 set aws_access_key_id <access-key>;
aws configure --profile=custom-s3 set aws_secret_access_key <secret-key>;
aws configure --profile=custom-s3 set region us-east-1;
```

Upload a file to NeorunBase as below, take a note that the S3 endpoint is `http://localhost:8080`.

```agsl
# upload file.
aws s3 cp ./values-dev_old.yaml s3://test-bucket/values-dev_old.yaml \
--profile=custom-s3 \
--endpoint-url http://localhost:8080 \
--no-verify-ssl
```

You can also test `ls`, `download`, `delete` of objects in NeorunBase.
```agsl
# ls.
aws s3 ls \
s3://test-bucket \
--profile=custom-s3 \
--endpoint=http://localhost:8080 \
--no-verify-ssl \
--recursive \
--human-readable \
--summarize \
;


# download.
aws s3 cp \
--profile=custom-s3 \
--endpoint=http://localhost:8080 \
s3://test-bucket/values-dev_old.yaml values-dev_downloaded.yaml \
;

# delete.
aws s3 rm \
--profile=custom-s3 \
--endpoint=http://localhost:8080 \
s3://test-bucket/values-dev_old.yaml \
;
```

## Stop Example Servers

In order to stop the running example servers, run this.

```agsl
bin/stop-example-servers.sh
```

