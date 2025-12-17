# syntax=docker/dockerfile:1
ARG PYTHON_VERSION=3.10

FROM python:${PYTHON_VERSION}-slim AS python-base
ARG TEST_ENV

WORKDIR /app

ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    PORT=${PORT:-9090} \
    PIP_CACHE_DIR=/.cache \
    WORKERS=1 \
    THREADS=8 \
    VITH_CHECKPOINT=/app/models/sam_vit_h_4b8939.pth \
    MOBILESAM_CHECKPOINT=/app/models/mobile_sam.pt \
    ONNX_CHECKPOINT=/app/models/sam_onnx_quantized_example.onnx

# Update the base OS
RUN --mount=type=cache,target="/var/cache/apt",sharing=locked \
    --mount=type=cache,target="/var/lib/apt/lists",sharing=locked \
    set -eux; \
    apt-get update; \
    apt-get upgrade -y; \
    apt install --no-install-recommends -y  \
        wget git libopencv-dev python3-opencv cmake protobuf-compiler; \
    apt-get autoremove -y

# Copy and run the model download script
COPY download_models.sh .
# Fix line endings and make executable before running
RUN sed -i 's/\r$//' /app/download_models.sh && \
    chmod +x /app/download_models.sh && \
    bash /app/download_models.sh

# install base requirements
COPY requirements-base.txt .
RUN --mount=type=cache,target=${PIP_CACHE_DIR},sharing=locked \
    pip install --upgrade pip && \
    pip install -r requirements-base.txt

# install custom requirements
COPY requirements.txt .
RUN --mount=type=cache,target=${PIP_CACHE_DIR},sharing=locked \
    pip install -r requirements.txt

# install test requirements if needed
COPY requirements-test.txt .
# build only when TEST_ENV="true"
RUN --mount=type=cache,target=${PIP_CACHE_DIR},sharing=locked \
    if [ "$TEST_ENV" = "true" ]; then \
      pip install -r requirements-test.txt; \
    fi

COPY . .

# keep your glibc workaround if staying on Debian trixie
ENV GLIBC_TUNABLES=glibc.rtld.execstack=2

# ensure a compatible ONNX/ORT pair BEFORE running the converter
RUN --mount=type=cache,target=${PIP_CACHE_DIR},sharing=locked \
    pip install --upgrade "onnx==1.15.0" "onnxruntime==1.16.3"

# Remove unsupported kwarg optimize_model=... from onnxconverter.py
RUN python3 - <<'PY'
import re, io
p = "onnxconverter.py"
s = open(p, "r", encoding="utf-8").read()
orig = s
# drop ", optimize_model=VALUE" no matter the whitespace/newlines
s = re.sub(r',\s*optimize_model\s*=\s*[^,)\n]+', '', s, flags=re.IGNORECASE)
# clean up possible leftover commas
s = re.sub(r'\(\s*,', '(', s)        # "(" followed by comma
s = re.sub(r',\s*,', ',', s)         # double comma -> single
s = re.sub(r',\s*\)', ')', s)        # comma right before ")"
open(p, "w", encoding="utf-8").write(s)
print("Patched optimize_model kwarg:", "changed" if s != orig else "not found")
PY

# (optional) sanity check
RUN grep -n "quantize_dynamic" onnxconverter.py || true

# now run the conversion
RUN python3 onnxconverter.py

EXPOSE 9090

CMD gunicorn --bind :${PORT:-9090} --workers $WORKERS --threads $THREADS --timeout 0 _wsgi:app