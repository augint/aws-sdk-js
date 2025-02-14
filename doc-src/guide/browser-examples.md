# @title Examples in the Browser

# Examples in the Browser

All of these examples assume that the AWS library is loaded, configured,
and authenticated with the correct credentials.

The common preamble code can be summarized as follows:

    <script src="https://sdk.amazonaws.com/js/aws-sdk-2.3.5.min.js"></script>
    <script type="text/javascript">
      // See the Configuring section to configure credentials in the SDK
      AWS.config.credentials = ...;

      // Configure your region
      AWS.config.region = 'us-west-2';
    </script>

## Basic Usage Example

The following example shows basic usage of the SDK to list objects in an
Amazon S3 bucket:

    <div id="status"></div>
    <ul id="objects"></ul>

    <script type="text/javascript">
      var bucket = new AWS.S3({params: {Bucket: 'myBucket'}});
      bucket.listObjects(function (err, data) {
        if (err) {
          document.getElementById('status').innerHTML =
            'Could not load objects from S3';
        } else {
          document.getElementById('status').innerHTML =
            'Loaded ' + data.Contents.length + ' items from S3';
          for (var i = 0; i < data.Contents.length; i++) {
            document.getElementById('objects').innerHTML +=
              '<li>' + data.Contents[i].Key + '</li>';
          }
        }
      });
    </script>

## Amazon Elastic Compute Cloud (Amazon EC2)

### Amazon EC2: Creating an Instance with Tags (`runInstances`, `createTags`)

The Amazon EC2 API has two distinct operations for creating instances and
attaching tags to instances. In order to create an instance with tags, you can
call both of these operations in series. The following example adds a "Name"
tag to a new instance, which the Amazon EC2 console recognizes and displays
in the Name field of the instance list.

```javascript
var ec2 = new AWS.EC2();

var params = {
  ImageId: 'ami-1624987f', // Amazon Linux AMI x86_64 EBS
  InstanceType: 't1.micro',
  MinCount: 1, MaxCount: 1
};

// Create the instance
ec2.runInstances(params, function(err, data) {
  if (err) { console.log("Could not create instance", err); return; }

  var instanceId = data.Instances[0].InstanceId;
  console.log("Created instance", instanceId);

  // Add tags to the instance
  params = {Resources: [instanceId], Tags: [
    {Key: 'Name', Value: 'instanceName'}
  ]};
  ec2.createTags(params, function(err) {
    console.log("Tagging instance", err ? "failure" : "success");
  });
});
```

Note that you can add up to 10 tags to an instance, and they can be all added
in a single call to `createTags`.

## Amazon S3

### Uploading data into an object

<p class="note">
  In order to upload files in the browser, you should ensure that you
  have configured CORS for your Amazon S3 bucket and exposed the "ETag"
  header via the <code>&lt;ExposeHeader&gt;ETag&lt;/ExposeHeader&gt;</code>
  declaration. See the {file:browser-configuring.md Configuring} section
  for more information on configuring CORS for an Amazon S3 bucket.
</p>

The following example will upload the contents of a `<textarea>` tag to an
object in S3:

    <textarea id="data"></textarea>
    <button id="upload-button">Upload to S3</button>
    <div id="results"></div>

    <script type="text/javascript">
      var bucket = new AWS.S3({params: {Bucket: 'myBucket'}});

      var textarea = document.getElementById('data');
      var button = document.getElementById('upload-button');
      var results = document.getElementById('results');
      button.addEventListener('click', function() {
        results.innerHTML = '';

        var params = {Key: 'data.txt', Body: textarea.value};
        bucket.upload(params, function (err, data) {
          results.innerHTML = err ? 'ERROR!' : 'SAVED.';
        });
      }, false);
    </script>

### Uploading a local file using the File API

<p class="note">
  In order to upload files in the browser, you should ensure that you
  have configured CORS for your Amazon S3 bucket and exposed the "ETag"
  header via the <code>&lt;ExposeHeader&gt;ETag&lt;/ExposeHeader&gt;</code>
  declaration. See the {file:browser-configuring.md Configuring} section
  for more information on configuring CORS for an Amazon S3 bucket.
</p>

The following example uses the [HTML5 File API](http://www.w3.org/TR/FileAPI/)
to upload a file on disk to S3:

    <input type="file" id="file-chooser" /> 
    <button id="upload-button">Upload to S3</button>
    <div id="results"></div>

    <script type="text/javascript">
      var bucket = new AWS.S3({params: {Bucket: 'myBucket'}});

      var fileChooser = document.getElementById('file-chooser');
      var button = document.getElementById('upload-button');
      var results = document.getElementById('results');
      button.addEventListener('click', function() {
        var file = fileChooser.files[0];
        if (file) {
          results.innerHTML = '';

          var params = {Key: file.name, ContentType: file.type, Body: file};
          bucket.upload(params, function (err, data) {
            results.innerHTML = err ? 'ERROR!' : 'UPLOADED.';
          });
        } else {
          results.innerHTML = 'Nothing to upload.';
        }
      }, false);
    </script>

### Getting a pre-signed URL for a getObject operation

A pre-signed URL allows you to give one-off access to other users who may not
have direct access to execute the operations. Pre-signing generates a valid
URL signed with your credentials that any user can access. By default, the SDK
sets all URLs to expire within 15 minutes, but this value can be adjusted.

To generate a simple pre-signed URL that allows any user to view the contents
of a private object in a bucket you own, you can use the following call to
`getSignedUrl()`:

```javascript
var s3 = new AWS.S3();
var params = {Bucket: 'myBucket', Key: 'myKey'};
s3.getSignedUrl('getObject', params, function (err, url) {
  console.log("The URL is", url);
});
```

### Controlling Expires time with pre-signed URLs

As mentioned above, pre-signed URLs will expire in 15 minutes by default
when generated by the SDK. This value is adjustable with the `Expires`
parameter, an integer representing the number of seconds that the URL will be
valid, and can be set with any call to `getSignedUrl()`:

```javascript
var s3 = new AWS.S3();

// This URL will expire in one minute (60 seconds)
var params = {Bucket: 'myBucket', Key: 'myKey', Expires: 60};
var url = s3.getSignedUrl('getObject', params, function (err, url) {
  if (url) console.log("The URL is", url);
});
```

## Amazon DynamoDB

### Listing tables

The following example will list all tables in a DynamoDB instance.

```javascript
var db = new AWS.DynamoDB();
db.listTables(function(err, data) {
  console.log(data.TableNames);
});
```

### Reading and writing items in a table

The following example puts an item in a DynamoDB table and then reads it back
using the hash key.

```javascript
var table = new AWS.DynamoDB({params: {TableName: 'MY_TABLE'}});
var key = 'UNIQUE_KEY_ID';

// Write the item to the table
var itemParams = {Item: {id: {S: key}, data: {S: 'data'}}};
table.putItem(itemParams, function() {
  // Read the item from the table
  table.getItem({Key: {id: {S: key}}}, function(err, data) {
    console.log(data.Item); // print the item data
  });
});
```

## Amazon SQS

### Creating a queue

The following example creates a queue resource in Amazon SQS.

```javascript
var sqs = new AWS.SQS();
sqs.createQueue({QueueName: 'MY_QUEUE_NAME'}, function (err, data) {
  if (data) {
    var url = data.QueueUrl; // use this queue URL to operate on the queue
  }
});
```

### Sending a message

The following example sends a message to the queue created in the previous
example.

```javascript
// using Queue URL variable (`url`) from previous example
var queue = new AWS.SQS({params: {QueueUrl: url}});
queue.sendMessage({MessageBody: 'THE MESSAGE TO SEND'}, function (err, data) {
  if (!err) console.log('Message sent.');
});
```

### Receiving a message

The following example receives the message from the queue sent in the
previous example.

```javascript
var queue = new AWS.SQS({params: {QueueUrl: url}}); // using url to queue
queue.receiveMessage(function (err, data) {
  if (data) {
    console.log(data.Messages); // message data in Messages structure
  }
});
```

## Amazon SNS

### Publishing to a topic

The following example publishes a message to an SNS topic resource. The topic
can be identified with a topic ARN.

```javascript
var sns = new AWS.SNS({params: {TopicArn: 'ARN_FOR_SNS_TOPIC'}});
sns.publish({Message: 'THE MESSAGE TO PUBLISH'}, function (err, data) {
  if (!err) console.log('Message published');
});
```
