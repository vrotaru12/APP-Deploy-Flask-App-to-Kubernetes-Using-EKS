# Deploying a Flask API

## Steps to run the API locally using the Flask server (no containerization)
The following steps describe how to run the Flask API locally with the standard Flask server, 
so that you can test endpoints before you containerize the app:
1. Install python dependencies. These dependencies are kept in a requirements.txt file. To install them, use pip:
   '''
   pip install -r requirements.txt
   '''
2. Set up the environment. You do not need to create an env_file to run locally but you do need the following two variables available in your terminal environment. The following environment variable is required:

JWT_SECRET - The secret used to make the JWT, for the purpose of this course the secret can be any string.
The following environment variable is optional:

LOG_LEVEL - The level of logging. This will default to 'INFO', but when debugging an app locally, you may want to set it to 'DEBUG'. To add these to your terminal environment, run the 2 lines below.

'''
export JWT_SECRET='myjwtsecret'
export LOG_LEVEL=DEBUG
'''
3. Run the app using the Flask server, from the top directory, run:
'''
 python main.py
'''

4. To try the API endpoints, open a new shell and run the following commands, replacing `EMAIL` and `PASSWORD` with any values:

To try the /auth endpoint, use the following command:

'''
export TOKEN=`curl -d '{"email":"<EMAIL>","password":"<PASSWORD>"}' -H "Content-Type: application/json" -X POST localhost:8080/auth  | jq -r '.token'`
'''

This calls the endpoint 'localhost:8080/auth' with the {"email":"<EMAIL>","password":"<PASSWORD>"} as the message body. The return value is a JWT token based on the secret you supplied. We are assigning that secret to the environment variable 'TOKEN'. To see the JWT token, run:

'''
echo $TOKEN
'''

To try the /contents endpoint which decrypts the token and returns its content, run:
'''
curl --request GET 'http://127.0.0.1:8080/contents' -H "Authorization: Bearer ${TOKEN}" | jq .
'''
You should see the email that you passed in as one of the values.





# Containerize the Flask App and Run Locally

## The following steps describe how to complete the Dockerization part of the project. After you complete these steps, you should have the Flask application up and running in a Docker container.

1. If you haven't installed Docker already, you should install now using these installation instructions.

2. Create a Dockerfile named Dockerfile in the app repo. Your Dockerfile should:

Use the python:stretch image as a source image
Set up an app directory for your code
Install needed Python requirements
Define an entrypoint which will run the main app using the Gunicorn WSGI server
Gunicorn should be run with the arguments as follows: `gunicorn -b :8080 main:APP`.
```
FROM python:stretch

COPY . /app
WORKDIR /app

RUN pip install --upgrade pip
RUN pip install -r requirements.txt

EXPOSE 8080


ENTRYPOINT ["gunicorn", "-b", ":8080", "-w","3", "main:APP"]
```

3. Create a file named `env_file` and use it to set the environment variables which will be run locally in your container. Here, we do not need the export command, 
just an equals sign:

  `<VARIABLE_NAME>=<VARIABLE_VALUE>`
In this file, you should set both JWT_SECRET and LOG_LEVEL, similar to how they were set as environment variables when you ran the Flask app locally.

Add the env setup to the `buildspec.yml`:
```
env:
- name: JWT_SECRET
value: JWT_SECRET_VALUE
```
In this file we need to add as well the following lines in `pre-buid` phase section:
```
- python -m pip install --upgrade pip
- python -m pip install -r requirements.txt
- python -m pytest test_main.py
```

4. Build a local Docker image with the tag `jwt-api-test`.
```
docker build --tag jwt-api-test .
```

5. Run the image locally, using the Gunicorn server.

   You can pass the name of the env file using the flag --env-file=<YOUR_ENV_FILENAME>.
   You should expose the port 8080 of the container to the port 80 on your host machine.
```
docker run -p 80:8080 --env-file=env_file jwt-api-test
```
   Note: update Docker file correctly to have requirements.txt included and the correct port

6. To use the endpoints, you can use the same curl commands as before, except using port 80 this time:

To try the /auth endpoint, use the following command:
'''
export TOKEN=`curl -d '{"email":"test@test.com","password":"test"}' -H "Content-Type: application/json" -X POST localhost:80/auth  | jq -r '.token'`
'''
To try the /contents endpoint which decrypts the token and returns its content, run:
'''
curl --request GET 'http://127.0.0.1:80/contents' -H "Authorization: Bearer ${TOKEN}" | jq .
'''
You should see the email that you passed in as one of the values.



# Deployment to Kubernetes

## Create an EKS Cluster and IAM Role
Before you can deploy your application, you will need to create an EKS cluster and set up an IAM role that CodeBuild can use to interact with EKS. You can follow the steps below to do this from the command line.

### Create a Kubernetes (EKS) Cluster
1. Create an EKS cluster named `simple-jwt-api`.
Run the following commands:
```
ecsctl create cluster --name simple-jwt-api
```
Then check the progress in aws console by selecting the correct region.
Once the status is 'CREATE_COMPLETE', check the health of your nodes:
```
kubectl get nodes
```
### Set Up an IAM Role for the Cluster
The next steps are provided to quickly set up an IAM role for your cluster.
1. Create an IAM role that CodeBuild can use to interact with EKS:
   Set an environment variable `ACCOUNT_ID` to the value of your AWS account id. You can do this with awscli:

   '''
   `ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)`
   '''
   Create a role policy document that allows the actions "eks:Describe*" and "ssm:GetParameters". 
   You can do this by setting an environment variable with the role policy:
   '''
   TRUST="{ \"Version\": \"2012-10-17\", \"Statement\": [ { \"Effect\": \"Allow\", \"Principal\": { \"AWS\": \"arn:aws:iam::${ACCOUNT_ID}:root\" }, \"Action\": \"sts:AssumeRole\" } ] }"
   '''
   Create a role named 'UdacityFlaskDeployCBKubectlRole' using the role policy document:
   '''
   aws iam create-role --role-name UdacityFlaskDeployCBKubectlRole --assume-role-policy-document "$TRUST" --output text --query 'Role.Arn'
   ''
   Create `tmp` directory and add in a role policy document ,called `iam-role-policy.file` that also allows the actions "eks:Describe*" and "ssm:GetParameters". You can create the document in your tmp directory:
   '''
   echo '{ "Version": "2012-10-17", "Statement": [ { "Effect": "Allow", "Action": [ "eks:Describe*", "ssm:GetParameters" ], "Resource": "*" } ] }' > ./iam-role-policy   
   '''
   Attach the policy to the `UdacityFlaskDeployCBKubectlRole`. You can do this using awscli:

   ```
   aws iam put-role-policy --role-name UdacityFlaskDeployCBKubectlRole --policy-name eks-describe --policy-document file://./iam-role-policy
   ```

   You have now created a role named `UdacityFlaskDeployCBKubectlRole`. 
   There are some permitions you need to add as well manually in order to be able to create a pipeline in the next steps:
   `AmazonEKSClusterPolicy`
   `AdministarorAccess`
   `AmazonEKSWorkerNodePolicy`
   `AmazonEKSServicePolicy`
   `AmazonEKS_CNI_Policy`




Before doing th next step check if your kubectl is properly installed. You have a kubectl.exe in kube folder on disk C. 
You should have a config file as well in this directory. If you do not have then run the following commands to generate it :
```
aws eks --region <region-code> update-kubeconfig --name <cluster_name>
kubectl get svc 
```

the file now is generated and you won't have any problems


2. Grant the role access to the cluster. The `aws-auth ConfigMap` is used to grant role based access control to your cluster.
   ```
   kubectl get -n kube-system configmap/aws-auth -o yaml > ./aws-auth-patch.yml
   ```
   this command will generate a file called ` aws-auth-patch.yml`

3. Configure the `aws-auth-patch.yml` file with the new role
Here we will actually declare the new Role and patch the configuration back to our account. This step will in fact enable the UdacityFlaskDeployCBKubectlRole to perform the operations as expected.

the missing piece here is the followig config for role we created:
```
  - groups:
      - system:masters
      rolearn: arn:aws:iam::<YOUR_ACCOUNT_ID>:role/UdacityFlaskDeployCBKubectlRole
      username: build
```
4. Patch the modified aws-auth-patch.yml
```
kubectl patch configmap/aws-auth -n kube-system --patch "$(cat ./aws-auth-patch.yml)"
```




# Deployment to Kubernetes using CodePipeline and CodeBuild
You should now be ready to deploy your application using CodePipeline and CodeBuild. Follow the steps below to complete your project.
1. Create repository on Github and add the project over there 
2.  Fill the `ci-cd-codepipeline.cfn.yml` file:
fill the `Default` key with your own information:

--> Your EksClusterName
--> Your GitSourceRepo
--> Your GitBranch
--> Your GitHubUser
--> Your KubectlRoleName

3. Create the `CloudFormation` stack on aws.amazon.com:
Select the correct location
 ---> go to https://eu-west-2.console.aws.amazon.com/cloudformation/home?region=eu-west-2#/
 ---> Go to https://github.com/settings/tokens/ and generate a token on GitHub and save it because you will use it in stack creation.
      (Ensure you select `repo` rights before generating token )  You should generate the token with full control of private repositories.
 ---> Create stack
 ---> Select 'Upload a template file'
 ---> Choose `ci-cd-codepipeline.cfn.yml` that you updated in step 2 on your proj(on computer). This will fill with into next step
 ---> Choose a name for this stack `simple-jwt-api-stack-test`
 ---> Create

 Once the status of the stack is `CREATE_COMPLETE` then you can check the pipeline here:
 https://eu-west-2.console.aws.amazon.com/codesuite/codepipeline/pipelines/simplee-jwt-api-stack-test-CodePipelineGitHub-JO4YL9YA45DT/view?region=eu-west-2
 and you shoul see created  `simple-jwt-api-stack-test-CodePipelineGitHub-JO4YL9YA45DT` pipeline
 
 4. Grab the EKS Cluster endpoint URL
```
kubectl get services simple-jwt-api -o wide
```
get the `External-ip` of your simplet-jwt service and use it in the following step to test your endponts:

```
export URL="a403fa53a96514f12948dd65f87550c1-1498351496.us-east-2.elb.amazonaws.com"
export TOKEN=`curl -d '{"email":"test@test.com","password":"test"}' -H "Content-Type: application/json" -X POST $URL/auth  | jq -r '.token'`
curl --request GET $URL:80/contents -H "Authorization: Bearer ${TOKEN}" | jq
```

### Dynamically loading the secret through buildspec.yml

What we actually want to do here is to replace the `JWT_SECRET_VALUE` that we declared above with the value
 we loaded from AWS SSM
 
During [`pre-build` phase](https://github.com/jungleBadger/FSND-Deploy-Flask-App-to-Kubernetes-Using-EKS/blob/2bff3c7387f04773fd591e1d0193ef4ac6b92f74/buildspec.yml#L11) add the following snippet

```yaml
 - sed -i 's@JWT_SECRET_VALUE@'"$JWT_SECRET"'@' simple-jwt-api.yml
```

## Troubleshooting

in case of any errors check the pods of kubectl:
```
kubectl get pods

kubectl describe pod <pod>
```

## Testing
You can simply list your pods, attach a shell and validate if the secret is loaded in there.
```
kubectl get pods
kubectl exec -it <POD_ID> /bin/bash
echo $JWT_SECRET
```
