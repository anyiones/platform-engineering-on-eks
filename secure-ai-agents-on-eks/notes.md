
## Lab 0: The AI Gateway

Every model call goes through an Envoy AI Gateway in front of Bedrock. Agents hold no AWS credentials, because the gateway signs requests with its own identity (EKS Pod Identity).

    kubectl get svc ai-gateway -n envoy-gateway-system
ClusterIP service, reachable only inside the cluster. Accepts OpenAI (`/v1/chat/completions`) and Anthropic formats and translates both to Bedrock's Converse API.

    kubectl get aigatewayroute bedrock -n envoy-ai-gateway-system \
      -o jsonpath='{range .spec.rules[*]}{.matches[0].headers[0].value}{" -> "}{.backendRefs[0].modelNameOverride}{"\n"}{end}'
Prints the alias table (e.g. `nova-lite -> us.amazon.nova-2-lite-v1:0`). Model names are a platform decision. Edit the table and every agent follows, with no rebuild or redeploy.

    kubectl run gw-test --rm -i --restart=Never --image=curlimages/curl -q -- \
      curl -sS http://ai-gateway.envoy-gateway-system.svc.cluster.local/v1/chat/completions \
      -H 'Content-Type: application/json' \
      -d '{"model":"nova-lite","messages":[{"role":"user","content":"Say hi in 3 words"}]}' | jq
Throwaway pod calls the gateway from inside the cluster. Response `model` was `us.amazon.nova-2-lite-v1:0`, so the alias was rewritten.

Same command with `"model":"claude-sonnet"` returned `us.anthropic.claude-sonnet-4-5-20250929-v1:0`. Same endpoint and request shape, different provider.

Gap: the gateway does not authenticate callers, so any pod in the cluster can spend Bedrock tokens. Production fix: pass a projected service account token.

## Lab 0: The AI Gateway

Every model call goes through an Envoy AI Gateway in front of Bedrock. Agents hold no AWS credentials, because the gateway signs requests with its own identity (EKS Pod Identity).

    kubectl get svc ai-gateway -n envoy-gateway-system
ClusterIP service, reachable only inside the cluster. Accepts OpenAI (`/v1/chat/completions`) and Anthropic formats and translates both to Bedrock's Converse API.

    kubectl get aigatewayroute bedrock -n envoy-ai-gateway-system \
      -o jsonpath='{range .spec.rules[*]}{.matches[0].headers[0].value}{" -> "}{.backendRefs[0].modelNameOverride}{"\n"}{end}'
Prints the alias table (e.g. `nova-lite -> us.amazon.nova-2-lite-v1:0`). Model names are a platform decision. Edit the table and every agent follows, with no rebuild or redeploy.

    kubectl run gw-test --rm -i --restart=Never --image=curlimages/curl -q -- \
      curl -sS http://ai-gateway.envoy-gateway-system.svc.cluster.local/v1/chat/completions \
      -H 'Content-Type: application/json' \
      -d '{"model":"nova-lite","messages":[{"role":"user","content":"Say hi in 3 words"}]}' | jq
Throwaway pod calls the gateway from inside the cluster. Response `model` was `us.amazon.nova-2-lite-v1:0`, so the alias was rewritten.

Same command with `"model":"claude-sonnet"` returned `us.anthropic.claude-sonnet-4-5-20250929-v1:0`. Same endpoint and request shape, different provider.

Gap: the gateway does not authenticate callers, so any pod in the cluster can spend Bedrock tokens. Production fix: pass a projected service account token.

## Lab 1: Building your first agent

Strands agent on EKS. Model calls go to Bedrock through the Envoy AI Gateway (no AWS credentials in agent code). One in-process tool: `lookup_order`, backed by DynamoDB.

    cat ~/environment/modules/200-strands-agents/customer-agent/tools.py
`@tool` turns a Python function into something the model can call. Strands reads the name, type hints and docstring, and the docstring acts as the prompt that tells the model when to use it. Tool errors are returned as data, not exceptions, so the agent can explain them to the customer.

    envsubst < k8s.yaml | kubectl apply -f -
Fills the `${ACCOUNT_ID}` and `${AWS_REGION}` placeholders in the image reference, then applies the Deployment and Service.

    kubectl rollout status deployment/customer-agent --timeout=120s
    kubectl get pods -l app=customer-agent
    kubectl exec deploy/customer-agent -- env | grep -E '^(MODEL_BASE_URL|MODEL_ID|ORDERS_TABLE)='
Confirms the pod is Running and that config comes from the `agent-config` ConfigMap (gateway URL, model alias, table name).

MCP preview: the tool lives inside the agent, so every agent needing it copies code, dependency and IAM permission. Lab 3 moves it to an MCP server that agents discover at runtime.
