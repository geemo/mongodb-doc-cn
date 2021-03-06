# MongoDB Wire Protocol

On this page

* [Introduction](https://docs.mongodb.com/manual/reference/mongodb-wire-protocol/#introduction)
* [TCP/IP Socket](https://docs.mongodb.com/manual/reference/mongodb-wire-protocol/#tcp-ip-socket)
* [Messages Types and Formats](https://docs.mongodb.com/manual/reference/mongodb-wire-protocol/#messages-types-and-formats)
* [Standard Message Header](https://docs.mongodb.com/manual/reference/mongodb-wire-protocol/#standard-message-header)
* [Client Request Messages](https://docs.mongodb.com/manual/reference/mongodb-wire-protocol/#client-request-messages)
* [Database Response Messages](https://docs.mongodb.com/manual/reference/mongodb-wire-protocol/#database-response-messages)

## Introduction

The MongoDB Wire Protocol is a simple socket-based, request-response style protocol. Clients communicate with the database server through a regular TCP/IP socket.

## TCP/IP Socket

Clients should connect to the database with a regular TCP/IP socket. There is no connection handshake.

### Port

The default port number for[`mongod`](https://docs.mongodb.com/manual/reference/program/mongod/#bin.mongod)and[`mongos`](https://docs.mongodb.com/manual/reference/program/mongos/#bin.mongos)instances is 27017. The port number for[`mongod`](https://docs.mongodb.com/manual/reference/program/mongod/#bin.mongod)and[`mongos`](https://docs.mongodb.com/manual/reference/program/mongos/#bin.mongos)is configurable and may vary.

### Byte Ordering

All data in the MongoDB wire protocol is little-endian.

## Messages Types and Formats

There are two types of messages, client requests and database responses.

NOTE

* This page uses a C-like
  `struct`
  to describe the message structure.
* The types used in this document \(
  `cstring`
  ,
  `int32`
  , etc.\) are the same as those defined in the
  [BSON specification](http://bsonspec.org/#/specification)
  .
* To denote repetition, the document uses the asterisk notation from the
  [BSON specification](http://bsonspec.org/#/specification)
  . For example,
  `int64*`
  indicates that one or more of the specified type can be written to the socket, one after another.
* The standard message header is typed as
  `MsgHeader`
  . Integer constants are in capitals \(e.g.
  `ZERO`
  for the integer value of 0\).

## Standard Message Header

In general, each message consists of a standard message header followed by request-specific data. The standard message header is structured as follows:

```c
struct MsgHeader {
    int32   messageLength; // total message size, including this
    int32   requestID;     // identifier for this message
    int32   responseTo;    // requestID from the original request
                           //   (used in responses from db)
    int32   opCode;        // request type - see table below
}
```

| Field | Description |
| :--- | :--- |
| `messageLength` | The total size of the message in bytes. This total includes the 4 bytes that holds the message length. |
| `requestID` | A client or database-generated identifier that uniquely identifies this message. For the case of client-generated messages \(e.g.[OP\_QUERY](https://docs.mongodb.com/manual/reference/mongodb-wire-protocol/#wire-op-query)and[OP\_GET\_MORE](https://docs.mongodb.com/manual/reference/mongodb-wire-protocol/#wire-op-get-more)\), it will be returned in the`responseTo`field of the[OP\_REPLY](https://docs.mongodb.com/manual/reference/mongodb-wire-protocol/#wire-op-reply)message. Clients can use the`requestID`and the`responseTo`fields to associate query responses with the originating query. |
| `responseTo` | In the case of a message from the database, this will be the`requestID`taken from the[OP\_QUERY](https://docs.mongodb.com/manual/reference/mongodb-wire-protocol/#wire-op-query)or[OP\_GET\_MORE](https://docs.mongodb.com/manual/reference/mongodb-wire-protocol/#wire-op-get-more)messages from the client. Clients can use the`requestID`and the`responseTo`fields to associate query responses with the originating query. |
| `opCode` | Type of message. See[Request Opcodes](https://docs.mongodb.com/manual/reference/mongodb-wire-protocol/#wp-request-opcodes). |

### Request Opcodes

NOTE

Starting with MongoDB 2.6 and[`maxWireVersion`](https://docs.mongodb.com/manual/reference/command/isMaster/#isMaster.maxWireVersion)`3`, MongoDB drivers use the[database commands](https://docs.mongodb.com/manual/reference/command/#collection-commands)[`insert`](https://docs.mongodb.com/manual/reference/command/insert/#dbcmd.insert),[`update`](https://docs.mongodb.com/manual/reference/command/update/#dbcmd.update), and[`delete`](https://docs.mongodb.com/manual/reference/command/delete/#dbcmd.delete)instead of`OP_INSERT`,`OP_UPDATE`, and`OP_DELETE`for acknowledged writes. Most drivers continue to use opcodes for unacknowledged writes.

IMPORTANT

`OP_COMMAND`and`OP_COMMANDREPLY`are cluster internal and should not be implemented by clients or drivers.

The`OP_COMMAND`and`OP_COMMANDREPLY`format and protocol are not stable and may change between releases without maintaining backwards compatibility.

The following are the supported`opCode`:

| Opcode Name | Value | Comment |
| :--- | :--- | :--- |
| `OP_REPLY` | 1 | Reply to a client request.`responseTo`is set. |
| `OP_UPDATE` | 2001 | Update document. |
| `OP_INSERT` | 2002 | Insert new document. |
| `RESERVED` | 2003 | Formerly used for OP\_GET\_BY\_OID. |
| `OP_QUERY` | 2004 | Query a collection. |
| `OP_GET_MORE` | 2005 | Get more data from a query. See Cursors. |
| `OP_DELETE` | 2006 | Delete documents. |
| `OP_KILL_CURSORS` | 2007 | Notify database that the client has finished with the cursor. |
| `OP_COMMAND` | 2010 | Cluster internal protocol representing a command request. |
| `OP_COMMANDREPLY` | 2011 | Cluster internal protocol representing a reply to an`OP_COMMAND`. |

## Client Request Messages

Clients can send request messages that specify all but the[OP\_REPLY](https://docs.mongodb.com/manual/reference/mongodb-wire-protocol/#wire-op-reply)opCode.[OP\_REPLY](https://docs.mongodb.com/manual/reference/mongodb-wire-protocol/#wire-op-reply)is reserved for use by the database.

Only the[OP\_QUERY](https://docs.mongodb.com/manual/reference/mongodb-wire-protocol/#wire-op-query)and[OP\_GET\_MORE](https://docs.mongodb.com/manual/reference/mongodb-wire-protocol/#wire-op-get-more)messages result in a response from the database. There will be no response sent for any other message.

You can determine if a message was successful with a getLastError command.

### OP\_UPDATE

The OP\_UPDATE message is used to update a document in a collection. The format of a OP\_UPDATE message is the following:

```c
struct OP_UPDATE {
    MsgHeader header;             // standard message header
    int32     ZERO;               // 0 - reserved for future use
    cstring   fullCollectionName; // "dbname.collectionname"
    int32     flags;              // bit vector. see below
    document  selector;           // the query to select the document
    document  update;             // specification of the update to perform
}
```

| Field | Description |
| :--- | :--- |
| `header` | Message header, as described in[Standard Message Header](https://docs.mongodb.com/manual/reference/mongodb-wire-protocol/#wp-message-header). |
| `ZERO` | Integer value of 0. Reserved for future use. |
| `fullCollectionName` | The full collection name; i.e. namespace. The full collection name is the concatenation of the database name with the collection name, using a`.`for the concatenation. For example, for the database`foo`and the collection`bar`, the full collection name is`foo.bar`. |
| `flags` | Bit vector to specify flags for the operation. The bit values correspond to the following:`0`corresponds to Upsert. If set, the database will insert the supplied object into the collection if no matching document is found.`1`corresponds to MultiUpdate.If set, the database will update all matching objects in the collection. Otherwise only updates first matching document.`2`-`31`are reserved. Must be set to 0. |
| `selector` | BSON document that specifies the query for selection of the document to update. |
| `update` | BSON document that specifies the update to be performed. For information on specifying updates see the[Update Operations](https://docs.mongodb.com/manual/applications/update)documentation from the MongoDB Manual. |

There is no response to an OP\_UPDATE message.

### OP\_INSERT

The OP\_INSERT message is used to insert one or more documents into a collection. The format of the OP\_INSERT message is

```c
struct {
    MsgHeader header;             // standard message header
    int32     flags;              // bit vector - see below
    cstring   fullCollectionName; // "dbname.collectionname"
    document* documents;          // one or more documents to insert into the collection
}
```

| Field | Description |
| :--- | :--- |
| `header` | Message header, as described in[Standard Message Header](https://docs.mongodb.com/manual/reference/mongodb-wire-protocol/#wp-message-header). |
| `flags` | Bit vector to specify flags for the operation. The bit values correspond to the following:`0`corresponds to ContinueOnError. If set, the database will not stop processing a bulk insert if one fails \(eg due to duplicate IDs\). This makes bulk insert behave similarly to a series of single inserts, except lastError will be set if any insert fails, not just the last one. If multiple errors occur, only the most recent will be reported by getLastError. \(new in 1.9.1\)`1`-`31`are reserved. Must be set to 0. |
| `fullCollectionName` | The full collection name; i.e. namespace. The full collection name is the concatenation of the database name with the collection name, using a`.`for the concatenation. For example, for the database`foo`and the collection`bar`, the full collection name is`foo.bar`. |
| `documents` | One or more documents to insert into the collection. If there are more than one, they are written to the socket in sequence, one after another. |

There is no response to an OP\_INSERT message.

### OP\_QUERY

The OP\_QUERY message is used to query the database for documents in a collection. The format of the OP\_QUERY message is:

```c
struct OP_QUERY {
    MsgHeader header;                 // standard message header
    int32     flags;                  // bit vector of query options.  See below for details.
    cstring   fullCollectionName ;    // "dbname.collectionname"
    int32     numberToSkip;           // number of documents to skip
    int32     numberToReturn;         // number of documents to return
                                      //  in the first OP_REPLY batch
    document  query;                  // query object.  See below for details.
  [ document  returnFieldsSelector; ] // Optional. Selector indicating the fields
                                      //  to return.  See below for details.
}
```

| Field | Description |
| :--- | :--- |
| `header` | Message header, as described in[Standard Message Header](https://docs.mongodb.com/manual/reference/mongodb-wire-protocol/#wp-message-header). |
| `flags` | Bit vector to specify flags for the operation. The bit values correspond to the following:`0`is reserved. Must be set to 0.`1`corresponds to TailableCursor. Tailable means cursor is not closed when the last data is retrieved. Rather, the cursor marks the final object’s position. You can resume using the cursor later, from where it was located, if more data were received. Like any “latent cursor”, the cursor may become invalid at some point \(CursorNotFound\) – for example if the final object it references were deleted.`2`corresponds to SlaveOk.Allow query of replica slave. Normally these return an error except for namespace “local”.`3`corresponds to OplogReplay. Internal replication use only - driver should not set.`4`corresponds to NoCursorTimeout. The server normally times out idle cursors after an inactivity period \(10 minutes\) to prevent excess memory use. Set this option to prevent that.`5`corresponds to AwaitData. Use with TailableCursor. If we are at the end of the data, block for a while rather than returning no data. After a timeout period, we do return as normal.`6`corresponds to Exhaust. Stream the data down full blast in multiple “more” packages, on the assumption that the client will fully read all data queried. Faster when you are pulling a lot of data and know you want to pull it all down. Note: the client is not allowed to not read all the data unless it closes the connection.`7`corresponds to Partial. Get partial results from a mongos if some shards are down \(instead of throwing an error\)`8`-`31`are reserved. Must be set to 0. |
| `fullCollectionName` | The full collection name; i.e. namespace. The full collection name is the concatenation of the database name with the collection name, using a`.`for the concatenation. For example, for the database`foo`and the collection`bar`, the full collection name is`foo.bar`. |
| `numberToSkip` | Sets the number of documents to omit - starting from the first document in the resulting dataset - when returning the result of the query. |
| `numberToReturn` | Limits the number of documents in the first[OP\_REPLY](https://docs.mongodb.com/manual/reference/mongodb-wire-protocol/#wire-op-reply)message to the query. However, the database will still establish a cursor and return the`cursorID`to the client if there are more results than`numberToReturn`. If the client driver offers ‘limit’ functionality \(like the SQL LIMIT keyword\), then it is up to the client driver to ensure that no more than the specified number of document are returned to the calling application. If`numberToReturn`is`0`, the db will use the default return size. If the number is negative, then the database will return that number and close the cursor. No further results for that query can be fetched. If`numberToReturn`is`1`the server will treat it as`-1`\(closing the cursor automatically\). |
| `query` | BSON document that represents the query. The query will contain one or more elements, all of which must match for a document to be included in the result set. Possible elements include`$query`,`$orderby`,`$hint`,`$explain`, and`$snapshot`. |
| `returnFieldsSelector` | Optional. BSON document that limits the fields in the returned documents. The`returnFieldsSelector`contains one or more elements, each of which is the name of a field that should be returned, and and the integer value`1`. In JSON notation, a`returnFieldsSelector`to limit to the fields`a`,`b`and`c`would be:{a:1,b:1,c:1} |

The database will respond to an OP\_QUERY message with an[OP\_REPLY](https://docs.mongodb.com/manual/reference/mongodb-wire-protocol/#wire-op-reply)message.

### OP\_GET\_MORE

The OP\_GET\_MORE message is used to query the database for documents in a collection. The format of the OP\_GET\_MORE message is:

```c
struct {
    MsgHeader header;             // standard message header
    int32     ZERO;               // 0 - reserved for future use
    cstring   fullCollectionName; // "dbname.collectionname"
    int32     numberToReturn;     // number of documents to return
    int64     cursorID;           // cursorID from the OP_REPLY
}
```

| Field | Description |
| :--- | :--- |
| `header` | Message header, as described in[Standard Message Header](https://docs.mongodb.com/manual/reference/mongodb-wire-protocol/#wp-message-header). |
| `ZERO` | Integer value of 0. Reserved for future use. |
| `fullCollectionName` | The full collection name; i.e. namespace. The full collection name is the concatenation of the database name with the collection name, using a`.`for the concatenation. For example, for the database`foo`and the collection`bar`, the full collection name is`foo.bar`. |
| `numberToReturn` | Limits the number of documents in the first[OP\_REPLY](https://docs.mongodb.com/manual/reference/mongodb-wire-protocol/#wire-op-reply)message to the query. However, the database will still establish a cursor and return the`cursorID`to the client if there are more results than`numberToReturn`. If the client driver offers ‘limit’ functionality \(like the SQL LIMIT keyword\), then it is up to the client driver to ensure that no more than the specified number of document are returned to the calling application. If`numberToReturn`is`0`, the db will used the default return size. |
| `cursorID` | Cursor identifier that came in the[OP\_REPLY](https://docs.mongodb.com/manual/reference/mongodb-wire-protocol/#wire-op-reply). This must be the value that came from the database. |

The database will respond to an OP\_GET\_MORE message with an[OP\_REPLY](https://docs.mongodb.com/manual/reference/mongodb-wire-protocol/#wire-op-reply)message.

### OP\_DELETE

The OP\_DELETE message is used to remove one or more documents from a collection. The format of the OP\_DELETE message is:

```c
struct {
    MsgHeader header;             // standard message header
    int32     ZERO;               // 0 - reserved for future use
    cstring   fullCollectionName; // "dbname.collectionname"
    int32     flags;              // bit vector - see below for details.
    document  selector;           // query object.  See below for details.
}
```

| Field | Description |
| :--- | :--- |
| `header` | Message header, as described in[Standard Message Header](https://docs.mongodb.com/manual/reference/mongodb-wire-protocol/#wp-message-header). |
| `ZERO` | Integer value of 0. Reserved for future use. |
| `fullCollectionName` | The full collection name; i.e. namespace. The full collection name is the concatenation of the database name with the collection name, using a`.`for the concatenation. For example, for the database`foo`and the collection`bar`, the full collection name is`foo.bar`. |
| `flags` | Bit vector to specify flags for the operation. The bit values correspond to the following:`0`corresponds to SingleRemove. If set, the database will remove only the first matching document in the collection. Otherwise all matching documents will be removed.`1`-`31`are reserved. Must be set to 0. |
| `selector` | BSON document that represent the query used to select the documents to be removed. The selector will contain one or more elements, all of which must match for a document to be removed from the collection. |

There is no response to an OP\_DELETE message.

### OP\_KILL\_CURSORS

The OP\_KILL\_CURSORS message is used to close an active cursor in the database. This is necessary to ensure that database resources are reclaimed at the end of the query. The format of the OP\_KILL\_CURSORS message is:

```c
struct {
    MsgHeader header;            // standard message header
    int32     ZERO;              // 0 - reserved for future use
    int32     numberOfCursorIDs; // number of cursorIDs in message
    int64*    cursorIDs;         // sequence of cursorIDs to close
}
```

| Field | Description |
| :--- | :--- |
| `header` | Message header, as described in[Standard Message Header](https://docs.mongodb.com/manual/reference/mongodb-wire-protocol/#wp-message-header). |
| `ZERO` | Integer value of 0. Reserved for future use. |
| `numberOfCursorIDs` | The number of cursor IDs that are in the message. |
| `cursorIDs` | “Array” of cursor IDs to be closed. If there are more than one, they are written to the socket in sequence, one after another. |

If a cursor is read until exhausted \(read until[OP\_QUERY](https://docs.mongodb.com/manual/reference/mongodb-wire-protocol/#wire-op-query)or[OP\_GET\_MORE](https://docs.mongodb.com/manual/reference/mongodb-wire-protocol/#wire-op-get-more)returns zero for the cursor id\), there is no need to kill the cursor.

### OP\_COMMAND

WARNING

`OP_COMMAND`is cluster internal and should not be implemented by clients or drivers.

The`OP_COMMAND`format and protocol is not stable and may change between releases without maintaining backwards compatibility.

`OP_COMMAND`is a wire protocol message used internally for intra-cluster database command requests issued by one MongoDB server to another. The receiving database sends back an[OP\_COMMANDREPLY](https://docs.mongodb.com/manual/reference/mongodb-wire-protocol/#wire-op-commandreply)as a response to a`OP_COMMAND`.

```c
struct {
   MsgHeader header;     // standard message header
   cstring database;     // the name of the database to run the command on
   cstring commandName;  // the name of the command
   document metadata;    // a BSON document containing any metadata
   document commandArgs; // a BSON document containing the command arguments
   inputDocs;            // a set of zero or more documents
}
```

| Field | Description |
| :--- | :--- |
| `header` | Standard message header, as described in[Standard Message Header](https://docs.mongodb.com/manual/reference/mongodb-wire-protocol/#wp-message-header). |
| `database` | The name of the database to run the command on. |
| `commandName` | The name of the command. See[Database Commands](https://docs.mongodb.com/manual/reference/command/#database-commands)for a list of database commands. |
| `metadata` | Available for the system to attach any metadata to internal commands that is not part of the command parameters proper, as supplied by the client driver |
| `commandArgs` | A BSON document containing the command arguments.See the documentation for the specified`commandName`for information its arguments |
| `inputDocs` | Zero or more documents acting as input to the command. Useful for commands that can require a large amount of data sent from the client, such as a batch insert. |

## Database Response Messages

### OP\_REPLY

The`OP_REPLY`message is sent by the database in response to an[OP\_QUERY](https://docs.mongodb.com/manual/reference/mongodb-wire-protocol/#wire-op-query)or[OP\_GET\_MORE](https://docs.mongodb.com/manual/reference/mongodb-wire-protocol/#wire-op-get-more)message. The format of an OP\_REPLY message is:

```c
struct {
    MsgHeader header;         // standard message header
    int32     responseFlags;  // bit vector - see details below
    int64     cursorID;       // cursor id if client needs to do get more's
    int32     startingFrom;   // where in the cursor this reply is starting
    int32     numberReturned; // number of documents in the reply
    document* documents;      // documents
}
```

| Field | Description |
| :--- | :--- |
| `header` | Message header, as described in[Standard Message Header](https://docs.mongodb.com/manual/reference/mongodb-wire-protocol/#wp-message-header). |
| `responseFlags` | Bit vector to specify flags. The bit values correspond to the following:`0`corresponds to CursorNotFound. Is set when`getMore`is called but the cursor id is not valid at the server. Returned with zero results.`1`corresponds to QueryFailure. Is set when query failed. Results consist of one document containing an “$err” field describing the failure.`2`corresponds to ShardConfigStale. Drivers should ignore this. Only[`mongos`](https://docs.mongodb.com/manual/reference/program/mongos/#bin.mongos)will ever see this set, in which case, it needs to update config from the server.`3`corresponds to AwaitCapable. Is set when the server supports the AwaitData Query option. If it doesn’t, a client should sleep a little between getMore’s of a Tailable cursor. Mongod version 1.6 supports AwaitData and thus always sets AwaitCapable.`4`-`31`are reserved. Ignore. |
| `cursorID` | The`cursorID`that this OP\_REPLY is a part of. In the event that the result set of the query fits into one OP\_REPLY message,`cursorID`will be 0. This`cursorID`must be used in any[OP\_GET\_MORE](https://docs.mongodb.com/manual/reference/mongodb-wire-protocol/#wire-op-get-more)messages used to get more data, and also must be closed by the client when no longer needed via a[OP\_KILL\_CURSORS](https://docs.mongodb.com/manual/reference/mongodb-wire-protocol/#wire-op-kill-cursors)message. |
| `startingFrom` | Starting position in the cursor. |
| `numberReturned` | Number of documents in the reply. |
| `documents` | Returned documents. |

### OP\_COMMANDREPLY

WARNING

`OP_COMMANDREPLY`is cluster internal and should not be implemented by clients or drivers.

The`OP_COMMANDREPLY`format and protocol is not stable and may change between releases without maintaining backwards compatibility.

The`OP_COMMANDREPLY`is a wire protocol message used internally for replying to intra-cluster[OP\_COMMAND](https://docs.mongodb.com/manual/reference/mongodb-wire-protocol/#wire-op-command)requests issued by one MongoDB server to another.

The format of an`OP_COMMANDREPLY`is:

```c
struct {
   MsgHeader header;       // A standard wire protocol header
   document metadata;      // A BSON document containing any required metadata
   document commandReply;  // A BSON document containing the command reply
   document outputDocs;    // A variable number of BSON documents
}
```

| Field | Description |
| :--- | :--- |
| `header` | Standard message header, as described in[Standard Message Header](https://docs.mongodb.com/manual/reference/mongodb-wire-protocol/#wp-message-header). |
| `metadata` | Available for the system to attach any metadata to internal commands that is not part of the command parameters proper, as supplied by the client driver. |
| `commandReply` | A BSON document containing the command reply. |
| `outputDocs` | Useful for commands that can return a large amount of data, such as find or aggregate.This field is not currently in use. |



