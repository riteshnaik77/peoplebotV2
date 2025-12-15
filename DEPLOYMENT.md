# Deployment Guide (Python 3.10)

## Prerequisites
- Ubuntu/Debian server with sudo access
- Python 3.10, git, and (optionally) nginx

Install basics:
```bash
sudo apt update
sudo apt install python3.10 python3.10-venv python3.10-dev git nginx
```

## Get the Code
```bash
sudo mkdir -p /opt/cursor-app && sudo chown $USER:$USER /opt/cursor-app
git clone <your-repo-url> /opt/cursor-app
cd /opt/cursor-app
```

## Virtual Env (Python 3.10) and Dependencies
```bash
python3.10 -m venv venv
source venv/bin/activate
pip install --upgrade pip
pip install -r requirements.txt   # or requirements_for_server.txt if applicable
```

## Environment Variables
Create `/opt/cursor-app/.env` with your keys:
```
MODEL_PROVIDER=gemini
GOOGLE_API_KEY=...
OPENAI_API_KEY=...
GROQ_API_KEY=...
OPENAI_MODEL=gpt-4o-mini
GEMINI_MODEL=gemini-2.5-flash
GROQ_MODEL=openai/gpt-oss-120b
GROQ_REASONING_EFFORT=high
FLASK_ENV=production
FLASK_APP=app.py
```

## Smoke Test (Manual)
```bash
source venv/bin/activate
python app.py   # or: flask run --host 0.0.0.0 --port 8000
# then visit http://SERVER_IP:8000
```

## Production with Gunicorn
```bash
pip install gunicorn
gunicorn -w 4 -b 0.0.0.0:8000 app:app
```

## systemd Service (auto-start on reboot)
```bash
sudo tee /etc/systemd/system/cursor-app.service <<'EOF'
[Unit]
Description=Cursor Flask App
After=network.target

[Service]
User=www-data
Group=www-data
WorkingDirectory=/opt/cursor-app
Environment="PATH=/opt/cursor-app/venv/bin"
EnvironmentFile=/opt/cursor-app/.env
ExecStart=/opt/cursor-app/venv/bin/gunicorn -w 4 -b 0.0.0.0:8000 app:app
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable cursor-app
sudo systemctl start cursor-app
sudo systemctl status cursor-app
```

## Nginx Reverse Proxy (optional but recommended)
```bash
sudo tee /etc/nginx/sites-available/cursor-app.conf <<'EOF'
server {
    listen 80;
    server_name your.domain.com;  # or _

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
EOF

sudo ln -s /etc/nginx/sites-available/cursor-app.conf /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

## SSL (if you have a domain)
```bash
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d your.domain.com
```

## Logs & Troubleshooting
- App logs: `journalctl -u cursor-app -f`
- Nginx logs: `/var/log/nginx/error.log`
- Restart app: `sudo systemctl restart cursor-app`

## Updating the App
```bash
cd /opt/cursor-app
git pull
source venv/bin/activate && pip install -r requirements.txt
sudo systemctl restart cursor-app
```



