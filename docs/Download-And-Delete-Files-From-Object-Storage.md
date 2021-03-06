# Go OCI SDK Delete and Download Muiltple Files From OCI Object Storage Bucket

###  Requirement:
We need to install [Go programming language](https://golang.org/dl/) in your host.

We need to setup config file for OCI SDK to get correct user credential, tenancy, compartment_id ...etc. Refer [my blog for example][1]:

We need to use OCI object storage for our backup purpose.  
Delete and download files from OCI bucket Refer [GitHub OCI Go examples](https://github.com/oracle/oci-go-sdk/tree/master/example)

Set http_proxy and https_proxy in the workstation to access internet if necessary.

To install Oracle OCI Go sdk
```
 go get -u github.com/oracle/oci-go-sdk
```
###  Solution:
####  Delete multiple files example:
```
package main

import (
	"context"
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

func getListfiles(ctx context.Context, c objectstorage.ObjectStorageClient, namespace, bucketname,prefix_files_name string ) []string {
	request := objectstorage.ListObjectsRequest{
	    NamespaceName: &namespace,
		BucketName:    &bucketname,
		Prefix:       &prefix_files_name,
		}
	r, err := c.ListObjects(ctx, request)
	if err != nil {
		fmt.Println("Error:", err)
		return nil
		} 
	var filelist []string
	for _, v := range r.ListObjects.Objects {
//	    fmt.Printf("%s\n",*v.Name)
		filelist = append(filelist,*v.Name)
		}
	if filelist == nil {
	   fmt.Printf("Error: No files with prefix %s found in bucket %s to delete",prefix_files_name,bucketname)
		return nil
		}
//	fmt.Println(filelist)
	return filelist
}

func deleteObject(ctx context.Context, c objectstorage.ObjectStorageClient, namespace, bucketname, objectname string) (err error) {
	request := objectstorage.DeleteObjectRequest{
		NamespaceName: &namespace,
		BucketName:    &bucketname,
		ObjectName:    &objectname,
	}
	_, err = c.DeleteObject(ctx, request)
	if err != nil {
		fmt.Println("Error:", err)
		return err
		} 
	fmt.Printf("You have deleted file %s in bucket %s\n",objectname,bucketname)
	return err
}

func main() {
    //get compartment id from env as defaultconfigprovider don't have info	
//	compartmentID := os.Getenv("OCI_COMPARTMENT_ID")
    Bucketname := flag.String("bucketname", "", "The name of bucket for file to delete ")
	prefix_files_name := flag.String("prefix", "", "The prefix filenames to delete, No wildcard needed, ie \"livesql\" will match \"livesql*\" ")
	flag.Parse()
	o, err := objectstorage.NewObjectStorageClientWithConfigurationProvider(common.DefaultConfigProvider())
	if err != nil {
		fmt.Println("Error:", err)
		return
		} 
	
	ctx := context.Background()
	namespace := getNamespace(ctx, o)
	if *Bucketname == "" || *prefix_files_name =="" { 
	  fmt.Println("No bucketname or No prefix_files_name is given, delete_files -h to see help")
	  return
	  } else {
	  fmt.Printf("You are going to delete files with prefix %s in bucket: %s\n",*prefix_files_name,*Bucketname)
	  }
	  listfiles := getListfiles(ctx,o,namespace,*Bucketname,*prefix_files_name)
    
	  for _, f := range listfiles {
	    err = deleteObject(ctx, o, namespace, *Bucketname, f)
	    if err != nil {
		fmt.Println("Error:", err)
		return
		} 
	}
	return
}
```
####  download multiple files example:
```
package main

import (
        "context"
        "fmt"
        "flag"
        "strings"
        "os"
        "io/ioutil"
        "github.com/oracle/oci-go-sdk/common"
        "github.com/oracle/oci-go-sdk/objectstorage"
)

func check(e error) {
    if e != nil {
        panic(e)
    }
}

func getNamespace(ctx context.Context, c objectstorage.ObjectStorageClient) string {
        request := objectstorage.GetNamespaceRequest{}
        r, err := c.GetNamespace(ctx, request)
        check(err)
//      fmt.Println("get namespace")
        return *r.Value
}

func getListfiles(ctx context.Context, c objectstorage.ObjectStorageClient, namespace, bucketname,prefix_files_name string ) []string {
        request := objectstorage.ListObjectsRequest{
            NamespaceName: &namespace,
                BucketName:    &bucketname,
                Prefix:       &prefix_files_name,
                }
        r, err := c.ListObjects(ctx, request)
        check(err)
        var filelist []string
        for _, v := range r.ListObjects.Objects {
//          fmt.Printf("%s\n",*v.Name)
                filelist = append(filelist,*v.Name)
                }
        if filelist == nil {
           fmt.Printf("Error: No files with prefix %s found in bucket %s to download",prefix_files_name,bucketname)
                return nil
                }
//      fmt.Println(filelist)
        return filelist
}

func downloadObject(ctx context.Context, c objectstorage.ObjectStorageClient, namespace, bucketname, objectname,Save_location string) (err error) {
        request := objectstorage.GetObjectRequest{
                NamespaceName: &namespace,
                BucketName:    &bucketname,
                ObjectName:    &objectname,
        }
        r, err := c.GetObject(ctx, request)
        check(err)
        var fullname []string
        filedata, err := ioutil.ReadAll(r.Content)
        check(err)
//      fmt.Println(Save_location)
        fullname = append(fullname,Save_location)
        fullname = append(fullname,"/")
        fullname = append(fullname,objectname)
//      fmt.Println(fullname)
        Save_location = strings.Join(fullname,"")
//      fmt.Println(Save_location)
        f, err := os.Create(Save_location)
        check(err)
        defer f.Close()

        _, err = f.Write(filedata)
        check(err)

        fmt.Printf("You have downloaded file %s from bucket %s to %s\n",objectname,bucketname,Save_location)
        return err
}

func main() {
    //get compartment id from env as defaultconfigprovider don't have info
//    compartmentID := os.Getenv("OCI_COMPARTMENT_ID")
        Bucketname := flag.String("bucketname", "", "The name of bucket for file to download ")
        prefix_files_name := flag.String("prefix", "", "The prefix filenames to download, No wildcard needed, ie \"livesql\" will match \"livesql*\" ")
        Save_location := flag.String("save_location", "/tmp/", "The full path of location for files to be saved")
        flag.Parse()
        o, err := objectstorage.NewObjectStorageClientWithConfigurationProvider(common.DefaultConfigProvider())
        check(err)

        ctx := context.Background()
        namespace := getNamespace(ctx, o)
        if *Bucketname == "" || *prefix_files_name ==""  {
          fmt.Println("No bucketname prefix_files_name is given, download_files -h to see help")
          return
          } else {
          fmt.Printf("You are going to download files with prefix %s in bucket: %s\n",*prefix_files_name,*Bucketname)
          }
          listfiles := getListfiles(ctx,o,namespace,*Bucketname,*prefix_files_name)

          for _, f := range listfiles {
            err = downloadObject(ctx, o, namespace, *Bucketname, f,*Save_location)
            check(err)
        }
        return
}
```
[1]: http://www.henryxieblogs.com/2018/10/prepare-config-file-for-python3-oci-sdk.html
