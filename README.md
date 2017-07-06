# Spark-on-Yarn
Running Spark On YARN


When it comes to running different distributed applications besides Spark, running Spark in standalone mode (with the embedded cluster manager) is not a good choice for better cluster resources utilization. It would be better, in terms of resources scheduling and utilization, if there is a single cluster manager that has a global view of what is running and want to run on the cluster.
<p>Without that single cluster manager, there are two main approaches for resources sharing and allocation:</p>
&nbsp;

<ol>
	<li>
	<p>Availing all cluster resources to all types of applications in the same time. However, that would lead to a great unfairly managed contention on resources.</p>
	</li>
	<li>
	<p>Dividing the pool of resources into smaller pools; a pool for each application type. However, that would lead to inefficient utilization of resources as some applications might require more resources than allocated in the corresponding pool while in the same time less resources are sufficient for other applications. Hence, a dynamic way of resources allocation leads to better resources utilization.</p>
	</li>
</ol>
&nbsp;

<p>There are different cluster managers that can do the job and overcome the issues highlighted above. Choosing one depends on the types of applications being run on the cluster as they all should speak the language of the manager. One of the cluster managers, that Spark applications can run on, is Apache YARN. The design of Apache YARN allows different YARN applications to coexist on the same cluster, so a Spark application can run at the same time as other types of applications (like Hadoop MapReduce jobs) which brings great benefits for manageability and cluster utilization.</p>
&nbsp;

<p>In this post we will illustrate what are the benefits of running Spark on YARN, how to run Spark on YARN and we will mention important notes to take care of when running Spark on YARN.</p>
&nbsp;

<p>First we will try to understand the architecture of both Spark and YARN.</p>

<h2>Spark Architecture:</h2>

<p>Spark Architecture consists of Driver Program , Executors and Cluster Manager.</p>
&nbsp;

<div style="text-align: center;"><img alt="" src="https://spark.apache.org/docs/latest/img/cluster-overview.png" style="height:250px; width:534px" /></div>
&nbsp;

<p><strong>Driver Program:</strong>The driver program is responsible for managing the job flow and scheduling tasks that will run on the executors.</p>

<p><strong>Executors:</strong> Executors are processes that run computation and store data for a Spark application.</p>

<p><strong>Cluster Manager:</strong>Cluster Manager is responsible for starting executor processes and where and when they will be run. Spark supports pluggable cluster manager, it supports (YARN, Mesos, and its own &ldquo;standalone&rdquo; cluster manager)</p>
&nbsp;

<h2>YARN Architecture:</h2>

<p>YARN Architecture consists of Resource Manager, Node Manager, Application Master and Container.</p>
&nbsp;

<div style="text-align: center;"><img alt="" src="http://hadoop.apache.org/docs/current/hadoop-yarn/hadoop-yarn-site/yarn_architecture.gif" style="height:385px; width:622px" /></div>
&nbsp;

<p><strong>Resource Manager:</strong> manages the use of resources across the cluster.</p>

<p><strong>Node Manager:</strong> launches and monitors containers on cluster machines.</p>

<p><strong>Application Master:</strong> manages the lifecycle of an application running on the cluster.</p>

<p><strong>Container: </strong> It represents a collection of physical resources (CPU cores + memory) on a single node at a cluster. Those resources are allocated for the use of a worker slave.</p>
&nbsp;

<h2>Spark on YARN:</h2>

<p>When running Spark on YARN each Spark executor runs as YARN container. Spark supports two modes for running on YARN, yarn-cluster mode and yarn-client mode.</p>
&nbsp;

<h3>YARN-Client Mode:</h3>

<ul>
	<li>In yarn-client mode, the driver runs in the client process, and the application master is only used for requesting resources from YARN.</li>
	<li>yarn-client mode makes sense for interactive and debugging uses where you want to see your application&rsquo;s output immediately (on the client process side).</li>
</ul>
&nbsp;

<div style="text-align: center;"><img alt="" src="http://blog.cloudera.com/wp-content/uploads/2014/05/spark-yarn-f22.png" style="height:493px; width:620px" /></div>
&nbsp;

<h3>YARN-Cluster Mode:</h3>

<ul>
	<li>In yarn-cluster mode, the Spark driver runs inside an application master process which is managed by YARN on the cluster, and the client can go away after initiating the application.</li>
	<li>yarn-cluster mode makes sense for production jobs.</li>
</ul>
&nbsp;

<div style="text-align: center;"><img alt="" src="http://blog.cloudera.com/wp-content/uploads/2014/05/spark-yarn-f31.png" style="height:493px; width:620px" /></div>
&nbsp;

<h2>Why Run on YARN?</h2>

<p>Running Spark on YARN has some benefits:</p>

<ul>
	<li>YARN allows to dynamically share the cluster resources between different frameworks that run on YARN. For example, you can run a mapreduce job after that you can run a Spark job without any changes in YARN configurations.</li>
	<li>You can use YARN schedulers for categorizing, isolating, and prioritizing workloads.</li>
	<li>YARN is the only cluster manager for Spark that supports security. With YARN, Spark can use secure authentication between its processes.</li>
</ul>

<h2>How to Run on YARN</h2>

<p>We used cloudera manager to install Spark and YARN. There wasn&rsquo;t any special configuration to get Spark just run on YARN, we just changed Spark&rsquo;s master address to yarn-client or yarn-cluster.</p>
&nbsp;

<p>We want to mention some important issues that we have met during running Spark on YARN:</p>

<ul>
	<li>Spark copies the Spark assembly JAR file to HDFS each time you run spark-submit. You can avoid doing this copy each time by manually uploading the Spark assembly JAR file to your HDFS. Then, set the SPARK_JAR environment variable to this HDFS path<br />
	&nbsp;
	<pre class="prettyprint linenums lang-c_pp prettyprint linenums" data-pbcklang="bash" data-pbcktabsize="4">
hdfs dfs -mkdir -p /user/spark/share/lib
hdfs dfs -put $SPARK_HOME/assembly/lib/spark-assembly_*.jar &nbsp;\ &nbsp;&nbsp;&nbsp;&nbsp;
/user/spark/share/lib/spark-assembly.jar
export SPARK_JAR=hdfs://&lt;nn&gt;:&lt;port&gt;/user/spark/share/lib/spark-assembly.jar
</pre>
	</li>
	<br />
	<li>Important configuration during submitting the job:
	<pre class="prettyprint linenums lang-c_pp prettyprint linenums" data-pbcklang="bash" data-pbcktabsize="4">
--executor-cores NUM &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Number of cores per executor (Default: 1)
--num-executors NUM &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Number of executors to launch (Default: 2)
--executor-memory NUM &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Amount of memory to use per executor process.
--driver-memory NUM &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Amount of memory to use for the driver process.
</pre>
	</li>
	<br />
	<li>We noticed that YARN uses more memory than we set for each executor, after searching we discovered that YARN uses:
	<ul>
		<li>executor memory + spark.yarn.executor.memoryOverhead for the executor.</li>
		<li>driver memory + spark.yarn.driver.memoryOverhead for the driver.</li>
		<li>We found that this memory overhead is the amount of off-heap memory (in megabytes) to be allocated per executor or driver. This is the memory that accounts for things like VM overheads, interned strings, other native overheads, etc. This tends to grow with the executor size (typically 6-10%).</li>
	</ul>
	</li>
	<br />
	<li>The local directories, used by Spark executors in saving map output files and RDDs that are stored on disk, will be the local directories configured for YARN (Hadoop YARN config yarn.nodemanager.local-dirs). If the user specifies spark.local.dir, it will be ignored.</li>
	<br />
	<li>Sharing application files (e.g. jar) with executors:
	<ul>
		<li>In yarn-client mode and Spark Standalone mode a link to the jar at the client machine is created and all executors receive this link to download the jar.</li>
		<li>In yarn-cluster mode, the jar is uploaded to hdfs before running the job and all executors download the jar from hdfs, so it takes some time at the beginning to upload the jar.</li>
	</ul>
	</li>
</ul>

<h2>Comparison Between Spark on YARN and Standalone mode:</h2>

<style>
	table.custom thead th, table.custom tbody tr td{
    	text-align: center !important;
	}
</style>

<table align="center" border="1" cellpadding="1" cellspacing="1" class="custom">
	<tbody>
		<tr>
			<td rowspan="2" scope="row"></td>
			<td colspan="2" rowspan="1"><strong>Spark On YARN</strong></td>
			<td colspan="1" rowspan="2" scope="col"><strong>Spark Standalone</strong></td>
		</tr>
		<tr>
			<td><strong>yarn-cluster</strong></td>
			<td><strong>yarn-client</strong></td>
		</tr>
		<tr>
			<th scope="row">Driver runs in</th>
			<td>Application Master</td>
			<td>Client</td>
			<td>Client</td>
		</tr>
		<tr>
			<th scope="row">Who requests resources</th>
			<td colspan="2" rowspan="1">Application Master</td>
			<td>Client</td>
		</tr>
		<tr>
			<th scope="row">Who starts executor processes</th>
			<td colspan="2" rowspan="1">YARN NodeManager</td>
			<td>Spark Worker (Slave)</td>
		</tr>
		<tr>
			<th scope="row">Support for Sparkshell</th>
			<td>No</td>
			<td>Yes</td>
			<td>Yes</td>
		</tr>
		<tr>
			<th scope="row">Sharing jar with executors</th>
			<td>uploads jar to hdfs</td>
			<td>creates link for the jar on the client</td>
			<td>creates link for the jar on the client</td>
		</tr>
		<tr>
			<th scope="row">Share cluster resources among different frameworks</th>
			<td colspan="2" rowspan="1">Yes</td>
			<td>No</td>
		</tr>
	</tbody>
</table>
&nbsp;

<h2>Example:</h2>

<h3>Running SparkPi in Standalone Mode</h3>

<pre class="prettyprint linenums lang-c_pp prettyprint linenums" data-pbcklang="bash" data-pbcktabsize="4">
spark-submit \
--master spark://$SPARK_MASTER_IP:$SPARK_MASTER_PORT \
--class org.apache.spark.examples.SparkPi \
$SPARK_HOME/examples/lib/spark-examples_version.jar 10
</pre>

<h3>Running SparkPi in YARN Client Mode</h3>

<pre class="prettyprint linenums lang-c_pp prettyprint linenums" data-pbcklang="bash" data-pbcktabsize="4">
spark-submit \
--master yarn-client \
--class org.apache.spark.examples.SparkPi \
$SPARK_HOME/examples/lib/spark-examples_version.jar 10
</pre>

<h3>Running SparkPi in YARN Cluster Mode</h3>

<pre class="prettyprint linenums lang-c_pp prettyprint linenums" data-pbcklang="bash" data-pbcktabsize="4">
spark-submit \
--master yarn-cluster \
--class org.apache.spark.examples.SparkPi \
$SPARK_HOME/examples/lib/spark-examples_version.jar 10
</pre>
