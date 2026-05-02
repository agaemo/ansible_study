FROM python:3.12-slim

RUN apt-get update && apt-get install -y --no-install-recommends \
    openssh-client \
    sshpass \
    && rm -rf /var/lib/apt/lists/* \
    && pip install --no-cache-dir ansible

WORKDIR /ansible

CMD ["/bin/bash"]
