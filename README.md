# Description

Argo Workflows is an open source container-native workflow engine for orchestrating parallel jobs on Kubernetes. Argo Workflows is implemented as a Kubernetes CRD (Custom Resource Definition).

## Deployment steps

**STEP 01**: We can check the last argo workflows release from this link, previous to deployment:

```
https://github.com/argoproj/argo-workflows/releases
```

Create the argo namespace first to deploy argo workflows resources inside

```
$ kubectl create namespace argo
```

Execute the kubectl command to deploy all argo workflows resources

```
$ kubectl apply -n argo -f https://github.com/argoproj/argo-workflows/releases/download/v3.5.4/install.yaml

customresourcedefinition.apiextensions.k8s.io/clusterworkflowtemplates.argoproj.io created
customresourcedefinition.apiextensions.k8s.io/cronworkflows.argoproj.io created
customresourcedefinition.apiextensions.k8s.io/workflowartifactgctasks.argoproj.io created
customresourcedefinition.apiextensions.k8s.io/workfloweventbindings.argoproj.io created
customresourcedefinition.apiextensions.k8s.io/workflows.argoproj.io created
customresourcedefinition.apiextensions.k8s.io/workflowtaskresults.argoproj.io created
customresourcedefinition.apiextensions.k8s.io/workflowtasksets.argoproj.io created
customresourcedefinition.apiextensions.k8s.io/workflowtemplates.argoproj.io created
serviceaccount/argo created
serviceaccount/argo-server created
role.rbac.authorization.k8s.io/argo-role created
clusterrole.rbac.authorization.k8s.io/argo-aggregate-to-admin created
clusterrole.rbac.authorization.k8s.io/argo-aggregate-to-edit created
clusterrole.rbac.authorization.k8s.io/argo-aggregate-to-view created
clusterrole.rbac.authorization.k8s.io/argo-cluster-role created
clusterrole.rbac.authorization.k8s.io/argo-server-cluster-role created
rolebinding.rbac.authorization.k8s.io/argo-binding created
clusterrolebinding.rbac.authorization.k8s.io/argo-binding created
clusterrolebinding.rbac.authorization.k8s.io/argo-server-binding created
configmap/workflow-controller-configmap created
service/argo-server created
priorityclass.scheduling.k8s.io/workflow-controller created
deployment.apps/argo-server created
deployment.apps/workflow-controller created
```

**STEP 02**: Authentication client mode 

By default argo Workflows is deployed with the client mode authentication configured, so we need to authenticate with a token to access to any argo workflows resource. We can create one executing this command:

```
$ kubectl -n argo exec $(kubectl get pod -n argo -l 'app=argo-server' -o jsonpath='{.items[0].metadata.name}') -- argo auth token

Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6Ik1mSVQ0b29vYlJOd01wZ2plWWVtcHFaandOM2xXZXB4MFhCREV6MUtGaTAifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiXSwiZXhwIjoxNzQwNDIzNDUyLCJpYXQiOjE3MDg4ODc0NTIsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJhcmdvIiwicG9kIjp7Im5hbWUiOiJhcmdvLXNlcnZlci02OTg3NzY5Y2I4LWRsbHdnIiwidWlkIjoiZTczYjk4NjUtZjJkZC00ZGIxLTk4ZTUtOTRiYTVmNmQ1ODJiIn0sInNlcnZpY2VhY2NvdW50Ijp7Im5hbWUiOiJhcmdvLXNlcnZlciIsInVpZCI6ImEwNWE5YmQ2LTM4MzItNGI2Ny04MTY5LTNlM2Q5NWZlZjZmNiJ9LCJ3YXJuYWZ0ZXIiOjE3MDg4OTEwNTl9LCJuYmYiOjE3MDg4ODc0NTIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDphcmdvOmFyZ28tc2VydmVyIn0.FH1aJW0PwUmRJLndBO3KyVzxAoRwDVWHOrVn165-R5P9M9rqcyXNZj6h1hOGz5P4opnxDhK99XSVm9CHETCdQcR2kW91bX5X1YP9iClTquL-sMQUiqvAaPPHwueDHhI_jDFQCEigpPLtBJyuRDqxmKo4p8G5OTTsUIrEbLMhhulbbNLHuDvx1HU4VenSWRRFDbniNOTj1I6LMhqQV13VGSVOFJMRj1dwdycZ9YR_xsKNypsZxHKSpT94hCdqcxX44wn2pEdXrSprINyhpsgLcpZu-0B8rblV4spSW9QIlHYFODm3befKoF4x6YAq6rm5vY5TwHt07YtraK20ql3tVQ
```

Other way to obtain a token is creating a long-lived API token as a secret for the argo service account like this in the namespace argo. After we can recover a token getting the secret like this 

```
$ kubectl apply -n argo -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: argo.service-account-token
  annotations:
    kubernetes.io/service-account.name: argo
type: kubernetes.io/service-account-token
EOF

$ ARGO_TOKEN="Bearer $(kubectl get secret -n argo argo.service-account-token -o=jsonpath='{.data.token}' | base64 --decode)"
echo $ARGO_TOKEN

```

The result of this command only return the token. We must add the Bearer after the token like this:

```
Bearer $ARGO_TOKEN
```

**STEP 02**: Authentication server mode 

If you want disable any authentication, we can set the authentication mode for Argo Workflows to "server" executing this command to patch the argo-server deployment:

```
$ kubectl patch deployment \
  argo-server \
  --namespace argo \
  --type='json' \
  -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/args", "value": [
  "server",
  "--auth-mode=server"
]}]'
```

**STEP 03**: Set permissions to default service account inside argo namespace

We must set the cluster role admin to the default service account located in argo name space to have permissions to deploy the worksflows in argo if we don't specify any service account in deployment like this:

```
$ kubectl create rolebinding default-admin --clusterrole=admin --serviceaccount=argo:default --namespace=argo

$  argo submit -n argo --watch https://raw.githubusercontent.com/argoproj/argo-workflows/main/examples/hello-world.yaml
```

**Be carefoul to use this Argo workflows template in Mac ARM because not works for this architecture**. Use this one for example:

```
apiVersion: argoproj.io/v1alpha1
kind: Workflow                  # new type of k8s spec
metadata:
  generateName: hello-world-    # name of the workflow spec
spec:
  entrypoint: echotest
  templates:
  - name: echotest
    container:
      image: alpine
      command: ["sh","-c"]
      args: ["echo","hello"]
```

If don't want set this admin permission for the default service account in argo namespace, you must specifie the service account to use when deploy a workflow like this:

```
$ argo submit -n argo --watch https://raw.githubusercontent.com/argoproj/argo-workflows/main/examples/hello-world.yaml --serviceaccount argo
Name:                hello-world-f8dps
Namespace:           argo
ServiceAccount:      argo
Status:              Succeeded
Conditions:          
 PodRunning          False
 Completed           True
Created:             Sat Mar 02 18:47:31 +0100 (10 seconds ago)
Started:             Sat Mar 02 18:47:31 +0100 (10 seconds ago)
Finished:            Sat Mar 02 18:47:41 +0100 (now)
Duration:            10 seconds
Progress:            1/1
ResourcesDuration:   5s*(1 cpu),5s*(100Mi memory)

STEP                  TEMPLATE  PODNAME            DURATION  MESSAGE
 ✔ hello-world-f8dps  echotest  hello-world-f8dps  4s   
```

**STEP 04**: Port forward Argo UI

To connect to UI we must execut a kuberneted port-forward:

```
kubectl -n argo port-forward deployment/argo-server 2746:2746
```

Now from Argo UI we must login in client mode and paste the previous token including the Bearer attribute:

![Items](https://github.com/masalinas/poc-argo-workflow/assets/1216181/8e965ffb-e1d1-4063-b53e-105849abafc2)

**STEP 05**: Install Argo CLI

Go to this URI and download the CLI for your platform

```
https://github.com/argoproj/argo-workflows/releases/tag/v3.5.4
```

Install on Mac

```
$ gzip -g argo-darwin-arm64.gz
$ chmod +x argo-darwin-arm64
$ mv argo-darwin-arm64 argo
$ sudo mv argo /usr/local/bin
```

**STEP 06**: Execute a workflow

Create a simple hello-world argo workflow like this:

```
argo-hello-world.yaml

apiVersion: argoproj.io/v1alpha1
kind: Workflow                  # new type of k8s spec
metadata:
  generateName: hello-world-    # name of the workflow spec
spec:
  entrypoint: echotest
  templates:
  - name: echotest
    container:
      image: alpine
      command: ["sh","-c"]
      args: ["echo","hello"]
```

Deploy the workflow from argo CLI in the argo namespace. We don't specify the default service account if executed set the argo as default service account in this namespace (Check STEP 03)

```
$ argo submit -n argo argo-hello-world.yaml
```

**STEP 07**: See results

```
$ argo watch -n argo delightful-dragon
Name:                delightful-dragon
Namespace:           argo
ServiceAccount:      unset (will run with the default ServiceAccount)
Status:              Succeeded
Conditions:          
 PodRunning          False
 Completed           True
Created:             Tue Feb 27 20:05:59 +0100 (1 minute ago)
Started:             Tue Feb 27 20:05:59 +0100 (1 minute ago)
Finished:            Tue Feb 27 20:06:09 +0100 (1 minute ago)
Duration:            10 seconds
Progress:            1/1
ResourcesDuration:   3s*(100Mi memory),3s*(1 cpu)
Parameters:          
  message:           hello argo

STEP                  TEMPLATE  PODNAME            DURATION  MESSAGE
 ✔ delightful-dragon  argosay   delightful-dragon  3s  
```

**STEP 08**: Use the Argo API

First load the Bearer Token

```
ARGO_TOKEN="Bearer $(kubectl get secret -n argo argo.service-account-token -o=jsonpath='{.data.token}' | base64 --decode)"
```

Now using the curl tool we can send request against the Argo Server. This requets is a Get to recover all 

```
curl --insecure -H "Authorization: $ARGO_TOKEN" https://localhost:2746/api/v1/workflows/argo
{"metadata":{"resourceVersion":"281377"},"items":[{"metadata":{"name":"hello-world-84zvb","generateName":"hello-world-","namespace":"argo","uid":"47e50feb-e293-4820-816d-f54b6c8632b8","resourceVersion":"276477","generation":3,"creationTimestamp":"2024-02-25T20:23:52Z","labels":{"workflows.argoproj.io/completed":"true","workflows.argoproj.io/phase":"Succeeded"},"annotations":{"workflows.argoproj.io/pod-name-format":"v2"},"managedFields":[{"manager":"argo","operation":"Update","apiVersion":"argoproj.io/v1alpha1","time":"2024-02-25T20:23:52Z","fieldsType":"FieldsV1","fieldsV1":{"f:metadata":{"f:generateName":{}},"f:spec":{}}},{"manager":"workflow-controller","operation":"Update","apiVersion":"argoproj.io/v1alpha1","time":"2024-02-25T20:24:02Z","fieldsType":"FieldsV1","fieldsV1":{"f:metadata":{"f:annotations":{".":{},"f:workflows.argoproj.io/pod-name-format":{}},"f:labels":{".":{},"f:workflows.argoproj.io/completed":{},"f:workflows.argoproj.io/phase":{}}},"f:status":{}}}]},"spec":{"templates":[{"name":"echotest","inputs":{},"outputs":{},"metadata":{},"container":{"name":"","image":"alpine","command":["sh","-c"],"args":["echo","hello"],"resources":{}}}],"entrypoint":"echotest","arguments":{}},"status":{"phase":"Succeeded","startedAt":"2024-02-25T20:23:52Z","finishedAt":"2024-02-25T20:24:02Z","progress":"1/1","nodes":{"hello-world-84zvb":{"id":"hello-world-84zvb","name":"hello-world-84zvb","displayName":"hello-world-84zvb","type":"Pod","templateName":"echotest","templateScope":"local/hello-world-84zvb","phase":"Succeeded","startedAt":"2024-02-25T20:23:52Z","finishedAt":"2024-02-25T20:23:56Z","progress":"1/1","resourcesDuration":{"cpu":4,"memory":4},"outputs":{"exitCode":"0"},"hostNodeName":"minikube"}},"conditions":[{"type":"PodRunning","status":"False"},{"type":"Completed","status":"True"}],"resourcesDuration":{"cpu":4,"memory":4},"artifactRepositoryRef":{"default":true,"artifactRepository":{}},"artifactGCStatus":{"notSpecified":true},"taskResultsCompletionStatus":{"hello-world-84zvb":true}}},{"metadata":{"name":"hello-world-cnq2g","generateName":"hello-world-","namespace":"argo","uid":"5f0767b5-856f-4c9b-97a7-01cd634b5792","resourceVersion":"276236","generation":3,"creationTimestamp":"2024-02-25T20:21:34Z","labels":{"workflows.argoproj.io/completed":"true","workflows.argoproj.io/creator":"system-serviceaccount-argo-argo","workflows.argoproj.io/phase":"Succeeded"},"annotations":{"workflows.argoproj.io/pod-name-format":"v2"},"managedFields":[{"manager":"argo","operation":"Update","apiVersion":"argoproj.io/v1alpha1","time":"2024-02-25T20:21:34Z","fieldsType":"FieldsV1","fieldsV1":{"f:metadata":{"f:generateName":{},"f:labels":{".":{},"f:workflows.argoproj.io/creator":{}}},"f:spec":{}}},{"manager":"workflow-controller","operation":"Update","apiVersion":"argoproj.io/v1alpha1","time":"2024-02-25T20:21:44Z","fieldsType":"FieldsV1","fieldsV1":{"f:metadata":{"f:annotations":{".":{},"f:workflows.argoproj.io/pod-name-format":{}},"f:labels":{"f:workflows.argoproj.io/completed":{},"f:workflows.argoproj.io/phase":{}}},"f:status":{}}}]},"spec":{"templates":[{"name":"echotest","inputs":{},"outputs":{},"metadata":{},"container":{"name":"","image":"alpine","command":["sh","-c"],"args":["echo","hello"],"resources":{}}}],"entrypoint":"echotest","arguments":{}},"status":{"phase":"Succeeded","startedAt":"2024-02-25T20:21:34Z","finishedAt":"2024-02-25T20:21:44Z","progress":"1/1","nodes":{"hello-world-cnq2g":{"id":"hello-world-cnq2g","name":"hello-world-cnq2g","displayName":"hello-world-cnq2g","type":"Pod","templateName":"echotest","templateScope":"local/hello-world-cnq2g","phase":"Succeeded","startedAt":"2024-02-25T20:21:34Z","finishedAt":"2024-02-25T20:21:38Z","progress":"1/1","resourcesDuration":{"cpu":4,"memory":4},"outputs":{"exitCode":"0"},"hostNodeName":"minikube"}},"conditions":[{"type":"PodRunning","status":"False"},{"type":"Completed","status":"True"}],"resourcesDuration":{"cpu":4,"memory":4},"artifactRepositoryRef":{"default":true,"artifactRepository":{}},"artifactGCStatus":{"notSpecified":true},"taskResultsCompletionStatus":{"hello-world-cnq2g":true}}}]}
```

## Submit a workflow from Argo API REST

First create a workflow template in Argo UI like this one:

```
metadata:
  name: arm-hello-world
  namespace: argo
  labels:
    example: 'true'
spec:
  workflowMetadata:
    labels:
      example: 'true'
  entrypoint: echotest
  templates:
    - name: echotest
      container:
        image: alpine
        command: ["sh","-c"]
        args: ["echo","hello"]
  ttlStrategy:
    secondsAfterCompletion: 300
  podGC:
    strategy: OnPodCompletion
```

![Captura de pantalla 2024-03-16 a las 17 23 20](https://github.com/masalinas/doc-argo-workflow/assets/1216181/780b6a02-fd18-459a-b39b-8effa9bdcf7f)

Second submit this workflow template using the argo API REST like this

```
curl -X POST https://localhost:2746/api/v1/workflows/argo/submit --insecure -H "Authorization: $ARGO_TOKEN" -d '{"namespace": "argo", "resourceKind": "WorkflowTemplate", "resourceName": "arm-hello-world"}'
{"metadata":{"name":"arm-hello-world-mrcbm","generateName":"arm-hello-world-","namespace":"argo","uid":"827d2664-1961-4d5c-8374-dd900cf9f675","resourceVersion":"481424","generation":1,"creationTimestamp":"2024-03-16T16:16:01Z","labels":{"workflows.argoproj.io/creator":"system-serviceaccount-argo-argo","workflows.argoproj.io/workflow-template":"arm-hello-world"},"managedFields":[{"manager":"argo","operation":"Update","apiVersion":"argoproj.io/v1alpha1","time":"2024-03-16T16:16:01Z","fieldsType":"FieldsV1","fieldsV1":{"f:metadata":{"f:generateName":{},"f:labels":{".":{},"f:workflows.argoproj.io/creator":{},"f:workflows.argoproj.io/workflow-template":{}}},"f:spec":{},"f:status":{}}}]},"spec":{"arguments":{},"workflowTemplateRef":{"name":"arm-hello-world"}},"status":{"startedAt":null,"finishedAt":null,"storedTemplates":{"namespaced/arm-hello-world/echotest":{"name":"echotest","inputs":{},"outputs":{},"metadata":{},"container":{"name":"","image":"alpine","command":["sh","-c"],"args":["echo","hello"],"resources":{}}}}}}
```

## Some links

The API documentation for this request can be check from the Argo UI like this:

![Items (1)](https://github.com/masalinas/poc-argo-workflow/assets/1216181/304c0834-ffbc-4cd9-95b2-83074322b11d)
