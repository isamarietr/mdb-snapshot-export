# Export a Snapshot to S3
## Setup
### Create a S3 Bucket
Create your bucket using the AWS Management Console or by using the following AWS CLI command:

```
aws s3api create-bucket --bucket it-bucket-demo --region us-east-1
```

*You can change `it-bucket-demo` to something else and change the region to any other S3 AWS region you wish*
### Create and Authorize an AWS IAM Role

This part is better done using the Atlas UI. Let's check it out.

1. Open your Atlas Project **Integrations**

 	![](figure1.png)
 	
2. Select **AWS IAM Role Access**

 	![](figure2.png)
 	
3. Click on **Authorize an AWS IAM Role**

	![](figure3.png)
	
4. The **Authorize AWS IAM Role** wizard takes you through setting up your AWS role pretty easily. Click **Next**.

	![](figure4.png)
	
5. This is where things get interesting... In the page shown below you are: 

  a) Creating Cloud Provider Access in Atlas, and 
  
  b) Creating an AWS IAM Role with a trust policy (for Atlas to assume your role)
  
  You can also do this using the API. Check out the [documentation](https://docs.atlas.mongodb.com/security/set-up-unified-aws-access/) for more info: 
	![](figure5.png)

*Note: Once you issue the command is Step 3, you will get a JSON back. You will enter the value of **"Arn"** in Step 4.*
	![](figure5e.png)

6. Once you've validated and finished the setup, you should get the following message.

	![](figure6.png)

7. Check out the Role Details for this newly created role and copy the **External ID**. You will need it to create an export bucket.

	![](figure7.png)
	
	![](figure8.png)


### Check Access Roles (Optional)

Use the [Cloud Provider Access Roles API](https://docs.atlas.mongodb.com/reference/api/cloud-provider-access-get-roles/#example-request) to see your recently created AWS IAM Role

```
curl --user "{PUBLIC-KEY}:{PRIVATE-KEY}" -X GET --digest \
     --header "Accept: application/json" \
     "https://cloud.mongodb.com/api/atlas/v1.0/groups/{GROUP-ID}/cloudProviderAccess/?pretty=true"
```

## Export Snapshot to S3

To Export an existing snapshot to your S3 bucket you need to create an export bucket and create an export job.

Check out [Cloud Backup Snapshot Export API](https://docs.atlas.mongodb.com/reference/api/cloud-backup/export/exports/) documentation for more info.

You can also import the [Export Snapshot Collection](export_to_s3_postman_collection.json) to Postman. If you have the [MongoDB APIs Postman Collection](https://www.postman.com/mongodb-devrel/workspace/mongodb-public/overview) already in your Postman workspace, you can use the same environment variables, otherwise use [this environment file](Atlas.postman_environment.json).

### Create an Export Bucket

This step **does not** create an S3 bucket, you should've created a S3 bucket in the Setup section. This is simply telling Atlas which buckets can be used for exporting snapshots.

Make sure that `bucketName` is the simple name of your S3 bucket (not the ARN) and that the `iamRoleId` has the same value as the **External ID** for your AWS IAM role in Atlas UI (or the value of `roleId` returned from the `cloudProviderAccess` API)

```
curl --user "{PUBLIC-KEY}:{PRIVATE-KEY}" -X POST --digest \
     --header "Accept: application/json" \
     --header "Content-Type: application/json" \
     "https://cloud.mongodb.com/api/atlas/v1.0/groups/{GROUP-ID}/backup/exportBuckets?pretty=true" \
     --data '{
       "iamRoleId": "{ROLE-EXTERNAL-ID}",
       "bucketName": "{YOUR-BUCKET-NAME}",
       "cloudProvider": "AWS"
     }'
```

From the results, capture the value for `_id`, this will be your `exportBucketId` value when exporting the snapshot.

### Get the Snapshot to be exported

To export a snapshot, you will need a Snapshot ID. Use the [Get All Cloud Backups API](https://docs.atlas.mongodb.com/reference/api/cloud-backup/backup/get-all-backups/#example-request) to retrieve the ID of the snapshot to be exported.

```
curl --user "{PUBLIC-KEY}:{PRIVATE-KEY}" --digest --include \
     --header "Accept: application/json" \
     --request GET "https://cloud.mongodb.com/api/atlas/v1.0/groups/{GROUP-ID}/clusters/{CLUSTER-NAME}/backup/snapshots?pretty=true"
```

From the results, find the Snapshot ID for the snapshot you want to export, this will be your `snapshotId` value when exporting the snapshot.


![](figure9.png)


### Create an Export Job

Once the export bucket has been configured, you can start an export job. 

```
curl --user "{PUBLIC-KEY}:{PRIVATE-KEY}" --digest \
     --header "Accept: application/json" \
     --header "Content-Type: application/json" \
     --include \
     --request POST "https://cloud.mongodb.com/api/atlas/v1.0/groups/{GROUP-ID}/clusters/{CLUSTER-NAME}/backup/exports/" \
     --data '{
              "snapshotId" : "{SNAPSHOT-ID}",
                "exportBucketId" : "{BUCKET-ID}",
                "customData": [
                  {
                    "key": "exported by",
                    "value": "myName"
                  }
               ]
            }'
```

Then the job will be Queued and later on started! 

## Restoring from an S3 Snapshot

This doesn't seem to be possible from the `restoreJobs` API at the time...

```
{
    "detail": "This resource requires access through an access list of ip ranges.",
    "error": 403,
    "errorCode": "RESOURCE_REQUIRES_ACCESS_LIST",
    "parameters": [
        "100.40.199.197"
    ],
    "reason": "Forbidden"
}
```

However, you can use `mongorestore` to restore a cluster using these files. Some alternatives are: 1) Download and restore, or 2) Pipe S3 to mongoimport

*Note: You may be tempted to add your S3 bucket to an Atlas Data Lake and use `$out` to write the data to a cluster, but considering that all data sources in a Data Lake are read-only, this won't work.*

Download and mongoimport:

```
> gzip -d your_snapshot_file.json.gz
> mongoimport -d database -c collection "mongodb+srv://user:pwd@cluster_host" your_snapshot_file.json
```
OR...

Pipe S3 to mongoimport:

```
aws s3 cp s3://your_snapshot_file_s3_uri - | gzip -d - | mongoimport -d database -c collection "mongodb+srv://user:pwd@cluster_host"
```
