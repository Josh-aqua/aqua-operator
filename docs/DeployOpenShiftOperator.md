The Aqua Security Operator is used to deploy and manage the Aqua Cloud Native Security Platform (CSP) and its components:
* Server (aka “console”)
* Database (optional; you can map an external database as well) 
* Gateway 
* Enforcer
* Scanner
* KubeEnforcer

Use the Aqua Operator to: 
* Deploy Aqua CSP on OpenShift
* Scale up Aqua Security components with extra replicas
* Assign metadata tags to Aqua CSP components
* Automatically scale the number of Aqua scanners based on the number of images in the scan queue
	
You can view all Aqua Operator CR options for managing the Aqua CSP platform [Custom Resources](https://github.com/aquasecurity/aqua-operator/tree/master/deploy/crds). 
   
## Prerequisites 

Make sure you have a license and access to the Aqua registry. To obtain a license, please contact Aqua Security at https://www.aquasec.com/about-us/contact-us/.

It is advised that you read about the [Aqua Environment and Configuration](https://docs.aquasec.com/docs/purpose-of-this-section) before deploying and using the Operator. 

## Deploying the Aqua Operator

1. Create a new namespace/project called "aqua" for the Aqua deployment.
2. Install the Aqua Operator from Red Hat's OperatorHub and add it to the "aqua" namespace. 

## Deploying the Aqua CSP custom resources
First, make sure to set the requirements that are specified [here](FirstSteps.md)

Before you start, you will need to supply two secrets for the deployment: 
* A secret for the Docker registry
* A secret for the database

You can list the secrets in the custom resource YAML files or you can define them in the OpenShift project (see example below):
```bash
oc create secret docker-registry aqua-registry --docker-server=registry.aquasec.com --docker-username=<AQUA_USERNAME> --docker-password=<AQUA_PASSWORD> --docker-email=<user email> -n aqua
oc create secret generic aqua-database-password --from-literal=db-password=<password> -n aqua
oc secrets link aqua-sa aqua-registry --for=pull -n aqua
```

## AquaCSP CRDs ##
The AquaCSP Operator includes the following CRDs -

**AquaCSP** provides the fastest methods to deploy Aqua in a single cluster. AquaCSP defines how to deploy the server, Kube-Enforcer, and Enforcer in the target cluseter. Please see the example [CR](https://github.com/aquasecurity/aqua-operator/blob/5.3/deploy/crds/operator_v1alpha1_aquacsp_cr.yaml) for the listing of all fields and configurrtions.
* You can set the enforcement mode using the "enforcerMode" properties in the CR file.
* You can deploy a Route by setting the "route" property to "true".
* The default service type for the console and gateway is ClusterIP. You can change this to a different service type.
* You can choose to deploy a different version of Aqua CSP by setting the "version" property or change the image "tag".
* You can choose to use an external databaes providing the 'externalDb' details.
* You can omit the Enforcer and Kube-Enforcer components by removing them from the CR.

**AquaKubeEnforcer** is used to deploy the KubeEnforcer in any cluster. Please see the example [CR](https://github.com/aquasecurity/aqua-operator/blob/5.3.0/deploy/crds/operator_v1alpha1_aquakubeenforcer_cr.yaml)
* You need to provide a token to identify the KubeEnforcer
* You can set the target gateway using the **config.gateway_address** property
* You can choose to deploy a different version of the KubeEnforcer by setting the **image.tag** property

**AquaEnforcer**
is used to deploy the AquaEnforcer in any cluster. Please see the example [CR](https://github.com/aquasecurity/aqua-operator/blob/5.3.0/deploy/crds/operator_v1alpha1_aquaenforcer_cr.yaml)
* You need to provide a token to identify the AquaEnforcer
* You can set the target gateway using the **gateway.host** and **gateway.port** properties
* You can choose to deploy a different version of the AquaEnforcer by setting the **image.tag** property

****


* You can instruct the Aqua CSP CR to automatically deploy the Enforcer by setting the "enforcer" and "enforcerMode" properties in the CR file.
* You can also instruct the Aqua CSP CR to automatically deploy the KubeEnforcer by setting the "kubeEnforcer" property in the CR file (with "registry" and "tag").  
* If you want to deploy the Enforcers manually, you will need to first get a security token. Access Aqua console and create a new Enforcer Group. Copy the group's "token" and use it in the AquaEnforcer/AquaKubeEnforcer CR (see the example below).
* You can instruct the Aqua CSP CR to automatically deploy a Route by setting the "route" property to "true".
* The default service type for the console and gateway is ClusterIP. You can change this to a different service type.
* You can choose to deploy a different version of Aqua CSP by setting the "version" property or change the image "tag".
	
#### Example: Deploying Aqua CSP

Here is an example of a simple deployment: 
```yaml
---
apiVersion: operator.aquasec.com/v1alpha1
kind: AquaCsp
metadata:
  name: aqua
  namespace: aqua
spec:
  infra:                                    
    serviceAccount: "aqua-sa"               
    namespace: "aqua"                       
    version: "5.3"                          
    requirements: true                      
  common:
    imagePullSecret: "aqua-registry"        # Optional: If already created image pull secret then mention in here
    dbDiskSize: 10
    databaseSecret:                         # Optional: If already created database secret then mention in here
      key: "db-password"
      name: "aqua-database-password"      
  database:                                 
    replicas: 1                            
    service: "ClusterIP"
    image:
      registry: "registry.aquasec.com"
      repository: "database"
      tag: "<<IMAGE TAG>>"
      pullPolicy: Always                    
  gateway:                                  
    replicas: 1                             
    service: "ClusterIP"
    image:
      registry: "registry.aquasec.com"
      repository: "gateway"
      tag: "<<IMAGE TAG>>"
      pullPolicy: Always                     
  server:                                   
    replicas: 1                             
    service: "LoadBalancer" 
    image:
      registry: "registry.aquasec.com"
      repository: "server"
      tag: "<<IMAGE TAG>>"
      pullPolicy: Always  
  route: true                               # Optional: If defined and set to true, the Operator will create a Route to enable access to the console
```

If you haven't used the "route" option in the Aqua CSP CR, you should define a Route manually to enable external access to the Aqua Server (Console).


#### Example: Deploying AquaCsp with KubeEnforcer and Enforcer

Here is an example of a AquaCsp deployment with KubeEnforcer + Enforcer: 
```yaml
---
apiVersion: operator.aquasec.com/v1alpha1
kind: AquaCsp
metadata:
  name: aqua
  namespace: aqua
spec:
  infra:                                    
    serviceAccount: "aqua-sa"               
    namespace: "aqua"                       
    version: "5.3"                          
    requirements: true                      
  common:
    imagePullSecret: "aqua-registry"        # Optional: If already created image pull secret then mention in here
    dbDiskSize: 10
    databaseSecret:                         # Optional: If already created database secret then mention in here
      key: "db-password"
      name: "aqua-database-password"      
  database:                                 
    replicas: 1                            
    service: "ClusterIP"
    image:
      registry: "registry.aquasec.com"
      repository: "database"
      tag: "<<IMAGE TAG>>"
      pullPolicy: Always                    
  gateway:                                  
    replicas: 1                             
    service: "ClusterIP"
    image:
      registry: "registry.aquasec.com"
      repository: "gateway"
      tag: "<<IMAGE TAG>>"
      pullPolicy: Always                     
  server:                                   
    replicas: 1                             
    service: "LoadBalancer" 
    image:
      registry: "registry.aquasec.com"
      repository: "server"
      tag: "<<IMAGE TAG>>"
      pullPolicy: Always 
  enforcer:                                 # Optional: If defined, the Operator will create the default Enforcer 
    enforcerMode: false                     # Defines whether the default Enforcer will work in "Enforce" (true) or "Audit Only" (false) mode
  kubeEnforcer:                             # Optional: If defined, the Operator will create KubeEnforcer
    registry: "registry.aquasec.com"        
    tag: "<<IMAGE TAG>>" 
  route: true                               # Optional: If defined and set to true, the Operator will create a Route to enable access to the console
```


#### Example: Deploying AquaCsp with split database

Here is an example of a split database deployment, this will create a different database for audits: 
```yaml
---
apiVersion: operator.aquasec.com/v1alpha1
kind: AquaCsp
metadata:
  name: aqua
  namespace: aqua
spec:
  infra:                                    
    serviceAccount: "aqua-sa"               
    namespace: "aqua"                       
    version: "5.3"                          
    requirements: true                      
  common:
    imagePullSecret: "aqua-registry"        # Optional: If already created image pull secret then mention in here
    dbDiskSize: 10
    databaseSecret:                         # Optional: If already created database secret then mention in here
      key: "db-password"
      name: "aqua-database-password"
    splitDB: true      
  database:                                 
    replicas: 1                            
    service: "ClusterIP"
    image:
      registry: "registry.aquasec.com"
      repository: "database"
      tag: "<<IMAGE TAG>>"
      pullPolicy: Always                    
  gateway:                                  
    replicas: 1                             
    service: "ClusterIP"
    image:
      registry: "registry.aquasec.com"
      repository: "gateway"
      tag: "<<IMAGE TAG>>"
      pullPolicy: Always                     
  server:                                   
    replicas: 1                             
    service: "LoadBalancer" 
    image:
      registry: "registry.aquasec.com"
      repository: "server"
      tag: "<<IMAGE TAG>>"
      pullPolicy: Always  
  route: true                               # Optional: If defined and set to true, the Operator will create a Route to enable access to the console
```

#### Example: Deploying AquaCsp with external database

Here is an example of deployment that use external database: 
```yaml
---
apiVersion: operator.aquasec.com/v1alpha1
kind: AquaCsp
metadata:
  name: aqua
  namespace: aqua
spec:
  infra:                                    
    serviceAccount: "aqua-sa"               
    namespace: "aqua"                       
    version: "5.3"                          
    requirements: true                      
  common:
    imagePullSecret: "aqua-registry"        # Optional: If already created image pull secret then mention in here
    dbDiskSize: 10      
  externalDb:
    host: "<<EXTERNAL DATABASE IP>>"
    port: "<<EXTERNAL DATABASE PORT>>"
    username: "<<EXTERNAL DATABASE USER NAME>>"
    password: "<<EXTERNAL DATABASE PASSWORD>>"    # Optional: you can specify the database password secret in common.databaseSecret                     
  gateway:                                  
    replicas: 1                             
    service: "ClusterIP"
    image:
      registry: "registry.aquasec.com"
      repository: "gateway"
      tag: "<<IMAGE TAG>>"
      pullPolicy: Always                     
  server:                                   
    replicas: 1                             
    service: "LoadBalancer" 
    image:
      registry: "registry.aquasec.com"
      repository: "server"
      tag: "<<IMAGE TAG>>"
      pullPolicy: Always  
  route: true                               # Optional: If defined and set to true, the Operator will create a Route to enable access to the console
```

### Example: Deploying AquaCsp with split external database

Here is an example of deployment that use external databases with split DB: 
```yaml
---
apiVersion: operator.aquasec.com/v1alpha1
kind: AquaCsp
metadata:
  name: aqua
  namespace: aqua
spec:
  infra:                                    
    serviceAccount: "aqua-sa"               
    namespace: "aqua"                       
    version: "5.3"                          
    requirements: true                      
  common:
    imagePullSecret: "aqua-registry"        # Optional: If already created image pull secret then mention in here
    dbDiskSize: 10
    splitDB: true      
  externalDb:
    host: "<<EXTERNAL DATABASE IP>>"
    port: "<<EXTERNAL DATABASE PORT>>"
    username: "<<EXTERNAL DATABASE USER NAME>>"
    password: "<<EXTERNAL DATABASE PASSWORD>>"    # Optional: you can specify the database password secret in common.databaseSecret
  auditDB:
    information:
      host: "<<AUDIT EXTERNAL DB IP>>"
      port: "<<AUDIT EXTERNAL DB PORT>>"
      username: "<<AUDIT EXTERNAL DB USER NAME>>"
      password: "<<AUDIT EXTERNAL DB PASSWORD>>"  # Optional: you can specify the database password secret in auditDB.secret
    secret:                                       # Optional: the secret that hold the audit database password. will create one if not provided
      key: 
      name:                     
  gateway:                                  
    replicas: 1                             
    service: "ClusterIP"
    image:
      registry: "registry.aquasec.com"
      repository: "gateway"
      tag: "<<IMAGE TAG>>"
      pullPolicy: Always                     
  server:                                   
    replicas: 1                             
    service: "LoadBalancer" 
    image:
      registry: "registry.aquasec.com"
      repository: "server"
      tag: "<<IMAGE TAG>>"
      pullPolicy: Always  
  route: true                               # Optional: If defined and set to true, the Operator will create a Route to enable access to the console
```


#### Example: Deploying Aqua Enforcer(s)

If you haven't deployed any Enforcers, or if you want to deploy additional Enforcers, follow the instructions [here](https://github.com/aquasecurity/aqua-operator/blob/master/deploy/crds/operator_v1alpha1_aquaenforcer_cr.yaml).

Here is an example of a simple Enforcer deployment: 
```yaml
---
apiVersion: operator.aquasec.com/v1alpha1
kind: AquaEnforcer
metadata:
  name: aqua
spec:
  infra:                                    
    serviceAccount: "aqua-sa"                
    version: "5.3"                          # Optional: auto generate to latest version
  common:
    imagePullSecret: "aqua-registry"        # Optional: if already created image pull secret then mention in here
  deploy:                                   # Optional: information about Aqua Enforcer deployment
    image:                                  # Optional: take the default value and version from infra.version
      repository: "enforcer"                # Optional: default = enforcer
      registry: "registry.aquasec.com"      # Optional: default = registry.aquasec.com
      tag: "<<IMAGE TAG>>"                  # Optional: default = 5.3
      pullPolicy: "IfNotPresent"            # Optional: default = IfNotPresent
  gateway:                                  # Required: data about the gateway address
    host: aqua-gateway
    port: 8443
  token: "<<your-token>>"                   # Required: The Enforcer group token can use an existing secret instead (you can create a token from the Aqua console)
```

#### Example: Deploying Aqua KubeEnforcer

Before deploying AquaKubeEnforcer, you need to run the following commands:

```bash
oc create serviceaccount aqua-kube-enforcer-sa -n aqua
oc adm policy add-cluster-role-to-user cluster-reader system:serviceaccount:aqua:aqua-kube-enforcer-sa
oc adm policy add-scc-to-user nonroot system:serviceaccount:aqua:aqua-kube-enforcer-sa
oc adm policy add-scc-to-user hostaccess system:serviceaccount:aqua:aqua-kube-enforcer-sa
```

Here is an example of a KubeEnforcer deployment:
```yaml
apiVersion: operator.aquasec.com/v1alpha1
kind: AquaKubeEnforcer
metadata:
  name: aqua
spec:
  config:
    gateway_address: "aqua-gateway:8443"      # Required: provide <<AQUA GW IP OR DNS: AQUA GW PORT>>
    cluster_name: "aqua-secure"               # Required: provide your cluster name
    imagePullSecret: "aqua-registry"          # Optional: needed in case spec.registry is not defined
  image:
    registry: "registry.aquasec.com"
    tag: "<<KUBE_ENFORCER_TAG>>"
    repository: kube-enforcer
    pullPolicy: Always
  registry:                                 # Optional: required only if spec.config.imagePullSecret does not exist
    url: "registry.aquasec.com"
    username: "<<YOUR_USER_NAME>>"
    password: "<<YOUR_PASSWORD>>"
    email: "<<YOUR_EMAIL_ADDERESS>>"
  token: "<<KUBE_ENFORCER_GROUP_TOKEN>>"    # Optional: The KubeEnforcer group token (if not provided manual approval will be required)
 ```

#### Example: Deploy Aqua Scanner:

You can deploy more scanners, here is an example for Aqua Scanner deployment:
```yaml
apiVersion: operator.aquasec.com/v1alpha1
kind: AquaScanner
metadata:
  name: aqua
  namespace: aqua
spec:
  infra:
    serviceAccount: aqua-sa
    version: '5.3'
  deploy:
    replicas: 1
    image:
      registry: "registry.aquasec.com"
      repository: "scanner"
      tag: "<<IMAGE TAG>>"
  login:
    username: "<<YOUR AQUA USER NAME>>"
    password: "<<YOUR AQUA USER PASSWORD>>"
    host: 'http://aqua-server:8080'    #Required: provide <<(http:// or https://)Aqua Server IP OR DNS: Aqua Server Port>>
```
