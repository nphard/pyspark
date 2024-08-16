# install in kubernetes
using the apache spark.
```bash
helm repo add spark-operator https://kubeflow.github.io/spark-operator 
helm install spark-operator spark-operator/spark-operator \
    --namespace spark-operator \
    --create-namespace
```

# RDD

