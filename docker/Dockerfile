FROM python:3.13-slim AS builder

# Define build arguments with default values
ARG BRANCH=main
# This will be overridden by the GitHub Actions workflow
# when building the Docker image for production.
ARG REPO_URL=https://github.com/Dispatcharr/Dispatcharr
# Add timestamp argument
ARG TIMESTAMP

ENV PATH="/dispatcharrpy/bin:$PATH" \
    VIRTUAL_ENV=/dispatcharrpy \
    DJANGO_SETTINGS_MODULE=dispatcharr.settings \
    PYTHONUNBUFFERED=1 \
    DISPATCHARR_BUILD=1

RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    build-essential \
    curl \
    gcc \
    git \
    libpcre3 \
    libpcre3-dev \
    python3-dev \
    wget && \
    python -m pip install virtualenv && \
    virtualenv /dispatcharrpy && \
    git clone -b ${BRANCH} ${REPO_URL} /app && \
    cd /app && \
    rm -rf .git

# Update version.py with build timestamp if provided
RUN if [ -n "$TIMESTAMP" ]; then \
    echo "Updating timestamp to ${TIMESTAMP} in version.py" && \
    sed -i "s|__timestamp__ = None.*|__timestamp__ = '${TIMESTAMP}'    # Set during CI/CD build process|" /app/version.py && \
    cat /app/version.py; \
    fi

RUN cd /app && \
    pip install --no-cache-dir -r requirements.txt

# Use a dedicated Node.js stage for frontend building
FROM node:20 AS frontend-builder
WORKDIR /app/frontend
COPY --from=builder /app /app
RUN corepack enable && corepack prepare yarn@stable --activate && \
    yarn install && \
    yarn build && \
    find . -maxdepth 1 ! -name '.' ! -name 'dist' -exec rm -rf '{}' \;

FROM python:3.13-slim

ENV PATH="/dispatcharrpy/bin:$PATH" \
    VIRTUAL_ENV=/dispatcharrpy \
    DJANGO_SETTINGS_MODULE=dispatcharr.settings \
    PYTHONUNBUFFERED=1

# Copy the virtual environment and application from the builder stage
COPY --from=builder /dispatcharrpy /dispatcharrpy
COPY --from=builder /app /app
COPY --from=frontend-builder /app/frontend/dist /app/frontend/dist

# Run collectstatic after frontend assets are copied
RUN cd /app && python manage.py collectstatic --noinput

# Install base dependencies with memory optimization
RUN apt-get update && \
    DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
    curl \
    ffmpeg \
    libpcre3 \
    libpq-dev \
    nginx \
    procps \
    streamlink \
    wget \
    gnupg2 \
    lsb-release \
    libva-drm2 \
    libva-x11-2 \
    libva-dev \
    libva-wayland2 \
    vainfo \
    i965-va-driver \
    intel-media-va-driver \
    mesa-va-drivers && \
    cp /app/docker/nginx.conf /etc/nginx/sites-enabled/default && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Set up Redis repository in a separate step
RUN curl -fsSL https://packages.redis.io/gpg | gpg --dearmor -o /usr/share/keyrings/redis-archive-keyring.gpg && \
    chmod 644 /usr/share/keyrings/redis-archive-keyring.gpg && \
    echo "deb [signed-by=/usr/share/keyrings/redis-archive-keyring.gpg] https://packages.redis.io/deb $(lsb_release -cs) main" | tee /etc/apt/sources.list.d/redis.list && \
    apt-get update && \
    DEBIAN_FRONTEND=noninteractive apt-get install -y redis-server && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Set up PostgreSQL repository and install in a separate step
RUN echo "=== setting up postgres ====" && \
    sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && \
    wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | gpg --dearmor -o /usr/share/keyrings/postgresql-keyring.gpg && \
    echo "deb [signed-by=/usr/share/keyrings/postgresql-keyring.gpg] http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list && \
    apt-get update && \
    DEBIAN_FRONTEND=noninteractive apt-get install -y postgresql-14 postgresql-contrib-14 && \
    mkdir -p /data && \
    apt-get remove -y gnupg2 lsb-release && \
    apt-get clean && \
    apt-get autoremove -y && \
    rm -rf /tmp/* /var/lib/apt/lists/* /var/tmp/*

WORKDIR /app

CMD ["/app/docker/entrypoint.sh"]
