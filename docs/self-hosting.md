# Self-Hosting the Public Speech and Translation Stack

This guide describes how to run the three services behind the public endpoints:

| Service | Software | Container port | Public endpoint |
|---|---|---|---|
| Translation and language identification | `linguonnx` + `ovos-plugin-linguonnx` behind `ovos-translate-server` | 9686 | `translate.openvoiceos.pt` |
| Text to speech | `phoonnx` behind `ovos-tts-server` | 9666 | `phoonnx.openvoiceos.pt` |
| Speech to text | `onnx-asr` + `ovos-stt-plugin-onnx-asr` behind `ovos-stt-server` | 8080 | `stt.openvoiceos.pt` |

All three run offline after the model cache is warm. All three are CPU-only.

The audience is a sysadmin with Docker, a reverse proxy and bulk storage.

## No warranty

These are best-effort community services. There is no service level agreement,
no support contract and no guarantee of availability, correctness or data
retention. A self-hoster owns their own uptime. Plan for the host to fail, and
decide in advance what your clients do when it does.

Every service here has a local fallback. Use the public host to evaluate the
stack. Run your own host for anything you depend on.

## Host requirements

| Resource | Minimum | Comfortable |
|---|---|---|
| CPU | 8 cores | 24 cores |
| RAM | 16 GB | 32 GB |
| Root disk | 40 GB | 100 GB |
| Bulk disk | 200 GB | 1 TB |

The bulk disk holds the model caches. Put the caches on bulk storage. Do not
put them on the root disk. See [Hardware sizing](#hardware-sizing) for the
memory arithmetic.

## Layout

Each service is one Docker Compose stack with a directory per service:

```
/opt/stacks/ovos-translate-servers/
    .env
    compose.yaml
    linguonnxsrv/
        Dockerfile
        prefetch.py
        conf/mycroft.conf
    promote-linguonnx.sh
/opt/stacks/ovos-tts-servers/
    .env
    compose.yaml
    phoonnx-src/
        Dockerfile
/opt/stacks/ovos-stt-servers/
    .env
    compose.yaml
    onnxasr/
        Dockerfile
        multi/mycroft.conf
```

The `.env` file holds the host port numbers, the uid and gid, and the cache
paths. Keep secrets out of the compose files.

## Translation and language identification: linguonnx

This is the largest and the most demanding service. Read this section fully
before you deploy it.

### The easy path: the published image

`ovos-plugin-linguonnx` publishes `ghcr.io/openvoiceos/ovos-plugin-linguonnx`.
The image runs both plugins behind `ovos-translate-server` on port 9686:
`opm.lang.translate` as the translation engine, `opm.lang.detect` as the
detection engine.

```bash
docker run -p 9686:9686 \
    -v /media/data/.cache/linguonnx:/home/ovos/.cache/linguonnx \
    -v /media/data/.cache/huggingface:/home/ovos/.cache/huggingface \
    ghcr.io/openvoiceos/ovos-plugin-linguonnx:dev
```

Use this image unless you need a dependency set it does not carry.

### The custom path: build your own

Build your own image when you need extra dependency groups. The public host
does, because three model families need them:

* `distance` supplies `orthography2ipa`. It makes `pivot_ranking: auto` rank
  pivot languages by phonological distance.
* `opennmt` supplies `sacremoses` and `subword-nmt`. Without it the OpenNMT
  models sit in the routing graph and raise `ImportError` when a route picks
  one.
* `indic` supplies the IndicTrans2 preprocessing, with the same failure mode.

```dockerfile
FROM python:3.12-slim

RUN apt-get update \
    && apt-get install -y --no-install-recommends git \
    && rm -rf /var/lib/apt/lists/* \
    && useradd -m -u 1000 ovos

RUN pip install --no-cache-dir \
        "ovos-translate-server @ git+https://github.com/OpenVoiceOS/ovos-translate-server.git@<commit>" \
        "linguonnx[distance,opennmt,indic] @ git+https://github.com/TigreGotico/linguonnx.git@<commit>" \
        "ovos-plugin-linguonnx @ git+https://github.com/OpenVoiceOS/ovos-plugin-linguonnx.git@<commit>"

COPY prefetch.py /usr/local/bin/prefetch.py

# Create the cache directories, owned by the runtime user, BEFORE the USER
# instruction. Docker creates a missing bind-mount parent as root. A container
# that runs as uid 1000 then cannot create ~/.cache/huggingface inside it, and
# every request for an uncached model answers HTTP 500.
RUN mkdir -p /home/ovos/.cache/huggingface /home/ovos/.cache/linguonnx \
    && chown -R ovos:ovos /home/ovos/.cache

USER ovos
ENV HOME=/home/ovos
ENV HF_HOME=/home/ovos/.cache/huggingface
ENV OMP_NUM_THREADS=8

ENTRYPOINT ["ovos-translate-server", \
            "--tx-engine", "ovos-translate-plugin-linguonnx", \
            "--detect-engine", "ovos-lang-detect-plugin-linguonnx", \
            "--port", "9686", "--host", "0.0.0.0"]
```

Pin every dependency to an explicit commit. A floating branch makes the
deployed build unreproducible, and a rebuild then changes behaviour without a
change to your files.

Set `HF_HOME` explicitly. An uncached side file then has a writable target on
bulk storage. Without it the path resolves into the container layer on the
root disk.

Set `OMP_NUM_THREADS`. Left unbounded, `onnxruntime` takes every core and
starves the other services on the host.

### Compose

```yaml
services:
  ovos_tx_linguonnx:
    container_name: ovos_tx_linguonnx
    build: ./linguonnxsrv
    image: local/ovos-tx-server-linguonnx:latest
    restart: always
    ports:
      - 9689:9686
    volumes:
      - /media/data/.cache/linguonnx:/home/ovos/.cache/linguonnx
      - /media/data/.cache/huggingface:/home/ovos/.cache/huggingface
      - ./linguonnxsrv/conf:/home/ovos/.config/mycroft:ro,z
    mem_limit: 12g
    environment:
      PUID: 1000
      PGID: 1000
      OMP_NUM_THREADS: 8
```

**Keep `mem_limit`.** It is the only bound that holds when a sizing assumption
is wrong. Without it, an out-of-memory condition lets the kernel choose a
victim across the whole host. The kernel can kill any of the other services
instead of this one. With it, one container restarts and the rest keep
serving.

### Configuration

The plugin reads `mycroft.conf`. The key path is
`language.translation_module` → `ovos-translate-plugin-linguonnx` for
translation, `language.detection_module` →
`ovos-lang-detect-plugin-linguonnx` for language identification.

```json
{
  "language": {
    "translation_module": "ovos-translate-plugin-linguonnx",
    "detection_module": "ovos-lang-detect-plugin-linguonnx",
    "ovos-translate-plugin-linguonnx": {
      "prefer": "dedicated",
      "max_hops": 2,
      "pivot_ranking": "auto",
      "include_noncommercial": false,
      "precision": "int8",
      "model_cache_size": 4,
      "num_beams": 4,
      "max_new_tokens": 512,
      "max_model_mb": 500,
      "oversize_fallback": true,
      "count_cached_as_free": false
    },
    "ovos-lang-detect-plugin-linguonnx": {
      "model": "glotlid-int8",
      "collapse_varieties": true
    }
  }
}
```

What each choice buys:

| Key | Effect |
|---|---|
| `prefer: dedicated` | Prefers a bilingual model over a large multilingual one. Routes run in 165 MB to 237 MB instead of 4945 MB. |
| `max_hops: 2` | Allows one pivot language between source and target. |
| `precision: int8` | Quantized models. Smaller cache, less memory, faster inference. |
| `model_cache_size: 4` | Retains four loaded models. |
| `max_model_mb: 500` | Routing cap. Applies before anything loads. |
| `count_cached_as_free: false` | Makes the cap apply to models already on disk. Without it a warm cache exempts big models and the cap does nothing. |
| `oversize_fallback: true` | A pair that nothing under the cap covers escalates to the smallest model that does. Keeps coverage wide. |

Add `max_concurrent_translations` when the host serves other work. It is the
only setting that bounds how many models are in flight at once. See
[Hardware sizing](#hardware-sizing).

### Do not cap the download budget near the routing cap

`LINGUONNX_MAX_DOWNLOAD_MB` is a cold-download ceiling, not a routing knob. It
also ceilings how far `oversize_fallback` may escalate, because the fallback
never admits a model the downloader would refuse.

Set it to `500` beside `max_model_mb: 500` and escalation collapses. Routable
languages on the default registry drop from 586 to 249, the same as having no
fallback at all.

Leave `LINGUONNX_MAX_DOWNLOAD_MB` at its default of 8192 MB.

### Model cache

Models are not baked into the image. The permissive int8 registry is about
109 GB. A warm cache on the public host measures 118 GB across 200 model
directories, plus 4.7 GB of raw HuggingFace blobs.

Mount both cache directories onto bulk storage:

```yaml
- /media/data/.cache/linguonnx:/home/ovos/.cache/linguonnx
- /media/data/.cache/huggingface:/home/ovos/.cache/huggingface
```

Mounting only one of them leaves the other on the root disk.

### Prefetch before you serve

A cold request downloads and loads a model while the client waits. A 4.9 GB
model takes about 349 seconds. A warm load is under one second.

Prefetch the whole selection once, after you build the image, against the same
cache volume the server mounts. Use the model manager's prefetch path. Do not
call `load()`: that builds an inference session you throw away.

```python
from linguonnx import load_translator
from linguonnx.model_manager import prefetch

tx = load_translator(precision="int8", include_noncommercial=False)
for model_id in sorted(tx.models):
    # enforce_budget=False: the request-path size limit exists to stop a
    # request from downloading a model. This is not the request path.
    prefetch(model_id, kind="translate", enforce_budget=False)

prefetch("glotlid-int8", kind="lid", enforce_budget=False)
```

Read the selection back from the translator, as above. The prefetch set then
follows the server configuration and the two cannot drift.

Run the prefetch with `HF_HUB_DISABLE_XET=1`:

```bash
docker compose run --rm -e HF_HUB_DISABLE_XET=1 \
    --entrypoint python ovos_tx_linguonnx /usr/local/bin/prefetch.py
```

The xet transfer backend wedges threads in uninterruptible sleep on some
networks. Those threads do not respond to a signal, and the partial state they
leave poisons the shared cache for later runs.

Retry every failure before you believe it. The HuggingFace Hub answers a rate
limit with a 429, and the client surfaces that as `FileNotFoundError`. A
transient rate limit therefore reads as a missing file.

### A model is cached only when its blobs exist

Large graphs keep their weights in an external data blob beside the `.onnx`
file. Both spellings occur: `*.onnx_data` and `*.onnx.data`. A model directory
that holds the `.onnx` file but not the blob looks cached and fails at request
time, not at fetch time.

Check for the blobs, and check that they are not empty:

```bash
find /media/data/.cache/linguonnx/models -name '*.onnx' | while read -r f; do
  for d in "${f}_data" "${f}.data"; do [ -s "$d" ] && continue 2; done
  grep -qs external_data "$f" && echo "MISSING BLOB: $f"
done
```

Re-run the prefetch for anything this reports.

## Text to speech: phoonnx

`phoonnx` runs behind `ovos-tts-server` on port 9666.

```dockerfile
FROM python:3.11-slim

RUN apt-get update && apt-get install -y --no-install-recommends \
        espeak-ng libsndfile1 git build-essential \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app
COPY . /app

RUN pip install --no-cache-dir --upgrade pip \
    && pip install --no-cache-dir torch --index-url https://download.pytorch.org/whl/cpu \
    && pip install --no-cache-dir "setuptools<81" ".[all]" ovos-tts-server \
    && (python -m spacy download en_core_web_sm || true) \
    && (python -m unidic download || true)

RUN useradd -m -u 1000 ovos
USER ovos

# Pre-fill the voice index so the server starts on an empty cache.
RUN phoonnx-voices update-cache || true

EXPOSE 9666
ENTRYPOINT ["ovos-tts-server", "--engine", "ovos-tts-plugin-phoonnx", "--cache"]
```

Notes on the build:

* `espeak-ng` does phonemization. `libsndfile1` reads reference audio for voice
  cloning.
* Install the CPU-only `torch` wheel first. Some phonemizer dependencies pull
  `torch` transitively, and the default index serves multi-gigabyte CUDA
  wheels this image never uses.
* `setuptools<81` keeps `pkg_resources` available for the plugin manager.
* `phoonnx-voices update-cache` fills the voice index at build time. The server
  then starts cleanly against an empty model cache.
* `--cache` persists synthesized audio across restarts.

Compose:

```yaml
services:
  phoonnx:
    container_name: phoonnx
    build: ./phoonnx-src
    image: local/phoonnx:latest
    restart: always
    ports:
      - 9666:9666
    volumes:
      # voice index, downloaded voices and the synthesis cache
      - /media/data/.cache/phoonnx:/home/ovos/.cache
    healthcheck:
      test: ["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://127.0.0.1:9666/status')"]
      interval: 60s
      timeout: 10s
      retries: 3
      start_period: 60s
```

The voice cache goes on bulk storage for the same reason as the translation
cache: the pre-fetched voice set is larger than a root filesystem should hold.

Select the voice through `mycroft.conf` under the TTS plugin key. Mount the
config read-only, as the translation service does. Without a config mount the
server serves its built-in default voice.

## Speech to text: onnx-asr

`onnx-asr` runs behind `ovos-stt-server` on port 8080.

```dockerfile
FROM python:3.12-slim

RUN apt-get update && apt-get install -y --no-install-recommends libsndfile1 git \
    && rm -rf /var/lib/apt/lists/* \
    && useradd -m -u 1000 ovos

RUN pip install --no-cache-dir \
    "ovos-stt-server @ git+https://github.com/OpenVoiceOS/ovos-stt-server.git@<commit>" \
    "ovos-stt-plugin-onnx-asr @ git+https://github.com/OpenVoiceOS/ovos-stt-plugin-onnx-asr.git@<commit>" \
    "onnx-asr[cpu,hub]" python-multipart pydub

USER ovos
ENV HOME=/home/ovos

ENTRYPOINT ["ovos-stt-server", "--engine", "ovos-stt-plugin-onnx-asr", \
            "--port", "8080", "--host", "0.0.0.0"]
```

`libsndfile1` and `pydub` decode the uploaded audio. `python-multipart` accepts
the upload itself. The `hub` extra fetches models from the HuggingFace Hub.

Compose:

```yaml
services:
  ovos_stt_onnxasr:
    container_name: ovos_stt_onnxasr
    build: ./onnxasr
    image: local/ovos-stt-server-onnxasr:latest
    restart: always
    ports:
      - 7074:8080
    volumes:
      - ./onnxasr/multi:/home/ovos/.config/mycroft:ro,z
      - /media/data/.cache/huggingface:/home/ovos/.cache/huggingface
```

Configuration, in `mycroft.conf`:

```json
{
  "stt": {
    "module": "ovos-stt-plugin-onnx-asr",
    "ovos-stt-plugin-onnx-asr": {
      "model": "nemo-parakeet-tdt-0.6b-v3"
    }
  }
}
```

One multilingual model serves every language on the public host. Point the
HuggingFace cache at bulk storage so the model survives a container rebuild.

## Hardware sizing

Size the translation service from the largest model a route can reach. Do not
size it from the cache settings.

Peak resident memory follows this formula, from `linguonnx`:

```
peak RSS  ~=  86 MB
            + 1.84 x ( max_loaded_mb + concurrency x largest declared size in flight )
```

The 1.84 multiplier already contains the per-process overhead. Do not add
overhead a second time.

One `madlad400-3b-mt-int8` in flight declares 4945 MB and measures 9081 MB.
Two exceed a 12 GiB limit. The full table of measured sizes and multipliers is
in [`linguonnx/docs/models.md`](https://github.com/TigreGotico/linguonnx/blob/dev/docs/models.md).

Concurrency is set by the web server, not by the library.
`ovos-translate-server` declares synchronous endpoints, so Starlette runs them
on a threadpool of 40 threads. Forty requests for forty different pairs make
forty models resident. Bound that with `max_concurrent_translations`.

**A small host: 12 GiB for the container**

Cap the routes, then allow some concurrency.

| Setting | Value |
|---|---|
| `max_model_mb` | 500 to 1300 |
| `max_loaded_mb` | 1500 |
| `max_concurrent_translations` | 2 |
| container `mem_limit` | 12g |

`max_model_mb: 1300` drops the 4945 MB model from the graph. The largest
reachable model is then about 1207 MB, and four concurrent translations
compute to 11730 MB.

Keep `oversize_fallback: true` for coverage. Understand the trade: an escalated
route puts the 4945 MB model back in flight. If you enable it, size for one
large model in flight, which means `max_concurrent_translations: 1`, or accept
that the container limit is what catches the second one.

**A host that must reach every language: 24 GiB or more**

| Setting | Value |
|---|---|
| `max_model_mb` | unset |
| `max_loaded_mb` | 2000 |
| `max_concurrent_translations` | 2 |
| container `mem_limit` | 24g |

`86 + 1.84 x (2000 + 2 x 4945)` is 21867 MB.

**No cache setting bounds peak RSS.** A model that a thread is decoding with is
resident because of that thread, not because the cache kept it. Eviction frees
nothing while the thread runs. `max_loaded_mb` bounds what the cache retains
and nothing more.

Keep the container memory limit whatever the settings say.

The speech services are far smaller. One `onnx-asr` model and the phoonnx voice
set fit comfortably in 2 GB to 4 GB each. Disk, not memory, is their
constraint.

## Verification checklist

A green healthcheck is not proof. Use each service the way a client uses it.

**1. Every service answers status**

```bash
curl -fsS http://localhost:9689/status   # translation
curl -fsS http://localhost:9666/status   # tts
curl -fsS http://localhost:7074/status   # stt
```

The translation status lists every routable language. Count it. A short list
means the registry or the routing cap is not what you intended.

**2. Translate, in both directions, through a pivot**

```bash
curl -fsS "http://localhost:9689/translate/en/gl/hello%20world"
curl -fsS "http://localhost:9689/translate/pt/eu/Bom%20dia%2C%20o%20meu%20nome%20e%20Joao."
```

Read the output. An empty `200` is a failure, not a success: a broken model can
return an empty string for every input. A fluent answer in the wrong language
is also a failure, and nothing raises.

**3. Detect a language**

```bash
curl -fsS "http://localhost:9689/detect/bom%20dia"
```

**4. Synthesize speech and play it**

```bash
curl -fsS "http://localhost:9666/synthesize/hello%20world" -o out.wav
soxi out.wav && aplay out.wav
```

A non-empty file is not enough. Listen to it.

**5. Transcribe that file back**

```bash
curl -fsS --data-binary @out.wav "http://localhost:7074/stt?lang=en"
```

The round trip through both speech services is the strongest single check.

**6. Prove a route resolves as you expect**

```bash
docker compose exec ovos_tx_linguonnx python -c "
from linguonnx import load_translator
tx = load_translator(precision='int8', include_noncommercial=False,
                     max_model_mb=500, oversize_fallback=True)
print(len(tx.available_languages))
print(tx.route('en', 'ca').model_ids)
print(tx.route('en', 'cv').model_ids)
print(tx.route('en', 'cv').waived_size_cap)
"
```

`en -> ca` must route through a small bilingual model, about 157 MB. `en -> cv`
has nothing under the cap and must escalate through the oversize fallback to a
4945 MB model, with `waived_size_cap` reporting the cap it waived. A pair that resolves to a different model than
you sized for invalidates the memory arithmetic.

**7. Prove the cache is complete**

Run the external-blob check from
[A model is cached only when its blobs exist](#a-model-is-cached-only-when-its-blobs-exist).
It must report nothing.

**8. Break it on purpose**

Restart every container and repeat steps 1 to 5. Disconnect the host from the
network and repeat them again: a warm cache serves every one of these without
the network. Request an uncached pair and confirm the failure is a readable
error, not a timeout with no explanation.

**9. Watch memory under a real pair**

```bash
docker stats --no-stream ovos_tx_linguonnx
```

Translate the pair that escalates to the largest model. Watch the peak against
your arithmetic. If it exceeds what the formula predicted, lower
`max_concurrent_translations` before you serve traffic.

## Operating notes

* Pin every dependency to a commit. Record the pins in the Dockerfile.
* Rebuild against a new pin in a staging container on a separate port and a
  separate cache. Verify it with this checklist. Then swap it in.
* Tag the running image before you replace it. A rollback is then one command.
* Prefetch the delta into the staging cache before the swap, not after.
* Re-pin every dependency together. Re-pinning one and leaving another rebuilds
  the service against a stale component, which changes behaviour silently.
* Keep the reverse proxy timeouts generous enough for a slow route and short
  enough to shed a stuck one.
* Watch the disk. The translation cache grows with the registry.
