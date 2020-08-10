```
 sudo -u hdfs spark2-submit --executor-cores 2 --master yarn  --deploy-mode cluster --name  sparkPi --executor-memory 2g --num-executors 3  \
 --class org.apache.spark.examples.SparkPi \
   --conf spark.executor.extraJavaOptions="-XX:+UseG1GC"  \
 /opt/cloudera/parcels/SPARK2/lib/spark2/examples/jars/spark-examples_2.11-2.4.0.cloudera2.jar
```


> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTI5MTU0OTA5Nl19
-->