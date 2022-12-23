### Readme for pipeline setup and deployment


Local K8s setup(Kind)

1. Install docker
2. Install docker-compose 
3. Install kind
4. Install kubectl

```
kind create cluster --name airflow-cluster --config kind-local-cluster-setup.yaml
```

You can do some checks

```
kubectl cluster-info
kubectl get nodes -o wide
```

To deploy Airflow on Kubernetes, the first step is to create a namespace.

```
kubectl create namespace airflow
kubectl get namespaces
```

Then, thanks to Helm, you need to fetch the official Helm of Apache Airflow that will magically get deployed on your cluster. Or, almost magically 😅

```
helm repo add apache-airflow https://airflow.apache.org
helm repo update
helm search repo airflow
helm install airflow apache-airflow/airflow --namespace airflow --debug
```

After a few minutes, you should be able to see your Pods running, corresponding to the different Airflow components.

```
kubectl get pods -n airflow
```

Don’t hesitate to check the current Helm release of your application with

```
helm ls -n airflow
```

Basically, each time your deploy a new version of your Airflow chart (after a modification or an update), you will obtain a new release. One of the most important field to take a look at is REVISION. This number will increase, if you made a mistake you can rollback to a previous revision with helm rollback.


To access the Airflow UI, open a new terminal and execute the following command

```
kubectl port-forward svc/airflow-webserver 8080:8080 -n airflow --context kind-airflow-cluster
```


Listen on port 8080 on all addresses, forwarding to 8080
```
kubectl port-forward svc/airflow-webserver 8080:8080 -n airflow --context kind-airflow-cluster --address 0.0.0.0
```


Make sure database settings  present in variables.yaml (POSTGRES_HOST and POSTGRES_PORT ) and the ones present in values.yml are pointing to the sbaiv2 database 

If you have some variables or connections that you want to export each time your Airflow instance gets deployed, you can define a ConfigMap. Open variables.yaml. This ConfigMap will export the environment variables under data. Great to have some bootstrap connections/variables.

```kubectl apply -f variables.yaml```



### Build the custom docker image and load it into your local Kubernetes cluster

```
# Run below commands from project root directory
docker build -t airflow-custom:1.0.0 -f pipeline/infra-setup/Dockerfile .
kind load docker-image airflow-custom:1.0.0 --name airflow-cluster
``` 
Note : If you are using MAC M1 machine please use following command to build image :

```
docker build --platform linux/amd64 -t airflow-custom:1.0.0 -f pipeline/infra-setup/Dockerfile . 
``` 

### Logs with Airflow on Kubernetes
With the KubernetesExecutor, when a task is triggered a POD is created. Once the task is completed, the corresponding POD gets deleted and so the logs.  Therefore, you need to find a way to store your logs somewhere so that you can still access them. For local development, the easiest way is to configure a HostPath PV. Let’s do it! First thing first, you should already have a folder data/ created next to the file kind-custer.yaml.

Next, to provide a durable location to prevent data from being lost, you have to set up the Persistent Volume.

```
kubectl apply -f pv.yaml
kubectl get pv -n airflow
```

Then, you create a Persistent Volume Claim so that you bind the PV with Airflow.
```
kubectl apply -f pvc.yaml 
kubectl get pvc -n airflow
```

And finally, deploy Airflow on Kubernetes again.
```
helm ls -n airflow 
helm upgrade --install airflow apache-airflow/airflow -n airflow -f values.yaml --timeout=10m --debug 
helm ls -n airflow
```



Some useful commands related to Helm(Rollback to specific deployment)
```
helm rollback airflow <release-version> -n airflow
```

In case of timeout error please increase timeout by adding --timeout=15m. 

The logs should be accessible in infra-setup/data folder. Please check that this folder has the right permissions. You can do so by executing the following command:

```
chmod 777 ./data
```

### Next Steps

- If the installation is done on a VM, it's possible to create a network tunnel to be able to view the Django admin and Airflow UI on local machine by running the following command with the port numbers for Airflow or Django: 

```
ssh -i<pemfile> -N -L 8000:<VM IP ADDRESS>:8000 <USERNAME>@<VM IP ADDRESS> -- to be executed on local machine 
```

- The pipelines are setup to ingest data from files hosted on s3. If you're using this storage service, you need to make sure your machine has IAM role to access ,list , delete and update files on s3 bucket you’re using for dags. 
When working with VMs, please contact gamma platform for creating IAM role to enable access to the s3 bucket.


### Debugging and useful commands

Here are some commands to help you debug errors. 

- To list pods and check which one is failing :
```
kubectl get pods -n airflow 
```

- To check logs for specific pod : 
```
Kubectl logs pod-name -n airflow 
```
- To execute scheduler pod and access airflow folders:
```
kubectl exec scheduler-pod-name -n airflow -it -- /bin/bash 
```
- To restore a specific version deployed 
```
helm rollback airflow <version-number> -n airflow 
```
-  To describe pod execution :  
```
kubectl describe pod pod-name –n airflow 
```

- In case of DAG Processor timeout error please increase timeout for the following parameters(AIRFLOW__CORE__DAG_FILE_PROCESSOR_TIMEOUT) in values.yaml file in pipeline/infra .Make sure to build the image and deploy it again. 
