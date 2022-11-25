Install Argo CD CLI
To interact with the API Server we need to deploy the CLI:

sudo curl --silent --location -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/download/v2.4.7/argocd-linux-amd64

sudo chmod +x /usr/local/bin/argocd


argocd login openshift-gitops-server-openshift-gitops.apps.training1.openshift.labs.sass.ro

use admin and password from oc cli command : 

 oc extract  secret/openshift-gitops-cluster -n openshift-gitops  --to=-
 
 
 argocd login openshift-gitops-server-openshift-gitops.apps.training1.openshift.labs.sass.ro
WARNING: server certificate had error: x509: certificate signed by unknown authority. Proceed insecurely (y/n)? y
WARN[0003] Failed to invoke grpc call. Use flag --grpc-web in grpc calls. To avoid this warning message, use flag --grpc-web.
Username: admin
Password:
'admin:login' logged in successfully
Context 'openshift-gitops-server-openshift-gitops.apps.training1.openshift.labs.sass.ro' updated
