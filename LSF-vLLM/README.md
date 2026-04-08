IBM LSF for vLLM Persistent Inference Service
==========================================

Overview
--------
This repository shows how to run a long-running vLLM inference service under IBM LSF,
validate it through a standard OpenAI-compatible API, access it from a Jupyter notebook,
and reuse the same service from a downstream batch job.

What this implementation demonstrates
-------------------------------------
- IBM LSF launching and managing a persistent inference runtime as a service job
- vLLM exposing an OpenAI-compatible endpoint
- endpoint discovery through a small registry file written by the service job
- interactive validation using curl and Jupyter
- downstream reuse through a separate IBM LSF batch job

Repository layout
-----------------
- scripts/start_vllm_lsf.sh
  Starts the vLLM container, waits for readiness, writes the registry file, and keeps the
  service attached to the IBM LSF job lifecycle.
- scripts/resolve_endpoint.py
  Reads the registry file for a given IBM LSF job ID and prints the resolved base URL.
- scripts/batch_client.py
  Reads a prompt corpus and sends requests to the registered vLLM service.
- notebook/LSF_vLLM_Client.ipynb
  Jupyter notebook for interactive validation against the IBM LSF-managed runtime.
- corpus/prompts.txt
  Sample prompt corpus for downstream batch validation.

Prerequisites
-------------
- IBM LSF installed and operational
- podman installed
- python3 installed
- curl installed
- network access from the execution host to pull the vLLM image and model
- a single-node IBM LSF setup is sufficient for this implementation

Note:
Replace "your-host" with the hostname or IP address of the system where the vLLM service is running.

The examples below assume you are running as the same user for all steps.

Step 1: Create the working directories
--------------------------------------

```bash
mkdir -p ~/lsf_vllm_poc/{logs,registry,cache,corpus,results,notebook}
```

```bash
cp corpus/prompts.txt ~/lsf_vllm_poc/corpus/prompts.txt
```

```bash
cp scripts/start_vllm_lsf.sh ~/lsf_vllm_poc/
cp scripts/resolve_endpoint.py ~/lsf_vllm_poc/
cp scripts/batch_client.py ~/lsf_vllm_poc/
```

```bash
chmod +x ~/lsf_vllm_poc/start_vllm_lsf.sh
chmod +x ~/lsf_vllm_poc/resolve_endpoint.py
chmod +x ~/lsf_vllm_poc/batch_client.py
```

Step 2: Review the service script defaults
------------------------------------------

```bash
MODEL=Qwen/Qwen3-0.6B PORT=8001 API_KEY=local-vllm-key
```

Step 3: Submit the persistent service job
-----------------------------------------

```bash
JOBID=$(
  bsub -J vllm_service        -q normal        -n 1        -R 'rusage[mem=12GB]'        -oo ~/lsf_vllm_poc/logs/vllm.%J.out        -eo ~/lsf_vllm_poc/logs/vllm.%J.err        ~/lsf_vllm_poc/start_vllm_lsf.sh   | awk '{print $2}' | tr -d '<>'
)

echo "Submitted service JOBID=$JOBID"
```

Step 4: Monitor the service startup
-----------------------------------

```bash
bjobs
bjobs -l ${JOBID}
bpeek ${JOBID}
```

```bash
podman ps -a | grep vllm-job-${JOBID}
podman logs -f vllm-job-${JOBID}
```

Step 5: Wait for the registry file
----------------------------------

```bash
until [[ -f ~/lsf_vllm_poc/registry/${JOBID}.json ]]; do
  sleep 2
done

cat ~/lsf_vllm_poc/registry/${JOBID}.json
```

Step 6: Resolve the endpoint
----------------------------

```bash
python3 ~/lsf_vllm_poc/resolve_endpoint.py ${JOBID}
```

```bash
ENDPOINT=$(python3 ~/lsf_vllm_poc/resolve_endpoint.py ${JOBID})
echo "${ENDPOINT}"
```

Step 7: Validate the service with curl
--------------------------------------

```bash
curl -sS "${ENDPOINT}/models"   -H "Authorization: Bearer local-vllm-key"
```

```bash
curl -sS "${ENDPOINT}/chat/completions"   -H "Content-Type: application/json"   -H "Authorization: Bearer local-vllm-key"   -d '{
    "model": "Qwen/Qwen3-0.6B",
    "messages": [
      {"role": "user", "content": "Explain the top 5 deserts in the world."}
    ],
    "temperature": 0,
    "max_tokens": 120
  }'
```

Step 8: Validate the service from Jupyter
-----------------------------------------

```bash
python3 -m venv ~/lsf_vllm_poc/notebook/.venv
source ~/lsf_vllm_poc/notebook/.venv/bin/activate
pip install --upgrade pip
pip install notebook jupyterlab requests openai ipykernel
python -m ipykernel install --user --name lsf-vllm --display-name "Python (lsf-vllm)"
```

```bash
jupyter notebook --no-browser --ip=0.0.0.0 --port 8888 --allow-root
```

```bash
ssh -L 8888:127.0.0.1:8888 user@your-host
```

```
http://127.0.0.1:8888
```

```
http://127.0.0.1:8001/v1
```

Step 9: Validate downstream batch reuse
---------------------------------------

```bash
python3 ~/lsf_vllm_poc/batch_client.py ${JOBID} ~/lsf_vllm_poc/corpus/prompts.txt
```

```bash
cat ~/lsf_vllm_poc/results/batch_${JOBID}.jsonl
```

```bash
BATCH_JOBID=$(
  bsub -J vllm_batch        -q normal        -n 1        -R 'rusage[mem=1GB]'        -oo ~/lsf_vllm_poc/logs/batch.%J.out        -eo ~/lsf_vllm_poc/logs/batch.%J.err        "python3 ~/lsf_vllm_poc/batch_client.py ${JOBID} ~/lsf_vllm_poc/corpus/prompts.txt"   | awk '{print $2}' | tr -d '<>'
)

echo "Submitted batch JOBID=$BATCH_JOBID"
```

```bash
bjobs
bpeek ${BATCH_JOBID}
cat ~/lsf_vllm_poc/results/batch_${JOBID}.jsonl
```

Cleanup
-------

```bash
bkill ${BATCH_JOBID}
bkill ${JOBID}
```

Troubleshooting
---------------

```bash
bpeek ${JOBID}
podman ps -a
podman logs vllm-job-${JOBID}
```

```bash
curl -sS http://127.0.0.1:8001/v1/models -H "Authorization: Bearer local-vllm-key"
```

Success criteria
----------------
1. the service job is RUN under IBM LSF
2. the registry file exists
3. /v1/models works
4. /v1/chat/completions works
5. the Jupyter notebook can call the service successfully
6. the batch client works locally
7. the IBM LSF batch job completes successfully
8. the result file contains responses for the full prompt corpus

Step 10: Validate using Open WebUI (Linux)
------------------------------------------

```bash
podman run -d   -p 3000:8080   -e OPENAI_API_BASE_URL=http://your-host:8001/v1   -e OPENAI_API_KEY=local-vllm-key   -e WEBUI_SECRET_KEY=my-openwebui-secret   -v open-webui:/app/backend/data   --name open-webui   ghcr.io/open-webui/open-webui:main
```

```bash
podman ps
podman logs -f open-webui
```

http://localhost:3000

- Settings → Connections → OpenAI
- Base URL: http://your-host:8001/v1
- API Key: local-vllm-key

Model:
Qwen/Qwen3-0.6B

Test:
Say one short line about LSF-managed model serving.
