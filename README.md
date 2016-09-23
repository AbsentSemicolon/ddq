DeDuplicated Queue (DDQ)
========================

[![Build Status][travis-image]][Travis CI]
[![Dependencies][dependencies-image]][Dependencies]
[![Dev Dependencies][devdependencies-image]][Dev Dependencies]
[![codecov.io][codecov-image]][Code Coverage]

About
-----

This module was created because of a need for a system which acts like a queue for tasks but doesn't allow for duplication. DDQ uses an `async` methodology to complete certain tasks, like sending messages and what happens when data is found to process. We used `callbacks` instead of `promises` to make the process faster and easier to get around.

DDQ also extends the Event Emitter module so it's very easy to use events for when there is data or an error has occured and inform the software of these events.

A couple terms which are used in these docs and could be within the code are `producer` and `consumer`. The `producer` is the software creating tasks; the one which sends messages to DDQ to put into the Queue. The `consumer` is the opposite. This is another piece of software `listening` to DDQ to see if there are any tasks for it to take action on.

Usage
-----

DDQ is separated into parts and each component of it is strict on what it does. DDQ's job is to take messages and commands from the user and relays to the `backend` to do something. DDQ also handles certain business logic like being paused when too many messages are in flight or if the user wants to stop polling for a time. The `backend` has the job of doing what DDQ tells it to do, but could also be responsible for polling for messages emitting when it's found one and cleaning up tasks.

Setting Up
----------

Setting up DDQ is pretty straight forward. It does require some configuration to get everything working together. Your project would need to include DDQ in it's `package.json` and also include a `backend` module or write one yourself. In the `config` value for `backend` this will set what module you're using. In the code this looks for `ddq-backend-whatYouHaveInConfigValueForBackend`. This should be in your `node_modules` directory, so it's easily used.

Another config value is the `heartbeatDelay`. DDQ uses a method on the wrapped message, which we'll get to later which is called every so often to update the task/job in the queue that it's being processed. This would set that update to happen per millisecond. So 1000 is 1 sec.

Also, servers might not be able to handle a certain number of processes, or you might only want to handle a few at a time, so setting the `maxProcessingMessages` to a number will make it so only up to that number of processes are created. DDQ will automatically tell the backend to pause the polling when this limit is reached. It will resume polling once the number of processing messages is lower than the max.

Other config values would be more specific for what your backend needs, like table name, how often to poll the storage mechanism and so on. These are passed to the backend using the `backendConfig` values.

    var DeDuplicatedQueue, deduplicatedQueue;

    DeDuplicatedQueue = require("ddq");

    /* Pass in the config, including which backend to use and DDQ will find it
     * and use it.
     */
    deduplicatedQueue = new DeDuplicatedQueue({
        backend: "mock",
        backendConfig: {
            host: "localhost",
            password: "someReallyNiceLongSecurePassword",
            pollingDelay: 5000,
            port: 3306,
            table: "query",
            user: "hopefullyNotRoot"
        },
        heartbeatDelay: 1000,
        maxProcessingMessages: 10
    });

Sending a Message
-----------------

A producer would be one to send a message. Using the `sendMessage` method you would send in the `message` you want to use and a `callback`. The callback is a way to handle when the `message` is successfully added to the queue, or not.

    deduplicatedQueue.sendMessage("sample message", (err) => {
        if (err) {
            // Take action if there is an error.
        }
    });

Listening
---------

Once you have an instance of DDQ, you then need to `listen` to events which will be emitted from it. DDQ will send two types of events, `data` and `error`. When `data` is emitted you'll receive a message from the queue and a `callback` from DDQ. Once the process is complete you'll need to call the `callback` with an argument whether there was an error when processing.

When the `data` event is triggered from the backend DDQ will use methods on the data received from the `backend`. These methods include: `heartbeat`, `heartbeatKill`, `requeue`, and `remove`. The only piece of information the software running DDQ will receive is the message and the callback from DDQ in order to say whether the processing of the message was successful or not. When the message is being processed DDQ with call the `heartbeat` method on the `wrappedMessage` using `heartbeatDelay` from the config to tell the `backend` to update the heartbeat at that interval.

When the `data` is done processing the callback will be called and DDQ will take action whether the software told it was successful or not. First the `heartbeatKill` method will be called. This tells the `backend` DDQ is done processing. When not successful the message needs to be requeued. This of course uses the `requeue` method. When the message is successful, we don't need it hanging around and thus the `remove` method is invoked. Since there is the separation of work, DDQ doesn't care what the `backend` does with these commands, just that it called them when appropriate.


    // Starts listening for events.
    deduplicatedQueue.listen();

    /* DDQ uses event emitters and you'll need to listen for them so your
     * applications can take actions.
     */
    deduplicatedQueue.on("data", (message, callback) => {
        // Take action on the data.
        if (somethingBadHappened) {
            callback(new Error("Something bad happened"));
        } else {
            callback();
        }
    });

The other event DDQ will emit is `error`. This alerts the code listening there was an error either coming from DDQ or passed back from the backend.

    deduplicatedQueue.on("error", () => {
        // Take action on the error.
    });

Pausing
-------

At some point your consumer might want to pause the polling of messages so it can do a task without having messages coming at it. You can do this by simply calling the DDQ method `pausePolling`. This only works if you are listening for events though. This also sets the flag `pausedByUser` to true within your instance of DDQ.  This doesn't use a `callback`, for if there is an error it should be emitted and picked up by the `listener`.

    // Tells the backend to stop polling.
    deduplicatedQueue.pausePolling();

Resume Polling
--------------

Once you feel you should resume polling you can call the `resumePolling` method. This will set the flag `pausedByUser` back to false and check if the `pausedByLimits` flag was set before telling the `backend` to resume it's polling. We don't want to resume polling if the limit has been reached. This also doesn't use a `callback`, for if there is an error it should be emitted and picked up by the `listener`.

    // Tells the backend to resume polling.
    deduplicatedQueue.resumePolling();

Closing
-------

At some time you'll want to stop listening to DDQ which you will call `close`. This will call the backend to close it's connections and emitting.

    // Stop listening to events and close the connection to the database.
    deduplicatedQueue.close((err) => {
        if (err) {
            // Take action if there is an error.
        }
    });

[Code Coverage]: https://codecov.io/github/tests-always-included/ddq?branch=master
[codecov-image]: https://codecov.io/github/tests-always-included/ddq/coverage.svg?branch=master
[Dev Dependencies]: https://david-dm.org/tests-always-included/ddq#info=devDependencies
[devdependencies-image]: https://david-dm.org/tests-always-included/ddq/dev-status.png
[Dependencies]: https://david-dm.org/tests-always-included/ddq
[dependencies-image]: https://david-dm.org/tests-always-included/ddq.png
[travis-image]: https://secure.travis-ci.org/tests-always-included/ddq.png
[Travis CI]: http://travis-ci.org/tests-always-included/ddq
