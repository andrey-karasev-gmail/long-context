FROM public.ecr.aws/docker/library/node:22-bookworm-slim@sha256:f3a68cf41a855d227d1b0ab832bed9749469ef38cf4f58182fb8c893bc462383

RUN apt-get update && apt-get install -y --no-install-recommends \
        tmux \
        python3 \
        python3-pip \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.lock /tmp/requirements.lock
RUN pip3 install --no-cache-dir --break-system-packages --require-hashes -r /tmp/requirements.lock

WORKDIR /app

COPY app/package.json app/package-lock.json ./
COPY app/packages/core/package.json ./packages/core/
COPY app/packages/api/package.json ./packages/api/
COPY app/packages/ui/package.json ./packages/ui/
COPY app/packages/cli/package.json ./packages/cli/

RUN npm config set strict-ssl false \
    && npm ci

# Pre-cache the override target versions so solve.sh can run npm install offline
RUN npm install --no-save \
    lodash@4.17.21 \
    semver@7.6.3 \
    axios@1.6.8 \
    express@4.19.2 \
    react@18.3.1

COPY app/ .
