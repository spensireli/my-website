<ul>
<a href="url"><img src="/media/flux/flux-horizontal-color.png" align="left" height="180" width="600" ></a>
<a href="url"><img src="/media/flux/code-commit.png" align="left" height="200" width="600" ></a>
</ul>


# Flux2 and AWS Code Commit
This blog will walk through how to configure an AWS Code Commit repository to use Flux2 as a continuious deployment GitOps solution. The same steps can be replicated for any variant of Git, such as GitHub or GitLab. 

### What is Flux?
Flux is a Cloud Native Computing Foundation (CNCF) incubation project focused on GitOps for Kubernetes. GitOps is a way for engineers to manage applications in a declarative state within the source code control system Git and automatically apply changes to their environments when changes are pushed to the codebase. 

For more information on what it is GitOps is I suggest checking out:

- https://www.weave.works/technologies/gitops/
- https://www.cloudbees.com/gitops/what-is-gitops

### Why Flux?
There are many GitOps solutions out there and it has quickly become a buzzword. Though from my perspective one of the upcoming leaders is Flux. It’s simple, Flux makes the configuration and deployment easy to use. It’s OpenSource and there are no-strings attached, you point it to a Git project and Flux will continue to poll for changes of the Git project and apply them when they occur. 

Flux was specifically designed for Kubernetes and offers out of the box integrations to a number of tools such as Helm, Kustomize, and Harbor. I have found Flux to be a flexible GitOps solution for managing multiple Kubernetes environments. If you are managing more than one Kubernetes cluster Flux can offer a way to simplify cluster management. 

## Assumptions
To perform the steps mentioned in this blog the following assumptions are made. 

- You have an AWS account.
- You have access to AWS Code Commit.
- You have a Kubernetes cluster. 
- Your AWS CLI Profile is configured with the account. 
- Flux CLI has been installed [here](https://fluxcd.io/docs/installation/)


## Creating CodeCommit Git Project

Before we can do anything we must first have a place to store our deployment files and a user that can access the git repository.

Lets create a repository, I will name this `fluxexample`.
```bash
[spensireli ~]$ aws codecommit create-repository --repository-name "fluxexample" --repository-description "example of gitops with flux"
{
    "repositoryMetadata": {
        "accountId": "<REDACTED>",
        "repositoryId": "<REDACTED>",
        "repositoryName": "fluxexample",
        "repositoryDescription": "example of gitops with flux",
        "lastModifiedDate": "2022-01-02T15:39:00.536000-05:00",
        "creationDate": "2022-01-02T15:39:00.536000-05:00",
        "cloneUrlHttp": "https://git-codecommit.us-east-1.amazonaws.com/v1/repos/fluxexample",
        "cloneUrlSsh": "ssh://git-codecommit.us-east-1.amazonaws.com/v1/repos/fluxexample",
        "Arn": "arn:aws:codecommit:us-east-1:<REDACTED>:fluxexample"
    }
}
```

Next I will create a dedicated service account I will use to access the code repository from the Kubernetes cluster.
```bash
[spensireli ~]$ aws iam create-user --user-name fluxserviceaccount
{
    "User": {
        "Path": "/",
        "UserName": "fluxserviceaccount",
        "UserId": "USER_ID",
        "Arn": "arn:aws:iam::ACCOUNT_ID:user/fluxserviceaccount",
        "CreateDate": "2022-01-02T20:44:12+00:00"
    }
}
```

We must give this service account permission to AWS CodeCommit so I will be using the managed AWS policy "AWSCodeCommitFullAccess"
```bash
[spensireli ~]$ aws iam attach-user-policy --policy-arn arn:aws:iam::aws:policy/AWSCodeCommitFullAccess --user-name fluxserviceaccount
```

Once the service account has full access we must create service specific credentials to pull and push to the code repository. After running this command copy and note the username and password, they will be used again.
```bash
[spensireli ~]$ aws iam create-service-specific-credential --user-name fluxserviceaccount --service-name codecommit.amazonaws.com
{
    "ServiceSpecificCredential": {
        "CreateDate": "2022-01-02T20:50:06+00:00",
        "ServiceName": "codecommit.amazonaws.com",
        "ServiceUserName": "<REDACTED>",
        "ServicePassword": "<REDACTED>",
        "ServiceSpecificCredentialId": "<REDACTED>",
        "UserName": "fluxserviceaccount",
        "Status": "Active"
    }
}
```

Now we can clone the repository. If your normal user account does not have permission to access code commit with configured ssh keys, you may use the service account in this example. Please note that this is not best practice, and it is recommended to only use the service account for deployment in Kubernetes or your pipeline. 

```bash
[spensireli repos]$ git clone https://git-codecommit.us-east-1.amazonaws.com/v1/repos/fluxexample
Cloning into 'fluxexample'...
Username for 'https://git-codecommit.us-east-1.amazonaws.com': 
Password for '<REDACTED>
warning: You appear to have cloned an empty repository.
```



## Creating the Deployment Files
In the cloned code repository create two directories.
```bash
mkdir namespaces
mkdir workloads
```

We will store our Kubernetes definitions for namespace creation in `namespaces` and the application deployments in `workloads`.

In the namespace directory create the namespace definition.

`example-ns.yaml`:
```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: flux-example

```

Now in the `workloads` directory create the application definition.
`example-workload.yaml`:
```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flux-example-deployment
  namespace: flux-example
spec:
  selector:
    matchLabels:
      app: flux-example
  replicas: 1
  template:
    metadata:
      labels:
        app: flux-example
    spec:
      containers:
        - name: flux-example
          image: nginxdemos/hello:latest
          ports:
            - containerPort: 443
```

Commit the files that you created to your code repository. 

## Bootstrapping Flux
We can now install Flux into our Kubernetes cluster and sync the files. To do this we must boostrap Flux. The commands for this vary based on Git being used; i.e. GitHub and GitLab have different bootstrapping commands, where Code Commit uses the generic Git bootstrap commands. 

To bootstrap code commit we will run the following commands with the credentials of the codecommit user we created earlier. You may note that we are specifying `--branch=master` , as you may have guessed this will pull updates only from the branch labeled `master`. 
```bash
flux bootstrap git \
    --url=https://git-codecommit.us-east-1.amazonaws.com/v1/repos/fluxexample \
    --username=<REDACTED> \
    --password=<REDACTED> \
    --token-auth=true \
    --branch=master
```

Once you have ran the command above you will see a number of checks performed, when complete you should see `all components are healthy`. 

Lets get all of our pods and see what is happening...

```
$ kubectl get pods -o wide --all-namespaces
NAMESPACE      NAME                                           READY   STATUS    RESTARTS   AGE    IP             NODE                         NOMINATED NODE   READINESS GATES
flux-example   flux-example-deployment-5cfdbf48bd-pd8td       1/1     Running   0          40s    192.168.0.10   ip-10-0-3-126.ec2.internal   <none>           <none>
flux-system    helm-controller-779b58df6b-8h92j               1/1     Running   0          53s    192.168.0.5    ip-10-0-3-126.ec2.internal   <none>           <none>
flux-system    kustomize-controller-5db6bfc56d-wsnl8          1/1     Running   0          53s    192.168.0.6    ip-10-0-3-126.ec2.internal   <none>           <none>
flux-system    notification-controller-7ccfbfbb98-vv7h9       1/1     Running   0          53s    192.168.0.7    ip-10-0-3-126.ec2.internal   <none>           <none>
flux-system    source-controller-565f8fbbff-6qxcc             1/1     Running   0          53s    192.168.0.8    ip-10-0-3-126.ec2.internal   <none>           <none>
kube-system    aws-load-balancer-controller-8b686b6d9-rnjq8   1/1     Running   1          23d    10.0.3.126     ip-10-0-3-126.ec2.internal   <none>           <none>
kube-system    coredns-998b8799c-rsgvb                        1/1     Running   0          23d    192.168.0.3    ip-10-0-3-126.ec2.internal   <none>           <none>
kube-system    coredns-998b8799c-zvk8j                        1/1     Running   0          23d    192.168.0.4    ip-10-0-3-126.ec2.internal   <none>           <none>
kube-system    kube-proxy-zbxlr                               1/1     Running   0          2d2h   10.0.3.126     ip-10-0-3-126.ec2.internal   <none>           <none>
kube-system    nvidia-device-plugin-daemonset-l5q4w           1/1     Running   0          2d2h   192.168.0.2    ip-10-0-3-126.ec2.internal   <none>           <none>
kube-system    weave-net-g6d6q                                2/2     Running   2          2d2h   10.0.3.126     ip-10-0-3-126.ec2.internal   <none>           <none>
redis          redis-master-0                                 1/1     Running   0          2d     192.168.0.9    ip-10-0-3-126.ec2.internal   <none>           <none>
```

As you can see a namespace called `flux-example` was created and the deployment was applied. 

## Updating The Deployment
Lets say we want to change our `replicas` from 1 to 2. Perform a `git pull` on your repository. Modify the file `workloads/example-workload.yaml` , change
`replicas: 1` to `replicas: 2`.

Commit the file and merge back into the target branch. Once the changes have been committed lets get our pods in the `flux-example` namespace.

```
$ kubectl get pods -o wide -n flux-example
NAME                                       READY   STATUS    RESTARTS   AGE     IP             NODE                         NOMINATED NODE   READINESS GATES
flux-example-deployment-5cfdbf48bd-ltwsk   1/1     Running   0          85s     192.168.0.11   ip-10-0-3-126.ec2.internal   <none>           <none>
flux-example-deployment-5cfdbf48bd-pd8td   1/1     Running   0          5m39s   192.168.0.10   ip-10-0-3-126.ec2.internal   <none>           <none>
```

As you can see we now have two replicas! Flux automatically polls the Git repository on a regular basis for changes. When it detects a change to the branch it pulls down the latest and greatest files and deploys them. 

## Enhancements
The example here is only to get your feet wet with Flux. There are many more kustomizable options... Like the use of Kustomize. 

I advise checking out this fantastic example that leverages Kustomize and Helm to perform deployments with Flux.

- https://github.com/fluxcd/flux2-kustomize-helm-example

Furthermore we used Password Authentication for our project. I additionally advise investigating how to use SSH keys and give it a try yourself! This will enhance the security of your Flux deployment. 

If you are wondering how to use secrets in your Git project, check out `sealed-secrets`.

- https://github.com/bitnami-labs/sealed-secrets

## Takeaways

- Flux is a powerful GitOps solution that is free.
- Flux is a pull based model instead of a push based model. 
- Flux can be highly customizable with Kustomize.
- Flux integrates with Helm and can deploy Helm Charts.
- Flux can leverage sealed-secrets.