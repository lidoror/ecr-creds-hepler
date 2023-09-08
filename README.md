# ECR CREDENTIALS HELPER

## About
This project helps you to renew the AWS ecr certificate with a simple cronjob with ClusterRoleBinding and ClusterRole.


I have added at the end of the readme file an explanation about the cronjob (how it's working) and the permission (what permissions the cronjob has) so feel free to look in the explanation if you don't understand something.

the project came to solve a common problem on the web the renewal of the ecr token that refreshes every few hours and I did it in the simplest way no charts, no Deployments only simple cornjob with permission.

## Motives
One of the motives I had for creating this simple method is not wanting to check a suspicious helm chart or Deployment that somebody else created with a docker image that doesn't have any explanation of what running in the image or what the helm chart is creating.
The image I use here is a simple AWS image.


## How does it work?


the cronjob runs every few hours (default of 8) and authenticates with aws via access key and access key ID after the authentication the cronjob creates a secret in each namespace and we use this secret to download our images from the ecr.

the cronjob takes the access key and access key ID from another secret that we create before creating the cronjob so everything is secured We don't need to write our secrets in the cronjob as a clear text.

the whole project is deployed to the kube-system namespace if you want to change it just change the namespace section in the YAML file to your preferred namespace


## where do i use the created ecr secret?


```
apiVersion: v1 
kind: Pod
metadata:
  name: someApp
  labels:
    name: someApp
spec:
  imagePullSecrets: <CreatedSecretName> 
  containers:
  - name: someApp
    image: someImage
    resources:
      limits:
        memory: "128Mi"
        cpu: "500m"
    ports:
      - containerPort: <Port>

```

## Usage

First we create a secret and fill the account info

```
apiVersion: v1
kind: Secret
metadata:
  name: aws-conf
  namespace: kube-system
data:
  aws_access_key_id: <access key id base64>
  aws_secret_access_key: <access key  base64>
  region: <region the ecr is located base64>
  output: <the command output base64 (i use json)>
  account_id: <your account id base64>

```

then we apply the ecr cronjob with the cluster premmission.\
the yaml file have four components

1. the service account that we attach the premission to.
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ecr-registry-helper-sa
  namespace: kube-system
```

2. the cluster role with the premmission to the cluster (read premmssion to namespace and premmission to get ,delete and create secrets)

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: ecr-registry-helper-cluster-role-check
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get","delete","create"]
  - apiGroups: [ "" ]
    resources: [ "namespaces" ]
    verbs: [ "get", "list", "watch" ]
```

3. the cluster role binding that combine between the cluster role and the service account

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: ecr-registry-helper-check
subjects:
  - kind: ServiceAccount
    name: ecr-registry-helper-sa
    namespace: default
roleRef:
  kind: ClusterRole
  name: ecr-registry-helper-cluster-role-check
  apiGroup: rbac.authorization.k8s.io
```

4. and finnally the cronjob 

```
apiVersion: batch/v1
kind: CronJob
metadata:
  name: ecr-registry-helper
  namespace: kube-system
spec:
  schedule: "*/8 * * * *"
  successfulJobsHistoryLimit: 3
  suspend: false
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: ecr-registry-helper-sa
          restartPolicy: Never
          containers:
            - name: ecr-creds
              image: amazon/aws-cli:2.10.3
              imagePullPolicy: IfNotPresent
              command:
                - "/bin/sh"
                - "-c"
                - |
                  echo setting up kubectl
                  curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
                  chmod +x ./kubectl
                  
                  echo getting AWS_ACCESS_KEY_ID
                  export AWS_ACCESS_KEY_ID=$(./kubectl get secret -n default aws-conf -o jsonpath='{.data.aws_access_key_id}' | base64 -d)
                  echo getting AWS_SECRET_ACCESS_KEY
                  export AWS_SECRET_ACCESS_KEY=$(./kubectl get secret -n default aws-conf  -o jsonpath='{.data.aws_secret_access_key}' | base64 -d)
                  
                  echo "exporting AWS_DEFAULT_REGION"
                  export AWS_DEFAULT_REGION=$(./kubectl get secret -n default aws-conf  -o jsonpath='{.data.region}' | base64 -d)
                  echo "exporting AWS_DEFAULT_OUTPUT"
                  export AWS_DEFAULT_OUTPUT=$(./kubectl get secret -n default aws-conf  -o jsonpath='{.data.output}' | base64 -d)
                  
                  echo initializing variables
                  AWS_REGION="${AWS_DEFAULT_REGION}"
                  AWS_ACCOUNT=$(./kubectl get secret -n default aws-conf  -o jsonpath='{.data.account_id}' | base64 -d)
                  echo getting ecr token
                  ECR_TOKEN=$(aws ecr get-login-password --region $AWS_REGION)
                  
                  
                  for NS in $(./kubectl get ns --no-headers -o custom-columns=":metadata.name")
                  do
                  
                    echo "deleting secret in namespace: ${NS}"
                    ./kubectl delete secret --ignore-not-found ecr-docker-creds -n $NS
                  
                    echo creating secret in namespace: $NS
                    ./kubectl create secret docker-registry ecr-docker-creds \
                    --docker-server=https://${AWS_ACCOUNT}.dkr.ecr.${AWS_REGION}.amazonaws.com \
                    --docker-username=AWS \
                    --docker-password=${ECR_TOKEN} --namespace=$NS
             
                  done
                  
                  echo "done"



```

