# Google Spark Operator

A better way to submit Spark job to a Kubernetes cluster.

## Installation

```sh
helm repo add spark-operator https://googlecloudplatform.github.io/spark-on-k8s-operator
helm install sparkoperator spark-operator/spark-operator --namespace spark-operator --create-namespace
```

## Overview

```txt
kubectl get deploy,pod
NAME                                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/sparkoperator-spark-operator   1/1     1            1           21h

NAME                                                READY   STATUS    RESTARTS   AGE
pod/sparkoperator-spark-operator-7995777d5c-xp6w4   1/1     Running   0          16h
```

## Sparkctl

`sparkctl` can be used instead of `kubectl` to interact with the operator.

## Submit jobs

```sh
sparkctl create -n spark-operator pypi.yaml -d
SparkApplication "pyspark-pi" created

OR

kubectl apply -f pypi.yaml

```

```sh
sparkctl list -n spark-operator
+------------+-----------+----------------+-----------------+
|    NAME    |   STATE   | SUBMISSION AGE | TERMINATION AGE |
+------------+-----------+----------------+-----------------+
| pyspark-pi | SUBMITTED | 12s            | N.A.            |
+------------+-----------+----------------+-----------------+
```

```sh
kubectl get pods
NAME                                            READY   STATUS              RESTARTS   AGE
pyspark-pi-driver                               1/1     Running             0          24s
pythonpi-9ab8b08779d76b79-exec-1                1/1     Running             0          16s
pythonpi-9ab8b08779d76b79-exec-2                0/1     ContainerCreating   0          16s
pythonpi-9ab8b08779d76b79-exec-3                0/1     ContainerCreating   0          15s
sparkoperator-spark-operator-7995777d5c-xp6w4   1/1     Running             0          16h
```

```sh
sparkctl list -n spark-operator
+------------+-----------+----------------+-----------------+
|    NAME    |   STATE   | SUBMISSION AGE | TERMINATION AGE |
+------------+-----------+----------------+-----------------+
| pyspark-pi | COMPLETED | 21h            | 21h             |
+------------+-----------+----------------+-----------------+
```

```sh
sparkctl event pyspark-pi -n spark-operator
+------------+--------+----------------------------------------------------+
|    TYPE    |  AGE   |                      MESSAGE                       |
+------------+--------+----------------------------------------------------+
| Normal     | 10m    | SparkApplication pyspark-pi                        |
|            |        | was added, enqueuing it for                        |
|            |        | submission                                         |
| Normal     | 10m    | SparkApplication pyspark-pi                        |
|            |        | was submitted successfully                         |
| Normal     | 10m    | Driver pyspark-pi-driver is                        |
|            |        | running                                            |
| Normal     | 10m    | Executor                                           |
|            |        | [pythonpi-e230df8779c8f671-exec-1]                 |
|            |        | is pending                                         |
| Normal     | 10m    | Executor                                           |
|            |        | [pythonpi-e230df8779c8f671-exec-1]                 |
|            |        | is running                                         |
| Normal     | 10m    | Executor                                           |
|            |        | [pythonpi-e230df8779c8f671-exec-1]                 |
|            |        | completed                                          |
| Normal     | 10m    | Driver pyspark-pi-driver                           |
|            |        | completed                                          |
| Normal     | 10m    | SparkApplication pyspark-pi                        |
|            |        | completed                                          |
+------------+--------+----------------------------------------------------+
````

## Read logs

```sh
sparkctl log pyspark-pi -n spark-operator | grep "Pi is roughly"
Pi is roughly 3.136840
```

## Code reference

```sh
docker run -it --rm gcr.io/spark-operator/spark-py:v3.1.1 cat /opt/spark/examples/src/main/python/pi.py
```

```python
import sys
from random import random
from operator import add

from pyspark.sql import SparkSession


if __name__ == "__main__":
    """
        Usage: pi [partitions]
    """
    spark = SparkSession\
        .builder\
        .appName("PythonPi")\
        .getOrCreate()

    partitions = int(sys.argv[1]) if len(sys.argv) > 1 else 2
    n = 100000 * partitions

    def f(_):
        x = random() * 2 - 1
        y = random() * 2 - 1
        return 1 if x ** 2 + y ** 2 <= 1 else 0

    count = spark.sparkContext.parallelize(range(1, n + 1), partitions).map(f).reduce(add)
    print("Pi is roughly %f" % (4.0 * count / n))

    spark.stop()
```
