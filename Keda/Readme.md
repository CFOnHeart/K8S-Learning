## An Keda Demo of Redis

### Preparation
1. aks
2. azure client

### Steps
#### 1. Install Redis
```bash
$Location="westus2"
$RESOURCE_GROUP="jun-dev"  # you should have created specific resource-group in your subscription already
$REDIS_NAME="redis-contoso-video-1"
az redis create --location $LOCATION --name $REDIS_NAME --resource-group $RESOURCE_GROUP --sku Basic --vm-size c0 --enable-non-ssl-port

$REDIS_HOST=$(az redis show -n $REDIS_NAME -g $RESOURCE_GROUP -o tsv --query "hostName")
$REDIS_KEY=$(az redis list-keys --name $REDIS_NAME --resource-group $RESOURCE_GROUP -o tsv --query "primaryKey")
```

#### 2. Deploy Keda
```bash
>> kubectl apply -f https://github.com/kedacore/keda/releases/download/v2.2.0/keda-2.2.0.yaml
namespace/keda unchanged
customresourcedefinition.apiextensions.k8s.io/clustertriggerauthentications.keda.sh configured
customresourcedefinition.apiextensions.k8s.io/scaledjobs.keda.sh configured
customresourcedefinition.apiextensions.k8s.io/scaledobjects.keda.sh configured
customresourcedefinition.apiextensions.k8s.io/triggerauthentications.keda.sh configured
serviceaccount/keda-operator unchanged
clusterrole.rbac.authorization.k8s.io/keda-external-metrics-reader unchanged
clusterrole.rbac.authorization.k8s.io/keda-operator configured
rolebinding.rbac.authorization.k8s.io/keda-auth-reader unchanged
clusterrolebinding.rbac.authorization.k8s.io/keda-hpa-controller-external-metrics unchanged
clusterrolebinding.rbac.authorization.k8s.io/keda-operator unchanged
clusterrolebinding.rbac.authorization.k8s.io/keda:system:auth-delegator unchanged
service/keda-metrics-apiserver created
deployment.apps/keda-metrics-apiserver created
deployment.apps/keda-operator created
apiservice.apiregistration.k8s.io/v1beta1.external.metrics.k8s.io unchanged
>> kubectl get all --namespace keda
NAME                                          READY   STATUS    RESTARTS   AGE
pod/keda-metrics-apiserver-55dc9f9498-wrmlw   0/1     Running   0          8s
pod/keda-operator-59dcf989d6-kqb2j            0/1     Running   0          8s

NAME                             TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)          AGE
service/keda-metrics-apiserver   ClusterIP   10.0.207.79   <none>        443/TCP,80/TCP   10s

NAME                                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/keda-metrics-apiserver   0/1     1            0           9s
deployment.apps/keda-operator            0/1     1            0           9s

NAME                                                DESIRED   CURRENT   READY   AGE
replicaset.apps/keda-metrics-apiserver-55dc9f9498   1         1         0       10s
replicaset.apps/keda-operator-59dcf989d6            1         1         0       10s
```

### 3. Deploy Redis client
```bash
>> kubectl apply -f ./redis-client-deployment.yaml -n ganjun
```

### 4. Deploy a scaler object 
```bash
kubectl apply -f ./keda-scaled-object.yaml -n ganjun
kubectl get ScaledObject -n ganjun # you should see this object ready
```

### 5. Watch the scaling
According to the  keda-scaled-object.yaml, we set the scale trigger when list(named as keda) in Redis length > 10. So we need to create a "keda" list and push many items into it. 
```bash
>> docker run -it --rm redis redis-cli -h $REDIS_HOST -a $REDIS_KEY
>> lpush keda Lorem ipsum dolor sit amet, consectetur adipiscing elit. Mauris eget interdum felis, ac ultricies nulla. Fusce vehicula mattis laoreet. Quisque facilisis bibendum dui, at scelerisque nulla hendrerit sed. Sed rutrum augue arcu, id maximus felis sollicitudin eget. Curabitur non libero rhoncus, pellentesque orci a, tincidunt sapien. Suspendisse laoreet vulputate sagittis. Vivamus ac magna lacus. Etiam sagittis facilisis dictum. Phasellus faucibus sagittis libero, ac semper lorem commodo in. Quisque tortor lorem, sollicitudin non odio sit amet, finibus molestie eros. Proin aliquam laoreet eros, sed dapibus tortor euismod quis. Maecenas sed viverra sem, at porta sapien. Sed sollicitudin arcu leo, vitae elementum
>> llen keda
```

Then you can see the pod auto-scaling by "kubectl get pod -w -n ganjun"

## Reference
+ https://docs.microsoft.com/en-us/learn/modules/aks-app-scale-keda/1-introduction
