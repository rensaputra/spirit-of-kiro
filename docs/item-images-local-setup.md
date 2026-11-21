# Running the Item Images Service Locally

The item images service generates pixel art for in-game items using Amazon Nova Canvas.
It caches results in a vector store (Redis) to avoid regenerating images for similar descriptions,
and stores final images in S3 behind a CloudFront distribution.

The game server points at a remote hosted instance by default. If that instance is unavailable,
follow these steps to run the service on your own machine.

## Prerequisites

- [Bun](https://bun.sh) installed locally
- Docker or Podman running
- AWS credentials configured with access to:
  - Amazon Bedrock (Nova Canvas) in `us-east-1`
  - S3 bucket `kiro-game-images-images-s3-imagesbucket-air3q3idvtae`
  - CloudFront distribution `d8a4w5rlouwtf.cloudfront.net`

## Steps

### 1. Start Redis

The service uses Redis as a vector store to cache and match previously generated images.

```bash
docker run -d --name redis -p 6379:6379 redis:latest
```

Verify it's running:

```bash
docker ps | grep redis
```

### 2. Install dependencies

```bash
cd item-images
bun install
```

### 3. Start the service

```bash
ENVIRONMENT=local bun run server.ts
```

Setting `ENVIRONMENT=local` tells the service to load `local.env`, which already contains
the S3 bucket name and CloudFront domain. The service starts on port `3001` by default.

Verify it's up:

```bash
curl localhost:3001
```

You should see `OK`.

### 4. Point the game server at the local service

Add the following line to `dev.env` in the project root:

```
ITEM_IMAGES_SERVICE_URL=http://localhost:3001
```

### 5. Restart the game server

The server container reads `dev.env` at startup, so it needs a restart to pick up the change.

```bash
# Docker
docker restart server

# Podman
podman restart server
```

### 6. Test image generation

Pull an item in the game using the red PULL lever. Check the `item-images` terminal output —
you should see either:

- `NEW: "..."` — a fresh image was generated via Nova Canvas (takes 5–10 seconds)
- `MATCH: "..."` — a cached image was returned from the vector store (takes ~600ms)

## Notes

- Nova Canvas calls go to `us-east-1` regardless of your default AWS region.
- The first request for any new item description triggers a full generation cycle (Nova Canvas → S3 upload). Subsequent requests with similar descriptions hit the vector cache and return quickly.
- If you stop and restart Redis, the vector cache is lost and all images will be regenerated on next request.
- Image dimensions are fixed at 320×320 pixels in a pixel art style.
