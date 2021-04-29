# companies-catalog

## to deploy  the apps on openshift
Please install : 
- oc cli : https://docs.openshift.com/container-platform/4.5/cli_reference/openshift_cli/getting-started-cli.html
- kn cli : https://docs.openshift.com/container-platform/4.5/serverless/installing_serverless/installing-kn.html#installing-kn
- kogito cli : https://docs.jboss.org/kogito/release/latest/html_single/#proc-kogito-operator-and-cli-installing_kogito-deploying-on-openshift


### connect to Openshift server

```shell
oc login https://ocp-url:6443 -u login -p password
```



### create new namespace
```shell
export PROJECT=companies-catalog
oc new-project $PROJECT
```
Routes include the project name, if you choose another one, don't forget to change it on the differents places/files (such kogito-realm) to use the correct urls. 
### add a github secret to checkout sources

```shell
oc create secret generic username \
    --from-literal=username=username \
    --from-literal=password=password \
    --type=kubernetes.io/basic-auth
```

### add a registry secret to build pull images from quay

``` shell
oc create secret docker-registry quay-secret \
    --docker-server=quay.io/username \
    --docker-username=username \
    --docker-password=password\
    --docker-email=email

oc secrets link builder quay-secret -n $PROJECT
oc secrets link default quay-secret --for=pull -n $PROJECT
```

### Clone the source from github

```git
git clone https://github.com/mouachan/companies-catalog.git cd companies-catalog
export TMP_DIR=tmp
mkdir $TMP_DIR
```
#### Install Openshift Serverless and knative-serving 

install openshift-serverless operator from OperatorHub

https://docs.openshift.com/container-platform/4.6/serverless/installing_serverless/installing-openshift-serverless.html

create a knative-serving instance
```shell
./manifest/scripts/knative-serving.sh
```
#### Install Openshift Pipelines Operator (Tekton) from Operator Hub

#### Build the application through tekton pipeline

```shell
tkn pipeline start build-and-deploy \
    -w name=shared-workspace,volumeClaimTemplateFile=https://raw.githubusercontent.com/mouachan/companies-catalog/main/manifest/build/pipeline/persistent_volume_claim.yaml \
    -p deployment-name=pipelines-companies-catalog \
    -p git-url=https://github.com/mouachan/companies-catalog.git \
    -p IMAGE=image-registry.openshift-image-registry.svc:5000/pipelines-companies/pipelines-companies-catalog \
```



#### Deploy application resources using the cli 
### Create a persistent mongodb

#### Option 1 : using oc cli

```shell
# check if persistent mongo exist

oc get templates -n openshift | grep mongodb

# if not create it
oc create -f https://raw.githubusercontent.com/openshift/origin/master/examples/db-templates/mongodb-persistent-template.json -n $PROJECT

#get paramters list
oc process --parameters -n openshift mongodb-persistent

#create the instannce
oc process mongodb-persistent -n openshift -p MONGODB_USER=admcomp -p MONGODB_PASSWORD=r3dh4t2021! -p MONGODB_DATABASE=companies -p MONGODB_ADMIN_PASSWORD=r3dh4t2021! \
| oc create -n $PROJECT -f -
```

#### wait until the mongodb is ready
```shell
oc get pods 
NAME               READY   STATUS      RESTARTS   AGE
mongodb-1-9zhxm    1/1     Running     0          41s
mongodb-1-deploy   0/1     Completed   0          47s 
```
####  Create  DB and collection

create the schema 
```shell
MONGO_POD_NAME=$(oc get pods --output=jsonpath={.items..metadata.name} -l 'name=mongodb')
oc exec -it $MONGO_POD_NAME -- bash -c  'mongo companies -u admcomp -p r3dh4t2021!' < ./manifest/create-schema.js  
```
add records
```shell
oc exec -it $MONGO_POD_NAME -- bash -c  'mongo companies -u admcomp -p r3dh4t2021!' < ./manifest/insert-records.js
```  

#### Deploy application using template

### Deploy the catalog service as a serverless application 

```shell
oc apply -f ./manifest/companies-svc-knative.yml 
```

### create the template
```shell
oc create -f manifest/companies-catalog-template.yaml
```  
### process the template
```
oc process companies-catalog -p MONGODB_USER=admcomp -p MONGODB_PASSWORD=r3dh4t2021! -p MONGODB_DATABASE=companies -p MONGODB_ADMIN_PASSWORD=r3dh4t2021! -p REGISTRY_SECRET=quay-secret -p IMAGE_SVC_CATALOG=quay.io/mouachan/companies-svc:1.0 -p SVC_CATALOG_NAME=companies-svc | oc create -f -  
```
### delete all resources 
```
oc delete configmap/companiesdb-init secret/mongodb service/mongodb persistentvolumeclaim/mongodb deploymentconfig.apps.openshift.io/mongodb service.serving.knative.dev/companies-svc template/companies-catalog 
```



##### Pipeline 

oc adm policy add-scc-to-user privileged -z default -n catalog-pipeline  
oc edit configmap config-defaults -n openshift-pipelines
oc create clusterrolebinding pipeline --clusterrole=cluster-admin --serviceaccount=catalog-pipeline:pipeline
oc secrets link pipeline quay-secret

