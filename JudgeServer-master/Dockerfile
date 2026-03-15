FROM debian:bookworm-slim AS builder
ARG TARGETARCH
ARG TARGETVARIANT

ENV DEBIAN_FRONTEND=noninteractive
WORKDIR /app

RUN --mount=type=cache,target=/var/cache/apt,id=apt-cahce-1-$TARGETARCH$TARGETVARIANT-builder,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,id=apt-cahce-2-$TARGETARCH$TARGETVARIANT-builder,sharing=locked \
    <<EOS
set -ex
rm -f /etc/apt/apt.conf.d/docker-clean
echo 'Binary::apt::APT::Keep-Downloaded-Packages "1";' > /etc/apt/apt.conf.d/keep-cache
echo 'APT::Install-Recommends "0";' > /etc/apt/apt.conf.d/no-recommends
echo 'APT::AutoRemove::RecommendsImportant "0";' >> /etc/apt/apt.conf.d/no-recommends
apt-get update
apt-get install -y libtool make cmake libseccomp-dev gcc python3 python3-venv
EOS

COPY Judger-1.5/ /app/
RUN <<EOS
set -ex
mkdir /app/build
cmake -S . -B build
cmake --build build --parallel $(nproc)
EOS

RUN <<EOS
set -ex
cd bindings/Python
python3 -m venv .venv
.venv/bin/pip3 install setuptools
.venv/bin/python3 setup.py sdist
cd dist
ls -la
EOS

FROM debian:bookworm-slim
ARG TARGETARCH
ARG TARGETVARIANT

ENV DEBIAN_FRONTEND=noninteractive
WORKDIR /app

RUN --mount=type=cache,target=/var/cache/apt,id=apt-cahce-1-$TARGETARCH$TARGETVARIANT-final,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,id=apt-cahce-2-$TARGETARCH$TARGETVARIANT-final,sharing=locked \
    <<EOS
set -ex
rm -f /etc/apt/apt.conf.d/docker-clean
echo 'Binary::apt::APT::Keep-Downloaded-Packages "1";' > /etc/apt/apt.conf.d/keep-cache
echo 'APT::Install-Recommends "0";' > /etc/apt/apt.conf.d/no-recommends
echo 'APT::AutoRemove::RecommendsImportant "0";' >> /etc/apt/apt.conf.d/no-recommends
apt-get update
apt-get install -y ca-certificates curl gnupg
apt-get install -y python3-minimal \
    python3-venv \
    libpython3-stdlib \
    libpython3-dev \
    golang-go \
    openjdk-17-jdk \
    gcc \
    g++ \
    nodejs \
    strace
EOS

COPY --from=builder --chmod=755 --link /app/output/libjudger.so /usr/lib/judger/libjudger.so
COPY --from=builder /app/bindings/Python/dist/ /app/
RUN --mount=type=cache,target=/root/.cache/pip,id=pip-cahce-$TARGETARCH$TARGETVARIANT-final \
    <<EOS
set -ex
python3 -m venv .venv
CC=gcc .venv/bin/pip3 install --compile --no-cache-dir flask gunicorn idna psutil requests
.venv/bin/pip3 install *.tar.gz
EOS

COPY server/ /app/
RUN <<EOS
set -ex
chmod -R u=rwX,go=rX /app/
chmod +x /app/entrypoint.sh
gcc -shared -fPIC -o unbuffer.so unbuffer.c
useradd -u 901 -r -s /sbin/nologin -M compiler
useradd -u 902 -r -s /sbin/nologin -M code
useradd -u 903 -r -s /sbin/nologin -M -G code spj
mkdir -p /usr/lib/judger
EOS

RUN <<EOS
set -ex
gcc --version
g++ --version
python3 --version
java -version
node --version
EOS

HEALTHCHECK --interval=5s CMD [ "/app/.venv/bin/python3", "/app/service.py" ]
EXPOSE 8080
ENTRYPOINT [ "/app/entrypoint.sh" ]
