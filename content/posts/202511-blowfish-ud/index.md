---
title: "Blowfish + UD: Build your own Web3 Blog"
tags: ["Web3"]
date: 2025-11-02
draft: false
---

## Introduction

In the era of Web3, owning your content means more than just writing it — it means *truly controlling where and how it lives*. By combining the **Blowfish theme** (a beautiful, modern Hugo theme) with **Unstoppable Domains (UD)** and **IPFS**, you can create a fully decentralized, censorship-resistant blog that’s as easy to update as a traditional site — but lives forever on the blockchain.

This guide walks you through setting up a **Node.js + Hugo + Blowfish** stack on Linux, generating a static site, and publishing it to **IPFS** via **Unstoppable Domains** — all in under 30 minutes.

---

## Prerequisites

We’ll install:

- **Node.js** (v22.20.0) – for Blowfish tools
- **Git** – version control
- **Go** (1.25.1) – required by some Hugo tools
- **Hugo** (via Snap) – static site generator
- **Dart Sass** – included in Hugo Snap (no separate install needed)

---

## Step 0: Get Your Unstoppable Domain

1. Go to [unstoppabledomains.com](https://unstoppabledomains.com)
2. Buy a domain like `myblog.brave` or `decentralized.brave`

---

## Step 1: Install Dependencies (One-Shot Setup)

Open your terminal and run these commands **one after another**:

```bash
# Update system
sudo apt update

# Install Git
sudo apt install git -y

# Install Node.js v22.20.0
wget https://nodejs.org/dist/v22.20.0/node-v22.20.0-linux-x64.tar.xz
sudo tar -C /usr/local -xJf node-v22.20.0-linux-x64.tar.xz

# Install Go 1.25.1
wget https://dl.google.com/go/go1.25.1.linux-amd64.tar.gz
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf go1.25.1.linux-amd64.tar.gz

# Add Node.js and Go to PATH (one-liner)
echo 'export PATH=$PATH:/usr/local/node-v22.20.0-linux-x64/bin:/usr/local/go/bin' >> $HOME/.profile
source $HOME/.profile

# Install Hugo via Snap (includes Dart Sass)
sudo snap install hugo
```

> **Pro Tip**: After running `source $HOME/.profile`, verify installations:
>
> ```bash
> node -v        # → v22.20.0
> go version     # → go1.25.1
> hugo version   # → hugo v0.x.x
> ```

---

## Step 2: Create Your Blog with Blowfish

Blowfish provides a CLI tool to scaffold a new Hugo site with the theme pre-installed.

```bash
npx blowfish-tools new my-web3-blog
cd my-web3-blog
```

This creates a fully functional Hugo site with Blowfish theme, sample content, and config.

---

## Step 3: Configure Hugo for IPFS (Relative URLs)

Edit the main config file:

```bash
nano hugo.toml
```

Add or update these lines:

```toml
relativeURLs = true
baseURL = "/"
```

> `relativeURLs = true` ensures all links (CSS, JS, images) use relative paths — **critical for IPFS**.  
> `baseURL = "/"` lets the site work from any domain or gateway.

Save and exit (`Ctrl+O` → `Enter` → `Ctrl+X`).

---

## Step 4: Write Your First Post

```bash
hugo new posts/my-first-web3-post.md
```

Edit the file:

```bash
nano content/posts/my-first-web3-post.md
```

Add front matter and content:

```markdown
---
title: "Welcome to My Decentralized Blog"
date: 2025-11-02
draft: false
---

This blog is hosted on **IPFS** and resolved via **Unstoppable Domains**.

No servers. No takedowns. Just freedom.
```

---

## Step 5: Build & Preview Locally

```bash
hugo server
```

Visit `http://localhost:1313` — your blog is live locally!

---

## Step 6: Build for Production

```bash
hugo --minify
```

This generates the static site in the `public/` folder.

---

## Step 7: Publish to IPFS via Unstoppable Domains

### Option A: Using Unstoppable Domains (Recommended – Easiest)

1. Go to **My Domains** on [unstoppabledomains.com](https://unstoppabledomains.com)
2. Select your domain (e.g., `myblog.brave`)
3. Click **"Website"**
4. Click **"Upload website files to IPFS"**
5. **Drag your entire `public/` folder** (not individual files) into the upload window
6. Click **Launch Website**

Your site is now live on IPFS and accessible at your domain.

> **To update later**: Rebuild with `hugo --minify`, then re-upload the new `public/` folder. UD will generate a new CID and update your domain automatically.

---

### Option B: Using Pinata (For Power Users)

1. Sign up at [pinata.cloud](https://pinata.cloud)
2. Get your **JWT** from the API keys section
3. Install Pinata CLI:

```bash
npm install -g @pinata/sdk
```

4. Deploy:

```bash
pinata deploy public/ --key YOUR_JWT_KEY
```

5. Copy the returned **CID** (e.g., `QmExample123...`)
6. In UD domain management:
   - Add a **Website Record**
   - Set type: `ipfs`
   - Value: Your CID

Your site is now live at:  
`https://myblog.brave`

---

## Why This Stack Rocks

| Feature                 | Benefit                              |
|-------------------------|--------------------------------------|
| **Blowfish Theme**      | Stunning, customizable, dark mode    |
| **Hugo**                | Blazing fast static sites            |
| **IPFS**                | Immutable, distributed hosting       |
| **Unstoppable Domains** | Human-readable `.brave` URLs         |
| **No Servers**          | Zero maintenance, zero cost          |

---

## Resources

- Blowfish Theme: [https://blowfish.page](https://blowfish.page)
- Hugo Docs: [https://gohugo.io](https://gohugo.io)
- Unstoppable Domains: [https://unstoppabledomains.com](https://unstoppabledomains.com)

---

## Final Thoughts

You now own a **truly decentralized blog**. No hosting bills. No domain seizures. No middlemen.

Write freely. Publish forever.

---

**Your Web3 Blog is Live**  

*Built with 💚 using Blowfish + Hugo + IPFS + UD*