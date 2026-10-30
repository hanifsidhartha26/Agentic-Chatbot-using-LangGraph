# Agentic Chatbot with LangGraph

A Streamlit chatbot using LangGraph, Groq, OpenAI embeddings, and SQLite-backed conversation checkpoints.

## Deploy to AWS EC2 with Docker

The included [Dockerfile](./Dockerfile) builds the Streamlit app and serves it on port `8501`. These instructions deploy it on a single Ubuntu EC2 instance.

### 1. Launch and secure an EC2 instance

Launch an Ubuntu Server EC2 instance and configure its security group:

- Allow SSH (TCP 22) only from your IP address.
- For a temporary test, allow TCP 8501 only from your IP address.
- For public production access, use a domain and HTTPS reverse proxy or load balancer on TCP 80/443. Do not expose port 8501 to the whole internet.

Connect to the instance using the SSH command shown in the EC2 Console.

### 2. Install Docker and get the project

Run these commands on the EC2 instance. Replace the repository URL with your GitHub repository URL:

```bash
sudo apt update
sudo apt install -y docker.io git
sudo systemctl enable --now docker
sudo mkdir -p /opt/agentic-chatbot
sudo chown "$USER":"$USER" /opt/agentic-chatbot
git clone https://github.com/<YOUR-USERNAME>/<YOUR-REPOSITORY>.git /opt/agentic-chatbot
cd /opt/agentic-chatbot
```

### 3. Configure API keys

Create a protected environment file on the instance:

```bash
sudo install -m 600 /dev/null /etc/agentic-chatbot.env
sudo nano /etc/agentic-chatbot.env
```

Add your API keys (replace the example values):

```dotenv
GROQ_API_KEY=your_groq_api_key
OPENAI_API_KEY=your_openai_api_key
OPENWEATHER_API_KEY=your_openweather_api_key
```

`GROQ_API_KEY` and `OPENAI_API_KEY` are required. `OPENWEATHER_API_KEY` is optional and is only needed for the weather tool. Do not add this file to Git or the Docker build context; pass it to the container with `--env-file`.

### 4. Build and start the container

Build the image from the project directory:

```bash
cd /opt/agentic-chatbot
sudo docker build -t agentic-chatbot:latest .
```

Start the app as a container that restarts after an instance or Docker restart:

```bash
sudo docker run -d \
  --name agentic-chatbot \
  --restart unless-stopped \
  --env-file /etc/agentic-chatbot.env \
  --publish 8501:8501 \
  --mount type=bind,source=/opt/agentic-chatbot,target=/app \
  agentic-chatbot:latest
```

The bind mount keeps the app's `chatbot.db` SQLite checkpoint file (and any `faiss_db` PDF index) on the EC2 instance when the container is replaced. Keep this deployment to one app instance; SQLite on a local disk is not suitable for horizontally scaled replicas. Back up the project data directory regularly.

For a temporary, IP-restricted test, open `http://<EC2-PUBLIC-IP>:8501`. To check startup or application logs:

```bash
sudo docker ps
sudo docker logs --tail 100 agentic-chatbot
```

### 5. Update the deployment

Pull the latest project changes and rebuild/recreate the container:

```bash
cd /opt/agentic-chatbot
git pull
sudo docker build -t agentic-chatbot:latest .
sudo docker rm -f agentic-chatbot
sudo docker run -d \
  --name agentic-chatbot \
  --restart unless-stopped \
  --env-file /etc/agentic-chatbot.env \
  --publish 8501:8501 \
  --mount type=bind,source=/opt/agentic-chatbot,target=/app \
  agentic-chatbot:latest
```

The bind mount points at the checked-out project directory, so the repository should contain only trusted application files. Never store `.env` files, API keys, or other secrets in that directory or include them in the Docker build context.

## Run locally

With Docker installed, configure a local environment file and build and run the image:

```bash
docker build -t agentic-chatbot .
docker run --rm -p 8501:8501 --env-file .env agentic-chatbot
```

Alternatively, install the dependencies from `requirements.txt`, set the required API keys in your shell or a local untracked `.env` file, then run:

```bash
streamlit run app.py
```
