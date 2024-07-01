# highlight in flink doc
* Flink is a streaming data system in its core, that executes “batch as a special case of streaming”, which 
is not the same as spark streaming, spark treat streaming as micro-batch, so in its design, the flink will 
have less latency than spark streaming.

* Furthermore, streaming applications often need to be complemented by batch (bounded stream) processing, for example when reprocessing data after bugs or data quality issues, or when bootstrapping new applications. A unified API and system make this much easier.