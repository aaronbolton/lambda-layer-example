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
