FROM espressif/idf:v5.5.1

RUN apt-get update && \
    apt-get install -y --no-install-recommends usbutils git socat && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

RUN git config --system core.filemode false && \
    git config --system core.autocrlf input
