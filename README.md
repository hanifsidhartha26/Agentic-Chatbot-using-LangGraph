# Agentic Chatbot with LangGraph

A Streamlit chatbot using LangGraph, Groq, OpenAI embeddings, and SQLite-backed conversation checkpoints.

## Deploy to AWS EC2

These instructions run the app on a single Ubuntu EC2 instance using `systemd`. This is a simple starting point for a small deployment; use a reverse proxy with HTTPS and a domain before making the app publicly accessible.

### Before you deploy

The current source needs these changes before it can be safely deployed:

1. **Rotate the Groq API key currently embedded in `backend.py`.** Treat it as exposed. Remove it from the source and read it from `GROQ_API_KEY` instead. Do not commit API keys or `.env` files.
2. **Add `langchain-groq` to `requirements.txt`.** `backend.py` imports `langchain_groq`, but the dependency is not currently listed.
3. Ensure `app.py`, `backend.py`, and `requirements.txt` are committed and pushed to the Git repository. The deployment steps below assume these files are available when you clone the repository.

The app also requires `OPENAI_API_KEY` for OpenAI embeddings. `OPENWEATHER_API_KEY` is optional and is only needed for the weather tool.

### 1. Create an EC2 instance

In the AWS Console, launch an Ubuntu Server 24.04 LTS instance. Choose an instance size appropriate for your expected traffic and workload.

Configure its security group:

- Allow SSH (TCP 22) **only from your IP address**.
- For a temporary test, allow TCP 8501 only from your IP address.
- For public production access, use a domain and HTTPS reverse proxy (TCP 80/443) instead of exposing port 8501 to everyone.

Connect to the instance using the SSH command shown in the EC2 Console.

### 2. Install system packages and the application

Run these commands on the EC2 instance, replacing the repository URL with your GitHub repository URL:

```bash
sudo apt update
sudo apt install -y python3-venv python3-pip git
sudo mkdir -p /opt/agentic-chatbot
sudo chown "$USER":"$USER" /opt/agentic-chatbot
git clone https://github.com/<YOUR-USERNAME>/<YOUR-REPOSITORY>.git /opt/agentic-chatbot
cd /opt/agentic-chatbot
python3 -m venv .venv
. .venv/bin/activate
python -m pip install --upgrade pip
pip install -r requirements.txt
```

Confirm that the dependency and credential fixes in the **Before you deploy** section have been made before installing and starting the app.

### 3. Configure API keys

Create a root-readable environment file on the instance:

```bash
sudo install -m 600 /dev/null /etc/agentic-chatbot.env
sudo nano /etc/agentic-chatbot.env
```

Add your keys (replace the placeholder values; do not include quotes):

```dotenv
GROQ_API_KEY=your_groq_api_key
OPENAI_API_KEY=your_openai_api_key
OPENWEATHER_API_KEY=your_openweather_api_key
```

`OPENWEATHER_API_KEY` may be omitted if you do not need weather lookups. Keep this file out of Git and restrict access to it.

### 4. Run the app as a service

Create a `systemd` service:

```bash
sudo tee /etc/systemd/system/agentic-chatbot.service >/dev/null <<'EOF'
[Unit]
Description=Agentic Chatbot Streamlit app
After=network.target

[Service]
Type=simple
User=ubuntu
WorkingDirectory=/opt/agentic-chatbot
EnvironmentFile=/etc/agentic-chatbot.env
ExecStart=/opt/agentic-chatbot/.venv/bin/streamlit run app.py --server.address=0.0.0.0 --server.port=8501 --server.headless=true
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now agentic-chatbot
sudo systemctl status agentic-chatbot
```

This service assumes the standard Ubuntu EC2 user is `ubuntu`. If you use another Linux distribution or user, update the `User` and application paths accordingly.

For a temporary, IP-restricted test, open `http://<EC2-PUBLIC-IP>:8501` in your browser. Check service logs if it does not start:

```bash
sudo journalctl -u agentic-chatbot -n 100 --no-pager
```

### 5. Production access and data

- Do not expose port 8501 to the public internet. Put Nginx, Caddy, or an Application Load Balancer with HTTPS in front of Streamlit, and allow inbound 80/443 as appropriate.
- The app stores conversation checkpoints in `chatbot.db` in its working directory. Keep that file on persistent storage and back it up. A single local SQLite database is intended for a single app instance, not a horizontally scaled deployment.
- If you use PDF retrieval, ensure the app's `faiss_db` index is present on the instance and persisted with the application data.
- For production, consider storing API keys in AWS Secrets Manager or Systems Manager Parameter Store instead of a local environment file.

## Run locally

After installing the requirements and configuring the required API keys in your environment (or a local, untracked `.env` file):

```bash
streamlit run app.py
```

Do not commit `.env` files, API keys, or local database files.
