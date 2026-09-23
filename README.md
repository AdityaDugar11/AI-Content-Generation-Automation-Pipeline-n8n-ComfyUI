# n8n AI Content Workflows

This folder contains n8n workflow exports and ComfyUI graph exports for generating images and short videos, writing social copy with Ollama, and publishing to Instagram and YouTube.

The JSON files are templates. They contain example account IDs, local paths, model names, and public URLs that must be reviewed before a workflow is enabled.

## Contents

| File | Use it for | Runs in | Required services |
| --- | --- | --- | --- |
| `image generation.json` | Basic Stable Diffusion text-to-image graph | ComfyUI | Checkpoint model |
| `image generation with juggernaut.json` | Juggernaut XL image generation graph | ComfyUI | Juggernaut checkpoint |
| `ltx video .json` | LTX video generation with GGUF loaders | ComfyUI | LTX models and VHS Video Combine |
| `Image Content Bot.json` | Scheduled idea, image generation, ImgBB upload, and Instagram image publishing | n8n | Ollama, ComfyUI, ImgBB, Meta Graph API |
| `Instagram automation.json` | Ollama script generation, local video handling, YouTube upload, and Instagram publishing | n8n | Ollama, Meta Graph API, YouTube OAuth |
| `ReelAgent V2 - Production.json` | Scheduled or webhook-driven reel generation, ComfyUI rendering, YouTube upload, and Instagram publishing | n8n | Ollama, ComfyUI, Meta Graph API, YouTube OAuth |

## How the pieces fit together

The ComfyUI files are independent graph templates. Import and test them in ComfyUI first. The n8n production workflow then sends a prompt to ComfyUI, waits for the result, reads the generated file, and sends it to the publishing services.

Typical automation flow:

1. A schedule or webhook starts an n8n workflow.
2. Ollama creates the topic, script, caption, and visual prompt.
3. n8n submits a job to ComfyUI at `/prompt` and polls `/history`.
4. n8n reads the generated file from disk.
5. Instagram receives a public HTTPS media URL and caption.
6. YouTube receives the video through the configured OAuth credential when that node is enabled.

## Requirements

- n8n with HTTP Request, Code, Read/Write Files from Disk, Wait, Webhook, Schedule Trigger, Ollama, and YouTube nodes.
- ComfyUI with its API enabled, normally at `http://127.0.0.1:8188`.
- Ollama, normally at `http://127.0.0.1:11434`, with a model suitable for the prompts in the workflow.
- The ComfyUI models and custom nodes required by the selected graph.
- A Meta developer app, an Instagram professional account, and the permissions required by the Instagram Graph API.
- An ImgBB API key for `Image Content Bot.json`.
- A YouTube OAuth credential for the two reel workflows.
- A public HTTPS host for video files. Instagram cannot fetch `localhost`, private LAN addresses, or a local filesystem path.

## First-time setup

### 1. Set up ComfyUI

Import one graph at a time into ComfyUI and run it manually before using n8n. The exported graphs reference these model files:

| Graph | Model files or nodes to verify |
| --- | --- |
| `image generation.json` | `v1-5-pruned-emaonly.ckpt` |
| `image generation with juggernaut.json` | `juggernautXL_juggXILightningByRD.safetensors` |
| `ltx video .json` | `t5-v1_1-xxl-encoder-Q4_K_M.gguf`, `ltx-video-2b-v0.9.1-q4_k_m.gguf`, `ltx-video-vae.safetensors`, GGUF loader nodes, VHS Video Combine |

Model filenames are part of the graph. Either install matching files or change the loader values in ComfyUI.

### 2. Set up Ollama

Start Ollama and confirm it responds from the machine running n8n. Review each Ollama node after import and set its model explicitly; the exports are not a portable model installation.

### 3. Import into n8n

In n8n, use **Workflows > Import from File** and import only the workflow you want to configure. Import the ComfyUI graphs into ComfyUI, not n8n.

After import, configure these n8n credentials:

| Credential | Used by |
| --- | --- |
| Ollama account | `Instagram automation.json`, `ReelAgent V2 - Production.json` |
| YouTube account | `Instagram automation.json`, `ReelAgent V2 - Production.json` |
| Meta access token | HTTP Request nodes that create and publish Instagram media |
| ImgBB API key | `Image Content Bot.json` |

The exported HTTP nodes currently include placeholder token values. Replace them with n8n credentials or protected expressions before testing.

### 4. Replace environment-specific values

Search the imported workflow for and replace:

- `17841475507362376`: the example Instagram professional account ID.
- `Your_facebook_access_Token`: the Meta access token placeholder.
- `christie-approbative-genevieve.ngrok-free.dev`: an example tunnel hostname that is not part of this repository.
- `/mnt/windows/Users/adity/Documents/Aditya/Agents/n8n/ReelAgent`: the original machine's output directory.
- `final_reel.mp4` and `final_reel_v2.mp4`: output names, if you choose different names.
- `http://127.0.0.1:8188` and `http://127.0.0.1:11434`: service URLs when n8n runs in another container or host.

n8n and ComfyUI must be able to see the same output directory. With Docker, mount that directory into both containers and use the container path in the Read/Write Files from Disk nodes.

### 5. Test before scheduling

Run manually and verify each stage in order:

1. Ollama returns parseable content.
2. ComfyUI returns a prompt/job ID and writes the expected file.
3. n8n can read that file.
4. The public media URL returns the media without authentication.
5. Meta creates a media container and its status becomes publishable.
6. YouTube upload succeeds if enabled.

Only enable the Schedule Trigger after a manual run succeeds.

## Webhook inputs

The webhook branches use `assigned_topic`. Example request body:

```json
{
  "assigned_topic": "How to build a focused morning routine"
}
```

The webhook paths in the exports are:

| Workflow | Path |
| --- | --- |
| `Instagram automation.json` | `trigger-workflow-two` |
| `ReelAgent V2 - Production.json` | `trigger-reel-production` |

The scheduled branches choose a topic internally. Treat webhook URLs as private until authentication and rate limiting are configured.

## Instagram and public media

Instagram publishing is a two-step Meta Graph API operation: create a media container, then publish its returned container ID. For video, Meta fetches the media from the URL supplied to the container request. The URL must be public HTTPS, stable during processing, and return the correct content type. A temporary tunnel is suitable for testing only; use durable storage or a controlled media host for production.

## Repository and secret hygiene

`.gitignore` excludes local environment files, n8n data, generated media, logs, backups, and machine-specific workflow copies. Keep real credentials in n8n's credential store or a secret manager. `.env.example` is documentation only; these exported workflows do not automatically read it.

Before pushing changes:

```bash
git status --short
git diff --check
rg -n 'access_token|api[_-]?key|Bearer |ngrok|/home/|/mnt/|[A-Za-z]:\\\\' --glob '*.json' .
```

Review every match. Placeholder values and documented example paths are expected; real tokens, private URLs, and personal filesystem paths are not.

## Troubleshooting

- **Missing ComfyUI node:** install the custom-node package used by the graph, then restart ComfyUI.
- **Missing model:** update the loader filename or place the exact model in the expected ComfyUI models directory.
- **n8n cannot read the file:** use a path visible inside the n8n runtime and grant the n8n process read access.
- **Instagram rejects the media:** verify public HTTPS access, media format, account permissions, token validity, and the account ID.
- **Ollama connection fails:** test the Ollama URL from the n8n runtime, not only from the host shell.
- **Webhook works locally but not remotely:** configure n8n's public URL and a reverse proxy or tunnel with authentication.

## License

MIT License