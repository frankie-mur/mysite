+++
title = 'Go with the Flow: Event Streaming with Go and Kafka'
date = 2024-07-27T11:39:57-07:00
draft = false
tags = ["golang", "kafka"]
+++
# What is Kafka?

Kafka is an event streaming platform used to collect and process data streams at scale, it was Initially developed at LinkedIn in 2011. It is now open-sourced and part of the Apache Software Foundation. It is a JVM application written in Java and Scala.

<div data-node-type="callout">
<div data-node-type="callout-emoji">💡</div>
<div data-node-type="callout-text">The name Kafka is inspired by the famous Bohemian writer Franz Kafka, and the term <em>Kafkaesque</em> meaning something that is extremely unpleasant, frightening, and confusing.</div>
</div>

**Notable Companies Using Kafka**:

* **LinkedIn**: Kafka originated at LinkedIn. They use it for tracking operational metrics, monitoring, and event sourcing.
    
* **Netflix**: They use Kafka for real-time monitoring and event processing.
    
* **Uber**: Uses Kafka for gathering metrics from its many services spread across its transportation platform.
    
* **Spotify**: Employs Kafka for tracking user activity and updating playlists in real-time.
    

## Events

* Simply put an event is "a thing that happened", it is a combination of Notification and State. Notable examples include
    
    * Internet of Things (If my thermostat reads 80)
        
    * User Interaction (User hovers over a black dress)
        
    * Microservice output
        
    * Business process change
        
* Kafa data model for events is a key-value pair
    
    * Event Key: "20393" (Representing a user ID)
        
        * Usually represented as a string or integer
            
    * Event Value: "Hovered over a black dress for 3 seconds"
        
        * Usually represented as one of JSON, Avro, Protocol Buffers
            

\*Note that I use the terms event and message interchangeably

## Topic

Now we need a way to store all these events...You can think of topics as named containers for similar events. They are really just append-only log files that contain events. All events must be sent to a topic, for example, we can create a topic named "user\_metadata", and send our event to it. Important things to note about topics are

* They are immutable, they cannot be deleted or destroyed
    
* Topics are durable, retention period is configurable
    
* They can only seek by offset, not indexed (this is what makes Kafka so fast!)
    

## Producer

A Producer is a client application that publishes messages to Kafka. A producer will set the Key and Value of an event and send to a specific topic.

## Consumer

A Consumer is also a client application that reads and processes events from Kafka. A Consumer must *subscribe* to a topic in order to receive that topic's events. There can be many consumers for a single topic

## Broker

A Broker refers to a single Kafka server instance in a Kafka cluster. The broker is responsible for receiving data (writes) from producers, storing this data, and serving it to consumers for reads. The Broker also manages the offset (or position) for consumer groups

---

This is all we need to know to get started, however, I would like to quickly mention a couple of other core Kafka components that we will not get to in this project:

**Partitions**: Segments within a Kafka topic that allow data to be distributed and processed in parallel.

**Replicas**: Copies of a Kafka partition that provide fault tolerance and data redundancy, ensuring data availability even if a server fails.

**Consumer groups**: A set of consumers working together to consume and process data from one or multiple topics, ensuring each message is processed once and only once by one member of the group.

**Kraft**: A way for Kafka to manage its internal organization without relying on an external helper, making it simpler and more self-contained.

## Project Time

Today we will create a Go project, simulating a real use case for Kafka. Real-time fraud detection, to get started make sure you have [Go](https://go.dev/doc/install) installed. Then create a directory where you want your project to live.

```bash
mkdir kafka-fraud-detection
```

Here we will be running Kafka on a Docker instance that can easily be spun up. Use the gist below as the content for your `docker-compose.yaml` file.

%[https://gist.github.com/frankie-mur/d1591e3f347a76998de9b1c7d3293d7f] 

```bash
docker-compose up
```

### Project Structure

Great now we have Kafka running in a docker container and ports forwarded to our local host this will allow us to connect to Kafka, let's set up the project structure we want to create a couple of folders

```bash
mkdir -p cmd/producer cmd/consumer pkg/models pkg/bankaccount
```

Finally, we want to initialize Go modules and install external packages. We will be using [`sarama`](https://github.com/IBM/sarama) it to establish a connection to our Kafka broker and [`faker`](https://github.com/go-faker/faker) to generate bank account creation events.

```bash
go mod init kafka-fraud-detection
go get github.com/IBM/sarama github.com/ github.com/bxcodec/faker/v4
```

### Create our Models and Faker Data

Let's start by setting up our BankAccount model and associated function to generate fake bank account events let's create a file `pkg/models/models.go`

```go
package models

type BankAccount struct {
	FirstName        string  `faker:"first_name"`
	LastName         string  `faker:"last_name"`
	CreditCardNumber string  `faker:"cc_number"`
	Amount           float64 `faker:"amount"`
}
```

Great, This is simulating Bank Account data that a consumer will need to verify. Now let's create that function in `/pkg/bankaccount/BankAccount.go`

```go
package bankaccount

import (
	"fmt"
	"github.com/bxcodec/faker/v4"
	"github.com/frankie-mur/go-kafka/pkg/models"
)

func GenerateBankAccount() (*models.BankAccount, error) {
	bankAccount := models.BankAccount{}
	err := faker.FakeData(&bankAccount)
	if err != nil {
		return nil, fmt.Errorf("failed to generate bank account: %w", err)
	}
	return &bankAccount, nil
}
```

This function uses our annotated BankAccount struct and the `faker` library to generate fake data and return that data.

### Setting up our Producer

Now that we have all that setup let's get to the fun part, setting up our Producers and Consumers. We will start with our Producer by creating a new file in `cmd/producer/producer.go` and creating the function

```go
const (
	KafkaServerAddress = "localhost:9092"
	KafkaTopic         = "bankaccount"
)

func setupProducer() (sarama.SyncProducer, error) {
	config := sarama.NewConfig()
	config.Producer.Return.Successes = true
	producer, err := sarama.NewSyncProducer([]string{KafkaServerAddress},
		config)
	if err != nil {
		return nil, fmt.Errorf("failed to setup producer: %w", err)
	}
	return producer, nil
}
```

Let's break down this code a bit

* Define a function `setupProducer` that creates and configures a Kafka producer client. This function will return a `SyncProducer` if it succeeds or an `error` if fails
    
* Create a `sarama.Config` struct to hold producer configuration set `Producer.Return.Successes` to true so the producer will return success metadata for each message.
    
* Call `sarama.NewSyncProducer` to create a new synchronous Kafka producer, passing the broker address and config. It returns the producer instance and any error from producer creation.
    

Simple enough, now let's create our main function that will call the created function above

```go
func main() {
    p, err := setupProducer()
	if err != nil {
		log.Fatalf("Failed to setup producer: %v", err.Error())
	}
	defer p.Close()
}
```

* Create our Kafka producer and assign it to a variable `p`
    
* Handle any errors creating the producer
    
* Defer closing the producer `p` connection unitl the function exits
    

#### Now let's create our message-sending function!

Now we have a producer and can send messages let's abstract that a bit and create another function that handles sending messages

```go
func sendKafkaMessage(producer sarama.SyncProducer, topic string, message []byte) error {
	msg := &sarama.ProducerMessage{
		Topic: topic,
		Value: sarama.ByteEncoder(message),
	}
	_, _, err := producer.SendMessage(msg)
	return err
}
```

* Define a function `sendKafkaMessage` that sends a message to Kafka using a provided `producer`. It accepts the producer, topic name, and message payload as parameters.
    
* Create a `sarama.ProducerMessage` struct with the topic name and message payload.
    
* use the `producer.SendMessage` method to send the message to Kafka asynchronously.
    
* Note: As you can see we only send a value, what about the key? In Kafka sending keys is optional. In our use case, we will omit the key.
    

Return any error from the send operation.

<div data-node-type="callout">
<div data-node-type="callout-emoji">🗒</div>
<div data-node-type="callout-text">If a topic is not yet created on a message send, it will automatically be created by Kafka. This behavior can be configured.</div>
</div>

Now back to our main function to add some code....

```go
func main() {
	p, err := setupProducer()
	if err != nil {
		log.Fatalf("Failed to setup producer: %v", err.Error())
	}
	defer p.Close()
    size := 10
	for i := 0; i < size; i++ {
		//Generate fake bank account data
		bank, err := bankaccount.GenerateBankAccount()
		if err != nil {
			fmt.Printf("Failed to generate bank account: %v", err.Error())
		}
		//Convert the bank account data to []byte
		data, err := json.Marshal(bank)
		if err != nil {
			fmt.Printf("Failed to marshal bank account: %v", err.Error())
		}
		//Send the message
		err = sendKafkaMessage(p, KafkaTopic, data)
        if err != nil {
			fmt.Printf("Failed to send message: %v", err.Error())
		}
	}
}
```

* Configure a size of 10, to send 10 messages (or events) to Kafka
    
* Generate random fake bank account data using `bankaccount.GenerateBankAccount()` function we created
    
* We must convert our bank data to `[]byte` to send as a message in kafka so we use `json.Marshal(bank)` to do that for us (basically just converting our struct to stringified json)
    
* Finally, we send the message and check for error
    

Done! Now that we can produce events lets create a Consumer to consume them!

### Setting up our Consumer

Similar to our `producer.go` we will start with writing a function to connect as a consumer to Kafka. Lets create a new file in `cmd/consumer/consumer.go`

```go
const (
	KafkaServerAddress = "localhost:9092"
	KafkaTopic         = "bankaccount"
)

func connectConsumer() (sarama.Consumer, error) {
	config := sarama.NewConfig()
	config.Consumer.Return.Errors = true
	config.Consumer.Offsets.Initial = sarama.OffsetNewest
	// Create new consumer
	conn, err := sarama.NewConsumer([]string{KafkaServerAddress}, config)
	if err != nil {
		return nil, err
	}
	return conn, nil
}
```

This code:

* Defines constants for the Kafka server address and topic name.
    
* Defines a `connectConsumer` function to create a Kafka consumer client.
    
* Creates a sarama.Config with settings:
    
    * `Consumer.Return.Errors` set to true will return any errors that occurred during the consumption of an event
        
    * `Consumer.Offsets.Initial` set to `sarama.OffsetNewest`, this will set the newly created Consumers offset to be the newest. This will make sure the Consumer only consumes events that happened during or after its creation.
        
* Calls `sarama.NewConsumer` to create the client, passing the broker address and config.
    
* Returns the new consumer instance and any error.
    

Great, now let's create our main function

```go
func main() {
	worker, err := connectConsumer()
	if err != nil {
		panic(err)
	}
	// Calling ConsumePartition. It will open one connection per broker
	// and share it for all partitions that live on it.
	consumer, err := worker.ConsumePartition(KafkaTopic, 0, sarama.OffsetNewest)
	if err != nil {
		panic(err)
	}
	fmt.Println("Consumer started ")
	sigchan := make(chan os.Signal, 1)
	signal.Notify(sigchan, syscall.SIGINT, syscall.SIGTERM)
	// Get signal for finish
	doneCh := make(chan struct{})
}
```

Some things to point out here:

* we call `worker.ConsumePartition()` to start consuming messages from partition 0 of the KafkaTopic, starting from the newest offset.
    
* we create sig chan channel to receive OS signals like Ctrl+C. To gracefully shutdown or log errors
    

Now we need to handle when a consumer receives events

```go
func main() {
...
    //spins up a goroutine to fetch messages from Kafka
	//and handle them, while also monitoring for shutdown signals
	go func() {
		for {
			select {
			case err := <-consumer.Errors():
				fmt.Println(err)
			case msg := <-consumer.Messages():
				//Handle the message received  from Kafka
				err := handleMessage(msg)
				if err != nil {
					fmt.Printf("Failed to handle message for consumer with key: %v, with error: %v\n", msg.Key, err)
				}
			case <-sigchan:
				fmt.Println("Interrupt is detected")
				doneCh <- struct{}{}
			}
		}
	}()

    <-doneCh
	fmt.Println("Processed all messages, awaiting more...")

	if err := worker.Close(); err != nil {
		panic(err)
	}

}
```

without going too much into goroutines when our consumer receives an Error or Message it will be caught in our `select` statement, let's define our \`handleMessage\` function...

```go
func handleMessage(msg *sarama.ConsumerMessage) error {
	fmt.Printf("Received message | Topic(%s) | Message(%s) \n", msg.Topic, string(msg.Value))
	//Parse the message back into our BankAccount struct
	bankAccount := models.BankAccount{}
	err := json.Unmarshal(msg.Value, &bankAccount)
	if err != nil {
		return fmt.Errorf("failed to parse message: %w", err)
	}
	//Call our fake Veritas API
	succeeded := callVeritas(bankAccount)
	if !succeeded {
		fmt.Printf("--UNAUTHORIZED-- bank account for user %s\n", bankAccount.FirstName)
	} else {
		fmt.Printf("--AUTHORIZED-- bank account for user %s\n", bankAccount.FirstName)
	}
	return nil
}

// Simulate calling an actual verification service
// randomly return true or false
func callVeritas(bankAccount models.BankAccount) bool {
	fmt.Printf("Calling Veritas for %s bank...\n", bankAccount.FirstName)
	return rand.Intn(2) == 0
}
```

* Notice we Unmarshal `msg.Value` (Remember these are key-value pairs!) to our BankAccount struct. Call our fake Authorization API that we call [`Vertias`](https://www.dictionary.com/e/translations/veritas-aequitas/), and print out the results.
    

#### Finished! We now have both our Producer and our Consumer, let's use them.

Let's start with running our consumer, go to the root of the project and run the command in your terminal

![Screenshot](https://cdn.hashnode.com/res/hashnode/image/upload/v1698274247768/5da84e9e-f59f-44fd-8474-9b41fc92b85c.png)

Our consumer is now running and listening for events of its subscribed topic...in our case the topic `bankaccount`.

Now let's send some events! Open a new terminal (or split) and once again go to the root of the project and run the command

![Screenshot](https://cdn.hashnode.com/res/hashnode/image/upload/v1698274350519/d850de71-9c50-4bda-b54b-014c11a0b030.png)

Did you see that?! If you were watching the Consumer terminal you would see something similar to this

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1698274401523/071a5f52-8626-4877-ad4b-fb6dd6317496.png)

We did it! If you were following along here is a quick overview of the entire flow

1. Our producer sent an event to Kafka with the topic `bankaccount` and the value `{"FirstName":"Jimmie","LastName":"Swaniawski","CreditCardNumber":"3558018114425614","Amount":644.78}`
    
2. Our consumer is subscribed to the topic `bankaccount`, therefore it receives our message
    
3. The message is Unmarshalled and sent to our (fake) Authorization service Veritas.
    
4. It is Verified (or in our case Unauthorized) and the result is printed to the console
    

<div data-node-type="callout">
<div data-node-type="callout-emoji">💡</div>
<div data-node-type="callout-text">Try adjusting the <code>size</code> variable in our Producer.go file. Set it to 100, 1000, 10000. This is the beauty of Kafka, it is built for high throughput.</div>
</div>

source code: [https://github.com/frankie-mur/go-kafka-fraud-detection/tree/main](https://github.com/frankie-mur/go-kafka-fraud-detection/tree/main)
