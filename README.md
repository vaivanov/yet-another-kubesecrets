# Yet Another Kube Secrets
#### CLI tool for managing opaque secrets in Kubernetes clusters as dotenv files based on kubectl

### Features

- Pulling secrets from cluster to dotenv file (in key=value format)
- Pushing a couple of values from dotenv file to k8s cluster
- CLI for CRUD values in secret

##### Problem
Default kubectl scripts for managing opaque secrets  allow just create or delete variables. Its not so easy to modify existing secret - we need to get secret from cluster, decode it from base64, add new variable, encode it and create secret again with new values. Of course we can just encode new variable and add it to existing json (or yaml) secret schema, but it so... opaque! Solutions like "vault" is overkill for small projects.

##### Solution
Creating script that allow sync values between dotenv file and k8s secret

### Requirements

* Install [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
for MacOS
```
brew install kubectl
```
* Install [jq](https://stedolan.github.io/jq/download/)
for MacOS
```
brew install jq
```

### Usage

`./kubesecrets ACTION SECRET [OPTIONS]`
                    
Argument  | Description
------------- | -------------
ACTION  | "create", "decode", "modify-key", "delete-key", "delete", "pull" or "push"
SECRET  | The secret to use
[-n=namespace] | The namespace to retrieve the secret from (optional, defaults to the namespace of the current context, KUBENS or 'default').
[-k=key] | key of variable (required for "modify-key" and "delete-key" actions)
[-v=value] | value of variable (required for "modify-key" action)
[-f=file] | path of dotenv file (required for "create", "pull" and "push" actions)
[-b] | backup current secret to temp file before actions (optional, defaults to false)
[--verbose] | verbose mode (optional, defaults to true)

#### Creating a secret from file
1) create **.env** file with variables, i.e.

```
DEBUG=true
ACCESS_KEY=;KJf:84oB2/.25d5guazZ)ct
```
2) run the create method

```
./kubesecrets create app-secrets -n=testing -f=.env
```
Output:
```
> checking that namespace exists
> "testing" founded
> checking arg file (-f): .env
> checking that secret exists in namespace
> "app-secrets" doesn't exist
> creating secret "app-secrets" in namespace "testing"
secret "app-secrets" created
> done
```

this method just a syntactic sugar for `kubectl create secret generic ... --from-env-file=.env

#### Decoding an existing secret
```
./kubesecrets decode app-secrets -n=testing
```
Output:
```
> checking that namespace exists
> "testing" founded
> checking that secret exists in namespace
> "app-secrets" founded
> getting secret "app-secrets" from namespace "testing" and decoding it
ACCESS_KEY=;KJf:84oB2/.25d5guazZ)ct
DEBUG=true
> done
```

#### Pulling a secret
Getting secret from k8s, decode it and merge with existing dotenv file (or create new file)
1) in example, create .env file
```
ACCESS_KEY=;KJf:84oB2/.25d5guazZ)ct
DEBUG=false
FRONTEND_URL=https://google.com
```
2) run
```
./kubesecrets pull app-secrets -n=testing -f=.env
```
script will get existing variables in secret
```
ACCESS_KEY=;KJf:84oB2/.25d5guazZ)ct
DEBUG=true
```
and merge it with existing file
result .env
```
ACCESS_KEY=;KJf:84oB2/.25d5guazZ)ct
DEBUG=true
FRONTEND_URL=https://google.com
```
DEBUG had overrided to true, new variable (FRONTEND_URL) didn't lost. Variables in k8s didn't change.

#### Pushing a secret
Taking variables from dotenv file and save it to k8s secret
1) in example, create .env file
```
FRONTEND_URL=https://google.com
BACKEND_URL=https://api.google.com
```
2) run
```
./kubesecrets push app-secrets -n=testing -f=.env
```
script will get existing variables in secret
```
ACCESS_KEY=;KJf:84oB2/.25d5guazZ)ct
DEBUG=true
```
and merge it with new variables in file. You can check changes with decode action.
```
./kubesecrets decode app-secrets -n=testing
```
Output:
```
ACCESS_KEY=;KJf:84oB2/.25d5guazZ)ct
BACKEND_URL=https://api.newgoogle.com
DEBUG=false
FRONTEND_URL=https://google.com
```

#### Deleting a secret
```
./kubesecrets delete app-secrets -n=testing
```
this method just a syntactic sugar for `kubectl delete secret ...` with confirmation dialog. Also you can use backup option for save temporary file

./kubesecrets delete app-secrets -n=testing

#### Deleting variable in a secret
```
./kubesecrets delete-key app-secrets -n=testing -k=BACKEND_URL
```
You can check changes with decode action.
Output after delete BACKEND_URL key execution:
```
ACCESS_KEY=;KJf:84oB2/.25d5guazZ)ct
DEBUG=false
FRONTEND_URL=https://google.com
```

#### Modifying variable in a secret

```bash
./kubesecrets modify-key app-secrets -n=testing -k=INTERNAL_URL -v="https://amazon.com"
```
You can check changes with decode action.
Output after modify INTERNAL_URL key execution:
```
ACCESS_KEY=;KJf:84oB2/.25d5guazZ)ct
DEBUG=false
FRONTEND_URL=https://google.com
INTERNAL_URL=https://amazon.com
```

### How to use variables in a secret as environments variables?
You must provide name of secret as **secretRef** variable in you deployment config. i.e.
```
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  name: django
  namespace: testing
  labels:
    deployment: django
spec:
  replicas: 1
  selector:
    matchLabels:
      pod: django
  template:
    metadata:
      labels:
        pod: django
    spec:
      containers:
        - name: django
          image: python:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 8000
          envFrom:
            - secretRef:
                name: app-secrets
```
[more examples (gist)](https://gist.github.com/troyharvey/4506472732157221e04c6b15e3b3f094 "learn")
