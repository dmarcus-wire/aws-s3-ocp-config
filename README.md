# aws-s3-ocp-config
Process for configuring AWS S3 for OpenShift connection

1. Administrator Creates Secret
2. Administrator Creates StorageClass
3. User Creates ObjectBucketClaim

# Administrator Creates Secret
This secret will contain the elevated/admin privileges needed by the provisioner to properly access and create S3 Buckets and IAM users and policies. The AWS Access ID and AWS Secret Key will be needed for this.
Create the Kubernetes Secret for the Provisioner's Owner Access.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: s3-bucket-owner [1]
  namespace: s3-provisioner [2]
type: Opaque
data:
  AWS_ACCESS_KEY_ID: *base64 encoded value* [3]
  AWS_SECRET_ACCESS_KEY: *base64 encoded value* [4]
```
1. Name of the secret, this will be referenced in StorageClass.
2. Namespace where the Secret will exist.
3. Your AWSACCESSKEY_ID base64 encoded.
4. Your AWSSECRETACCESS_KEY base64 encoded.

```shell
# kubectl create -f creds.yaml
  secret/s3-bucket-owner created
```

# Administrator Creates StorageClass
The StorageClass defines the name of the provisioner and holds other properties that are needed to provision a new bucket, including the Owner Secret and Namespace, and the AWS Region.

## Greenfield Example:
For Greenfield, a new, dynamic bucket will be generated.

```yaml
Create the Kubernetes StorageClass for the Provisioner.
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: s3-buckets [1]
provisioner: aws-s3.io/bucket [2]
parameters:
  region: us-west-1 [3]
  secretName: s3-bucket-owner [4]
  secretNamespace: s3-provisioner [5]
reclaimPolicy: Delete [6]
```
1. Name of the StorageClass, this will be referenced in the User ObjectBucketClaim.
2. Provisioner name
3. AWS Region that the StorageClass will serve
4. Name of the bucket owner Secret created above
5. Namespace where the Secret will exist
6. reclaimPolicy (Delete or Retain) indicates if the bucket can be deleted when the OBC is deleted. NOTE: the absence of the bucketName Parameter key in the storage class indicates this is a new bucket and its name is based on the bucket name fields in the OBC.

```shell
# kubectl create -f storageclass-greenfield.yaml
  storageclass.storage.k8s.io/s3-buckets created
```

## Brownfield Example:
For brownfield, the StorageClass defines the name of the provisioner and the name of the existing bucket. It also includes other properties needed by the target provisioner, including: the Owner Secret and Namespace, and the AWS Region

Create the Kubernetes StorageClass for the Provisioner.
```yaml 
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: s3-existing-buckets [1]
provisioner: aws-s3.io/bucket [2]
parameters:
  bucketName: my-existing-bucket [3]
  region: us-west-1 [4]
  secretName: s3-bucket-owner [5]
  secretNamespace: s3-provisioner [6]
```

1. Name of the StorageClass, this will be referenced in the User ObjectBucketClaim.
2. Provisioner name
3. Name of the existing bucket
4. AWS Region that the StorageClass will serve
5. Name of the bucket owner Secret created above
6. Namespace for that bucket owner secret 

NOTE: the storage class's reclaimPolicy is ignored for existing buckets.

```shell
# kubectl create -f storageclass-brownfield.yaml
  storageclass.storage.k8s.io/s3-buckets created
```

# User Creates ObjectBucketClaim
An ObjectBucketClaim follows the same concept as a PVC, in that it is a request for Object Storage, the user doesn't need to concern him/herself with the underlying storage, just that they need access to it. The user will work with the cluster/storage administrator to get the proper StorageClass needed and will then request access via the OBC.

```yaml
Greenfield Request Example:
Create the ObjectBucketClaim.
apiVersion: objectbucket.io/v1alpha1
kind: ObjectBucketClaim
metadata:
  name: myobc [1]
  namespace: s3-provisioner [2]
spec:
  generateBucketName: mybucket [3]
  bucketName: my-awesome-bucket [4]
  storageClassName: s3-buckets [5]
```
1. Name of the OBC
2. Namespace of the OBC
3. Name prepended to a random string used to generate a bucket name. It is ignored if bucketName is defined
4. Name of new bucket which must be unique across all AWS regions, otherwise an error occurs when creating the bucket. If present, this name overrides generateName
5. StorageClass name NOTE: if both generateBucketName and bucketName are omitted, and the storage class does not define a bucket name, then a new, random bucket name is generated with no prefix.

```shell
# kubectl create -f obc-brownfield.yaml
  objectbucketclaim.objectbucket.io/myobc created
```

## Brownfield Request Example:

Create the ObjectBucketClaim.
```yaml
apiVersion: objectbucket.io/v1alpha1
kind: ObjectBucketClaim
metadata:
  name: myobc [1]
  namespace: s3-provisioner [2]
spec:
  storageClassName: s3-existing-buckets [3]
```
  
1. Name of the OBC
2. Namespace of the OBC
3. StorageClass name 

NOTE: in the OBC here there is no reference to the bucket's name. This is defined in the storage class and is not a concern of the user creating the claim to this bucket. An OBC does have fields for defining a bucket name for greenfield use only.

```shell
# kubectl create -f obc-brownfield.yaml
  objectbucketclaim.objectbucket.io/myobc created
```

# Results and Recap
Let's pause for a moment and digest what just happened. 
After creating the OBC, and assuming the S3 provisioner is running, we now have the following Kubernetes resources:
1. a global ObjectBucket (OB) which contains: bucket endpoint info (including region and bucket name)
2. a reference to the OBC, and a reference to the storage class
3. Unique to S3, the OB also contains the bucket Amazon Resource Name (ARN)
   1. Note: there is always a 1:1 relationship between an OBC and an OB. 
4. a ConfigMap in the same namespace as the OBC, which contains the same endpoint data found in the OB. 
5. a Secret in the same namespace as the OBC, which contains the AWS key-pairs needed to access the bucket. 
6. And of course, we have a new AWS S3 Bucket which you should be able to see via the AWS Console. ObjectBucket

```shell
# kubectl get ob obc-s3-provisioner-my-awesome-bucket -o yaml
# kubectl get cm myobc -n s3-provisioner -o yaml
# kubectl get secret my-awesome-bucket -n s3-provisioner -o yaml
```

Create a Sample Pod to Access the Bucket.
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: photo1
  labels:
    name: photo1
spec:
  containers:
  - name: photo1
    image: docker.io/screeley44/photo-gallery:latest
    imagePullPolicy: Always
    envFrom:
    - configMapRef:
        name: my-awesome-bucket <1>
    - secretRef:
        name: my-awesome-bucket <2>
    ports:
    - containerPort: 3000
      protocol: TCP
```
1. Name of the generated configmap from the provisioning process
2. Name of the generated secret from the provisioning process 
[Note] Generated ConfigMap and Secret are same name as the OBC!

Lastly, expose the pod as a service so you can access the url from a browser. In this example, I exposed as a LoadBalancer
`# kubectl expose pod photo1 --type=LoadBalancer --name=photo1 -n your-namespace`

  # kubectl get svc photo1
  NAME                         TYPE           CLUSTER-IP       EXTERNAL-IP                                                               PORT(S)          AGE
  photo1                       LoadBalancer   100.66.124.105   a00c53ccb3c5411e9b6550a7c0e50a2a-2010797808.us-east-1.elb.amazonaws.com   3000:32344/TCP   6d
NOTE: This is just one example of a Pod that can utilize the bucket information, there are several ways that these pod applications can be developed and therefore the method of getting the actual values needed from the Secrets and ConfigMaps will vary greatly, but the idea remains the same, that the pod consumes the generated ConfigMap and Secret created by the provisioner.