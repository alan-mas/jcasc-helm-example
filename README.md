# jcasc-helm-example

## Pre-requisites
You need to have all these tools installed in your machine to start working with this Jenkins Configuration as code installation with helm example:
- Vault: https://www.vaultproject.io/docs/install
- Docker: https://docs.docker.com/get-docker/
- Kubernetes: https://kubernetes.io/releases/download/
- Any k8 cluster (like minikube): https://minikube.sigs.k8s.io/docs/start/
- Helm: https://helm.sh/docs/intro/install/
- Git: https://www.atlassian.com/git/tutorials/install-git

## Steps to follow
### Step 1 - Set your kubernetes cluster
First of all, we need to set-up a Kubernetes cluster in which we are going to deploy or jenkins.
So first you need to install [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/) which is the client you are going to use to interact with kubernetes, so after installing kubectl, we need to up a kubernetes cluster to interact with, in this case we are going to use [minikube](https://minikube.sigs.k8s.io/docs/start/) which is a local kubernetes cluster that you can start right away in your machine.

After folliwing the steps to install minikube and added the binary to your ´PATH´, we need to start our cluster using this command at your terminal:
```
minikube start
```

So now if you run:
```
kubectl get all
```

you will see a kubernetes service like this:
```
>kubectl get all
NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   9d
```

If so, congratulations! you have created your first kubernetes cluster locally using minikube!

### Step 2 - Start your vault server
Assuming that you already install vault in your machine, as it is one of the prerequisites for this repo, but in case that you have not please follow this [install tutorial](https://www.vaultproject.io/docs/install).

We normally can start a vault server using:
```
vault server -dev
```

But in this case, as we are going to run Jenkins inside our local kubernetes cluster, and also vault is running in our local machine, we need to expose vault so our jenkins (inside a kubernetes pod) is able to reach it, so we need to change our start command a little bit:

```
vault server -dev -dev-root-token-id root -dev-listen-address 0.0.0.0:8200
```

Then we need to export an environment variable for the vault CLI to address the Vault server.

export VAULT_ADDR=http://0.0.0.0:8200

Next step is to authenticate into vault, we can either use UI in our web browser http://0.0.0.0:8200 or use cmd commands like:
```
$ vault login root
Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.
```

After login, we need to create a secret at path `secret/jenkins` with `JENKINS_ADMIN_ID` and `JENKINS_ADMIN_PASSWORD`
```
vault kv put secret/jenkins JENKINS_ADMIN_ID='taco' JENKINS_ADMIN_PASSWORD='pastor'
```

You can now verify that secret is stored at path `secret/jenkins` running:
```
vault kv get secret/jenkins
```

Now that we already have our secrets in place and our vault server expose, we need to determine the vault address which we are going to redirect our jenkins ping.

To do that we need start a minukube ssh session:
```
minikube ssh
```

Within this SSH session, retrieve the value of the Minikube host.
```
$ dig +short host.docker.internal
192.168.65.2
```

After retrieving the value, we are going to retrieve the status of the Vault server to verify network connectivity.
```
$ dig +short host.docker.internal | xargs -I{} curl -s http://{}:8200/v1/sys/seal-status
{
  "type": "shamir",
  "initialized": true,
  "sealed": false,
  "t": 1,
  "n": 1,
  "progress": 0,
  "nonce": "",
  "version": "1.5.0",
  "migration": false,
  "cluster_name": "vault-cluster-44ba824c",
  "cluster_id": "adc0bb6a-e330-3e7a-e0c7-38061c3bf191",
  "recovery_seal": false,
  "storage_type": "inmem"
}
```

The output displays that Vault is initialized and unsealed. This confirms that pods in the cluster are able to reach Vault given that each pod is configured to use the gateway address.

Next, exit the Minikube SSH session:
```
exit
```
And there you go! now we have vault up and running with the secrets that we need and exposed and reachable from our minikube kubernetes cluster!

### Step 3 - Start our Jenkins with Helm and minikube cluster

You need to clone this repo:
```
git clone https://github.com/alan-mas/jcasc-helm-example.git
```

Now that you have the code at your local, go to the main directory and run our helm chart with:
```
> helm upgrade --install -f values.yml mijenkins jenkins/jenkins
Release "mijenkins" does not exist. Installing it now.
NAME: mijenkins
LAST DEPLOYED: Thu Apr  7 12:57:59 2022
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
1. Get your 'admin' user password by running:
  kubectl exec --namespace default -it svc/mijenkins -c jenkins -- /bin/cat /run/secrets/chart-admin-password && echo
2. Get the Jenkins URL to visit by running these commands in the same shell:
  echo http://127.0.0.1:8080
  kubectl --namespace default port-forward svc/mijenkins 8080:8080

3. Login with the password from step 1 and the username: admin
4. Configure security realm and authorization strategy
5. Use Jenkins Configuration as Code by specifying configScripts in your values.yaml file, see documentation: http:///configuration-as-code and examples: https://github.com/jenkinsci/configuration-as-code-plugin/tree/master/demos

For more information on running Jenkins on Kubernetes, visit:
https://cloud.google.com/solutions/jenkins-on-container-engine

For more information about Jenkins Configuration as Code, visit:
https://jenkins.io/projects/jcasc/


NOTE: Consider using a custom image with pre-installed plugins
```

After running and waiting a 1-2 minutes, you can run kubectl to see the status of the jenkins pods we just deployed with helm:
```
>kubectl get all
NAME              READY   STATUS    RESTARTS   AGE
pod/mijenkins-0   2/2     Running   0          41s

NAME                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)     AGE
service/kubernetes        ClusterIP   10.96.0.1       <none>        443/TCP     9d
service/mijenkins         ClusterIP   10.99.216.135   <none>        8080/TCP    41s
service/mijenkins-agent   ClusterIP   10.108.101.8    <none>        50000/TCP   41s

NAME                         READY   AGE
statefulset.apps/mijenkins   1/1     41s
```

As you can see pod mijenkins-0 has Running status, that means that is up and working, now we need to start the jenkins service:
```
kubectl --namespace default port-forward svc/mijenkins 8080:8080
```

After that if you visit `http://127.0.0.1:8080` or `http://localhost:8080` you will see the main Jenkins Dashboard.

Inside our `values.yml` file we have created a user that uses the `vault` secrets we just created, so in the right corner of the jenkins main dashboard click in `login` and then put the values we set with `JENKINS_ADMIN_ID` and `JENKINS_ADMIN_PASSWORD`

Now you should be able to login with that user and OLÉ! you now have a jenkins running in kubernetes, deployed with helm that can reach a vault that is outside the cluster!


## Useful links:
- https://octopus.com/blog/jenkins-helm-install-guide
- https://artifacthub.io/packages/helm/jenkinsci/jenkins
- https://github.com/jenkinsci/configuration-as-code-plugin/blob/master/README.md
- https://github.com/jenkinsci/configuration-as-code-plugin/blob/master/docs/features/secrets.adoc#using-credential-provider-plugins
- https://github.com/jenkinsci/configuration-as-code-plugin/issues/821
- https://www.digitalocean.com/community/tutorials/how-to-automate-jenkins-setup-with-docker-and-jenkins-configuration-as-code
- https://www.eficode.com/blog/start-jenkins-config-as-code
- https://github.com/RamazanKara/rancher-helm-migration/tree/21aa4d98a9619258cb310f306b8e8f6347f56882/charts/jenkins
- https://github.com/amitkshirsagar13/devops/blob/46b241d15f863206cc24ea978d00c7383f60531d/cicd/jenkins/jcasc.yaml
- https://piotrminkowski.com/2018/09/24/running-jenkins-server-with-configuration-as-code/
- https://craftech.io/blog/manage-your-kubernetes-secrets-with-hashicorp-vault/
- https://github.com/hashicorp/vault/issues/6091
- https://learn.hashicorp.com/tutorials/vault/kubernetes-external-vault?in=vault/kubernetes
- https://plugins.jenkins.io/active-directory/
- https://www.vaultproject.io/docs/commands/token/create