# Testing- vcluster with NVCF blueprint

I have been hearing a lot about NVCF (NVIDIA Cloud Functions) lately so thought to read about it and learn how it works with our vcluster. Actually I wanted to try what I was reading and learning, and personally test how i can implement that NVCF-style serverless workflow inside a vcluster. I emulated and tested this entire pipeline locally on my Mac without using any actual GPU hardware.

**What is NVCF?**

In simple terms, [**NVIDIA Cloud Functions (NVCF)**](https://developer.nvidia.com/dgx-cloud/nvcf) is a specialized cloud platform that lets developers run large AI models on demand without managing servers.

Instead of renting a powerful, expensive **NVIDIA GPU** server around the clock, you upload your AI model to NVIDIA's cloud. You receive a web link (**API Endpoint**) and pay only for the seconds your model spends processing requests.

**What real-word crisis it solves?**

If you deploy a heavy AI model (like a large chatbot or image generator) on regular cloud platforms, you face a huge dilemma:

- **The High Cost:** Leaving an expensive GPU running continuously when no one is using your app wastes thousands of dollars.
- **The "Cold Start" Timeout:** To save money, you could shut the server down when idle (**Scale-to-Zero**). But loading a massive AI model into memory takes 1 to 3 minutes. Standard web systems will time out and crash before the model finishes waking up.

**How NVCF solves this?**

NVCF acts like a **smart shock-absorber** using a simple 3-step loop:

1. **The Ingress Queue:** When a user hits the API, NVCF instantly drops the request into a secure waiting lounge (buffer stream). The user's connection stays open safely without timing out.
2. **Instant Scaling:** The platform detects the request, instantly spins up an NVIDIA GPU node in the background, and loads your AI model.
3. **Scale-to-Zero:** The warm GPU pulls the request from the queue, processes the inference instantly, and answers the user. If no more requests come in, NVCF **gracefully shuts the GPU down to absolute zero** to stop the billing clock.

**Nvidia open sources NVCF architecture:**

NVIDIA open-sourced the blueprint of their NVCF architecture under the Apache 2.0 license. Their official open-source package is built entirely on top of standard cloud-native tools: **NATS JetStream** for request buffering and **KEDA** for event-driven autoscaling.

Since I’m running this test inside a local **vcluster** without physical GPU hardware, I deployed the exact components used in the NVIDIA open-source blueprint (**KEDA + NATS JetStream**) to simulate and validate the identical 'Scale-to-Zero' loop natively.

**What is NATS Jetstream (Neural Autonomic Transport System)?**

**NATS:** A super-fast message broker that routes data using **subjects** (like text-based addresses, e.g., `ai.request`). It is strictly "fire-and-forget". if no service is listening to that subject, the message is lost.

**NATS Jetstream** acts as high-performance message stream (the waiting lounge). In simple terms, the storage layer for those subjects. When a user sends an AI request, NATS instantly catches it and holds it securely. This acts as a shock-absorber so the user's connection never drops or times out while the backend infrastructure is waking up.

**What is KEDA (Kubernetes based event-driven Autoscaling)?**

It is an open-source tool that allows you to scale any container in **Kubernetes** based on the number of events needing to be processed, rather than just relying on standard metrics like **CPU or memory usage**.

Standard Kubernetes can be slow to react to sudden bursts of traffic because it mostly looks at hardware metrics (like a server reaching 80% CPU).

**KEDA solves this** by looking directly at your event sources. For example, if you connect KEDA to a **NATS JetStream** message queue, KEDA can count exactly how many unread messages are waiting. If 1,000 messages suddenly arrive, KEDA will instantly scale up your worker containers from 0 to 50 to process the load, and then scale them right back down to 0 once the queue is empty.

It serves as the intelligent autoscaling agent. It polls the NATS headless metric endpoint via the global tracking account (`$G`) to dynamically spin up or tear down pods.

### **Putting it all together:**

Since NVCF's architecture blueprint is open-source, I emulated it inside a **vcluster** using two core tools: **NATS JetStream** and **KEDA**. NATS acts as a secure buffer stream to hold incoming requests so they never timeout, while KEDA acts as the brain that watches the queue to automatically scale our AI model from **0 to 1** on-demand, and back to **0** when idle.

## **Step performed in Lab (mac):**

1. kind create cluster --name nvcf-host
2. vcluster create my-ai-vcluster --namespace ai-team
3. Install Core Software (Inside **`vcluster`**)

#### Install KEDA Autoscaler

```yaml
helm repo add kedacore [https://kedacore.github.io/charts](https://kedacore.github.io/charts)
helm repo update
helm install keda kedacore/keda --namespace keda --create-namespace
```

#### Install NATS with JetStream & Monitoring Port 8222 enabled

```yaml
helm repo add nats [https://nats-io.github.io/k8s/helm/charts/](https://nats-io.github.io/k8s/helm/charts/)
helm repo update
helm install my-nats nats/nats \
--set config.jetstream.enabled=true \
--set config.jetstream.memoryStore.enabled=true \
--set config.cluster.enabled=true \
--set config.http.port=8222
```

#### The Core Manifest Logic: Worker & Manager:

To run a successful event-driven serverless system, we fundamentally need two things: a worker to do the actual compute processing, and a manager to control the infrastructure lifecycles. We created these two specific files to fulfill these exact roles:

### Create `dummy-ai-app.yaml` (The Worker):

> **Purpose:** This manifest deploys our core AI workload (simulating a heavy framework like an LLM or a diffusion model).
It represents our application worker. It is configured natively with replicas: 0 so that it stays completely turned off and asleep when there is no incoming traffic, dropping our computing overhead to absolute $0 when idle.
> 

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dummy-ai-model
  namespace: default
spec:
  replicas: 0 # Starts at absolute zero resource consumption
  selector:
    matchLabels:
      app: dummy-ai-model
  template:
    metadata:
      labels:
        app: dummy-ai-model
    spec:
      containers:
      - name: ai-worker
        image: python:3.9-slim
        command: ["python3", "-m", "http.server", "8080"]
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: dummy-ai-service
  namespace: default
spec:
  ports:
  - port: 8080
    targetPort: 8080
  selector:
    app: dummy-ai-model
```

### Create `nvcf-trigger.yaml` (The Manager)

> **Purpose:** This is the autoscaling rule-book (a KEDA ScaledObject) that serves as the controller configuration for our automation engine.
It acts as the smart operations manager. It instructs KEDA to constantly monitor our NATS JetStream data queue. The moment a request arrives, it commands the sleeping worker (dummy-ai-model) to scale up from 0 to 1 to process the request, and cuts it back to 0 after a 15-second cooldown period when the queue is empty.
> 

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: nvcf-autoscaler
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: dummy-ai-model
  minReplicaCount: 0
  maxReplicaCount: 5
  cooldownPeriod: 15
  triggers:
  - type: nats-jetstream
    metadata:
      natsServerMonitoringEndpoint: "my-nats-headless.default.svc.cluster.local:8222"
      account: "$G"
      stream: "AI_REQUESTS"
      consumer: "NVCF_WORKER"
      lag: "1"

```

YAML reference from [https://keda.sh/docs/2.21/scalers/nats-jetstream/](https://keda.sh/docs/2.21/scalers/nats-jetstream/)

5. **Apply then to the vcluster:**

```yaml
kubectl apply -f dummy-ai-app.yaml
kubectl apply -f nvcf-trigger.yaml
```

1. **Create the Stream and Consumer.**

Before sending any traffic, we must manually prepare the internal infrastructure channels inside NATS. KEDA is just an external inspector (autoscaler),it only monitors metrics, it does not create the queues or tracking systems for us.

**Register the permanent memory stream**

```yaml
kubectl run nats-setup --image=natsio/nats-box:latest --rm -it --restart=Never -- nats stream add AI_REQUESTS --server=nats://my-nats.default.svc.cluster.local:4222 --subjects="AI_REQUESTS" --ack --max-msgs=-1 --max-bytes=-1 --max-age=-1 --storage=memory --replicas=1 --defaults
```

**Purpose:** NATS is completely empty by default. This command creates a permanent storage cabinet inside NATS called `AI_REQUESTS` . Without this, incoming messages would instantly vanish in the air because there is no folder to hold them while our AI pod is slowly booting up.

**Register the consumer window for KEDA**

```yaml
kubectl exec -it deployment/my-nats-box -- nats consumer add AI_REQUESTS NVCF_WORKER --pull --deliver=all --ack=explicit --replay=instant --defaults
```

**Purpose:** This command creates a dedicated metrics tracking dashboard inside the stream called `NVCF_WORKER`. KEDA constantly checks this exact dashboard endpoint via HTTP. The moment this window reports that a message is pending and unprocessed, KEDA reads that number and triggers the deployment scale-up.

**Now check the use-case:**

First, open an independent terminal tab (vcluster context) and run the live monitoring hook to show your team the state of the pods:

```yaml
kubectl get pods -w
```

(Notice that `dummy-ai-model` doesn't exist yet. it's at `0` replicas).

Now, go to your main terminal window and trigger the automated delayed load script. It places a request in the queue, logs the step, holds it for 40 seconds to give KEDA time to cycle, processes the message, and exits:

```yaml
kubectl run auto-tester --image=natsio/nats-box:latest --rm -it --restart=Never -- /bin/sh -c 'nats pub -s nats://my-nats.default.svc.cluster.local:4222 AI_REQUESTS "Auto-Inference-Job" && echo "Waiting 40s for KEDA to scale up..." && sleep 40 && nats consumer next AI_REQUESTS NVCF_WORKER -s nats://my-nats.default.svc.cluster.local:4222 --count 1 --ack && echo "Job Acknowledged! Waiting for Scale-to-Zero..."’
```

Now, go back to terminal where POD watch (-w) is running and below is what you will see:

1. **0s - 10s:** The request hits NATS JetStream. KEDA reads the queue via the headless routing metrics port `8222` and commands the deployment to scale up. You will watch **`dummy-ai-model-xxxx` transition instantly into `Running` state.**
2. **40s:** The client utility reaches out, grabs the message, and responds with `Acknowledged message`, simulating a successful inference output clearing the database backlog.
3. **55s:** KEDA registers that the pipeline holds 0 pending records, initiates its internal clock, and automatically commands the cluster to delete the container. The pod prints **`Terminating` and completely vanishes back to 0 replicas,** achieving the full, perfect serverless AI loop!