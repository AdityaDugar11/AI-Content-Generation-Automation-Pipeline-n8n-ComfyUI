# AI Content Generation Automation Pipeline

n8n workflows and ComfyUI graphs for generating AI images and short-form videos, preparing social content with Ollama, and publishing to Instagram and YouTube.

## What is included

| File | Purpose | Main dependencies |
| --- | --- | --- |
| `image generation.json` | Basic ComfyUI text-to-image graph using a checkpoint, KSampler, VAE decode, and image output | ComfyUI, compatible checkpoint |
| `image generation with juggernaut.json` | Text-to-image graph configured for the Juggernaut checkpoint family | ComfyUI, Juggernaut checkpoint |
| `ltx video .json` | LTX video generation graph with GGUF CLIP/UNet loaders and VHS video output | ComfyUI, LTX video models, VHS Video Combine nodes |
| `Image Content Bot.json` | Scheduled content ideation, image generation, file handling, ImgBB upload, and Instagram publishing | n8n, Ollama, ComfyUI, ImgBB, Instagram Graph API |
| `Instagram automation.json` | Creates an educational reel script with Ollama, processes a video, uploads it to YouTube, and publishes it to Instagram | n8n, Ollama, Instagram Graph API, YouTube credentials |
| `ReelAgent V2 - Production.json` | Scheduled or webhook-driven reel production: topic selection, script/caption generation, ComfyUI rendering, file output, YouTube upload, and Instagram publishing | n8n, Ollama, ComfyUI, Instagram Graph API, YouTube credentials |

## Architecture

The production workflows follow this general path:

1. A schedule or webhook starts the workflow.
2. Ollama generates a topic, title, educational points, caption, and visual prompt.
3. n8n sends a ComfyUI API prompt to `http://127.0.0.1:8188/prompt`.
4. The workflow polls ComfyUI history until the generated asset is available.
5. The asset is read from disk and optionally uploaded to a public URL provider.
6. The workflow uploads to Instagram through the Meta Graph API and can upload the video to YouTube.

The standalone ComfyUI exports are graph templates. They are not n8n workflows and should be imported into ComfyUI separately.

## Requirements

- n8n with permission to use HTTP Request, Code, Read/Write Files from Disk, Wait, Webhook, Schedule Trigger, Ollama, and YouTube nodes.
- ComfyUI running locally with its API enabled, normally at `http://127.0.0.1:8188`.
- Ollama running locally, normally at `http://127.0.0.1:11434`, with the model referenced by each Ollama node available.
- ComfyUI models and custom nodes required by the selected graph, including the Juggernaut checkpoint or LTX/GGUF model files where applicable.
- A Meta developer app with Instagram Graph API permissions and an Instagram professional account.
- ImgBB API access for `Image Content Bot.json`.
- YouTube OAuth credentials for the YouTube upload nodes.
- A public HTTPS URL for Instagram video publishing. The exported workflows contain an example ngrok URL that must be replaced.

## Quick start

### 1. Install and start ComfyUI

Start ComfyUI and verify these endpoints from the n8n host:

```text
GET  http://127.0.0.1:8188/history
POST http://127.0.0.1:8188/prompt
```

Import one of the ComfyUI JSON graphs and confirm it runs by itself before connecting it to n8n.

### 2. Install and configure Ollama

Start Ollama and pull the model required by the Ollama node. The model name is stored inside the n8n export, so review it before running the workflow.

### 3. Import the n8n workflows

In n8n, choose **Workflows > Import from File** and import the desired JSON file. Configure the referenced credentials after import. Do not commit credentials or API tokens into workflow exports.

### 4. Update local paths and URLs

Search the imported workflow for these values and replace them for your environment:

- ComfyUI output directory paths in Read/Write Files from Disk and Code nodes.
- The example public video URL used by Instagram publishing.
- The Instagram account ID in Meta Graph API URLs.
- The ImgBB API key and Meta access token, stored as n8n credentials or secure variables.
- The Ollama and ComfyUI host URLs if those services are not on the same machine as n8n.

### 5. Test manually

Run each workflow manually with a test topic. Confirm that the file is created, the public media URL is reachable, and the Meta media container reaches a publishable state before enabling a schedule.

## Trigger inputs

The webhook-based workflows expect a topic field named `assigned_topic` in the incoming JSON. Example:

```json
{
  "assigned_topic": "How to build a focused morning routine"
}
```

The scheduled branches generate or select the topic internally. Review the Code nodes if your input field names or content niche differ.

## Instagram publishing notes

Instagram publishing uses the Meta Graph API in two stages:

1. Create a media container with the media URL and caption.
2. Publish the returned container ID after the media is available.

The video must be reachable from Meta over public HTTPS. Localhost paths and private LAN URLs will not work. The account ID, API version, permissions, token, and media URL all need to match your Meta app configuration.

## File and model conventions

The workflows currently reference Windows-style and mounted filesystem paths from the original development environment. Treat those paths as examples. Set the ComfyUI output directory and n8n file permissions explicitly for the machine running the services.

ComfyUI graph JSON files may depend on exact model filenames, node versions, and custom-node packages. If a graph imports with missing nodes or models, install the matching dependency or update the loader settings in ComfyUI.

## Security

- Keep Meta, ImgBB, Ollama, and YouTube credentials in n8n credentials or environment variables.
- Rotate any token that has been exposed outside n8n.
- Restrict webhook access and add authentication before exposing n8n to the internet.
- Review Code nodes before enabling schedules because they read and write files on the n8n host.

## License

MIT License