# **BigBroker Project Requirements**

## **Overview**
The BigBroker project is a message broker built using the **Bulldog BigQueue** library (available on GitHub). It establishes a server-push-based messaging system over **HTTP/2**. While clients initiate flow control, the broker manages the flow. The broker will provide features for high performance, scalability, and high availability.

---

## **Core Features**

### **Message Handling**
1. **Message Structure**:
   - All messages should be JSON objects with the following:
     - **Headers**: Contain metadata such as timestamps or unique IDs.
     - **Body**: Contains the actual payload.

2. **Message Consumption**:
   - Clients can establish **persistent connections** to consume messages.
   - Parameters for consumption include:
     - **Batch Size**: Number of messages in a batch (JSON array).
     - **Interval**: Time between message pushes.
   - The broker pushes messages to clients based on these parameters.
   - Consumers should be able to:
     - Dynamically adjust the batch size.
     - Disconnect if overwhelmed.

3. **Compression**:
   - Utilize **HTTP/2 header compression** or explicitly enable **GZIP** to reduce bandwidth.
   - JSON payloads should also be compressed to optimize storage and bandwidth.

4. **Message Filtering**:
   - Allow clients to filter messages based on header properties (e.g., `Priority`).

---

## **Consumer Management**
1. **Consumer Registry**:
   - Maintain a registry of all active consumers.
   - Expose APIs for server-side management:
     - View all consumers.
     - Stop message publishing to specific consumers.
     - Adjust batch size for specific consumers dynamically.
     - Support queue creation, deletion, and updates.

2. **Management API**:
   - Provide a REST API over a dedicated management port for the above operations.

---

## **High Availability and Replication**
1. **Multiple Nodes**:
   - Support high availability (HA) with multiple nodes.
   - Replicate the **consumer registry** across all nodes.
   - Clients will connect to multiple nodes using **HAProxy**.

2. **Queue Replication**:
   - Designate a **leader queue** for publishing and consumption.
   - On any `push`/`pop` operation:
     - Write to the leader queue.
     - Replicate the message in **rolling logs** (similar to Logback’s asynchronous appender) on follower nodes:
       - One log for **push** operations.
       - One log for **pop** operations.

3. **Log Processing**:
   - Each replica node processes logs to synchronize with the leader:
     - Logs are written into an **H2 database** with separate tables for `push` and `pop` logs.
     - Periodic cleanup with a query:  
       ```sql
       DELETE FROM push WHERE id IN (SELECT id FROM pop);
       ```

4. **Leader Election with ZooKeeper**:
   - Use **Apache ZooKeeper** for leader election:
     - Each node registers itself with ZooKeeper.
     - Upon leader failure, ZooKeeper facilitates a new leader election.
   - The newly elected leader:
     - Consumes remaining rolling logs for `push` and `pop`.
     - Executes cleanup (`DELETE` query).
     - Replays any unconsumed `push` messages to initialize its queue.
     - **Non-blocking behavior**: Serve publishers and consumers immediately upon election while recovering in the background.

---

## **Broker Console**
1. **Frontend Application**:
   - A **Vue.js** application built with the **Quasar** framework.
   - Secured with a username and password, statically defined in the configuration file.

2. **Features**:
   - Monitor all queues with metrics:
     - Push throughput (TPS).
     - Consume throughput (TPS).
   - View and manage consumers.
   - Create, update, and delete queues.
   - Monitor replication backlogs and leader election processes.

---

## **REST APIs for Monitoring**
1. **Leader Election Status**:
   - Publish the current leader node and its status.
   - Provide API to query leader election history.

2. **Throughput Metrics**:
   - Provide APIs to expose:
     - **Publisher TPS**: Transactions per second for publishing.
     - **Consumer TPS**: Transactions per second for consuming.
   - Metrics should be updated in real time and exposed for UI consumption.

3. **Consumer and Queue Details**:
   - APIs to list all queues and their statuses.
   - APIs to list all consumers, their batch size, and connection status.

4. **Replication Backlog**:
   - Provide APIs to expose replication backlog metrics for each node.

---

## **Health and Monitoring**
1. **Health URLs for Nodes**:
   - Each node in the cluster should expose a health check endpoint (e.g., `/actuator/health`).
   - These URLs will be used by **HAProxy** to monitor node health and distribute client requests accordingly.

2. **Prometheus & Grafana Integration**:
   - Expose Prometheus-compatible metrics endpoints:
     - `/actuator/prometheus`: For collecting broker metrics, including queue metrics, throughput (TPS), replication status, and node health.
   - Set up **Grafana dashboards** to visualize:
     - Publisher TPS and Consumer TPS.
     - Queue health and replication status.
     - Node health and leader election status.

---

## **Sample Client Project**
1. **Purpose**:
   - Provide a reference implementation for interacting with the message broker.
   - Serve as a starting point for developers building applications on top of BigBroker.

2. **Features**:
   - **Message Publishing**:
     - Allow clients to send messages with headers and body.
     - Support bulk publishing with a JSON array of messages.
   - **Message Consumption**:
     - Allow clients to establish a persistent connection.
     - Specify parameters like batch size and interval.
     - Dynamically adjust batch size or disconnect as needed.
     - Demonstrate filtering based on header properties (e.g., `Priority`).

3. **Implementation**:
   - Written in **Java** using **HTTP/2** features and **Spring Boot** framework.
   - Utilize **Jackson** for JSON handling.
   - Include basic examples of:
     - Configuring a connection to the broker.
     - Sending and receiving messages.
     - Handling dynamic adjustments (batch size, filters, etc.).

4. **Repository Setup**:
   - The sample project should have a clear structure with:
     - Documentation for setup and usage.
     - Scripts for running a local broker for testing.
     - Sample configuration files.

---

## **Docker Compose Setup**
Provide a `docker-compose.yml` to easily test the broker cluster setup locally.

### Example:
```yaml
version: "3.8"

services:
  zookeeper:
    image: zookeeper:3.8
    container_name: zookeeper
    ports:
      - "2181:2181"

  broker1:
    build: ./broker
    container_name: broker1
    environment:
      - SPRING_PROFILES_ACTIVE=node1
    ports:
      - "8081:8080"
      - "9091:9090" # Prometheus

  broker2:
    build: ./broker
    container_name: broker2
    environment:
      - SPRING_PROFILES_ACTIVE=node2
    ports:
      - "8082:8080"
      - "9092:9090" # Prometheus

  haproxy:
    image: haproxy:2.7
    container_name: haproxy
    volumes:
      - ./haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg
    ports:
      - "80:80" # Load Balancer

  prometheus:
    image: prom/prometheus
    container_name: prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
