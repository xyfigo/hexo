---
title: Minio Java SDK for Amazon S3 Compatible Cloud Storage
date: 2019-01-28 09:34:15
tags: Minio
categories: Technology
---

Minio Java客户端SDK提供了访问任何Amazon S3兼容对象存储服务的简单API接口。

# 环境要求

Java1.8以上版本

## Maven配置

```xml
<dependency>
    <groupId>io.minio</groupId>
    <artifactId>minio</artifactId>
    <version>6.0.0</version>
</dependency>
```

## gradle配置

```groovy
dependencies {
    compile 'io.minio:minio:6.0.0'
}
```

# API接口使用例子

## 文件上传

这个例子连接到对象存储服务，在服务器上创建一个bucket，并且上传一个文件到bucket。

连接服务器需要3个参数：

| Params     | Description                                                  |
| ---------- | ------------------------------------------------------------ |
| Endpoint   | URL to object storage service.                               |
| Access Key | Access key is like user ID that uniquely identifies your account. |
| Secret Key | Secret key is the password to your account.                  |

## FileUploader.java

```java
import java.io.IOException;
import java.security.NoSuchAlgorithmException;
import java.security.InvalidKeyException;

import org.xmlpull.v1.XmlPullParserException;

import io.minio.MinioClient;
import io.minio.errors.MinioException;

public class FileUploader {
  public static void main(String[] args) throws NoSuchAlgorithmException, IOException, InvalidKeyException, XmlPullParserException {
    try {
      // Create a minioClient with the Minio Server name, Port, Access key and Secret key.
      MinioClient minioClient = new MinioClient("https://play.minio.io:9000", "Q3AM3UQ867SPQQA43P2F", "zuf+tfteSlswRu7BJ86wekitnifILbZam1KYY3TG");

      // Check if the bucket already exists.
      boolean isExist = minioClient.bucketExists("asiatrip");
      if(isExist) {
        System.out.println("Bucket already exists.");
      } else {
        // Make a new bucket called asiatrip to hold a zip file of photos.
        minioClient.makeBucket("asiatrip");
      }

      // Upload the zip file to the bucket with putObject
      minioClient.putObject("asiatrip","asiaphotos.zip", "/home/user/Photos/asiaphotos.zip");
      System.out.println("/home/user/Photos/asiaphotos.zip is successfully uploaded as asiaphotos.zip to `asiatrip` bucket.");
    } catch(MinioException e) {
      System.out.println("Error occurred: " + e);
    }
  }
}
```

## 编译FileUploader

```shell
javac -cp "minio-6.0.0-all.jar"  FileUploader.java
```

## 运行FileUploader

```shell
java -cp "minio-6.0.0-all.jar:." FileUploader
/home/user/Photos/asiaphotos.zip is successfully uploaded as asiaphotos.zip to `asiatrip` bucket.

mc ls play/asiatrip/
[2016-06-02 18:10:29 PDT]  82KiB asiaphotos.zip
```

# API参考

The full API Reference is available here.

- [Complete API Reference](https://docs.minio.io/docs/java-client-api-reference)

### API Reference: Bucket Operations

- [`makeBucket`](https://docs.minio.io/docs/java-client-api-reference#makeBucket)
- [`listBuckets`](https://docs.minio.io/docs/java-client-api-reference#listBuckets)
- [`bucketExists`](https://docs.minio.io/docs/java-client-api-reference#bucketExists)
- [`removeBucket`](https://docs.minio.io/docs/java-client-api-reference#removeBucket)
- [`listObjects`](https://docs.minio.io/docs/java-client-api-reference#listObjects)
- [`listIncompleteUploads`](https://docs.minio.io/docs/java-client-api-reference#listIncompleteUploads)

### API Reference: Object Operations

- [`getObject`](https://docs.minio.io/docs/java-client-api-reference#getObject)
- [`putObject`](https://docs.minio.io/docs/java-client-api-reference#putObject)
- [`copyObject`](https://docs.minio.io/docs/java-client-api-reference#copyObject)
- [`statObject`](https://docs.minio.io/docs/java-client-api-reference#statObject)
- [`removeObject`](https://docs.minio.io/docs/java-client-api-reference#removeObject)
- [`removeIncompleteUpload`](https://docs.minio.io/docs/java-client-api-reference#removeIncompleteUpload)

### API Reference: Presigned Operations

- [`presignedGetObject`](https://docs.minio.io/docs/java-client-api-reference#presignedGetObject)
- [`presignedPutObject`](https://docs.minio.io/docs/java-client-api-reference#presignedPutObject)
- [`presignedPostPolicy`](https://docs.minio.io/docs/java-client-api-reference#presignedPostPolicy)

### API Reference: Bucket Policy Operations

- [`getBucketPolicy`](https://docs.minio.io/docs/java-client-api-reference#getBucketPolicy)
- [`setBucketPolicy`](https://docs.minio.io/docs/java-client-api-reference#setBucketPolicy)

## Full Examples

#### Full Examples: Bucket Operations

- [ListBuckets.java](https://github.com/minio/minio-java/tree/master/examples/ListBuckets.java)
- [ListObjects.java](https://github.com/minio/minio-java/tree/master/examples/ListObjects.java)
- [BucketExists.java](https://github.com/minio/minio-java/tree/master/examples/BucketExists.java)
- [MakeBucket.java](https://github.com/minio/minio-java/tree/master/examples/MakeBucket.java)
- [RemoveBucket.java](https://github.com/minio/minio-java/tree/master/examples/RemoveBucket.java)
- [ListIncompleteUploads.java](https://github.com/minio/minio-java/tree/master/examples/ListIncompleteUploads.java)

#### Full Examples: Object Operations

- [PutObject.java](https://github.com/minio/minio-java/tree/master/examples/PutObject.java)
- [PutObjectEncrypted.java](https://github.com/minio/minio-java/tree/master/examples/PutObjectEncrypted.java)
- [GetObject.Java](https://github.com/minio/minio-java/tree/master/examples/GetObject.java)
- [GetObjectEncrypted.Java](https://github.com/minio/minio-java/tree/master/examples/GetObjectEncrypted.java)
- [GetPartialObject.java](https://github.com/minio/minio-java/tree/master/examples/GetPartialObject.java)
- [RemoveObject.java](https://github.com/minio/minio-java/tree/master/examples/RemoveObject.java)
- [RemoveObjects.java](https://github.com/minio/minio-java/tree/master/examples/RemoveObjects.java)
- [StatObject.java](https://github.com/minio/minio-java/tree/master/examples/StatObject.java)

#### Full Examples: Presigned Operations

- [PresignedGetObject.java](https://github.com/minio/minio-java/tree/master/examples/PresignedGetObject.java)
- [PresignedPutObject.java](https://github.com/minio/minio-java/tree/master/examples/PresignedPutObject.java)
- [PresignedPostPolicy.java](https://github.com/minio/minio-java/tree/master/examples/PresignedPostPolicy.java)

#### Full Examples: Bucket Policy Operations

- [SetBucketPolicy.java](https://github.com/minio/minio-java/tree/master/examples/SetBucketPolicy.java)
- [GetBucketPolicy.Java](https://github.com/minio/minio-java/tree/master/examples/GetBucketPolicy.java)

#### Full Examples: Server Side Encryption

- [CopyObjectEncrypted.java](https://github.com/minio/minio-java/tree/master/examples/CopyObjectEncrypted.java)
- [CopyObjectEncryptedKms.java](https://github.com/minio/minio-java/tree/master/examples/CopyObjectEncryptedKms.java)
- [CopyObjectEncryptedS3.java](https://github.com/minio/minio-java/tree/master/examples/CopyObjectEncryptedS3.java)
- [PutGetObjectEncrypted.java](https://github.com/minio/minio-java/tree/master/examples/PutGetObjectEncrypted.java)
- [PutObjectEncryptedKms.java](https://github.com/minio/minio-java/tree/master/examples/PutObjectEncryptedKms.java)
- [PutObjectEncryptedS3.java](https://github.com/minio/minio-java/tree/master/examples/PutObjectEncryptedS3.java)

## Explore Further

- [Complete Documentation](https://docs.minio.io/)
- [Minio Java Client SDK API Reference](https://docs.minio.io/docs/java-client-api-reference)
- [Build your own Photo API Service - Full Application Example](https://docs.minio.io/docs/java-photo-api-service)