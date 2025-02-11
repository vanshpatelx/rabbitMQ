# rabbitMQ

Here’s how you can implement these RabbitMQ patterns using **TypeScript with amqplib**.  

---
---

## **Comparison Table (RabbitMQ Implementation)**

| **Scenario** | **RabbitMQ Exchange Type** | **Queue Strategy** | **Pod Distribution** |
|-------------|----------------------|------------------|------------------|
| Multiple Partitions, Multiple Services | Fanout | Separate queue per service | Each service gets all messages, pods get unique ones |
| Multiple Partitions, Single Service | Direct | Single queue per service | Load-balanced across pods |
| Single Partition, All Pods Must Receive | Fanout | Unique queue per pod | All pods get the message |

---


## **1️⃣ Multiple Partitions, Multiple Services**  
> **Goal:** Each service gets all messages, but within a service, each pod gets unique events.  
> **Solution:** Use a **fanout exchange** where each service has its own queue.
```bash
# Create fanout exchange
rabbitmqadmin declare exchange name=service_exchange type=fanout

# Create queues for each service
rabbitmqadmin declare queue name=service1_queue
rabbitmqadmin declare queue name=service2_queue

# Bind queues to the exchange
rabbitmqadmin declare binding source=service_exchange destination=service1_queue
rabbitmqadmin declare binding source=service_exchange destination=service2_queue
```

### **Producer (Publisher)**
```ts
import amqplib from 'amqplib';

async function publishMessage() {
  const connection = await amqplib.connect('amqp://localhost');
  const channel = await connection.createChannel();

  const exchange = 'service_exchange';
  await channel.assertExchange(exchange, 'fanout', { durable: true });

  const message = 'New event for all services!';
  channel.publish(exchange, '', Buffer.from(message));
  console.log(`Sent: ${message}`);

  setTimeout(() => connection.close(), 500);
}

publishMessage();
```

### **Consumer (Each Service)**
```ts
import amqplib from 'amqplib';

async function consumeMessages(serviceName: string) {
  const connection = await amqplib.connect('amqp://localhost');
  const channel = await connection.createChannel();

  const exchange = 'service_exchange';
  await channel.assertExchange(exchange, 'fanout', { durable: true });

  // Create a dedicated queue per service
  const queue = `${serviceName}_queue`;
  await channel.assertQueue(queue, { durable: true });

  // Bind queue to exchange
  await channel.bindQueue(queue, exchange, '');

  console.log(`${serviceName} waiting for messages...`);

  channel.consume(queue, (msg) => {
    if (msg) {
      console.log(`${serviceName} received: ${msg.content.toString()}`);
      channel.ack(msg);
    }
  });
}

// Run for different services
consumeMessages('service1');
consumeMessages('service2');
```
Each **service** gets the full message stream, but inside each service, messages are load-balanced among **pods**.

---

## **2️⃣ Multiple Partitions, Single Service**  
> **Goal:** Distribute messages across pods within a single service.  
> **Solution:** Use a **direct exchange** with a shared queue.

```bash
# Create direct exchange
rabbitmqadmin declare exchange name=single_service_exchange type=direct

# Create a single queue for the service
rabbitmqadmin declare queue name=service_queue

# Bind queue to the exchange
rabbitmqadmin declare binding source=single_service_exchange destination=service_queue
```

### **Producer (Publisher)**
```ts
import amqplib from 'amqplib';

async function publishMessage() {
  const connection = await amqplib.connect('amqp://localhost');
  const channel = await connection.createChannel();

  const exchange = 'single_service_exchange';
  await channel.assertExchange(exchange, 'direct', { durable: true });

  const message = 'Task for service pods';
  channel.publish(exchange, '', Buffer.from(message));
  console.log(`Sent: ${message}`);

  setTimeout(() => connection.close(), 500);
}

publishMessage();
```

### **Consumer (All Pods in One Service)**
```ts
import amqplib from 'amqplib';

async function consumeMessages(serviceQueue: string) {
  const connection = await amqplib.connect('amqp://localhost');
  const channel = await connection.createChannel();

  const exchange = 'single_service_exchange';
  await channel.assertExchange(exchange, 'direct', { durable: true });

  // Create a single queue shared across all pods
  await channel.assertQueue(serviceQueue, { durable: true });
  await channel.bindQueue(serviceQueue, exchange, '');

  console.log(`Pod connected to queue: ${serviceQueue}`);

  channel.consume(serviceQueue, (msg) => {
    if (msg) {
      console.log(`Pod received: ${msg.content.toString()}`);
      channel.ack(msg);
    }
  });
}

// Each pod should call this function with the same queue name
consumeMessages('service_queue');
```
RabbitMQ **load-balances** messages across pods automatically.

---

## **3️⃣ Single Partition, All Pods Must Receive**  
> **Goal:** Every pod in the service should receive every message.  
> **Solution:** Use a **fanout exchange**, and each pod dynamically creates a queue.

```bash
rabbitmqadmin declare exchange name=broadcast_exchange type=fanout
```

### **Producer (Publisher)**
```ts
import amqplib from 'amqplib';

async function publishMessage() {
  const connection = await amqplib.connect('amqp://localhost');
  const channel = await connection.createChannel();

  const exchange = 'broadcast_exchange';
  await channel.assertExchange(exchange, 'fanout', { durable: true });

  const message = 'Message for all pods!';
  channel.publish(exchange, '', Buffer.from(message));
  console.log(`Sent: ${message}`);

  setTimeout(() => connection.close(), 500);
}

publishMessage();
```

### **Consumer (Each Pod)**
```ts
import amqplib from 'amqplib';

async function consumeMessages() {
  const connection = await amqplib.connect('amqp://localhost');
  const channel = await connection.createChannel();

  const exchange = 'broadcast_exchange';
  await channel.assertExchange(exchange, 'fanout', { durable: true });

  // Each pod gets a unique queue
  const { queue } = await channel.assertQueue('', { exclusive: true });
  await channel.bindQueue(queue, exchange, '');

  console.log(`Pod queue ${queue} listening for messages...`);

  channel.consume(queue, (msg) => {
    if (msg) {
      console.log(`Pod received: ${msg.content.toString()}`);
      channel.ack(msg);
    }
  });
}

// Each pod runs this function
consumeMessages();
```
Each **pod receives every message**, ensuring proper broadcast.





```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: rabbitmq-config
data:
  enabled_plugins: |
    [rabbitmq_management, rabbitmq_prometheus].

---
apiVersion: v1
kind: Service
metadata:
  name: rabbitmq
  labels:
    app: rabbitmq
spec:
  clusterIP: None  # Required for StatefulSet
  ports:
    - port: 5672  # AMQP
      name: amqp
    - port: 15672 # Management UI
      name: http
    - port: 15692 # Prometheus Metrics
      name: metrics
  selector:
    app: rabbitmq

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: rabbitmq
spec:
  serviceName: "rabbitmq"
  replicas: 1
  selector:
    matchLabels:
      app: rabbitmq
  template:
    metadata:
      labels:
        app: rabbitmq
    spec:
      containers:
        - name: rabbitmq
          image: rabbitmq:management
          env:
            - name: RABBITMQ_DEFAULT_USER
              value: "admin"
            - name: RABBITMQ_DEFAULT_PASS
              value: "password"
          ports:
            - containerPort: 5672
            - containerPort: 15672
            - containerPort: 15692
          volumeMounts:
            - name: rabbitmq-storage
              mountPath: /var/lib/rabbitmq
      volumes:
        - name: rabbitmq-storage
          persistentVolumeClaim:
            claimName: rabbitmq-pvc

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: rabbitmq-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

### Implement Dead Letter Exchange (DLX)
```
import amqplib from 'amqplib';

async function publishMessage() {
  const connection = await amqplib.connect('amqp://rabbitmq');
  const channel = await connection.createChannel();

  // Declare the dead-letter exchange
  await channel.assertExchange('dlx_exchange', 'direct', { durable: true });
  
  // Declare the dead-letter queue
  await channel.assertQueue('dlq', { durable: true });
  
  // Bind DLQ to DLX
  await channel.bindQueue('dlq', 'dlx_exchange', 'dlx_routing');

  // Declare a normal queue with DLX
  await channel.assertQueue('main_queue', {
    durable: true,
    arguments: {
      'x-dead-letter-exchange': 'dlx_exchange',
      'x-dead-letter-routing-key': 'dlx_routing'
    }
  });

  // Send a message that will fail
  const message = 'This will be rejected!';
  channel.sendToQueue('main_queue', Buffer.from(message));

  console.log(`Sent: ${message}`);
  setTimeout(() => connection.close(), 500);
}

publishMessage();
```
```
import amqplib from 'amqplib';

async function consumeDLQ() {
  const connection = await amqplib.connect('amqp://rabbitmq');
  const channel = await connection.createChannel();

  await channel.assertQueue('dlq', { durable: true });

  console.log('DLQ waiting for messages...');

  channel.consume('dlq', (msg) => {
    if (msg) {
      console.log(`DLQ received: ${msg.content.toString()}`);
      channel.ack(msg);
    }
  });
}

consumeDLQ();
```
