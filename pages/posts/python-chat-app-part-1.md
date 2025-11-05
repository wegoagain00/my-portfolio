---
title: How I Deployed a Python Chat App (Manually) - Part 1
date: 2025-11-03
description: Part 1 of my DevOps project. I explore the "manual way" of running a Python and Redis chat app locally, highlighting the dependency hell and issues that Docker is built to solve.
tag: DevOps, Python, Redis, Troubleshooting, Project
author: Tawfiq (wegoagain)
---

**This blog is part of a 4-part series on building a real-time chat application:**
- **Part 1: The Manual Way**
- **Part 2: A Production-Ready Docker Setup**
- **Part 3: The Automated CI Pipeline with GitHub Actions & ECR**
- **Part 4: Deploying to Kubernetes with Terraform & EKS (coming soon)**

The project is a real-time chat application. It's a microservice based app that has two distinct components:
1. **The Web App:** A Python server using the Flask framework. It handles the user interface (the web page) and uses Socket.IO to manage the real-time, bi-directional chat messages.
2. **The Database:** A Redis in-memory database. Redis is incredibly fast, making it perfect for a chat app where messages need to be stored and retrieved almost instantly.
These two services need to be installed, configured, and must be able to find and talk to each other.

This application was created by Redis as a demo application, I used this already built project and will be implementing DevOps approach to this project - original link [here](https://github.com/redis-developer/basic-redis-chat-app-demo-python/tree/master?tab=readme-ov-file).

---

Before we jump into Docker, Kubernetes, and automated pipelines, we are going to run our application manually. As a DevOps engineer, my goal is to automate a process, but I can't automate what I don't understand.

We're going to set up the chat application the "old-fashioned" way. This will quickly show us all the problems and pain points that Docker is designed to solve in the Step 2 of this series.

## Step 1: Install and Run the Database (Redis)

First, our application needs its database. The app code is written to connect to a Redis server, so we need to install and run one on our local machine.

**What's Happening?** We are installing the Redis server directly onto our operating system. This is a "global" installation.

**On macOS (using Homebrew):**

```
# Install the Redis server
brew install redis

# Start the Redis service in the background
brew services start redis
```

**On Linux (using `apt`):**

```
# Install the Redis server
sudo apt update
sudo apt install redis-server

# The service usually starts automatically. If not:
sudo systemctl start redis-server
```

![alt text](/images/python-chat-app/img-1.png)

**Why are we doing this?** At this point, you have a Redis database running on your machine. By default, it listens for connections on `localhost`, port `6379`. You can test this by running `redis-cli ping`, and it should reply `PONG`.

## Step 2: Set Up the Python Application

Now we'll set up the web application. We can't just run `python app.py` because our main operating system doesn't have the libraries it needs (like Flask and Redis).

**What's Happening?** We are creating a **virtual environment**. This is a core Python concept that creates an isolated "bubble" for our project's dependencies, so we don't mess up our global Python installation.

1. **Clone the code:**

```
git clone https://github.com/wegoagain00/redis-chat-app-python.git
cd redis-chat-app-python
```

![alt text](/images/python-chat-app/img-2.png)

2. **Create the virtual environment:**

```
   python3 -m venv venv
   ```

(This creates a new folder named `venv`.)

3. **Activate the virtual environment:**

```
   # On macOS/Linux
   source venv/bin/activate
   ```

(Your terminal prompt will change to show `(venv)`.)

4. **Install the dependencies:**

```
pip install -r requirements.txt
```

(This reads the `requirements.txt` file and installs Flask and the `redis-py` library _inside_ your `venv` bubble.)

![alt text](/images/python-chat-app/img-3.png)

## Step 3: Time to Run the App

With everything setup now, let's run the app.

```
python app.py
```

## "The First Wall I Hit: The 'Greenlet' Error."

![alt text](/images/python-chat-app/img-6.png)

This error happened because I was using **Python 3.13** on my mac (it may run fine for you).

The solution is to not use Python 3.13 for this project. The most common and stable versions for production and tutorials are **3.11**, **3.10**, or **3.9**.

When I tried to run the app locally, I hit a massive wall of C-compilation errors. My Mac was running the latest Python 3.13, but the app's dependencies weren't compatible. This is a classic 'dependency hell' problem that stops projects dead.

***"This is the exact problem Docker solves.***" (You will see soon)

***Fix***:
What I did was Install python 3.9 using `brew install python@3.9`
![alt text](/images/python-chat-app/img-5.png)

1. Then ran `deactivate` so we can be in a new virtual environment.
2. We will now run `python3.9 -m venv venv` to start our virtual environment in python3.9
and then do same steps as above
3. `source venv/bin/activate` and `pip install -r requirements.txt`.

Now lets run `python app.py`  and we should be all good

---

Your terminal will show the app is running. Now, flask runs on default port 5000 so i was able to access on `http://127.0.0.1:5000`.

![alt text](/images/python-chat-app/img-4.png)


You can duplicate the tab after running this in the browser and simply login as a different user! Then message each other and you'll see it being reflected and the messages pulling through! Congratulations!
![alt text](/images/python-chat-app/img-7.png)


![alt text](/images/python-chat-app/img-8.png)



**Why did it work?** Let's look at the `chat/config.py` code. The app needs to know the address of the Redis database.

```
redis_endpoint_url = os.environ.get("REDIS_ENDPOINT_URL", "127.0.0.1:6379")
```

This line says: "Look for an environment variable named `REDIS_ENDPOINT_URL`. If you can't find one, just use the default value: `127.0.0.1:6379`."

Since we didn't set that variable, the app connected to `127.0.0.1:6379`, where our Redis server is already running.

---
## Congrats! The app is running. But, why is this a bad way to build software?

- **Problem 1: "It Works On My Machine"** This setup is incredibly fragile. What if your teammate is on Windows? Their Redis install is completely different. What if they have a different version of Python? (Like we did) The app might break.

- **Problem 2: Dependency Hell** What if you start a new project that needs an older version of Flask? Now you have two projects with conflicting libraries. Virtual environments help, but they're easy to forget and manage.

- **Problem 3: "Polluting" the Host Machine** You now have a Redis server running on your laptop all the time, even when you're not working on this project. It's just "junk" taking up resources.

- **Problem 4: No Clear Path to Production** How do you deploy this to a server? you'd have to SSH in and repeat all these manual steps perfectly. What if you forget to install Redis? The app breaks.

We've successfully run our application, but we've also created a messy, fragile, and non-portable system. We've perfectly demonstrated _why_ we need a better way.

**Now we're ready for** [Part 2](https://wegoagain00.vercel.app/posts/python-chat-app-part-2). We'll solve every single one of these problems by "containerising" our application with Docker. We'll package the app, the database, and all their dependencies into isolated, portable boxes that can run anywhere.
