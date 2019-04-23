## spark-summit-2019-demo
Demo created for [Life is but a Stream](https://databricks.com/sparkaisummit/north-america/sessions-single-2019?id=125) presentation at Spark AI Summit 2019 (San Francisco, CA). The Notebook found within this repository is meant to be run on a Databricks cluster. If you do not have access to one already, you can [signup](https://databricks.com/try-databricks) to run the demo.

If you simply want to see the results and visuals that were generated at each step of the demo, then you can view the _SEC Apache Logs.html_ page contents. Otherwise, follow the setup steps below to be able to generate these results for yourself. Note that this content was originally created using Spark 2.4.0 with the Databricks Runtime (DBR) Version 5.3.

### AWS Setup
1. Create/log into your AWS account and establish an access key and secret using IAM.
2. Create an S3 bucket and an associated [Databricks mount](https://docs.databricks.com/spark/latest/data-sources/aws/amazon-s3.html#mount-an-s3-bucket). Configure the mount name within the _SEC Apache Logs_ notebook, in the first code cell.
3. Create a Kinesis Data Stream called _spark-summit-2019_ with the default settings.
4. Create a DynamoDB table called _sec-company-lookup_, with a primary partition key called _cik_ (STRING). Override the default settings to configure the table to use ON-DEMAND capacity.
5. Create a Lambda function called _sec-company-lookup_, targeting _Node.js 8.10_, using the contents of _lambda.js_.
6. Create an AWS Gateway API that passes requests through to the _sec-company-lookup_ Lambda function. Deploy the API Gateway, and copy the public URL into Notebook cell 26.

### Data Setup
1. Download [SEC EDGAR web server logs](https://www.sec.gov/dera/data/edgar-log-file-data-set.html) to your local computer.
2. Upload the CSV content to a sub-folder within your AWS S3 bucket called _sec-logs_.
3. Download the [IANA status codes](https://www.iana.org/assignments/http-status-codes/http-status-codes-1.csv), and upload the CSV content to a sub-folder within your AWS S3 bucket called _http-status-codes_.
4. Launch a Databricks Spark cluster, which has the _AWS_ACCESS_KEY_ID_, _AWS_SECRET_ACCESS_KEY_, and _AWS_REGION_ environment variables configured to use your AWS IAM credentials.
5. Attach a new Notebook to this cluster, and run the following code. This will populate your DynamoDB table with the necessary _cik_ -> _company name_ information.

```scala
import com.amazonaws.services.dynamodbv2.AmazonDynamoDBClientBuilder;
import com.amazonaws.services.dynamodbv2.model.AttributeValue;
import scala.collection.JavaConverters._

// Ensure that this mount has already been established; it can point to any S3 bucket within your AWS account.
val dataMount = "/mnt/spark-summit-2019-demo"

spark
  .read
  .text(s"$dataMount/cik-ticker/company")
  .map(c => {
    val line = c.getString(0)
    val name = line.substring(0, 62).trim
    val cik = line.substring(74, 86).trim
   (cik, name)
  }).as[(String, String)]
  .distinct()
  .foreach(r => {
    val dynamoDB = AmazonDynamoDBClientBuilder
      .standard()
      .build()

    dynamoDB.putItem("sec-company-lookup", Map(
      ("cik" -> new AttributeValue(r._1)),
      ("name" -> new AttributeValue(r._2))
    ).asJava)
  })
```
  6. Run the following code to generate files representing the hourly activity within the SEC server logs.

```scala
import org.apache.spark.sql.types.StructType
import org.apache.spark.sql.catalyst.ScalaReflection
import org.apache.spark.sql.functions._

// Ensure that this mount has already been established; it can point to any S3 bucket within your AWS account.
val dataMount = "/mnt/spark-summit-2019-demo"

case class Log(ip: String, date: String, time: String, zone: Float, cik: Float, accession: String, extention: String, code: Float, size: Float, idx: Float, norefer: Float, noagent: Float, find: Float, crawler: Float, browser: String)

val logSchema = ScalaReflection.schemaFor[Log].dataType.asInstanceOf[StructType]

// Load the raw logs from S3.
val logs = spark
  .read
  .option("header", "true")
  .schema(logSchema)
  .csv(s"$dataMount/sec-logs/")

// Break them down by hour of the day.
for(i <- 0 to 23) {
  logs
    .filter(hour(format_string("%s %s", $"date", $"time")) === i)
    .repartition(1)
    .write
    .csv(s"$dataMount/sec-logs-hourly/$i/")
}
```

### Publishing Data
1. Ensure that Docker is installed and running on your computer. Clone the [Docker container provided by Zendesk](https://github.com/zendesk/docker-amazon-kinesis-agent).

```bash
git clone https://github.com/zendesk/docker-amazon-kinesis-agent.git
```
2. Traverse into the cloned repository, and update the _agent.json_ file to contain the name of your Kinesis data stream.

```json
{
  "cloudwatch.emitMetrics": true,
  "kinesis.endpoint": "kinesis.us-east-1.amazonaws.com",
  "cloudwatch.endpoint": "monitoring.us-east-1.amazonaws.com",
  "flows": [
    {
      "filePattern": "/tmp/log-input/*.*",
      "kinesisStream": "spark-summit-2019"
    }
  ]
}
```

3. Run the following commands to build and execute the container. When the container is running, this will mount the input folder to the local path _~/kinesis-logs/_. Create that in advance, or choose and configure a different local path.

```bash
docker build -t spark-summit-2019/kinesis-agent .
docker run -v ~/kinesis-logs:/tmp/log-input/ spark-summit-2019/kinesis-agent:latest
```

4. Now that the Kinesis data agent is running (via Docker), anytime a file is placed inside of the _~/kinesis-logs/_ folder it's contents will be parsed and published to the Kinesis data stream. You can drop the SEC web server logs into this location while running the Spark Structured Streaming portion of the demo Notebook to see the magic come to life.
