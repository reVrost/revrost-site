---
title: "Running Playwright-Go In AWS Lambda"
date: 2024-06-29T15:33:51+10:00
---

There is a few bit of articles around how to run playwright with puppetter
online but there is no content out there on how to run Playwright-Go
specifically in AWS Lambda. This is a guide on how to do it. At least how I've
done it anyway.

I was originally using SST.dev as an IAC for my lambda deployments. However,
annoyingly enough they don't support container image as lambda functions. So I
am currently doing it manually however I am contemplating to move to AWS CDK
because of this.

THe only way to run playwright or playwright-go is to use a container image for
your lambdas.

So first, you need a Dockerfile to build your lambda container with playwright
installed and your go binary:

```
# Start from the Golang base image to build the Go application
FROM golang:1.22 as build
WORKDIR /app
# Set the target OS and architecture for cross-compilation
ENV GOOS=linux
ENV GOARCH=arm64

# Copy go mod and sum files
COPY go.mod go.sum ./
# Copy the necessary source directories
COPY lambda/ lambda/
COPY pkg/ pkg/
COPY internal/ internal/
COPY cmd/ cmd/

# Download dependencies
RUN go mod download

# Build the Go application
RUN go build -o main ./lambda/scrape/main.go
RUN go build -o pwinstall ./cmd/pwinstall/main.go

# Start from the Amazon Linux 2023 base image for deployment (make sure this base supports ARM64)
FROM public.ecr.aws/lambda/provided:al2023

# Install Node.js using NodeSource and DNF (confirm Node.js and npm support ARM64 in this environment)
RUN curl -sL https://rpm.nodesource.com/setup_20.x | bash - && \
    dnf install -y nodejs

# Copy the built Go binary from the build stage
COPY --from=build /app/main ./main
COPY --from=build /app/pwinstall ./pwinstall

RUN chmod +x ./pwinstall
RUN ./pwinstall

# Install Playwright and its dependencies
RUN dnf install -y \
    nss \
    atk \
    cups-libs \
    libXcomposite \
    libXrandr \
    libxkbcommon \
    libXScrnSaver \
    pango \
    at-spi2-atk \
    at-spi2-core \
    libXdamage \
    libXfixes \
    mesa-libgbm \

# Copy the Playwright installation cache
RUN cp -r /root/.cache ./.cache

# Adjust permissions to allow execution
RUN chmod -R a+rx ./.cache/ms-playwright-go/

# Set a home directory environment variable
ENV HOME=.

# Set the entrypoint to the Go binary
ENTRYPOINT ["./main"]
```

Theres a few hacks here that I implemented to get this to work. The first is
`playwright-go` requires a go install command to be run. Now, I could do this
but I don't really want to have my lambda container which is based on AL2023 to
install go.

So I created a separate go binary that installs playwright-go and then I run
that binary in the container. This is the `pwinstall` binary. It basically just
has the following code:

```golang
package main

import "github.com/playwright-community/playwright-go"

func main() {
	err := playwright.Install()
	if err != nil {
		panic(err)
	}
}
```

Once the playwright-go is installed, I copy the cache folder where playwright is
installed to the lambda container.

And then just make sure to run it the browser with these flags so that they dont
crash in lambda:

```golang
browser, err := pw.Chromium.Launch(
	playwright.BrowserTypeLaunchOptions{
		Args: []string{
			"--headless=new",
			"--disable-gpu",
			"--single-process",
		},
	},
)
```

Also make sure to give your lambd enough memory. I firstly tried with 256MB and
it was struggling bumpng it up to 512MB seem to have fixed the issue.

And, thats it! You should now have a working playwright-go lambda function.
