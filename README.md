# AI Chat Service

A Go service that proxies chat requests to OpenAI's ChatGPT API. Designed for deploying on OpenChoreo as a `deployment/service` component to research egress control via an AI gateway.

## Why this sample matters for egress control

This service makes **outbound HTTPS calls to `api.openai.com`**, which is exactly the traffic pattern an AI gateway would intercept. When deployed on OpenChoreo:

- OpenChoreo creates a **default-deny ingress** NetworkPolicy for the component
- **Egress is currently unrestricted** -- this service can freely call `api.openai.com`
- An AI gateway would sit in the egress path, allowing you to enforce token limits, audit prompts, block sensitive data, and route across LLM providers

## API

### POST /chat

Send a message and get an AI-generated reply.

```bash
curl -X POST http://localhost:8080/chat \
  -H "Content-Type: application/json" \
  -d '{"message": "What is Kubernetes?"}'
```

Response:
```json
{
  "reply": "Kubernetes is an open-source container orchestration platform...",
  "model": "gpt-4o-mini",
  "usage": {
    "prompt_tokens": 25,
    "completion_tokens": 42,
    "total_tokens": 67
  }
}
```

### GET /healthz, GET /readyz

Health and readiness probes.

## Configuration

| Environment Variable | Required | Default | Description |
|---------------------|----------|---------|-------------|
| `OPENAI_API_KEY` | Yes | - | OpenAI API key |
| `OPENAI_BASE_URL` | No | `https://api.openai.com` | Base URL (override to point to an AI gateway) |
| `OPENAI_MODEL` | No | `gpt-4o-mini` | Model to use |
| `SYSTEM_PROMPT` | No | Generic helpful assistant | System prompt for the conversation |
| `LOG_LEVEL` | No | `info` | Log level (debug, info, warn, error) |

## Run locally

```bash
export OPENAI_API_KEY="sk-..."
go run . --port 8080
```

## Build and run with Docker

```bash
docker build -t ai-chat-service .
docker run -p 8080:8080 -e OPENAI_API_KEY="sk-..." ai-chat-service
```

## Deploy on OpenChoreo

```bash
kubectl apply -f deploy/openchoreo.yaml
```

Then test:
```bash
HOSTNAME=$(kubectl get httproute -A -l openchoreo.dev/component=ai-chat-service -o jsonpath='{.items[0].spec.hostnames[0]}')
PATH_PREFIX=$(kubectl get httproute -A -l openchoreo.dev/component=ai-chat-service -o jsonpath='{.items[0].spec.rules[0].matches[0].path.value}')

curl -X POST "http://${HOSTNAME}:19080${PATH_PREFIX}/chat" \
  -H "Content-Type: application/json" \
  -d '{"message": "Explain network policies in Kubernetes"}'
```

## Egress gateway testing

To route LLM traffic through an AI gateway, set `OPENAI_BASE_URL` to your gateway endpoint:

```yaml
env:
  - key: OPENAI_BASE_URL
    value: "https://ai-gateway.internal.example.com"
```

The service uses the standard OpenAI `/v1/chat/completions` endpoint, so any OpenAI-compatible gateway will work transparently.