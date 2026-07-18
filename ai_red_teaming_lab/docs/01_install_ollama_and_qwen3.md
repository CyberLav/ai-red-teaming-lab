# Installing Ollama and Qwen3

This guide walks through setting up a local AI environment for cybersecurity labs, OSCP style practice, and CTF work, using Ollama and the Qwen3 model family.

## Step 1: Install Ollama

1. Go to the official Ollama website: https://ollama.com
2. Download the Ollama for Windows installer.
3. Run the downloaded `.exe` file and follow the standard installation prompts.
4. Once installed, Ollama runs in your system tray and is accessible from your terminal (PowerShell or Command Prompt).

## Step 2: Verify the installation

Open PowerShell and run:

```
ollama --version
```

This confirms Ollama installed correctly and is on your PATH.

## Step 3: Pull and run Qwen3

In your terminal, pull and start the Qwen3 model. The 7B parameter version is a good balance of performance and resource usage for most laptops:

```
ollama run qwen3:7b
```

The model downloads automatically. Once the download reaches 100 percent, you get a prompt where you can interact with the model directly.

## Step 4: Manage your models

A few commands you will use often.

List installed models:

```
ollama ls
```

Remove a model to free disk space:

```
ollama rm qwen3:7b
```

Exit a chat session:

```
/bye
```

## Next

With Ollama and Qwen3 running, the next document, `02_garak_setup_journey.md`, covers installing Garak and pointing it at this local model.
