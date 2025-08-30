# 🚀 CI/CD Pipeline: GitHub → AWS EC2 (Node.js + PM2 + RabbitMQ)

This guide explains how to set up a **CI/CD pipeline** to automatically deploy your Node.js app from **GitHub** to an **AWS EC2 instance** using **GitHub Actions** and **PM2**, along with **RabbitMQ running in Docker**.

---

## 1️⃣ Connect to EC2

Use your `.pem` key to SSH into your instance:

```bash
ssh -i err.pem ubuntu@ec2-16-176-229-247.ap-southeast-2.compute.amazonaws.com
```
2️⃣ Pre-requisites on EC2
Once inside your EC2 instance, prepare the environment:

# Update system
```
sudo apt update && sudo apt upgrade -y
```

# Install Git, Node.js, and PM2
```
sudo apt install git -y
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs
sudo npm install -g pm2
```

# Setup project folder
```
mkdir -p ~/myapp && cd ~/myapp
git clone https://github.com/<your-username>/<your-repo>.git .
npm install
```

# Start app with PM2
```
pm2 start index.js --name myapp
pm2 save
```

# Check PM2
```
pm2 list
pm2 logs index
✅ Your EC2 is now ready to pull the latest code and restart automatically.
3️⃣ Install and Setup Docker + RabbitMQ on EC2
```
# Install Docker
```
sudo apt update
sudo apt install -y docker.io
```

# Start and enable Docker service
```
sudo systemctl start docker
sudo systemctl enable docker
```

# Give ubuntu user Docker permissions
```
sudo usermod -aG docker ubuntu
Run RabbitMQ Container
docker run -d --hostname my-rabbit --name rabbitmq \
  -p 5672:5672 -p 15672:15672 \
  rabbitmq:3-management
Verify RabbitMQ
```

# List running containers
```
docker ps
```

# Access RabbitMQ container shell
```
docker exec -it <container_id_or_name> bash

--Example:
docker exec -it bb05cd24cd0c bash
```

# List queues
```
rabbitmqctl list_queues
```

# Get messages from a queue
```
rabbitmqadmin get queue=user_events
```

# Exit container
```
exit
RabbitMQ Management UI is now available at:
👉 http://<EC2_PUBLIC_IP>:15672
(Default user/pass: guest / guest)
```

## Add GitHub Secrets
```
Go to GitHub Repo → Settings → Secrets and variables → Actions → New repository secret.
Add the following secrets:
EC2_HOST → Your EC2 public IP
EC2_USER → Usually ubuntu
EC2_SSH_KEY → Contents of your .pem key
(Optional) EC2_PATH → /home/ubuntu/myapp
```

## Create GitHub Actions Workflow
```
Inside your project, create a workflow file:
.github/workflows/ci-cd.yml
```
```
name: CI/CD Pipeline

on:
  push:
    branches: ["main"]   # deploy only when code pushed to main

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Code
        uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: "18"

      - name: Install Dependencies
        run: npm install

      - name: Run Tests
        run: npm test   # or skip if you don't have tests

  deploy:
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Checkout Code
        uses: actions/checkout@v3

      - name: Deploy to EC2
        uses: appleboy/ssh-action@v1.0.0
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USER }}
          key: ${{ secrets.EC2_SSH_KEY }}
          script: |
            cd ${{ secrets.EC2_PATH }}
            git pull origin main
            npm install --production
            pm2 restart myapp || pm2 start index.js --name myapp
            pm2 save
```

6️⃣ How It Works
```
Push code to the main branch
GitHub Actions triggers:
Runs npm install and npm test
If successful → SSH into EC2
Pulls latest code
Installs dependencies
Restarts app with PM2
```

7️⃣ Verify Deployment
```
Check if everything works:
# Check PM2 apps
pm2 list

# View app logs
pm2 logs index
Check RabbitMQ:
docker ps
docker exec -it rabbitmq bash
rabbitmqctl list_queues
rabbitmqadmin get queue=user_events
Open RabbitMQ Management UI:
👉 http://<EC2_PUBLIC_IP>:15672

```

8️⃣ Troubleshooting
```
Permission denied (publickey)
Make sure your .pem file has correct permissions:
chmod 400 err.pem
PM2 app not starting
Check logs:
pm2 logs myapp
Docker requires sudo
Run:
sudo usermod -aG docker ubuntu
Then logout and login again.
🎉 Congrats! You now have:
Automated CI/CD pipeline for Node.js → AWS EC2
RabbitMQ running in Docker with management UI
Quick SSH access to EC2 instance
```
---
8️⃣ PM2 Management

🔹 PM2 Kya Hai?
```
PM2 (Process Manager 2) ek production-grade process manager hai jo Node.js (aur dusre languages ke) applications ko manage karne ke liye use hota hai.

👉 Matlab simple shabdo me:

Ye background me tumhari app ko chalata hai continuously
Agar app crash ho jaye to auto restart kar deta hai
Ye tumhare app ke logs, monitoring, load balancing sab manage karta hai

```
## Common PM2 Commands
```
Start / Restart / Stop / Delete
pm2 start index.js --name myapp   # Start with name
pm2 restart myapp                 # Restart by name
pm2 restart 0                     # Restart by ID
pm2 restart all                   # Restart all apps

pm2 stop myapp                    # Stop by name
pm2 stop 0                        # Stop by ID
pm2 stop all                      # Stop all apps

pm2 delete myapp                  # Stop + remove from list
pm2 delete all                    # Delete all apps
Logs & Monitoring
pm2 list                          # List all apps
pm2 status                        # App status
pm2 show myapp                    # App details
pm2 describe myapp                 # App details

pm2 logs                          # Logs of all apps
pm2 logs myapp                    # Logs of specific app
pm2 monit                         # Monitoring dashboard
```

Cluster Mode
```
pm2 start index.js -i max         # One instance per CPU core
pm2 scale myapp 4                 # Scale app to 4 instances
```

Startup & Auto-Restart
```
pm2 startup                       # Generate startup script
pm2 save                          # Save current process list
pm2 resurrect                     # Restore apps after reboot
```

Maintenance
```
pm2 kill                          # Stop PM2 daemon + all apps
pm2 reset myapp                   # Reset app stats
pm2 flush                         # Clear all logs
```

✅ Example Workflow (Most Used)
```
pm2 start index.js --name myapp   # App start karo
pm2 list                          # Check app list
pm2 logs myapp                    # Logs check karo
pm2 restart myapp                 # Restart after code changes
pm2 stop myapp                    # Stop app
pm2 delete myapp                  # Stop + remove app
pm2 save                          # Save so it auto-starts after reboot
```
