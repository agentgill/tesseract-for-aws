# Using Tesseract with AWS

Using Tesseract with AWS is extremely difficult due to the size of the binaries and the dependencies.

Compile the latest stable version of Tesseract and use with Lambda Layers or Lambda Docker. :fire:

## Resources

- [Docs](https://tesseract-ocr.github.io/)
- [Github](https://github.com/tesseract-ocr/tesseract)

## Build Tesseract 5.5.0 (Latest Stable Version)

Build necessary dependencies for Tesseract.

### Build in Docker

Use Amazon Linux 2023 as the base image.

```bash
docker run --platform linux/amd64 -it -v "${PWD}":/layer-build amazonlinux:2023 bash
```

### Install Dependencies

```bash
# Update system and install development tools
dnf update -y
dnf groupinstall -y "Development Tools"
dnf install -y cmake gcc gcc-c++ make autoconf automake libtool pkgconfig
```

### Install library dependencies

```bash
dnf install -y zlib zlib-devel libjpeg libjpeg-devel libwebp libwebp-devel \
    libtiff libtiff-devel libpng libpng-devel wget libicu libicu-devel
```

### Install Leptonica using CMake with position-independent code

```bash
cd /tmp
wget https://github.com/DanBloomberg/leptonica/releases/download/1.83.1/leptonica-1.83.1.tar.gz
tar -xzvf leptonica-1.83.1.tar.gz
cd leptonica-1.83.1
mkdir build && cd build
# Add -DCMAKE_POSITION_INDEPENDENT_CODE=ON to ensure PIC is enabled
cmake -DCMAKE_INSTALL_PREFIX=/usr/local -DCMAKE_POSITION_INDEPENDENT_CODE=ON -DBUILD_SHARED_LIBS=ON ..
make
make install
ldconfig
```

### Install Tesseract 5.5.0 using CMake

```bash
cd /tmp
wget https://github.com/tesseract-ocr/tesseract/archive/refs/tags/5.5.0.tar.gz
tar -xzvf 5.5.0.tar.gz
cd tesseract-5.5.0
mkdir build && cd build
cmake -DCMAKE_INSTALL_PREFIX=/usr/local -DBUILD_SHARED_LIBS=ON -DBUILD_TRAINING_TOOLS=OFF -DCMAKE_POSITION_INDEPENDENT_CODE=ON ..
make
make install
ldconfig
```

### Download English language data

```bash
cd /usr/local/share/tessdata
wget https://github.com/tesseract-ocr/tessdata_best/raw/main/eng.traineddata
```

### Create Lambda Layer Structure

```bash
mkdir -p /layer-build/layer/lib
mkdir -p /layer-build/layer/bin
mkdir -p /layer-build/layer/tessdata
```

### Copy Tesseract binary

```bash
cp /usr/local/bin/tesseract /layer-build/layer/bin/

# Copy required libraries - The CMake build system likely defaulted to using /usr/local/lib64/ 
# cp /usr/local/lib/libtesseract.so* /layer-build/layer/lib/
cp /usr/local/lib64/libtesseract.so* /layer-build/layer/lib/
# cp /usr/local/lib/libleptonica.so* /layer-build/layer/lib/
cp /usr/local/lib64/libleptonica.so* /layer-build/layer/lib/
```

### Create symbolic links with expected names

```bash
cd /layer-build/layer/lib
ln -s libleptonica.so.6 liblept.so
ln -s libleptonica.so.6 liblept.so.6
```

### Copy language data

```bash
cp /usr/local/share/tessdata/eng.traineddata /layer-build/layer/tessdata/
```

### Find and copy all dependencies

```bash
ldd /usr/local/bin/tesseract | grep "=> /" | awk '{print $3}' | xargs -I '{}' cp -v '{}' /layer-build/layer/lib/
```

#### Create zip file for Lambda layer

```bash
cd /layer-build/layer
zip -r ../tesseract-layer.zip *
```

## Working with Lambda

### Using with Lambda Docker

If you need to use Tesseract with Lambda Docker, you can use the following Dockerfile as a starting point.

```dockerfile
FROM public.ecr.aws/lambda/python:3.13

# Install unzip utility
RUN dnf install -y unzip

# Copy tesseract layer and extract it to /opt
COPY tesseract-layer.zip /tmp/
RUN unzip /tmp/tesseract-layer.zip -d /opt && \
    rm /tmp/tesseract-layer.zip

# Copy application files
COPY . ${LAMBDA_TASK_ROOT}
COPY requirements.txt .
RUN pip install -r requirements.txt

CMD ["main.handler" ]

```

### Using the Lambda as a Layer

Create a Lambda Layer using the following command and associate it with your Lambda function.

```bash
aws lambda publish-layer-version --layer-name tesseract --zip-file fileb://tesseract-layer.zip --compatible-runtimes python3.13
```

## Example Lambda Function

```python
import subprocess
import os
from typing import Dict, Any

from aws_lambda_powertools import Logger
from aws_lambda_powertools.logging.formatter import LambdaPowertoolsFormatter
from aws_lambda_powertools.utilities.typing import LambdaContext

formatter = LambdaPowertoolsFormatter(utc=True)
logger = Logger(service="universal-ocr-engine", logger_formatter=formatter)


def handler(event: Dict[str, Any], context: LambdaContext) -> None:
    """Handle OCR processing requests"""

    # Setup & verify tesseract
    env = os.environ.copy()
    get_tesseract_version(env)


def get_tesseract_version(env: Dict[str, str]) -> None:
    """Get and log the installed Tesseract OCR version."""
    try:
        tesseract_v = subprocess.run(
            ["/opt/bin/tesseract", "--version"],
            env=env,
            stdout=subprocess.PIPE,
            stderr=subprocess.PIPE,
        )
        logger.info(f"tesseract_v: {tesseract_v.stdout.decode()}")
    except Exception as e:
        logger.error(f"Error running tesseract: {str(e)}")

```

### Successfully tested with Lambda Layers and Lambda Docker :fire:

```json
{"level":"INFO","location":"handler:69","message":"tesseract_v: tesseract 5.5.0\n leptonica-1.83.1\n  libjpeg 6b (libjpeg-turbo 2.1.4) : libpng 1.6.37 : libtiff 4.4.0 : zlib 1.2.11 : libwebp 1.2.4\n Found AVX2\n Found AVX\n Found FMA\n Found SSE4.1\n","timestamp":"2025-03-17 10:06:57,658+0000","service":"universal-ocr-engine","xray_trace_id":"1-67d7f441-6d3d372abd2e1c6106494145"}
```