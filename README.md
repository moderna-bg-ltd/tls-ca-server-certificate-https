# tls-ca-server-certificate-https

## TLS Basics
- With TLS, the client must trust a certificate authority (CA). The CA will sign something held by the server so the client can verify it. This is a bit like your mutual friend signing a note and you recognizing their handwriting. For more information, see How internet security works: TLS, SSL, and CA.

## Server Authentication
- The following command will create a CA certificate that can be used to sign a server’s certificate:
```
openssl req -x509 -nodes -newkey rsa:4096 -keyout ca.key -out ca.pem \
              -subj /O=me
```

This will output two files:

- ca.key is a private key.
- ca.pem is a public certificate.

You can then create a certificate for your server and sign it with your CA certificate:
```
openssl req -nodes -newkey rsa:4096 -keyout server.key -out server.csr \
              -subj /CN=recommendations
$ openssl x509 -req -in server.csr -CA ca.pem -CAkey ca.key -set_serial 1 \
              -out server.pem
```

This will produce three new files:

- server.key is the server’s private key.
- server.csr is an intermediate file.
- server.pem is the server’s public certificate.

You can add this to the Recommendations microservice `Dockerfile`. It’s very hard to securely add secrets to a Docker image, but there’s a way to do it with the latest versions of Docker, shown highlighted below:
```
# syntax = docker/dockerfile:1.0-experimental

# DOCKER_BUILDKIT=1 docker build . -f recommendations/Dockerfile \

#                     -t recommendations --secret id=ca.key,src=ca.key


FROM python


RUN mkdir /service

COPY infra/ /service/infra/

COPY protobufs/ /service/protobufs/

COPY recommendations/ /service/recommendations/

COPY ca.pem /service/recommendations/


WORKDIR /service/recommendations

RUN python -m pip install --upgrade pip

RUN python -m pip install -r requirements.txt

RUN python -m grpc_tools.protoc -I ../protobufs --python_out=. \

           --grpc_python_out=. ../protobufs/recommendations.proto

RUN openssl req -nodes -newkey rsa:4096 -subj /CN=recommendations \

                -keyout server.key -out server.csr

RUN --mount=type=secret,id=ca.key \

    openssl x509 -req -in server.csr -CA ca.pem -CAkey /run/secrets/ca.key \

                 -set_serial 1 -out server.pem


EXPOSE 50051

ENTRYPOINT [ "python", "recommendations.py" ]
```
The new lines are highlighted. Here’s an explanation:

- Line 1 is needed to enable secrets.
- Lines 2 and 3 show the command for how to build the Docker image.
- Line 11 copies the CA public certificate into the image.
- Lines 18 and 19 generate a new server private key and certificate.
- Lines 20 to 22 temporarily load the CA private key so you can sign the server’s certificate with it. However, it won’t be kept in the image.

Your image will now have the following files:
- ca.pem
- server.csr
- server.key
- server.pem

You can now update serve() in recommendations.py as highlighted:
```
def serve():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    recommendations_pb2_grpc.add_RecommendationsServicer_to_server(
        RecommendationService(), server
    )

    with open("server.key", "rb") as fp:
        server_key = fp.read()

    with open("server.pem", "rb") as fp:
        server_cert = fp.read()

    creds = grpc.ssl_server_credentials([(server_key, server_cert)])
    server.add_secure_port("[::]:443", creds)
    server.start()
    server.wait_for_termination()
```
Here are the changes:
- Lines 7 to 10 load the server’s private key and certificate.
- Lines 12 and 13 run the server using TLS. It will accept only TLS-encrypted connections now.

You’ll need to update marketplace.py to load the CA cert. You only need the public cert in the client for now, as highlighted:
```
recommendations_host = os.getenv("RECOMMENDATIONS_HOST", "localhost")
with open("ca.pem", "rb") as fp:
    ca_cert = fp.read()
creds = grpc.ssl_channel_credentials(ca_cert)
recommendations_channel = grpc.secure_channel(
    f"{recommendations_host}:443", creds
)
recommendations_client = RecommendationsStub(recommendations_channel)
```
You’ll also need to add COPY ca.pem /service/marketplace/ to the Marketplace Dockerfile.

`docker-compose up`, run `docker-compose build` after the latest changesbelow:
```
version: "3.8"
services:
    marketplace:
        environment:
            RECOMMENDATIONS_HOST: recommendations
        # DOCKER_BUILDKIT=1 docker build . -f marketplace/Dockerfile \
        #                   -t marketplace --secret id=ca.key,src=ca.key
        image: marketplace
        networks:
            - microservices
        ports:
            - 5000:5000
    recommendations:
        # DOCKER_BUILDKIT=1 docker build . -f recommendations/Dockerfile \
        #                   -t recommendations --secret id=ca.key,src=ca.key
        image: recommendations
        networks:
            - microservices
networks:
    microservices:
```

#### Mutual Authentication
The server now proves that it can be trusted, but the client does not. Luckily, TLS allows verification of both sides. Update the Marketplace Dockerfile as highlighted:
```
# syntax = docker/dockerfile:1.0-experimental
# DOCKER_BUILDKIT=1 docker build . -f marketplace/Dockerfile \
#                     -t marketplace --secret id=ca.key,src=ca.key
FROM python

RUN mkdir /service
COPY protobufs/ /service/protobufs/
COPY marketplace/ /service/marketplace/
COPY ca.pem /service/marketplace/
WORKDIR /service/marketplace
RUN python -m pip install -r requirements.txt
RUN python -m grpc_tools.protoc -I ../protobufs --python_out=. \
           --grpc_python_out=. ../protobufs/recommendations.proto
RUN openssl req -nodes -newkey rsa:4096 -subj /CN=marketplace \
                -keyout client.key -out client.csr
RUN --mount=type=secret,id=ca.key \
    openssl x509 -req -in client.csr -CA ca.pem -CAkey /run/secrets/ca.key \
                 -set_serial 1 -out client.pem
EXPOSE 5000
ENV FLASK_APP=marketplace.py
ENTRYPOINT [ "flask", "run", "--host=0.0.0.0"]
```

Update serve() in recommendations.py to authenticate the client as highlighted:
```
def serve():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    recommendations_pb2_grpc.add_RecommendationsServicer_to_server(
        RecommendationService(), server
    )

    with open("server.key", "rb") as fp:
        server_key = fp.read()
    with open("server.pem", "rb") as fp:
        server_cert = fp.read()
    with open("ca.pem", "rb") as fp:
        ca_cert = fp.read()

    creds = grpc.ssl_server_credentials(
        [(server_key, server_cert)],
        root_certificates=ca_cert,
        require_client_auth=True,
    )
    server.add_secure_port("[::]:443", creds)
    server.start()
    server.wait_for_termination()
```
This loads the CA certificate and requires client authentication.

Finally, update marketplace.py to send its certificate to the server as highlighted:
```
recommendations_host = os.getenv("RECOMMENDATIONS_HOST", "localhost")
with open("client.key", "rb") as fp:
    client_key = fp.read()
with open("client.pem", "rb") as fp:
    client_cert = fp.read()
with open("ca.pem", "rb") as fp:
    ca_cert = fp.read()
creds = grpc.ssl_channel_credentials(ca_cert, client_key, client_cert)
recommendations_channel = grpc.secure_channel(
    f"{recommendations_host}:443", creds
)
recommendations_client = RecommendationsStub(recommendations_channel)
```
