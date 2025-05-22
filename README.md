# STOMP.js

This library provides a STOMP client for Web browser (using Web Sockets) or node.js applications (either using raw TCP sockets or Web Sockets).

# Project Status

__This project is _no longer maintained_ ([some context about this decision](http://jmesnil.net/weblog/2015/09/04/stepping-out-from-personal-open-source-projects/)).__

__If you encounter bugs with it or need enhancements, you can fork it and modify it as the project is under the Apache License 2.0.__

## Web Browser support

The library file is located in `lib/stomp.js` (a minified version is available in `lib/stomp.min.js`).
It does not require any dependency (except WebSocket support from the browser or an alternative to WebSocket!)

Online [documentation][doc] describes the library API (including the [annotated source code][annotated]).

## node.js support

Install the 'stompjs' module

    $ npm install stompjs

In the node.js app, require the module with:

    var Stomp = require('stompjs');

To connect to a STOMP broker over a TCP socket, use the `Stomp.overTCP(host, port)` method:

    var client = Stomp.overTCP('localhost', 61613);

To connect to a STOMP broker over a WebSocket, use instead the `Stomp.overWS(url)` method:

    var client = Stomp.overWS('ws://localhost:61614');

## Development Requirements

For development (testing, building) the project requires node.js. This allows us to run tests without the browser continuously during development (see `cake watch`).

    $ npm install

## Building and Testing

[![Build Status](https://secure.travis-ci.org/jmesnil/stomp-websocket.png)](http://travis-ci.org/jmesnil/stomp-websocket)

To build JavaScript from the CoffeeScript source code:

    $ cake build

To run tests:

    $ cake test

To continuously run tests on file changes:

    $ cake watch


## Browser Tests

* Make sure you have a running STOMP broker which supports the WebSocket protocol
 (see the [documentation][doc])
* Open in your web browser the project's [test page](browsertests/index.html)
* Check all tests pass

## Use

The project contains examples for using stomp.js
to send and receive STOMP messages from a server directly in the Web Browser or in a WebWorker.

## Advanced ACK/NACK: Timeouts and Resending Messages

When reliable message processing is critical, especially over potentially unreliable networks, implementing a timeout and resend mechanism for message acknowledgements (ACK/NACK) becomes essential. This ensures that if the STOMP broker does not confirm an ACK/NACK within a certain timeframe, the message processing can be retried or handled appropriately.

### 1. Subscribing for Client-Side Acknowledgement

To manage ACKs/NACKs from the client-side, you must subscribe to a destination with the `ack` header set to either `'client'` or `'client-individual'`.

*   `'client'`: The client acknowledges all messages received on this subscription. If a message is NACKed, or the client disconnects before acknowledging, the broker might redeliver all messages received since the last acknowledged message.
*   `'client-individual'`: The client acknowledges each message individually. This provides finer-grained control over message processing.

```javascript
var subscription = client.subscribe("/queue/your-destination", function(message) {
  // process the message
  console.log("Received message:", message.body);

  // ... later, after successful processing
  // Acknowledge the message, potentially with a receipt
  var receiptId = "receipt-" + message.headers["message-id"]; // Ensure unique receipt ID
  message.ack({ receipt: receiptId });
  // Track this message for ACK confirmation (see below)
}, { ack: 'client-individual' });
```

### 2. Sending Receipts with ACKs/NACKs

To confirm that the broker has successfully processed your `message.ack()` or `message.nack()` (or `client.ack()` / `client.nack()` if you are using message IDs directly), you should include a `receipt` header. The value of this header should be a unique identifier for this specific acknowledgement.

```javascript
// When acknowledging a message
var receiptId = "ack-receipt-" + new Date().getTime(); // Example unique ID
message.ack({ receipt: receiptId });
pendingReceipts[receiptId] = {
  message: message,
  // ... other tracking info like retry count, timeoutId
};

// When NACKing a message
var nackReceiptId = "nack-receipt-" + new Date().getTime();
message.nack({ receipt: nackReceiptId });
pendingReceipts[nackReceiptId] = {
  message: message,
  // ...
};
```

### 3. Listening for Server Confirmation with `client.onreceipt`

The STOMP broker will send a `RECEIPT` frame back to the client once it has processed a frame that included a `receipt` header. You can listen for these `RECEIPT` frames using the `client.onreceipt` callback.

```javascript
client.onreceipt = function(frame) {
  var receiptId = frame.headers.receipt-id;
  console.log("Received ACK confirmation for:", receiptId);

  // Logic to handle confirmed ACKs (e.g., clear timeouts)
  if (pendingReceipts[receiptId]) {
    clearTimeout(pendingReceipts[receiptId].timeoutId);
    delete pendingReceipts[receiptId];
    console.log("Successfully ACKed and cleared message:", receiptId);
  }
};
```

### 4. Example: ACK Timeout and Resend Logic

Here's a conceptual JavaScript example demonstrating how to track messages awaiting ACK confirmation, use `setTimeout` for handling timeouts, `clearTimeout` upon receiving a receipt, and a simple resend strategy.

```javascript
const ACK_TIMEOUT_MS = 5000; // 5 seconds
const MAX_RESEND_ATTEMPTS = 3;
let pendingAcks = {}; // Stores messages awaiting ACK confirmation

// When subscribing and receiving a message:
client.subscribe("/queue/your-destination", function(message) {
  console.log("Processing message:", message.headers["message-id"]);
  // Simulate processing
  processMessage(message)
    .then(() => {
      sendAck(message);
    })
    .catch((error) => {
      console.error("Failed to process message, sending NACK:", message.headers["message-id"], error);
      // Optionally, send a NACK here, also with receipt and tracking
      // message.nack({ receipt: "nack-receipt-" + message.headers["message-id"] });
    });
}, { ack: 'client-individual' });

function sendAck(message) {
  const messageId = message.headers["message-id"];
  const receiptId = "ack-" + messageId + "-" + new Date().getTime();

  if (!pendingAcks[messageId]) {
    pendingAcks[messageId] = {
      message: message,
      attempts: 0,
      receiptId: null // Will be set when ACK is sent
    };
  }

  pendingAcks[messageId].attempts++;
  pendingAcks[messageId].receiptId = receiptId; // Update receipt ID for this attempt

  console.log(`Attempting to ACK message: ${messageId}, Attempt: ${pendingAcks[messageId].attempts}, Receipt: ${receiptId}`);
  message.ack({ receipt: receiptId });

  const timeoutId = setTimeout(() => {
    console.warn(`ACK timeout for message: ${messageId}, Receipt: ${receiptId}`);
    handleAckTimeout(messageId);
  }, ACK_TIMEOUT_MS);

  pendingAcks[messageId].timeoutId = timeoutId;
}

function handleAckTimeout(messageId) {
  const ackInfo = pendingAcks[messageId];
  if (!ackInfo) return; // Already ACKed and cleared

  if (ackInfo.attempts < MAX_RESEND_ATTEMPTS) {
    console.log(`Resending ACK for message: ${messageId}`);
    // It's important to note that resending an ACK for the *same message delivery*
    // might not be what you want. The broker might have already processed the first ACK,
    // and the receipt was lost.
    // More commonly, you might re-process the message or trigger an alert.
    // For this example, we'll simulate re-sending the ACK, assuming the original
    // message.ack() command itself might have failed to reach the broker or the
    // receipt was lost.
    // A true "resend the message" would involve the application logic that
    // originally sent the message to send it again, likely with a new message-id.
    // Here, we are retrying the ACK for the received message.
    sendAck(ackInfo.message); // Retry sending ACK for the same message
  } else {
    console.error(`Max ACK resend attempts reached for message: ${messageId}. Moving to dead-letter queue or error handling.`);
    // Implement dead-letter queue logic or other error handling here
    // For example, explicitly NACK the message if not done already
    // ackInfo.message.nack({ receipt: "nack-final-" + messageId });
    delete pendingAcks[messageId];
  }
}

// In client.onreceipt:
client.onreceipt = function(frame) {
  const receivedReceiptId = frame.headers["receipt-id"];
  console.log("Received receipt:", receivedReceiptId);

  // Find the message associated with this receipt
  for (const messageId in pendingAcks) {
    if (pendingAcks[messageId].receiptId === receivedReceiptId) {
      console.log(`ACK confirmed for message: ${messageId}, Receipt: ${receivedReceiptId}`);
      clearTimeout(pendingAcks[messageId].timeoutId);
      delete pendingAcks[messageId];
      break;
    }
  }
};

// Example `processMessage` function
function processMessage(message) {
  return new Promise((resolve, reject) => {
    // Simulate asynchronous processing
    setTimeout(() => {
      if (Math.random() > 0.2) { // Simulate 80% success rate
        console.log("Successfully processed message:", message.headers["message-id"]);
        resolve();
      } else {
        console.error("Simulated failure processing message:", message.headers["message-id"]);
        reject(new Error("Simulated processing failure"));
      }
    }, 500);
  });
}

// Ensure client is connected before subscribing
// client.connect(headers, function(frame) { ... subscribe here ... });
```

**Important Considerations for the Example:**
*   **Message Uniqueness:** The example uses `message.headers["message-id"]` which is typically provided by the broker and unique for each message delivery.
*   **Receipt Uniqueness:** `receiptId` must be unique for each attempt to ACK/NACK if you are retrying the ACK command itself.
*   **Resending Message vs. Resending ACK:** The example focuses on retrying the `message.ack()` command. If the goal is to have the *original sender* resend the message because it was never processed, that's a different pattern, often involving a NACK back to the broker to requeue (or dead-letter) the message, and then the original publisher re-publishing.
*   **NACK Handling:** Similar logic can be applied for NACKs if you need to ensure the broker has processed a NACK.

### 5. Encapsulation in a Service (e.g., Angular)

In frameworks like Angular, React, or Vue.js, or even in vanilla JavaScript applications with good structure, this ACK/NACK handling logic with timeouts and resends is best encapsulated within a dedicated service (e.g., `StompAckNackService`).

The responsibilities of such a service would include:

*   **Managing `pendingAcks`**: Tracking messages, their receipt IDs, timeout IDs, and retry counts.
*   **Handling `client.onreceipt`**: Centralizing the logic for processing incoming receipts and clearing timeouts.
*   **Exposing Methods**: Providing clear methods for components/other services to call, e.g., `acknowledgeMessage(message)` which would internally handle the receipt generation, timeout setup, and tracking.
*   **Configuration**: Allowing configuration of timeout durations and max retry attempts.
*   **Error Handling**: Defining strategies for what happens when max retries are exceeded (e.g., notifying the user, sending the message to a dead-letter queue via a NACK).

This separation of concerns makes the main application logic cleaner and the ACK/NACK handling reusable and easier to maintain.

## Authors

 * [Jeff Mesnil](http://jmesnil.net/)
 * [Jeff Lindsay](http://github.com/progrium)

[doc]: http://jmesnil.net/stomp-websocket/doc/
[annotated]: http://jmesnil.net/stomp-websocket/doc/stomp.html
