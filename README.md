# OpenWhisk building block - Message Hub Trigger
Create Message Hub data processing apps with Apache OpenWhisk on IBM Bluemix. This tutorial takes less than 5 minutes to complete. After this, move on to more complex serverless applications such as those tagged [_openwhisk-hands-on-demo_](https://github.com/search?q=topic%3Aopenwhisk-hands-on-demo+org%3AIBM&type=Repositories).

If you're not familiar with the OpenWhisk programming model [try the action, trigger, and rule sample first](https://github.com/IBM/openwhisk-action-trigger-rule). [You'll need a Bluemix account and the latest OpenWhisk command line tool](docs/OPENWHISK.md).

This example shows how to create an action that consumes Message Hub (Kafka) messages.

1. [Configure Message Hub](#1-configure-message-hub)
2. [Create OpenWhisk actions](#2-create-openwhisk-actions)
3. [Clean up](#3-clean-up)

# 1. Configure Message Hub
Log into Bluemix, provision a [Message Hub](https://console.ng.bluemix.net/catalog/services/message-hub) instance, and name it `kafka-broker`. On the "Manage" tab of the Message Hub console create a topic named "in-topic".

Extract the API key and the REST URL endpoint from the "Service Credentials" tab in Bluemix. Use them in the commands below.

```bash
# Bind Message Hub (Kafka) service as a package in OpenWhisk
wsk package bind kafka kafka-out-binding \
  --param api_key ${API_KEY} \
  --param kafka_rest_url ${KAFKA_REST_URL} \
  --param topic in-topic

# Create trigger to fire events when data is inserted
wsk trigger create message-received-trigger \
   --feed /_/Bluemix_kafka-broker_Credentials-1/messageHubFeed \
   --param isJSONData true \
   --param topic in-topic
```

# 2. Create OpenWhisk actions
## Create a file named `process-change.js`
```javascript
function main(params) {
  console.log("DEBUG: Received the following message as input: " + JSON.stringify(params));

  return new Promise(function(resolve, reject) {
    if (!params.messages || !params.messages[0] ||
      !params.messages[0].value || !params.messages[0].value.events) {
      reject("Invalid arguments. Must include 'messages' JSON array with 'value' field");
    }
    var msgs = params.messages;
    var out = [];
    for (var i = 0; i < msgs.length; i++) {
      var msg = msgs[i];
      for (var j = 0; j < msg.value.events.length; j++) {
        out.push(msg.value.events[j]);
      }
    }
    resolve({
      "events": out
    });
  });
}
```

## Create action sequence and map to trigger
```bash
# Upload action above that responds to messages received
wsk action create process-message process-message.js

# Unit test the new action directly
wsk action invoke \
  --blocking \
  --param name Tahoma \
  --param color Tabby \
  process-message

# Create rule that maps database change trigger to sequence
wsk rule create log-message-rule message-received-trigger process-message
```

## Enter data to fire a change
Begin streaming the OpenWhisk activation log.
```bash
wsk activation poll
```

In another terminal window, send a message to Kafka using its REST API.
```bash
DATA='{"records":[{"value":"'Tabby'"}]}'
curl -X POST -H "Content-Type: application/vnd.kafka.binary.v1+json" \
  -H "X-Auth-Token: $API_KEY" \
  --data "$DATA" \
  "$KAFKA_REST_URL/topics/in-topic"
```

View the OpenWhisk log to look for the change notification.

# 3. Clean up
## Remove the rule, trigger, action, and package

```bash
# Remove rule
wsk rule disable log-message-rule
wsk rule delete log-message-rule

# Remove trigger
wsk trigger delete message-received-trigger

# Remove actions
wsk action delete process-change

# Remove package
wsk package delete kafka-out-binding
```

# Troubleshooting
Check for errors first in the OpenWhisk activation log. Tail the log on the command line with `wsk activation poll` or drill into details visually with the [monitoring console on Bluemix](https://console.ng.bluemix.net/openwhisk/dashboard).

If the error is not immediately obvious, make sure you have the [latest version of the `wsk` CLI installed](https://console.ng.bluemix.net/openwhisk/learn/cli). If it's older than a few weeks, download an update.
```bash
wsk property get --cliversion
```

# License
[Apache 2.0](LICENSE.txt)
