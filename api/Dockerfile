# multistage docker build. This redices the size of the final docker image.
# stage 1 to build the app
FROM golang:alpine as builder

RUN mkdir /build 

ADD . /build/

WORKDIR /build 
 
# Use vendor mode to avoid fetching modules during build (helps when proxy/tls issues occur)
RUN if [ -d "vendor" ]; then \
		echo "Using vendor for modules"; \
		go build -mod=vendor -o main .; \
	else \
		go build -o main .; \
	fi

# stage 2 deploys the app built in stage 1
FROM alpine

RUN adduser -S -D -H -h /app appuser

USER appuser

COPY . /app

COPY --from=builder /build/main /app/

WORKDIR /app

EXPOSE 3000

CMD ["./main"]