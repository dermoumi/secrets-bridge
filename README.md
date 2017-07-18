Fork of https://github.com/abourget/secrets-bridge

Allows running `docker build` without a secrets-bridge server running,
allowing existing environment variables to be used as-is.

# Example

Using my secrets-bridge version:

    $ wget https://github.com/dermoumi/secrets-bridge/releases/download/v1.2.1-permissive/secrets-bridge-linux-amd64 -O secrets-bridge
    $ chmod +x secrets-bridge

Using the following Dockerfile:

    FROM alpine:latest
    ARG BRIDGE_CONF={}

    ENV H=default

    RUN apk update && \
     apk add ca-certificates && \
     update-ca-certificates && \
     apk add openssl

    ADD secrets-bridge /usr/bin/secrets-bridge
    RUN chmod +x /usr/bin/secrets-bridge

    # RUN secrets-bridge print TEST -c $BRIDGE_CONF
    RUN secrets-bridge exec -c $BRIDGE_CONF -e H=TEST -- sh -c 'echo \"$H\"'

Note: the `{}` default value to BRIDGE_CONF is important...

When we run docker build:

    $ docker build -t test-image .

The image should be built using the default $H value

    Step 7/7 : RUN secrets-bridge exec -c $BRIDGE_CONF -e H=TEST -- sh -c 'echo \"$H\"'
     ---> Running in ab10b47aa65c
    2017/07/18 08:43:40 --bridge-conf has an invalid value: building CA cert pool: invalid PEM encoding for ca_cert field
    2017/07/18 08:43:40 secrets-bridge: Calling subprocess with: ["sh" "-c" "echo \\\"$H\\\""]
    "default"
    2017/07/18 08:43:40 secrets-bridge: Call returned, exited with: %!s(<nil>)

When we run the secrets-bridge with the given secrets:

    $ ./secrets-bridge serve -d daemon.log -f ./bridge-conf -w --ssh-agent-forwarder --secret TEST=MY_SECRET --timeout=300
    2017/07/18 09:54:31 Setting up secrets-bridge daemon, pid=27341
    2017/07/18 09:54:31 Setup successfully.

And we run docker build using the file generated:

    $ docker build -t test-image --build-arg BRIDGE_CONF=`cat bridge-conf` .

The image should be built using the value given to `secrets-bridge` as a secret

    Step 7/7 : RUN secrets-bridge exec -c $BRIDGE_CONF -e H=TEST -- sh -c 'echo \"$H\"'
     ---> Running in 04dc1c296e22
    2017/07/18 08:48:13 secrets-bridge: server connection successful
    2017/07/18 08:48:13 secrets-bridge: Setting up SSH-Agent forwarder...
    2017/07/18 08:48:13 secrets-bridge: Calling subprocess with: ["sh" "-c" "echo \\\"$H\\\""]
    "MY_SECRET"
    2017/07/18 08:48:13 secrets-bridge: Call returned, exited with: %!s(<nil>)

And the `docker history test-image --no-trunc` will not show the secret value

    $ docker history test-image --no-trunc | grep MY_SECRET
