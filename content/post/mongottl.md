+++
author = "Brandon Nicoll"
date = 2014-07-29T18:51:08Z
description = ""
draft = false
slug = "mongottl"
title = "Automatic Data Purging with MongoDB TTL"

+++

<img src="/Title.jpg" style="max-width: 100%" />
A common database maintenance scenario is the need to purge unneeded or expired data after a period of time, such as log messages or perhaps sensitive user data. With MongoDB, this can be easily accomplished with a built-in process. By creating a specialized index, we can make use of Mongo's [TTL feature][1]. This feature allows us to either purge documents at a specific date or specify an amount of time before the document expires.

### Step 1: Design Decision
We have a relatively minor design choice to make regarding which field to create the index off of. Here are our options:

 - Specify a date the document should be deleted
 - Specify an amount of time the document has to live from the indexed field

Oftentimes, a collection will have been designed with some audit trail information in mind. There may be fields such as: createdBy, createDate, modifiedBy, modifiedDate. In this example, I already have a collection that contains these four audit fields. Therefore, I'm going to create my TTL index off of the createDate field and specify an expireAfterSeconds value. There are a few other rules to keep in mind when making this decision; the index must be [single field][2], [a date type][3], and the field may not already be part of another index. Lucky for me, my createDate meets the criteria. 

### Step 2: Creating the TTL Index

I'm using [Mongoose][4] in my project, so in order to create the index, all I have to do is add 'expires' to 'createDate':

```prettyprint lang-js
var mongoose = require('mongoose');
var Schema = mongoose.Schema;

var doomedDataSchema = new Schema({
  name:  String,
  title: String,
  reasonForDoom: String,
  notes: [{ note: String, noteDate: Date }],
  audit: {
    createdBy: String,
    createDate:  { type: Date, expires: 60*60*24*30}, 
    modifiedBy: String,
    modifiedDate: Date
  }
});
```

My complicated math formula will set the document to be deleted after thirty days. According to the [API docs][5] on this subject, we don't need to specify an ugly number, but can substitute with an easier-to-read string value, like so:
```prettyprint lang-js
    createDate:  { type: Date, expires: '30d'}, 
```
That's it! Next time our code is started, the Mongoose schema will use the [ensureIndex][6] command behind the scenes to create the index if it doesn't already exist. The service that removes expired documents runs every 60 seconds.

### Conclusion
The MongoDB TTL feature is a great tool to keep in mind when your project requires time-based data purging. This feature saves us the development time and overhead of writing our own, similar services and can be enabled from the Mongo shell, or in my case, using an ODM tool. 


  [1]: http://docs.mongodb.org/manual/tutorial/expire-data/
  [2]: http://docs.mongodb.org/manual/core/index-single/
  [3]: http://docs.mongodb.org/manual/reference/bson-types/#date
  [4]: http://mongoosejs.com/
  [5]: http://mongoosejs.com/docs/api.html#schema_date_SchemaDate-expires
  [6]: http://docs.mongodb.org/manual/reference/method/db.collection.ensureIndex/