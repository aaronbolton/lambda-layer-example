# Creating a Lambda Layer for pyodbc (SQL Server / MSSQL)

> Tested with: `public.ecr.aws/lambda/python:3.14` (Amazon Linux 2023)

---

## Prerequisites

- Docker installed and running
- AWS CLI configured

---

## 1. Pull the Lambda base image

```bash
docker pull public.ecr.aws/lambda/python:3.14
```

Verify it's available:

```bash
docker image ls
# REPOSITORY                     TAG       IMAGE ID       CREATED       SIZE
# public.ecr.aws/lambda/python   3.14      6779653486fa   ...           514MB
```

---

## 2. Create the layer directory

```bash
mkdir -p lambda-layer/python
```

---

## 3. Build the layer inside Docker

```bash
docker run --rm \
  --entrypoint bash \
  -v "$PWD/lambda-layer":/var/task \
  public.ecr.aws/lambda/python:3.14 \
  -c "
    # Install unixODBC
    dnf install -y unixODBC unixODBC-devel

    # Add Microsoft repo and install ODBC 18 driver
    curl https://packages.microsoft.com/config/rhel/9/prod.repo > /etc/yum.repos.d/mssql-release.repo
    ACCEPT_EULA=Y dnf install -y msodbcsql18

    # Install pyodbc into the python/ directory
    pip install pyodbc -t /var/task/python

    # Copy shared libraries
    mkdir -p /var/task/lib
    cp /usr/lib64/libodbc.so.2          /var/task/lib/
    cp /usr/lib64/libodbcinst.so.2      /var/task/lib/
    cp /opt/microsoft/msodbcsql18/lib64/libmsodbcsql-18.*.so.* /var/task/lib/

    # Copy ODBC config files
    mkdir -p /var/task/odbcinst
    cp /opt/microsoft/msodbcsql18/etc/odbcinst.ini /var/task/odbcinst/
  "
```

After this runs, your `lambda-layer/` directory should look like:

```
lambda-layer/
├── python/          # pyodbc package
├── lib/             # libodbc.so.2, libodbcinst.so.2, libmsodbcsql-18.x.so.x
└── odbcinst/        # odbcinst.ini
```

---

## 4. Zip the layer

```bash
cd lambda-layer
zip -r ../pyodbc-layer.zip .
cd ..
```

---

## 5. Publish to AWS

```bash
aws lambda publish-layer-version \
  --layer-name pyodbc-layer \
  --zip-file fileb://pyodbc-layer.zip \
  --compatible-runtimes python3.14 \
  --region eu-west-1
```

Copy the `LayerVersionArn` from the output — you'll need it in the next step.

---

## 6. Attach the layer to your Lambda function

```bash
aws lambda update-function-configuration \
  --function-name your-function-name \
  --layers arn:aws:lambda:eu-west-1:123456789012:layer:pyodbc-layer:1
```

---

## 7. Configure Lambda environment variables

Set these on your Lambda function so the shared libraries and ODBC config are found at runtime:

| Key | Value |
|---|---|
| `LD_LIBRARY_PATH` | `/opt/lib` |
| `ODBCSYSINI` | `/opt/odbcinst` |
| `ODBCINI` | `/opt/odbcinst/odbc.ini` |

Via CLI:

```bash
aws lambda update-function-configuration \
  --function-name your-function-name \
  --environment "Variables={LD_LIBRARY_PATH=/opt/lib,ODBCSYSINI=/opt/odbcinst,ODBCINI=/opt/odbcinst/odbc.ini}"
```

---

## 8. Use pyodbc in your function

```python
import pyodbc

def handler(event, context):
    conn = pyodbc.connect(
        "DRIVER={ODBC Driver 18 for SQL Server};"
        "SERVER=your-server.database.windows.net;"
        "DATABASE=your-db;"
        "UID=user;PWD=password;"
        "Encrypt=yes;TrustServerCertificate=no;"
    )
    cursor = conn.cursor()
    cursor.execute("SELECT 1")
    return {"status": "ok"}
```

---

## Notes

- Use `--entrypoint bash` when running the Lambda base image — it has a custom entrypoint that requires a handler name otherwise.
- Use `dnf` not `yum` — the Python 3.12 Lambda image is AL2023-based.
- Use the **RHEL 9** Microsoft repo URL — AL2023 is closer to RHEL 9 than RHEL 8.
- The final layer zip will be roughly **50–80 MB**, well within Lambda's 250 MB unzipped limit.
- The `libmsodbcsql-18.*.so.*` glob handles the exact version suffix automatically.

---

## Building for ARM64 from x86_64 (e.g., AWS CloudShell)

If you're building on an x86_64 machine (like AWS CloudShell) but need an ARM64 layer, use Docker's multi-arch support with the `--platform` flag:

### 1. Enable Docker buildx (if needed)

Most modern Docker installations have buildx enabled by default. Verify:

```bash
docker buildx version
```

### 2. Pull the ARM64 Lambda base image

```bash
docker pull --platform linux/arm64 public.ecr.aws/lambda/python:3.14-arm64
```

### 3. Create the layer directory

```bash
mkdir -p lambda-layer-arm64/python
```

### 4. Build the layer for ARM64

```bash
docker run --rm \
  --platform linux/arm64 \
  --entrypoint bash \
  -v "$PWD/lambda-layer-arm64":/var/task \
  public.ecr.aws/lambda/python:3.14-arm64 \
  -c "
    # Install unixODBC
    dnf install -y unixODBC unixODBC-devel

    # Add Microsoft repo and install ODBC 18 driver
    curl https://packages.microsoft.com/config/rhel/9/prod.repo > /etc/yum.repos.d/mssql-release.repo
    ACCEPT_EULA=Y dnf install -y msodbcsql18

    # Install pyodbc into the python/ directory
    pip install pyodbc -t /var/task/python

    # Copy shared libraries
    mkdir -p /var/task/lib
    cp /usr/lib64/libodbc.so.2          /var/task/lib/
    cp /usr/lib64/libodbcinst.so.2      /var/task/lib/
    cp /opt/microsoft/msodbcsql18/lib64/libmsodbcsql-18.*.so.* /var/task/lib/

    # Copy ODBC config files
    mkdir -p /var/task/odbcinst
    cp /opt/microsoft/msodbcsql18/etc/odbcinst.ini /var/task/odbcinst/
  "
```

> **Note:** Docker will automatically use QEMU emulation to run the ARM64 container on your x86_64 host. The build will be slower than native, but fully compatible.

### 5. Zip and publish the ARM64 layer

```bash
cd lambda-layer-arm64
zip -r ../pyodbc-layer-arm64.zip .
cd ..

aws lambda publish-layer-version \
  --layer-name pyodbc-layer-arm64 \
  --zip-file fileb://pyodbc-layer-arm64.zip \
  --compatible-runtimes python3.14 \
  --compatible-architectures arm64 \
  --region eu-west-1
```

### 6. Attach to your ARM64 Lambda function

```bash
aws lambda update-function-configuration \
  --function-name your-arm64-function \
  --layers arn:aws:lambda:eu-west-1:123456789012:layer:pyodbc-layer-arm64:1 \
  --architectures arm64
```

Then set the same environment variables as in step 7 above.

### Troubleshooting

- If `docker run --platform linux/arm64` fails with "exec format error", ensure Docker Desktop has multi-arch support enabled, or install `qemu-user-static`:
  ```bash
  docker run --rm --privileged multiarch/qemu-user-static --reset -p yes
  ```
- Verify the layer was built for the correct architecture by checking the shared libraries inside the zip:
  ```bash
  unzip -l pyodbc-layer-arm64.zip | grep libmsodbcsql
  file lambda-layer-arm64/lib/libmsodbcsql-18.*.so.*
  # Should output: "ELF 64-bit LSB shared object, ARM aarch64"
  ```
