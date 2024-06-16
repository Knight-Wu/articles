报错示例如下: 
```
Exception in thread "main" org.apache.spark.SparkException: Task not serializable
at org.apache.spark.util.ClosureCleaner$.ensureSerializable(ClosureCleaner.scala:166)
at org.apache.spark.util.ClosureCleaner$.clean(ClosureCleaner.scala:158)
at org.apache.spark.SparkContext.clean(SparkContext.scala:1242)
at org.apache.spark.rdd.RDD.map(RDD.scala:270)
at org.apache.spark.api.java.JavaRDDLike$class.mapToPair(JavaRDDLike.scala:99)
at org.apache.spark.api.java.JavaRDD.mapToPair(JavaRDD.scala:32)
at fr.aid.cim.spark.dao.GenrateCustumorJourney.AllCleintInteractions(GenrateCustumorJourney.java:91)
at fr.aid.cim.spark.dao.GenrateCustumorJourney.main(GenrateCustumorJourney.java:75)
at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
at java.lang.reflect.Method.invoke(Method.java:483)
at org.apache.spark.deploy.SparkSubmit$.launch(SparkSubmit.scala:328)
at org.apache.spark.deploy.SparkSubmit$.main(SparkSubmit.scala:75)
at org.apache.spark.deploy.SparkSubmit.main(SparkSubmit.scala)
Caused by: java.io.NotSerializableException: org.apache.spark.api.java.JavaSparkContext
at java.io.ObjectOutputStream.writeObject0(ObjectOutputStream.java:1184)
at java.io.ObjectOutputStream.defaultWriteFields(ObjectOutputStream.java:1548)
at java.io.ObjectOutputStream.writeSerialData(ObjectOutputStream.java:1509)
at java.io.ObjectOutputStream.writeOrdinaryObject(ObjectOutputStream.java:1432)
at java.io.ObjectOutputStream.writeObject0(ObjectOutputStream.java:1178)
at java.io.ObjectOutputStream.defaultWriteFields(ObjectOutputStream.java:1548)
at java.io.ObjectOutputStream.writeSerialData(ObjectOutputStream.java:1509)
at java.io.ObjectOutputStream.writeOrdinaryObject(ObjectOutputStream.java:1432)
at java.io.ObjectOutputStream.writeObject0(ObjectOutputStream.java:1178)
at java.io.ObjectOutputStream.writeObject(ObjectOutputStream.java:348)
at org.apache.spark.serializer.JavaSerializationStream.writeObject(JavaSerializer.scala:42)
at org.apache.spark.serializer.JavaSerializerInstance.serialize(JavaSerializer.scala:73)
at org.apache.spark.util.ClosureCleaner$.ensureSerializable(ClosureCleaner.scala:164)
... 14 more
```
把SparkContext 用static 修饰, 
[https://blog.csdn.net/LL9504/article/details/89310030](https://blog.csdn.net/LL9504/article/details/89310030)


> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbNjMxODExMDIzXX0=
-->