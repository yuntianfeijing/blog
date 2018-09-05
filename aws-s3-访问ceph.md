---
title: aws s3 访问ceph
date: 2018-09-05 11:31:28
tags: fs
---
## 使用亚马逊S3　GO-SDK 访问ceph ##

### 修改本机hosts文件 ###
添加ceph网关节点的域名，如:`10.17.90.9 test.mxy-d-wkm-dev-1.test.xesv5.com`
修改ceph网关节点的nginx配置，添加` /etc/nginx/conf.d/mxy-d-wkm-dev-1.test.xesv5.com.conf`文件,把请求转发到7480端口，注意upstream域名配置需要一直（s3 V4的验证里验证了域名,转发不能改变host）

```
upstream mxy-d-wkm-dev-1.test.xesv5.com {
    server 127.0.0.1:7480;
}
server {
    listen       80;
    server_name  mxy-d-wkm-dev-1.test.xesv5.com;
    #charset koi8-r;
    access_log  /var/log/ceph/host.access.log  main;
    location / {
        proxy_pass http://mxy-d-wkm-dev-1.test.xesv5.com;
    }
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}

```
### 代码 ###
```
package main
import (
	"fmt"
	"github.com/aws/aws-sdk-go/aws/session"
	"github.com/aws/aws-sdk-go/service/s3"
	"github.com/aws/aws-sdk-go/aws/credentials"
	"github.com/aws/aws-sdk-go/aws"
	"io/ioutil"
	"bytes"
)
func main() {
	sess, err := session.NewSession(&aws.Config{
		Region:      aws.String("us-east-1"),
		Credentials: credentials.NewStaticCredentials("W3W7DQADM05MG6BSJ9GB", "rfOOJQbwLEKYBTBRO3yWmXkMAGBgKSqkDVX8l4ka", ""),
		Endpoint: aws.String("http://mxy-d-wkm-dev-1.test.xesv5.com"),
		DisableSSL: aws.Bool(true),
		S3ForcePathStyle:aws.Bool(true),
	})
	if err != nil {
		panic(err)
	}
	data, err := sess.Config.Credentials.Get()
	if err != nil {
		panic(err)
	}
	fmt.Println(data)

	svc := s3.New(sess)

	//创建Bucket
	crtBk,err := svc.CreateBucket(&s3.CreateBucketInput{
		Bucket:aws.String("test"),
	})
	if err != nil {
		panic(err)
	}
	fmt.Println("crtBk:",crtBk)
	//查询Bucket列表
	listBk,err := svc.ListBuckets(nil)
	if err != nil {
		panic(err)
	}
	fmt.Println("listBk:",listBk.Buckets)
	//上传文件
	addObj,err := svc.PutObject(&s3.PutObjectInput{
		Bucket:aws.String("test"),
		Key:aws.String("zxl/test.txt"),
		Body:bytes.NewReader([]byte("test zxl")),
	})
	if err != nil {
		panic(err)
	}
	fmt.Println("addObj:",addObj)
	//下载文件
	optPut,err := svc.GetObject(&s3.GetObjectInput{
		Bucket: aws.String("test"),
		Key: aws.String("zxl/test.txt"),})
	if err != nil {
		panic(err)
	}
	outData,err := ioutil.ReadAll(optPut.Body)
	if err != nil {
		panic(err)
	}
	defer optPut.Body.Close()
	fmt.Println(string(outData))
	//删除上传的文件
	delObj,err := svc.DeleteObject(&s3.DeleteObjectInput{
		Bucket:aws.String("test"),
		Key:aws.String("zxl/test.txt"),
	})
	if err != nil {
		panic(err)
	}
	fmt.Println("delObj:",delObj)
	//删除Bucket
	delBk,err := svc.DeleteBucket(&s3.DeleteBucketInput{
		Bucket:aws.String("test"),
	})
	if err != nil {
		panic(err)
	}
	fmt.Println("delBk:",delBk)
}
```
