# LiteLLM as an OpenAI‑style API gateway on Windows (WSL + Docker)

A concise guide that shows how to run LiteLLM in Docker on Windows with WSL (Ubuntu) and expose an OpenAI‑style API on http://localhost:4000. Your local tools/scripts call LiteLLM, which forwards requests to OpenAI (or other providers) — so you can change providers by editing a single config file.

Audience: developers who use Windows and want a local, switchable LLM proxy for tooling, testing, and local development.

---

## Quick overview (what you are building)

You will end up with:
- A Windows host with WSL (Ubuntu) as your shell.
- Docker Desktop on Windows with WSL integration enabled.
- A LiteLLM Docker container running as a proxy/gateway on http://localhost:4000.
- Local programs call http://localhost:4000; LiteLLM forwards to OpenAI using the OpenAI API key provided to the container.

Mental architecture:
- Your tools/scripts → http://localhost:4000 (LiteLLM proxy)
- LiteLLM → OpenAI (or other providers) via their APIs
- Swap providers/models by editing litellm_config.yaml — no need to change all your apps.

---

## Prerequisites

- Windows 10/11 with WSL support
- Internet connection (to pull Docker image and call OpenAI)
- Docker Desktop for Windows
- An OpenAI API key (or equivalent provider key)

---

## Step 1 — One‑time Windows + WSL + Docker setup

1. Install and open WSL (Ubuntu)
   - Open PowerShell as Administrator and run:
     ```powershell
     wsl --install
     ```
   - Reboot if prompted.
   - Open "Ubuntu" from the Start menu and create your Linux username and password.

2. Install Docker Desktop for Windows
   - Download + install Docker Desktop and accept WSL2 integration during setup.
   - Open Docker Desktop.
   - Settings → Resources → WSL Integration → enable your Ubuntu distribution → Apply & Restart.

3. Confirm Docker works in WSL (Ubuntu)
   - In WSL:
     ```bash
     docker --version
     ```
   - If you see a version, Docker is connected to WSL. If you get a connection error, ensure Docker Desktop is running on Windows.

---

## Step 2 — Create a simple LiteLLM project in WSL

1. Create project folder and enter it:
   ```bash
   mkdir -p ~/litellm-docker-demo
   cd ~/litellm-docker-demo
   ```

2. Create configuration file `litellm_config.yaml`:
   ```bash
   nano litellm_config.yaml
   ```
   Paste the following contents:
   ```yaml
   model_list:
     - model_name: "gpt-4o"
       litellm_params:
         model: "openai/gpt-4o"
         api_key: os.environ/OPENAI_API_KEY

   litellm_settings:
     master_key: "sk-proxy-master-key-123"
   ```
   - Save and exit: Ctrl+O, Enter, Ctrl+X.
   Notes:
   - `model_name`: name to use in requests (e.g., "gpt-4o").
   - `litellm_params.model`: the provider model LiteLLM will call (here `openai/gpt-4o`).
   - `api_key` uses the environment variable `OPENAI_API_KEY` inside the container.
   - `master_key` is the proxy auth key your clients will use in the Authorization header.

---

## Step 3 — Set your OpenAI key and start LiteLLM in Docker

1. In the same WSL terminal:
   ```bash
   export OPENAI_API_KEY="sk-your-real-openai-key-here"
   ```
   Replace with your real OpenAI key.

2. Run the LiteLLM container (single-line command — copy exactly):
   ```bash
   sudo docker run --name litellm-proxy -v "$(pwd)/litellm_config.yaml:/app/config.yaml" -e OPENAI_API_KEY="$OPENAI_API_KEY" -p 4000:4000 ghcr.io/berriai/litellm:main-latest --config /app/config.yaml --port 4000
   ```
   What this does:
   - `--name litellm-proxy`: container name.
   - `-v "$(pwd)/litellm_config.yaml:/app/config.yaml"`: mount your config into the container.
   - `-e OPENAI_API_KEY="$OPENAI_API_KEY"`: pass your OpenAI key into the container.
   - `-p 4000:4000`: expose LiteLLM at http://localhost:4000.
   - Remaining args tell LiteLLM which config file and port to use.

Leave this terminal open to view container logs. If you change the config, stop and remove the container then re-run:

```bash
sudo docker stop litellm-proxy || true
sudo docker rm litellm-proxy || true
# then re-run the docker run command above
```

---

## Step 4 — Verify the proxy is running

Open a second WSL terminal.

1. Change to your project folder:
   ```bash
   cd ~/litellm-docker-demo
   ```

2. Health check:
   ```bash
   curl http://localhost:4000/health/readiness
   ```
   - HTTP 200 with small JSON → LiteLLM is up.
   - "Connection refused" → container not running:
     ```bash
     sudo docker start litellm-proxy
     ```

3. Test a chat completion through LiteLLM:
   ```bash
   curl -X POST http://localhost:4000/v1/chat/completions \
     -H "Content-Type: application/json" \
     -H "Authorization: Bearer sk-proxy-master-key-123" \
     -d '{"model":"gpt-4o","messages":[{"role":"user","content":"Say one short sentence that proves LiteLLM via Docker on WSL is working."}]}'
   ```
   - Successful response: JSON containing `choices` with the model reply.
   - If you see "Invalid model name":
     - Your OpenAI key may not support `gpt-4o`.
     - List available models directly from OpenAI:
       ```bash
       curl https://api.openai.com/v1/models -H "Authorization: Bearer $OPENAI_API_KEY"
       ```
     - Pick a supported chat model (e.g., `gpt-4.1-mini`) and update both `model_name` and `litellm_params.model` in `litellm_config.yaml` (use `openai/` prefix for the `model` value), restart the container, and update the `model` in your curl test.

---

## Step 5 — The “forever memory” concept (why this pattern helps)

- Your apps always call the same local endpoint (http://localhost:4000).
- LiteLLM routes those requests to whichever provider/model is configured in `litellm_config.yaml`.
- To change provider or model, only edit the config and restart the container — your apps stay unchanged.
- Key rules:
  - `model_name` in `litellm_config.yaml` must match the `"model"` field you send in JSON requests.
  - `litellm_params.model` must be a real model supported by the API key used inside the container.
  - `master_key` in config must match the `Authorization: Bearer ...` header you send.

---

## Troubleshooting

- Docker can't connect from WSL:
  - Ensure Docker Desktop is running on Windows and WSL integration is enabled.
  - `docker --version` should work inside WSL.
- Container logs:
  ```bash
  sudo docker logs -f litellm-proxy
  ```
- Ports in use or container won't start:
  - Check `sudo docker ps -a` and stop/remove existing container.
- Permission / sudo:
  - Using `sudo` avoids Docker group setup; if you prefer not to use `sudo`, add your WSL user to the Docker group (follow Docker docs).
- Invalid model or auth errors:
  - Confirm `OPENAI_API_KEY` is valid and has access to the requested model.
  - Confirm `Authorization: Bearer sk-proxy-master-key-123` matches `master_key` in the YAML.
- If health endpoint returns non-200, check container logs for errors.

---

## Security & recommended best practices

- Never commit your real `OPENAI_API_KEY` to source control.
- Consider putting `litellm_config.yaml` in `.gitignore` if it contains secrets.
- Use a strong `master_key` and rotate it periodically.
- Avoid exposing the container to the public Internet. Use localhost-only or firewall rules.
- For production or shared environments, prefer secure secret management (e.g., Docker secrets, environment variables in CI/CD, or a secret manager).
- Limit the OpenAI API key permissions if possible and monitor usage.

---

## Publishing this guide

To publish this as a README in a GitHub repo:

1. Initialize a git repo in the project directory (or your documentation repo):
   ```bash
   cd ~/litellm-docker-demo
   git init
   git add README-litellm-wsl-docker.md
   echo "litellm_config.yaml" >> .gitignore
   git add .gitignore
   git commit -m "Add LiteLLM WSL + Docker guide"
   ```
2. Create a repository on GitHub and push (replace `<owner>` and `<repo>`):
   ```bash
   git remote add origin https://github.com/<owner>/<repo>.git
   git branch -M main
   git push -u origin main
   ```
Alternatively, paste the markdown into a new GitHub Gist or a documentation site.

---

## Example TL;DR (copy-paste checklist)

- Install WSL and Ubuntu: `wsl --install`
- Install Docker Desktop and enable WSL integration
- Create project and `litellm_config.yaml`
- Export key: `export OPENAI_API_KEY="sk-..."`
- Run container (single line):
  ```bash
  sudo docker run --name litellm-proxy -v "$(pwd)/litellm_config.yaml:/app/config.yaml" -e OPENAI_API_KEY="$OPENAI_API_KEY" -p 4000:4000 ghcr.io/berriai/litellm:main-latest --config /app/config.yaml --port 4000
  ```
- Health check: `curl http://localhost:4000/health/readiness`
- Test chat: use the curl POST shown above with `Authorization: Bearer <master_key>`

---

If you’d like, I can:
- Create a ready-to-publish README file in a GitHub repository for you (I’ll need the repo in owner/name format and permission to create files), or
- Produce a short blog-post style version or a Gist-ready snippet instead. Which would you prefer?