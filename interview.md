# DevOps Home Assignment

**Submission:** A link to a public GitHub repository

---

## Background

You've just joined a small team maintaining **Linker** — an internal URL shortener service used by the company.

The service is a small Python application. Your job is to **containerize it, deploy it to Kubernetes, and set up a GitOps workflow** so that any change merged to `main` is automatically reflected in the cluster.

The team has no existing infrastructure for this service. You are starting from scratch.

---

## The Application

Below is the full source code of the service. Save it as `app.py`.

```python
import os
import hashlib
from flask import Flask, request, jsonify, redirect

app = Flask(__name__)

# In-memory store (no persistence needed for this assignment)
store = {}

BASE_URL = os.environ.get("BASE_URL", "http://localhost:8080")


@app.route("/health")
def health():
    return jsonify({"status": "ok"})


@app.route("/shorten", methods=["POST"])
def shorten():
    data = request.get_json()
    if not data or "url" not in data:
        return jsonify({"error": "missing 'url' field"}), 400

    original = data["url"]
    slug = hashlib.md5(original.encode()).hexdigest()[:6]
    store[slug] = original

    return jsonify({"short_url": f"{BASE_URL}/{slug}"}), 201


@app.route("/<slug>")
def resolve(slug):
    original = store.get(slug)
    if not original:
        return jsonify({"error": "not found"}), 404
    return redirect(original, code=302)


@app.route("/stats")
def stats():
    return jsonify({"total_links": len(store)})


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8080)
```

**Dependencies** (`requirements.txt`):
```
flask>=3.0.0
```

The app exposes:
- `POST /shorten` — accepts `{"url": "https://..."}`, returns a short URL
- `GET /<slug>` — redirects to the original URL
- `GET /stats` — returns total number of links stored
- `GET /health` — health check

---

## What We'd Like You to Build

### 1. Containerize the application
Write a `Dockerfile` that packages the application. Think about what makes a container image suitable for production.

### 2. Kubernetes manifests
Write the Kubernetes manifests needed to deploy this service to a cluster. The service should be accessible from outside the cluster. Think about what properties a production deployment should have.

The `BASE_URL` environment variable should be configurable without modifying the image.

### 3. GitOps with Argo CD
Set up an Argo CD `Application` that points to your repository and manages the deployment. The cluster state should always reflect what's in `main`.

### 4. CI pipeline
Add a GitHub Actions workflow that builds and pushes the Docker image on every push to `main`. You can push to [GitHub Container Registry (GHCR)](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry) — it's free with your GitHub account.

### 5. README
Write a `README.md` that explains:
- How to run the app locally with Docker
- How to bootstrap the deployment on a local cluster (e.g. `kind` or `minikube`)
- How to install and configure Argo CD to sync this repo
- How to verify everything is working end-to-end

---

## Repository Structure

There is no required structure. Organize the repository in whatever way you think is clean and maintainable. Your structure will be part of the evaluation.

---

## Notes

- You don't need a real domain or a cloud cluster. A local `kind` or `minikube` cluster is perfectly fine.
- The app uses in-memory storage — there's no database to worry about.
- If you make trade-offs or leave something incomplete, note it in the README.
- We value clarity and intentionality over completeness. A smaller, well-reasoned submission beats a large messy one.

---

## Submission

Send us the link to your GitHub repository. Make sure it's public.

--- 
## Bonus
- Helmchart usage

