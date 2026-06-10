# Stage 1: Download and extract the GNSS dataset
FROM alpine:latest AS data-fetcher

# Install download dependencies
RUN apk add --no-cache wget tar

WORKDIR /tmp/gnss-data
RUN wget -O dataset.tar.gz "https://sourceforge.net/projects/gnss-sdr/files/data/2013_04_04_GNSS_SIGNAL_at_CTTC_SPAIN.tar.gz/download" \
    && tar -xzf dataset.tar.gz \
    && rm dataset.tar.gz

# ===================================================================
# Stage 2: Final GNSS-SDR image
# ===================================================================
FROM carlesfernandez/docker-gnsssdr:latest

# Copy the extracted data from the first stage into the container workspace
COPY --from=data-fetcher /tmp/gnss-data/ /workspace/

WORKDIR /workspace