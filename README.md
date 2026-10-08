\# Payment Gateway \& Processor — Kubernetes DevOps Take-Home



\## Overview



This project deploys the provided Payment Gateway and Payment Processor microservices on a local Kubernetes cluster.



The objective is to:



\- Deploy both services on Kubernetes.

\- Enable communication between the Gateway and Processor.

\- Secure the internal service-to-service communication using a Kubernetes Secret.

\- Expose only the Payment Gateway outside the cluster.

\- Configure resource requests and limits.

\- Provide health and readiness checks.

\- Make the setup reproducible using Kubernetes manifests.

\- Apply production-minded DevOps practices such as service discovery, secrets management, resource management, observability, and reliability considerations.



The application consists of two services:



| Service | Role |

|---|---|

| Payment Gateway | Accepts payment requests through `POST /pay` and forwards them to the Processor |

| Payment Processor | Authenticates the Gateway, processes the payment, and returns the result |



The provided Docker images are used without modification.



\---



\## Architecture



```text

&#x20;                        Client

&#x20;                          |

&#x20;                          | HTTP POST /pay

&#x20;                          v

&#x20;               +----------------------+

&#x20;               |   Payment Gateway    |

&#x20;               |      Port 8080       |

&#x20;               |      Deployment      |

&#x20;               +----------+-----------+

&#x20;                          |

&#x20;                          | HTTP POST /process

&#x20;                          | Authorization: Bearer <token>

&#x20;                          | X-Request-ID

&#x20;                          |

&#x20;                          v

&#x20;               +----------------------+

&#x20;               | Payment Processor    |

&#x20;               |    ClusterIP Service |

&#x20;               |      Port 8080       |

&#x20;               +----------+-----------+

&#x20;                          |

&#x20;                          v

&#x20;               +----------------------+

&#x20;               | Payment Processor    |

&#x20;               |      Deployment      |

&#x20;               |      Port 8080       |

&#x20;               +----------------------+

```



\### Kubernetes Networking



The Payment Processor is exposed internally using a `ClusterIP` Service.



The Gateway communicates with it using Kubernetes DNS:



```text

http://payment-processor:8080

```



The Payment Gateway is exposed externally using a `NodePort` Service for local testing.



The Payment Processor is not exposed outside the Kubernetes cluster.



\---



\## Kubernetes Resources



The deployment consists of the following resources:



```text

payment-system namespace

│

├── payment-gateway Deployment

│   └── payment-gateway Pod

│

├── payment-gateway Service

│   └── NodePort

│

├── payment-processor Deployment

│   └── payment-processor Pod

│

├── payment-processor Service

│   └── ClusterIP

│

└── payment-service-token Secret

```



\---



\## Repository Structure



```text

payment-devops/

│

├── namespace.yaml

├── payment-gateway-deployment.yaml

├── payment-gateway-service.yaml

├── payment-processor-deployment.yaml

├── payment-processor-service.yaml

├── .gitignore

└── README.md

```



\---



\## Prerequisites



The following tools are required:



\- Docker Desktop

\- Minikube

\- kubectl

\- Git



Verify the installations:



```bash

docker --version

minikube version

kubectl version --client

git --version

```



The deployment uses the Docker driver for Minikube.



\---



\## 1. Start the Kubernetes Cluster



Start Minikube:



```bash

minikube start --driver=docker

```



Verify that the node is ready:



```bash

kubectl get nodes

```



Expected:



```text

NAME       STATUS   ROLES           AGE   VERSION

minikube   Ready    control-plane   ...   ...

```



\---



\## 2. Create the Namespace



Apply the namespace manifest:



```bash

kubectl apply -f namespace.yaml

```



Verify:



```bash

kubectl get namespace payment-system

```



All application resources are deployed into the `payment-system` namespace.



\---



\## 3. Create the Service Authentication Secret



The Gateway communicates with the Processor using a shared bearer token.



The token is stored in a Kubernetes Secret instead of being hard-coded in the Deployment manifests.



Create the Secret:



```bash

kubectl create secret generic payment-service-token \\

&#x20; --from-literal=SERVICE\_TOKEN=<YOUR\_TOKEN> \\

&#x20; -n payment-system

```



For PowerShell:



```powershell

kubectl create secret generic payment-service-token `

&#x20; --from-literal=SERVICE\_TOKEN=<YOUR\_TOKEN> `

&#x20; -n payment-system

```



Verify that the Secret exists:



```bash

kubectl get secret -n payment-system

```



The actual token must never be committed to Git.



\---



\## 4. Deploy the Payment Processor



Deploy the Processor:



```bash

kubectl apply -f payment-processor-deployment.yaml

kubectl apply -f payment-processor-service.yaml

```



Verify:



```bash

kubectl get pods -n payment-system

kubectl get svc -n payment-system

```



The Processor Service should be:



```text

payment-processor   ClusterIP   ...   8080/TCP

```



The Gateway can therefore access the Processor using:



```text

http://payment-processor:8080

```



\---



\## 5. Deploy the Payment Gateway



Deploy the Gateway:



```bash

kubectl apply -f payment-gateway-deployment.yaml

kubectl apply -f payment-gateway-service.yaml

```



Verify:



```bash

kubectl get pods -n payment-system

kubectl get svc -n payment-system

```



The Gateway Service is exposed using `NodePort`.



\---



\## 6. Verify the Deployment



Check the Pods:



```bash

kubectl get pods -n payment-system

```



Expected:



```text

NAME                                   READY   STATUS

payment-gateway-...                    1/1     Running

payment-processor-...                  1/1     Running

```



Check Services:



```bash

kubectl get svc -n payment-system

```



Expected:



```text

NAME                 TYPE        PORT(S)

payment-gateway      NodePort    8080:xxxxx/TCP

payment-processor    ClusterIP   8080/TCP

```



\---



\## 7. Access the Payment Gateway



With Minikube, retrieve the Gateway URL:



```bash

minikube service payment-gateway -n payment-system --url

```



Example:



```text

http://127.0.0.1:<PORT>

```



Use the URL returned by Minikube for the API tests below.



\---



\## API Endpoints



\### Payment Gateway



| Method | Endpoint | Purpose |

|---|---|---|

| POST | `/pay` | Accept and process a payment |

| GET | `/healthz` | Liveness check |

| GET | `/readyz` | Readiness check |

| GET | `/metrics` | Prometheus metrics |



\### Payment Processor



| Method | Endpoint | Purpose |

|---|---|---|

| POST | `/process` | Process a payment |

| GET | `/healthz` | Liveness check |

| GET | `/readyz` | Readiness check |

| GET | `/metrics` | Prometheus metrics |



\---



\## Health Check



Test the Gateway:



```bash

curl http://127.0.0.1:<PORT>/healthz

```



Expected:



```json

{

&#x20; "status": "ok",

&#x20; "service": "payment-gateway",

&#x20; "version": "1.0.0"

}

```



The health endpoint confirms that the application process is running.



\---



\## Readiness Check



Test the Gateway readiness endpoint:



```bash

curl http://127.0.0.1:<PORT>/readyz

```



The Gateway readiness check verifies that the Payment Processor is reachable.



Expected:



```json

{

&#x20; "status": "ready",

&#x20; "detail": "processor reachable"

}

```



If the Processor becomes unavailable, the Gateway readiness check can fail.



\---



\## Metrics



Both services expose a Prometheus-compatible metrics endpoint:



```text

/metrics

```



For example:



```bash

curl http://127.0.0.1:<PORT>/metrics

```



In a production Kubernetes environment, these metrics could be collected by Prometheus and visualized using Grafana.



\---



\## Make a Payment



The Gateway accepts payment requests through:



```text

POST /pay

```



Example:



```bash

curl -X POST http://127.0.0.1:<PORT>/pay \\

&#x20; -H "Content-Type: application/json" \\

&#x20; -d '{

&#x20;   "amount": 100.50,

&#x20;   "currency": "INR",

&#x20;   "payer": "acc-1",

&#x20;   "payee": "acc-2",

&#x20;   "reference": "inv-42"

&#x20; }'

```



PowerShell:



```powershell

curl http://127.0.0.1:<PORT>/pay `

&#x20; -Method POST `

&#x20; -ContentType "application/json" `

&#x20; -Body '{"amount":100.50,"currency":"INR","payer":"acc-1","payee":"acc-2","reference":"inv-42"}'

```



Expected response:



```json

{

&#x20; "payment\_id": "<payment-id>",

&#x20; "status": "approved",

&#x20; "amount": "100.50",

&#x20; "currency": "INR",

&#x20; "request\_id": "<request-id>",

&#x20; "processed\_at": "<timestamp>"

}

```



\---



\## End-to-End Request Flow



The complete request flow is:



```text

Client

&#x20; |

&#x20; | POST /pay

&#x20; v

Payment Gateway

&#x20; |

&#x20; | Validate request

&#x20; |

&#x20; | Generate X-Request-ID

&#x20; |

&#x20; | POST /process

&#x20; | Authorization: Bearer <SERVICE\_TOKEN>

&#x20; | X-Request-ID

&#x20; v

Payment Processor

&#x20; |

&#x20; | Authenticate request

&#x20; |

&#x20; | Apply payment decision

&#x20; |

&#x20; v

PaymentResponse

&#x20; |

&#x20; v

Payment Gateway

&#x20; |

&#x20; v

Client

```



The `X-Request-ID` allows a payment request to be traced across both services.



\---



\## Service-to-Service Authentication



The Payment Processor requires a bearer token for `/process`.



The token is injected into both containers through the Kubernetes Secret:



```text

payment-service-token

```



The Gateway sends the token to the Processor using:



```text

Authorization: Bearer <SERVICE\_TOKEN>

```



This prevents unauthenticated internal callers from invoking the Processor.



The Secret value is not stored in the Git repository.



\---



\## Configuration



The application supports environment-based configuration.



Important configuration values include:



| Variable | Service | Purpose |

|---|---|---|

| `SERVICE\_TOKEN` | Both | Shared authentication token |

| `PROCESSOR\_URL` | Gateway | Processor Service URL |

| `LOG\_LEVEL` | Both | Logging level |

| `REQUEST\_TIMEOUT\_S` | Gateway | Processor request timeout |

| `CONNECT\_TIMEOUT\_S` | Gateway | Connection timeout |

| `MAX\_RETRIES` | Gateway | Maximum retry attempts |

| `BACKOFF\_BASE\_S` | Gateway | Retry backoff |

| `DECLINE\_OVER` | Processor | Amount threshold for declined payments |

| `FAILURE\_RATE` | Processor | Optional failure injection |

| `LATENCY\_MS` | Processor | Optional artificial latency |



The Gateway uses:



```text

PROCESSOR\_URL=http://payment-processor:8080

```



This uses Kubernetes Service DNS instead of a hard-coded Pod IP.



\---



\## Reliability



The Gateway has built-in reliability mechanisms for communication with the Processor.



These include:



\- Connection timeout

\- Request timeout

\- Bounded retries

\- Backoff between retries

\- Handling of Processor connection failures

\- Handling of Processor 5xx responses



The Gateway maps downstream failures to meaningful HTTP responses.



| Condition | Gateway Response |

|---|---|

| Processor healthy | `200` |

| Processor timeout after retries | `504` |

| Processor unreachable after retries | `503` |

| Processor 5xx / invalid response | `502` |

| Invalid payment request | `422` |



This prevents downstream failures from being exposed as ambiguous errors to the client.



\---



\## Kubernetes Resource Management



CPU and memory requests and limits are defined for the application containers.



Example:



```yaml

resources:

&#x20; requests:

&#x20;   cpu: 100m

&#x20;   memory: 256Mi

&#x20; limits:

&#x20;   cpu: "1"

&#x20;   memory: 512Mi

```



Requests provide Kubernetes with the minimum resources required by the workload.



Limits prevent a container from consuming unlimited resources.



\---



\## Why Kubernetes Services Are Used



Pod IP addresses are temporary and can change whenever a Pod is recreated.



Therefore, the Gateway does not communicate directly with a Pod IP.



Instead, it uses:



```text

http://payment-processor:8080

```



The Kubernetes Service provides:



\- Stable DNS

\- Stable virtual IP

\- Service discovery

\- Load balancing across Processor Pods when replicas are increased



\---



\## Why ClusterIP for the Processor?



The Payment Processor is an internal backend service.



There is no reason for external clients to directly access it.



Therefore, it is exposed using:



```text

ClusterIP

```



This keeps the Processor internal to the Kubernetes cluster.



\---



\## Why NodePort for the Gateway?



This is a local Minikube deployment.



NodePort provides a simple way to expose the Gateway to the local developer machine without introducing an Ingress controller or cloud LoadBalancer.



For a production Kubernetes environment, an Ingress or cloud LoadBalancer would generally be preferred.



\---



\## Observability



The provided services expose:



```text

/healthz

/readyz

/metrics

```



These endpoints provide the foundation for monitoring the application.



In production, I would integrate these with:



\- Prometheus

\- Grafana

\- Centralized logging

\- Alerting

\- Kubernetes monitoring



The `X-Request-ID` propagated between the Gateway and Processor also provides request-level traceability.



\---



\## Security Considerations



\### Secrets



Sensitive values such as the service authentication token are stored in Kubernetes Secrets.



They should not be:



\- Hard-coded in application code

\- Committed to Git

\- Added directly to Deployment YAML files

\- Printed in logs



For production, I would use a dedicated secret-management solution such as:



\- AWS Secrets Manager

\- Azure Key Vault

\- HashiCorp Vault

\- External Secrets Operator



depending on the target platform.



\### Network Exposure



Only the Gateway is externally exposed.



The Processor remains internal using a ClusterIP Service.



For production, I would also introduce Kubernetes NetworkPolicies so that only the Gateway is allowed to communicate with the Processor.



\---



\## Production Readiness Considerations



Before handing this application over to a production team, I would consider the following improvements.



\### 1. High Availability



Run multiple replicas:



```yaml

spec:

&#x20; replicas: 2

```



This prevents a single Pod failure from taking the service offline.



A PodDisruptionBudget should also be considered.



\### 2. Autoscaling



Configure Horizontal Pod Autoscaling based on CPU, memory, or application-specific metrics.



This allows the workload to scale according to traffic.



\### 3. Ingress



Replace the local NodePort exposure with an Ingress or cloud LoadBalancer.



This provides:



\- TLS termination

\- Host/path-based routing

\- Better external traffic management



\### 4. Network Policies



Add NetworkPolicies so that:



```text

Client -> Gateway

Gateway -> Processor

```



is allowed while unnecessary communication is denied.



\### 5. Secret Management



Move sensitive credentials from manually created Kubernetes Secrets to a dedicated secret-management solution.



\### 6. Monitoring and Alerting



Integrate metrics with Prometheus/Grafana and configure alerts for:



\- High error rates

\- High latency

\- Pod restarts

\- CPU/memory saturation

\- Processor availability



\### 7. Centralized Logging



Send application and Kubernetes logs to a centralized logging platform.



The propagated `X-Request-ID` should be included in logs to make troubleshooting easier.



\### 8. Container Security



Before production deployment, I would also introduce:



\- Container image vulnerability scanning

\- Image signing

\- Non-root containers where supported

\- Minimal container privileges

\- Regular base-image updates

\- Dependency vulnerability scanning



\### 9. CI/CD



A production workflow should automatically:



```text

Git Commit

&#x20;   |

&#x20;   v

Build

&#x20;   |

&#x20;   v

Unit Tests

&#x20;   |

&#x20;   v

Security Scan

&#x20;   |

&#x20;   v

Build/Verify Image

&#x20;   |

&#x20;   v

Push Image

&#x20;   |

&#x20;   v

Deploy to Kubernetes

&#x20;   |

&#x20;   v

Smoke Tests

```



Deployments should use immutable image tags rather than relying on `latest`.



\### 10. Infrastructure as Code



For a real cloud environment, Kubernetes infrastructure and supporting resources should be managed through Infrastructure as Code such as Terraform.



\---



\## Troubleshooting



\### Check Pods



```bash

kubectl get pods -n payment-system

```



\### Check Services



```bash

kubectl get svc -n payment-system

```



\### Check Deployments



```bash

kubectl get deployments -n payment-system

```



\### View Gateway Logs



```bash

kubectl logs deployment/payment-gateway -n payment-system

```



\### View Processor Logs



```bash

kubectl logs deployment/payment-processor -n payment-system

```



\### Describe a Pod



```bash

kubectl describe pod <POD\_NAME> -n payment-system

```



\### Check Processor Endpoints



```bash

kubectl get endpoints payment-processor -n payment-system

```



\### Check the Secret



```bash

kubectl get secret payment-service-token -n payment-system

```



Do not expose the Secret value.



\---



\## Cleanup



To remove the application:



```bash

kubectl delete -f payment-gateway-service.yaml

kubectl delete -f payment-gateway-deployment.yaml

kubectl delete -f payment-processor-service.yaml

kubectl delete -f payment-processor-deployment.yaml

kubectl delete secret payment-service-token -n payment-system

kubectl delete namespace payment-system

```



To stop Minikube:



```bash

minikube stop

```



To completely remove the local Minikube cluster:



```bash

minikube delete

```



\---



\## Validation Performed



The deployment was validated end-to-end.



The following were verified:



\- Minikube cluster started successfully.

\- Kubernetes namespace created.

\- Payment Processor Pod running.

\- Payment Gateway Pod running.

\- Processor exposed through ClusterIP.

\- Gateway exposed through NodePort.

\- Gateway successfully resolved the Processor using Kubernetes DNS.

\- Service authentication using `SERVICE\_TOKEN` worked.

\- Gateway `/healthz` returned successfully.

\- Gateway `/readyz` confirmed Processor connectivity.

\- Payment request successfully reached the Processor.

\- Processor returned an approved payment response.

\- Gateway returned HTTP `200` with the payment result.



Example successful result:



```text

HTTP 200 OK

status: approved

currency: INR

amount: 100

```



\---



\## Local Environment Note



The provided container images are built for `arm64`, while the local Minikube node used for this setup is `amd64`.



Docker therefore runs the containers using architecture emulation.



This can introduce additional startup latency compared with running the images on their native architecture.



For a production environment, I would use images built for the target cluster architecture or publish multi-architecture images.



\---



\## Conclusion



The final Kubernetes setup provides a reproducible local deployment of the Payment Gateway and Payment Processor.



The design keeps the Processor internal, uses Kubernetes Service discovery, protects the service-to-service token using a Secret, exposes health/readiness/metrics endpoints, defines resource limits, and uses the application's built-in timeout and retry mechanisms for downstream reliability.



For production, the next steps would be to add high availability, autoscaling, ingress/TLS, network policies, centralized observability, dedicated secret management, image security scanning, and CI/CD automation.

