# Go OCI SDK Create Bucket and Upload Files Into OCI Object Storage

###  Requirement:
We need to install [Go programming language](https://golang.org/dl/) in your host.

We need to setup config file for OCI SDK to get correct user credential, tenancy, compartment_id ...etc. Refer [my blog for example][1]:

We need to use OCI object storage for our backup purpose.  
First to create Bucket, then we put all our backup files into the bucket. Refer [GitHub OCI Go examples](https://github.com/oracle/oci-go-sdk/tree/master/example)

Set http_proxy and https_proxy in the workstation to access internet if necessary.

To install Oracle OCI Go sdk
```
 go get -u github.com/oracle/oci-go-sdk
```
###  Solution:

####  Create bucket example:
```
package main
import (
	"context"
    "os"
	"fmt"
	"flag"
	"github.com/oracle/oci-go-sdk/common"
	"github.com/oracle/oci-go-sdk/objectstorage"
)
func getNamespace(ctx context.Context, c objectstorage.ObjectStorageClient) string {
	request := objectstorage.GetNamespaceRequest{}
	r, err := c.GetNamespace(ctx, request)
	if err != nil {
		fmt.Println("Error:", err)
		return "Error"
		} 
//	fmt.Println("get namespace")
	return *r.Value
}
func createBucket(ctx context.Context, c objectstorage.ObjectStorageClient, namespace, name ,compartmentID string) {
	request := objectstorage.CreateBucketRequest{
		NamespaceName: &namespace,
	}
	request.CompartmentId = &compartmentID
	request.Name = &name
	request.Metadata = make(map[string]string)
	request.PublicAccessType = objectstorage.CreateBucketDetailsPublicAccessTypeNopublicaccess
	_, err := c.CreateBucket(ctx, request)
	if err != nil {
		fmt.Println("Error:", err)
		return
		} 
	fmt.Printf("You have created a bucket with name %s in namespace %s\n",name,namespace)
}
func main() {
    //get compartment id from env as defaultconfigprovider don't have info	
	compartmentID := os.Getenv("OCI_COMPARTMENT_ID")
    Bucketname := flag.String("bucketname", "", "The name of bucket to be created")
	flag.Parse()
	o, err := objectstorage.NewObjectStorageClientWithConfigurationProvider(common.DefaultConfigProvider())
	if err != nil {
		fmt.Println("Error:", err)
		return
		} 
	ctx := context.Background()
	namespace := getNamespace(ctx, o)
	if *Bucketname == "" { 
	  fmt.Println("No bucketname is given, create_bucket -h to see help")
	  return
	  } else {
	  fmt.Printf("You are going to create a buckename named %s in namespace: %s\n",*Bucketname,namespace)
	  }
	createBucket(ctx, o, namespace, *Bucketname,compartmentID)
	return
}
```

####  Upload files into the bucket example:
```
package main
import (
	"context"
    "os"
	"io"
	"path/filepath"
	"fmt"
	"flag"
	"github.com/oracle/oci-go-sdk/common"
	"github.com/oracle/oci-go-sdk/objectstorage"
)

func getNamespace(ctx context.Context, c objectstorage.ObjectStorageClient) string {
	request := objectstorage.GetNamespaceRequest{}
	r, err := c.GetNamespace(ctx, request)
	if err != nil {
		fmt.Println("Error:", err)
		return "Error"
		} 
//	fmt.Println("get namespace")
	return *r.Value
}

func putObject(ctx context.Context, c objectstorage.ObjectStorageClient, namespace, bucketname, objectname string, contentLen int64, content io.ReadCloser, metadata map[string]string) error {
	request := objectstorage.PutObjectRequest{
		NamespaceName: &namespace,
		BucketName:    &bucketname,
		ObjectName:    &objectname,
		ContentLength: &contentLen,
		PutObjectBody: content,
		OpcMeta:       metadata,
	}
	_, err := c.PutObject(ctx, request)
	if err != nil {
		fmt.Println("Error:", err)
		return err
		} 
	fmt.Printf("You have uploaded file %s in bucket %s\n",objectname,bucketname)
	return err
}
func main() {
    //get compartment id from env as defaultconfigprovider don't have info	
//	compartmentID := os.Getenv("OCI_COMPARTMENT_ID")
    Bucketname := flag.String("bucketname", "", "The name of bucket for file to upload ")
	Filename := flag.String("filename", "", "The full path of files to be uploaded, wildcard can used ie: upload_files -bucketname testaa -filename \"filenam**\" ")
	flag.Parse()
	o, err := objectstorage.NewObjectStorageClientWithConfigurationProvider(common.DefaultConfigProvider())
	if err != nil {
		fmt.Println("Error:", err)
		return
		} 
	
	ctx := context.Background()
	namespace := getNamespace(ctx, o)
	if *Bucketname == "" || *Filename =="" { 
	  fmt.Println("No bucketname or No filename is given, upload_file -h to see help")
	  return
	  } else {
	  fmt.Printf("You are going to upload file %s in bucket: %s\n",*Filename,*Bucketname)
	  }
	
	filename,err := filepath.Glob(*Filename)
	if filename == nil {
	   fmt.Println("Error: No files found to upload")
	   }
//	fmt.Println(*Filename)
//	fmt.Println(filename)
	if err != nil {
		fmt.Println("Error:", err)
		return
		} 
	for _, f := range filename {
	  file,err := os.Open(f)
	  if err != nil {
		fmt.Println("Error:", err)
		return
		}
	  defer file.Close()
	  fi,err := file.Stat()
	  fname := filepath.Base(f)
	  err = putObject(ctx, o, namespace, *Bucketname, fname, fi.Size(), file, nil)
	  if err != nil {
		fmt.Println("Error:", err)
		return
		} 
	}
	return
}
```
[1]: http://www.henryxieblogs.com/2018/10/prepare-config-file-for-python3-oci-sdk.html
