# Chat with Gemma
Chabot demo using GKE, Cloud Run, TGI, Gradio and Gemma.  This is based on the [Serve Gemma open models using GPUs on GKE with Hugging Face TGI](https://cloud.google.com/kubernetes-engine/docs/tutorials/serve-gemma-gpu-tgi) tutorial.


## Prerequisites

You will need to follow these steps from tutorial mentioned above:

1. [Before you begin](https://cloud.google.com/kubernetes-engine/docs/tutorials/serve-gemma-gpu-tgi#before-you-begin)
2. [Get access to the model](https://cloud.google.com/kubernetes-engine/docs/tutorials/serve-gemma-gpu-tgi#model-access)
3. [Prepare your environment](https://cloud.google.com/kubernetes-engine/docs/tutorials/serve-gemma-gpu-tgi#prepare-environment)
4. [Create and configure Google Cloud resources](https://cloud.google.com/kubernetes-engine/docs/tutorials/serve-gemma-gpu-tgi#create-resources)
5. Clone this repository 
```
git clone https://github.com/gke-demos/chat-with-gemma.git
cd chat-with-gemma
```


## Serve Gemma using TGI

Deploy the TGI container which will download and serve Gemma:

```
kubectl apply -f k8s-manifests/gemma-tgi.yaml 
```

Wait for the deployment to be available

```
kubectl wait --for=condition=Available --timeout=700s deployment/tgi-gemma-deployment
```

Check the logs:

```
kubectl logs -f -l app=gemma-server
```

Wait to see output like the following as it can take a few minutes to download the model:

```
INFO text_generation_router: router/src/main.rs:237: Using the Hugging Face API to retrieve tokenizer config
INFO text_generation_router: router/src/main.rs:280: Warming up model
INFO text_generation_router: router/src/main.rs:316: Setting max batch total tokens to 666672
INFO text_generation_router: router/src/main.rs:317: Connected
WARN text_generation_router: router/src/main.rs:331: Invalid hostname, defaulting to 0.0.0.0
INFO text_generation_router::server: router/src/server.rs:1035: Built with `google` feature
INFO text_generation_router::server: router/src/server.rs:1036: Environment variables `AIP_PREDICT_ROUTE` and `AIP_HEALTH_ROUTE` will be respected.
```

Get the IP address on the internal load balancer:

```
export MODEL_SERVER=`kubectl get service llm-service -o json | jq -r .status.loadBalancer.ingress[].ip`
echo $MODEL_SERVER
```

Optionally, interact directly with the model:

1. Set up port forwarding
```
kubectl port-forward service/llm-service 8000:8000
```
2. Interact with the model using `curl`:
```
USER_PROMPT="Java is a"

curl -X POST http://localhost:8000/generate \
  -H "Content-Type: application/json" \
  -d @- <<EOF
{
    "inputs": "${USER_PROMPT}",
    "parameters": {
        "temperature": 0.90,
        "top_p": 0.95,
        "max_new_tokens": 128
    }
}
EOF
```

Output should look like:
```
{"generated_text":" general-purpose, high-level, class-based, object-oriented programming language. <strong>Is Java a statically typed language?</strong> Yes, Java is a statically typed language. Java also supports dynamic typing. Static typing means that the type of every variable is explicitly specified at the time of declaration. The type can be either implicit or explicit. Static typing means that if no types are assigned then it will be assumed as a primitive type.\n\n<h3>What is Java?</h3>\n\nJava is a general-purpose, class-based, object-oriented programming language. Java is one of the oldest programming languages that has gained a"}
```

## Deploy chat application

Deploy a simply chat application built using [Gradio](https://gradio.app/) to Cloud Run:

1. Replace MODEL_SERVER in the `cloud-run/gradio.yaml` manifest with the value of `$MODEL_SERVER` output above.

Example using sed:
```
sed -i "s/MODEL_SERVER/$MODEL_SERVER/" cloud-run/gradio.yaml 
```

2. Deploy Gradio to Cloud Run
```
gcloud run services replace cloud-run/gradio.yaml
```

Note: This deploys to `us-central`.  If you want to use different region, you can edit the value of the `cloud.googleapis.com/location` label in `cloud-run/gradio.yaml`

3. (Optional) Allow unauthenticated access
```
gcloud run services set-iam-policy gradio-app cloud-run/policy.yaml
```
4. Retrieve the URL and access the chat application in your favorite browser
```
gcloud run services describe gradio-app --format="value(status.url)" --region us-central1
```




