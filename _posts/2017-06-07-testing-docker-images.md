---
layout: post
title:  "Testing Docker images"
date:   2017-06-07 00:00:00
categories: docker testing
image-base: /assets/images/posts/2017-06-07-testing-docker-images
---

As part of my [OpenALPR Docker image](https://github.com/FlyingTopHat/OpenALPR-Docker), I had investigated methods for testing Docker images that wouldn't clutter the Dockerfile with dependencies or test artefacts. The approach I ended up using was creating a Bash script (or PowerShell if you're so inclined) to start the image and then modify it into a state ready for testing, e.g. copying test data into the container. I thought I'd share what it looks like...

![Process from polling image to processing via the Docker image]({{ page.image-base }}/openalpr-docker-process.png)

To provide some context around what I was testing; the image comprises of a Python script (the entry point) that feeds photos polled from a URL, which can be enhanced, to an Automatic License Plate Recognition (ALPR) library. The results from the ALPR library are then posted to a URL, and more importantly, printed to the standard output, which is monitored by the tests.

In my implementation, the [./test.sh](https://github.com/FlyingTopHat/OpenALPR-Docker/blob/master/test.sh) runs the image, starts a web-server inside of the container (removing the need for an additional image or host dependencies) and then copies the test data to the web-server's working directory to be served by the polling Python script. The docker logs are then monitored for the expected output.


```bash
#!/bin/bash
set -e
set -o pipefail

function find_in_docker_log {
    CONTAINER_NAME=$1
    TIMEOUT=$2
    SUBSTRING=$3
    COMMAND="docker logs -f $CONTAINER_NAME"
    expect -c "log_user 0; set timeout $TIMEOUT; spawn $COMMAND; expect \"$SUBSTRING\" { exit 0 } timeout { exit 1 }"
}

IMAGE_NAME=single_horizontal.jpg
CONTAINER_NAME=alpr_test

# Start the container
docker run --name $CONTAINER_NAME -d flyingtophat/alpr
    http://127.0.0.1/$IMAGE_NAME \ # URL to poll for photo
    http://127.0.0.1/ \ # URL to post the results (not used for test)
    --verbose \ # Increase verbosity to ensure data that would be posted is printed to stdout
    --interval 1 # Small increment to speed up tests

# Run the image with the localhost URL to poll
docker run --name $CONTAINER_NAME -d flyingtophat/alpr http://127.0.0.1/$IMAGE_NAME http://127.0.0.1/ --verbose

# Start web-server to serve test data
docker exec -d $CONTAINER_NAME python -m SimpleHTTPServer 80

# Copy test data to the working directory for Python's HTTP Server to serve
docker cp ./test_files/$TEST_DATA_FILENAME $CONTAINER_NAME:/opt/docker-alpr/

# Wait for license plate to appear in image's output
find_in_docker_log $CONTAINER_NAME $TIMEOUT "WR62XDF"
TEST_RESULT=$?

# Report test status
docker rm -f $CONTAINER_NAME
if [ $TEST_RESULT == 0 ]; then
    echo "SUCCESS"
else
    echo "FAILED"
    exit 1
fi
```

If you're interested in seeing it in action, check out the [Travis job](https://travis-ci.org/FlyingTopHat/OpenALPR-Docker).
