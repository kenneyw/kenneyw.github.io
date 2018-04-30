# Telling Your Data To “Back Off!” (or How To Effectively Use Streams)

<figure>
  <img src="{{site.url}}/2017-11-01/Full-Stream-ghostbusters-27831704-500-209.gif" alt="my alt text"/>
  <figcaption>Careful Use of Streams</figcaption>
</figure>

# Foreword

Our Curations engineering team at [Bazaarvoice](https://www.bazaarvoice.com) makes heavy use of serverless architecture. While this typically gives us the benefit of reduced costs, flexibility, and rapid development, it also requires us to ensure that our processes will run within the tight memory and lifecycle constraints of serverless instances.

In this article, I will describe an actual case where a scheduled job had started to fail, the discovery of the root cause and the refactor that resolved the issue. I will assume you have at least a rudimentary knowledge of Node JS and Amazon Web Services.

## Content Export

Months previously, we built a scheduled job using AWS Lambdas that would kick off an export of our social content every 24 hours. In a nutshell, its purpose was to:

1. Query for social content documents stored in a MongoDB server.
1. Transform each document to a row in CSV format.
Write out the CSV rows to a file in S3.
1. The resulting CSV files were meant for ingestion into an analytics tool to measure web site conversion impact.

It all worked fine for months. Then recently, this job began to fail.

## Drowning in Content

For your viewing pleasure, here is an excerpted, condensed section of the main code:

```javascript
function runExport(query) {
  MongoClient.connect(MONGODB_URL)
    .then((db) => {
      let exportCount = 0;
      db
        .collection('content')
        .find(query)
        .forEach(
          // iterator function to perform on each document
          (content) => {
            // transform content doc to a csv row, then push to S3 writer
            s3.write(csvTransform(content));
            exportCount++;
          },
          // done function, no more documents
          () => {
            db.close();
            s3.close();
            logger.info(`documents exported: ${exportCount}`);
          }
        );
    });
}
```

Each document from the database query is transformed into a CSV row, then pushed to our S3 writer which buffered all content until an `s3.close()`. At this point, the CSV data would be flushed out to the S3 destination bucket.

It was obvious from this naïve implementation that heap usage would grow unbounded, as documents from the database would be loaded into resident memory as CSV data. As we aggregated content over the previous months, the data pushed into our export process began to generate frequent “out of heap memory” errors in our logs.

## Memory Profile

To gain visibility into the memory usage, I wrote a quick and dirty `MemProfiler` class whose purpose was to sample memory usage using `process.memoryUsage()`, collecting the maximum heap used over the process lifetime.

```javascript
class MemProfiler {
  constructor(collectInterval) {
    this.heapUsed = {
      min : null,
      max : null,
    };
    if (collectInterval) {
      this.start(collectInterval);
    }
  }

  start(collectInterval = 500) {
    this.interval = setInterval(() => this.sample(), collectInterval);
  }

  stop() {
    if (this.interval) {
      clearInterval(this.interval);
    }
  }

  reset() {
    this.stop();
    this.heapUsed = {
      min : null,
      max : null,
    };
  }

  sample() {
    const mem = process.memoryUsage();
    if (this.heapUsed.min === null || mem.heapUsed < this.heapUsed.min) {
      this.heapUsed.min = mem.heapUsed;
    }
    if (this.heapUsed.max === null || mem.heapUsed > this.heapUsed.max) {
      this.heapUsed.max = mem.heapUsed;
    }
  }
}
```

I then added the MemProfiler to the `runExport` code:

```javascript
function runExport(query) {
  memProf.start();
  MongoClient.connect(MONGODB_URL)
    .then((db) => {
      let exportCount = 0;
      db
        .collection('content')
        .find(query)
        .forEach(
          (content) => {
            // transform content doc to a csv row, then push to S3 writer
            s3.write(csvTransform(content));
            exportCount++;
            memProf.sample();
          },
          () => {
            db.close();
            s3.close();
            logger.info(`documents exported: ${exportCount}`);
            console.log('heapUsed:');
            console.log(` max: ${memProf.heapUsed.max/1024/1024} MB`);
            console.log(` min: ${memProf.heapUsed.min/1024/1024} MB`);
          }
        );
    });
}
```

Running the export against a small set of 16,900 documents yielded this result:

```
documents exported: 16900
heapUsed:
max: 30.32764434814453 MB
min: 8.863059997558594 MB
```

Then running against a larger set of 34,000 documents:

```
documents exported: 34000
heapUsed:
max: 36.204505920410156 MB
min: 8.962921142578125 MB
```

*Testing with additional documents increased the heap memory usage linearly as we expected.*

## Enter the Stream

The necessary code rewrite had one goal in mind: ensure the memory profile was constant, whether a total of one document was processed or a million. Since this was a batch process, we could trade off memory usage for processing time.

Let’s reiterate the primary operations:

1. Fetch a document
1. Transform the document to a CSV row
1. Write the CSV row to S3

If I may present an analogy, we have widgets moving down the assembly line from one station to the next. The conveyor belt on which the widgets are transported is moving at a constant rate, which means the throughput at each station is constant. We could increase the speed of that conveyor belt, but only as fast as can be handled by the slowest station.

In Node JS, streams are the assembly line stations, and pipes are the conveyor belt. Let’s examine the three primary types of streams:

1. **Readable streams**: This is typically an upstream data source that feeds data to a writable or transform stream. In our case, this is the MongoDB find query.
1. **Transform streams**: This typically takes upstream data, performs a calculation or transformation operation on it and feeds it out downstream. In our case, this would take a MongoDB document and transform it to a CSV row.
1. **Writable streams**: This is typically the terminating point of a data flow. In our case this is the writing of a CSV row to S3.

## Readable Stream

Conveniently, the MongoDB Node JS driver’s `find()` function returns a Cursor object that extends the Node JS Readable stream class. We can use it directly as a native stream. How convenient!

```javascript
const streamContentReader = db.collection('content').find(query);
```

## Transform Stream

All we need to do is extend the Transform stream class, and override the `_transform()` method to transform the content to a CSV row, push it out the other end, and signal upstream that it is ready for the next content.

```javascript
class CsvTransformer extends Transform {
  constructor(options) {
    super(options);
  }

  _transform(content, encoding, callback) {
    // write row
    const { _id, authoredAt, client, channel, mediaType, permalink , text, textLanguage } = content;
    const csvRow = `${_id},${authoredAt},${client},${channel},${mediaType},${permalink}${text},${textLanguage}\n`;
    this.push(csvRow);
    // signal upstream that we are ready to take more data
    callback();
  }
}
```

## Writable Stream

Let’s mock the S3 Writer for now. It does nothing here, but realistically, we would buffer up incoming data, then flush out the buffer in chunks over the network for best throughput.

```javascript
class S3Writer extends Writable {
  constructor(options) {
    super(options);
  }

  _write(data, encoding, callback) {
    // TODO: probably use a fixed size buffer
    // that we write to and then flush out to S3 using
    // multipart data upload
    callback();
  }
}
```

We can now create the three streams:

```javascript
const streamContentReader = db.collection('content').find(query);
const streamCsvTransformer = new CsvTransformer({ objectMode: true });
const streamS3Writer = new S3Writer();
```

## Pipe ‘Em Up

Streams alone do nothing. What makes them useful is connecting the output from one stream to the input of another. In stream terminology, this is called piping, and is accomplished using the stream’s [`pipe`](https://nodejs.org/api/stream.html#stream_readable_pipe_destination_options) method:

```javascript
streamContentReader.pipe(streamCsvTransformer).pipe(streamS3Writer);
```

Although the native Node JS pipe method is perfectly functional, I highly recommend using the [`pump`](https://www.npmjs.com/package/pump-promise) library. In addition to being more readable by passing in an the ordered list streams to pipe together, we can also invoke `then()` when the pipe has finished, as well as handle `close` or `error` events emitted by the streams:

```javascript
pump([ streamContentReader, streamCsvTransformer, streamS3Writer ])
  .then(() => {
    console.log(`documents exported: ${exportCount}`);
  })
  .catch((err) => console.error(err));
```

## CSV and S3 Write Streams

I purposely did not implement the S3Writer above, because npm is a treasure trove of solutions! The [s3-write-stream](https://www.npmjs.com/package/s3-write-stream) module will take care of buffering, using multipart upload and handling retries, so we don’t have to get our hands too dirty. And wouldn’t you know it, there is also the [csv-write-stream](https://www.npmjs.com/package/csv-write-stream) module that will generate properly escaped CSV rows.

As an exercise, you the reader may want to try using the these or other streaming modules. Trust me, once you get the hang of streaming, you will enjoy it!

## Back Off!

Let’s touch back on the issue of memory usage.

What if a write stream becomes blocked for a period of time? Maybe we are writing data to a third-party service over HTTP and network congestion or service throttling is slowing the write rate. The readable stream will happily pump data in as fast as it can. Won’t that cause increased heap usage as all that data gets backed up and buffered?

The quick answer is “no”. In the implementation of the `_write` method of your writable stream, when done with the data chunk (after writing to a file for example), call `callback`. This signals to the incoming stream that it is ready to receive the next chunk. You can also provide a `highWaterMark` to the writable stream constructor to give a specific number of objects to buffer before pausing the incoming stream. This is called **back pressure control**. This is all handled internally by the Node JS streams implementation. No more unbounded buffering!

For a deep dive into the concept of back pressure control, read this great article on [Backpressuring in Streams](https://nodejs.org/en/docs/guides/backpressuring-in-streams/).


## Stream ‘Em If You Got ’em!

Our team is now in the process of utilizing Node JS streams whenever we read in volumes of data, process that data, then write out the results. We are now seeing great gains in both stability and maintainability!

### References

* [Node JS Streams API](https://nodejs.org/api/stream.html)
* [Article: Backpressuring in Streams](https://nodejs.org/en/docs/guides/backpressuring-in-streams/)
* [MongoDB Cursor Stream](http://mongodb.github.io/node-mongodb-native/2.2/api/Cursor.html)
* [Pump Module](https://www.npmjs.com/package/pump)
* [CSV Write Stream Module](https://www.npmjs.com/package/csv-write-stream)
* [S3 Write Stream Module](https://www.npmjs.com/package/s3-write-stream)
