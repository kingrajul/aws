# ML Threat Model 
<--Second DRAFT May 30, 2020-->

Our goal is to understand the security threats to the AWS ML/AI components and recommend countermeasures. We will understand the process, identify the data elements and components involved in each process, and evaluate the threats scenarios based on STRIDE methodology. We will analyze the available countermeasures offered by Amazon and come up with a set of requirements to apply these countermeasure. The analysis is generic for any organization with PII and PCI data.

## High-level flow 
Machine learning is about teaching a computer to make predictions or inferences. 

It is a three-step process:

**Step 1**: Generate example data: You need to fetch the data set, clean it up, and prepare or transform it so it can be used. In Amazon SageMaker, you preprocess example data in Jupyter notebook. You use a notebook to fetch your data, explore it, prepare it for model training.  Typically, you import the data with Pandas as DataFrame, explore it, clean it, and export it as NumPy array to be ready to be fed to an ML algorithm. 

Components involved: 
1. The Dataset which could be located in an S3 bucket 
2. Jupyter notebook instance running on an EC2 instance. 
3. Attached EBS volume to the EC2 instance 

**Step 2**: Train the Model: You need an algorithm and compute resources for training. The process is simple, you create a training job, SageMaker launches the ML compute instances, and uses the training code and the training dataset to train the model. It saves the resulting model artifacts and other output in the S3 bucket you specified for that purpose. 
When you create a training job, SageMaker replicates the entire dataset on ML compute instance by default. 
When you run a model training job, the SageMaker container uses the `/opt/ml/input` directory, which contains the JSON file that configures the hyper-parameters for the algorithm and the network layout used for the distributed training. The `/opt/ml/input` directory also contains files that specify the channels through which SageMaker accesses the S3 data. 
When a container is spun, we assume (to be validated) that it runs on a dedicated EC2 instance that is not shared with other AWS clients. To ensure this being the case, we want to ensure that it runs from within our own VPC. 

Components involved: 
1. EC2 container registry - AWS ECR that contains Docker Container Image
2. Container and respective EC2 instance that runs it. We assume it will run from within designated VPC
3. Amazon SageMaker - Jupyter notebook instance running on an EC2 instance 
4. Attached EBS volume to the EC2 instance 
5. Training dataset in S3 Bucket - URL is required by training job
6. Trained model - output of the training - URL is required by the training job 
   

**Step 3**: Deploy the model: After you train your model, you can deploy it to get prediction, either in batch form or online deployment.  Model files are update at `/opt/model`
The entire directory structure is as follows: 

```
	/opt/ml
	├── input
	│   ├── config
	│   │   ├── hyperparameters.json
	│   │   └── resourceConfig.json
	│   └── data
	│       └── <channel_name>
	│           └── <input data>
	├── model
	│
	├── code
	│
	├── output
	│
	└── failure
```

Components involved: 
1. Model - its S3 path where model artifacts are stored 
2. Docker registry path for Images that contains the inference code. 
3. Docker image 

SageMaker requires that Docker containers run without privileged access. 
Docker containers send messages to `Stdout` and `Stderr`  files. Amazon SageMaker sends these messages to CloudWatch Logs in your account. 

## Security Threat Analysis 
For this exercise, the threats are identified by using STRIDE - Spoofing, Tempering, Repudiation, Information Disclosure, Denial-of-Service, Elevation of privilege. Each of the threat impacts an information security property. 

* Spoofing - Authentication - Impersonating something or someone else. 
* Tampering - Integrity - Modifying data or code. 
* Repudiation - Non-repudiation - Claiming to have not performed an action 
* Information Disclosure - Confidentiality - Exposing information to someone not authorized to see it. 
* Denial of Service - Availability - Deny or degrade service to users 
* Elevation of Privilege - Authorization - Gain capabilities without proper authorization. 

**Applying STRIDE**  

STRIDE can be applied either Per Element or per Interaction - In this case, we will apply it on per Element basis. 

## Assets to protect 
There are five assets to protect.

|Tag|Element|Description|
|---|---|---|
|A01| Training Data |At rest in S3 and Container directory Structure In Transit S3> Model training Container|
|A02| Trained Models |At rest in S3 and in transit S3>Deployed docker container|
|A03| Docker Image |At rest in ECR|
|A04| Notebook instance |EBS volume attached to EC2 instance|
|A05| Python program  |PY file created by a Data Scientist goes a code repository, it has been excluded from the model|


Threat Table (STRIDE)

1. Training data and model (A01, A02)  Located in S3 bucket 
	   1. Unauthorized write access to the training data. 
	   2. Unauthorized modifications without proper logging.  
	   3. Unauthorized read access to confidential data. 
	   4. Unauthorized deletion of objects.
	   5. An instance spun with a role that has excessive access to the data. 
	   6. Direct internet access to the S3 data, public access goes undetected. 
2. Training data and model (A01, A02)  Transfer between Container and S3, SageMaker EC2 and S3 and Inter-container communication 
	   1. Eavesdropping and tampering if the data in communication is not protected using TLS.
3. Container images with directory structure containing data (A03) - Located in ECR
	   1. A user with access to the ECR can access the custom images created for a particular training 
4. EC2 running Container (A04)
	   1. Unencrypted EC2 EBS volume can be mapped to another machine causing sensitive data loss


## Available Controls 
|Tag|Element|Description|
|---|---|---|
|C01 |IAM Policy|IAM policy is used to grant permission to the roles defined. These roles are used to run the EC2 instances and access and encrypt and decrypt the data from S3|
|C02 |MFA |Used for admin accounts, not particularly used in this use case|
|C03 |Transport Layer Security |Control to ensure data in transit is protected.|
|C04 |Logging |Control to ensure that |
|C05 |Encryption SSE-C| Encryption applied on S3 buckets to ensure data at rest encryption|
|C06 |Client-Side Encryption |Encrypt data elements prior to moving to S3, this control is not applied in this threat model|
|C07 |Restricted Internet Access |The notebook EC2 instances and the containers running on EC2 run from within the VPC that does not have any internet access.|
|C08 |Mutual TLS |Inter-container communication|
|C09 |AWS Config |To ensure that the planned configuration is applied properly|
|C10 |AWS Account |Isolated account to ensure no interference from other accounts|


## Security Considerations 
1. Accounts (C10) - SageMaker should be run on its own account. 
2. IAM (C01)- Use Roles - There must be only two roles required. 
   * 1st role is created for use by Service Catalog to provision Sagemaker Notebook, attach a pre-build lifecycle config to it, and also attach a specific IAM role to the notebook instance (so that way users can't manipulate the role that gets passed to the notebook and the containers).
   * 2nd role is a limited privilege which allows users to login to Sagemaker console and launch the notebook server created for them (start, stop, launch, etc.). You will have to define the policy properly here so they can launch their own notebook server and not someone else's. Once inside the notebook servers, they can pass the IAM role assigned to the Notebook Server to the training jobs they launch from a notebook and all other jobs. Again make sure through policy as well that this is the only role they are able to pass. 
3. MFA (C02) - Ensure that the root account has MFA enabled 
4. Transport layer Security (C03, C08)
   * All traffic should use TLS 
   * Inter-node training commmunication must be encrypted using two way SSL communication 
5. Logging (C04) - Ensure that the logging is enabled in CloudTrail and monitoring is performed by CloudWatch. 
   * At least the following events must be logged 
     * invocation and latency of endpoints 
     * The Health of instance nodes 
     * All user actions, all role actions and all service actions within SageMaker 
     * Log files moved to enterprise logging solution 
6. Data at Rest Encryption (C05) 
   * Running EC2 instances 
     * Use AWS Key Management Service KMS Customer provided CMK 
     * KMS is accepted by all notebooks and all sagemaker jobs e.g. 
     * Training, Tuning, batch transf0rm, endpoint 
     * Directory structure all under `/opt/ml/` and `/tmp` must be encrypted 
   * S3 
     * Create a custom policy for assignment and use bucket policies to primarily enforce the best practice of minimum privilege. Bucket policies can have `deny` statements. 
     * Ensure that the IAM policy on the two roles to grant access. These roles are assigned to EC2 instances from where S3 will be accessed. 
     * Use an encrypted bucket for training data and hosting models (SSE-C)
     * Use AWS KMS with the following conditions: 
         * IP Restriction - Specified IP 
         * A principal with specific tag/specified value (used for principles running outside of the VPC, such as out of VPC lambda function)
         * Specific VPC endpoint 
         * Specific VPCs 
	Example policy 
		```
		"Condition": {
			"NotIpAddress":{
				"aws:SourceIp":[
					"Your Organization's proxy IP"
				]
			}` 


		"Condition": {
			"NotIpAddress":{
				"aws:SourceIp":[
					"Your Organization's proxy IP"
				]
			}
			"StringNotEquals":{
				"aws:PrincipalTag/App_Grp":[
					"This can be specific application tag",
					"This can be lambda tag"
				]
			}
			"aws:sourceVpc":[
				"VPC ARN-1"
			]
			"aws:sourceVpc":[
				"VPC ARN-2"
			]

		}
		```
7. Key ownership (C05) - The key ownership should be with the account that SageMaker is running. 
8. Running SageMaker in the VPC - By default, all the SageMaker notebooks have internet access. 
   * Use private VPC for security 
   * SageMaker needs access to training data and trained model hosting, so ensure that access through VPC endpoints or through private link. 
9. Running Training jobs (C07)
   * Training and Inference containers are Internet-enabled by default 
   * Ensure that the containers are run in a private VPC 
   * SageMaker needs access to training data and trained model 
10. Protecting the PII (C06)
   * Identify any PII like SSN and PAN and tokenize it during the ETL process prior to moving to the AWS cloud. 
  
