# Build Stage

# We're using alpine instead of the golang-alpine to configure everything
FROM alpine:3.16.2 as build

# Install dependencies
RUN apk add go git

ENV GOPATH /go
# Add go to the path variable
ENV PATH $GOPATH/bin:/usr/local/go/bin:$PATH
# Setup the go specific folder structure
RUN mkdir -p "$GOPATH/src" "$GOPATH/bin" && chmod -R 777 "$GOPATH"

WORKDIR $GOPATH/src
# Clone repository which we will use
RUN git clone https://github.com/betauia/Beetroot

WORKDIR Beetroot/cmd/beetroot

# Download/Install repository dependencies
RUN go get
# Build the binary file
RUN go build

# Running Stage

FROM alpine:3.16.2
RUN apk add tzdata
# Copy created binary file to the runnning stage
COPY --from=build ./go/src/Beetroot/cmd/beetroot/beetroot .
# Execute the binary file when the container is starting
CMD ./beetroot
