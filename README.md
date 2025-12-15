# Databricks Unity Catalog to AWS Glue Federation

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

Query your Databricks Unity Catalog tables from Amazon Athena, EMR, Redshift Spectrum, and other AWS services.

---

## Background

At re:Invent 2024, AWS announced federated catalogs for Glue, allowing it to connect to external catalog sources like Databricks Unity Catalog via the Iceberg REST API.

The problem? Getting it working requires:

- Creating a Secrets Manager secret with a specific format (`USER_MANAGED_CLIENT_APPLICATION_CLIENT_SECRET`)
- Setting up an IAM role with the right trust policy
- Attaching a `secrets-access` policy to the role (this one isn't obvious)
- Registering the Glue connection with Lake Formation using `--with-federation`
- Creating the federated catalog with the correct configuration

I spent a few hours debugging all of this. This script automates the whole thing.

---

## How It Works

```
┌─────────────────────────┐         ┌─────────────────────────┐
│  Databricks             │         │  AWS                    │
│  Unity Catalog          │         │                         │
│                         │         │  ┌─────────────────┐    │
│  ┌─────────────────┐    │  ────>  │  │  Glue Catalog   │    │
│  │  my_catalog     │    │ Iceberg │  │  (Federated)    │    │
│  │   └── default   │    │  REST   │  └────────┬────────┘    │
│  │        └── tbl  │    │   API   │           │             │
│  └─────────────────┘    │         │           v             │
│                         │         │  ┌─────────────────┐    │
└─────────────────────────┘         │  │ Athena / EMR /  │    │
                                    │  │ Redshift / etc  │    │
                                    │  └─────────────────┘    │
                                    └─────────────────────────┘
```

After running this script, you can query your Databricks tables directly from Athena:

```sql
SELECT * FROM "my-federated-catalog"."default".my_table LIMIT 10;
```

---

## Quick Start

```bash
git clone https://github.com/YOURUSERNAME/databricks-glue-federation.git
cd databricks-glue-federation
./setup.sh
```

The script will prompt you for everything it needs.

---

## Prerequisites

- AWS CLI installed and configured (`aws configure`)
- A Databricks Service Principal with OAuth credentials
- Access to the Unity Catalog you want to federate

### What you'll need to provide:

| Value | Where to find it |
|-------|------------------|
| Workspace URL | Browser address bar when logged into Databricks |
| Catalog Name | Databricks Catalog Explorer |
| OAuth Client ID | Account Console > Service Principals |
| OAuth Client Secret | Created when setting up the Service Principal |
| S3 Bucket | Your catalog's storage location |

---

## Usage

### Interactive mode

```bash
./setup.sh
```

### With a config file

```bash
cp examples/config.example.env .env
# edit .env with your values
./setup.sh --config .env
```

### Dry run

```bash
./setup.sh --dry-run
```

---

## What Gets Created

| Resource | Name | Purpose |
|----------|------|---------|
| Secrets Manager | `{prefix}-oauth-secret` | Stores OAuth credentials |
| IAM Role | `{prefix}-glue-role` | Allows Glue to access S3 and secrets |
| Glue Connection | `{prefix}-connection` | Connects to Databricks Iceberg REST API |
| Glue Catalog | `{prefix}-catalog` | Federated catalog visible in AWS |

---

## Cleanup

```bash
./cleanup.sh --prefix your-prefix
```

---

## Troubleshooting

### "Access Denied for the given secret ID"

The IAM role needs an explicit policy to access the secret. The script handles this, but if you're doing manual setup:

```json
{
  "Effect": "Allow",
  "Action": ["secretsmanager:GetSecretValue", "secretsmanager:DescribeSecret", "secretsmanager:PutSecretValue"],
  "Resource": ["arn:aws:secretsmanager:REGION:ACCOUNT:secret:YOUR-SECRET-*"]
}
```

### "The data source is not registered"

The Glue connection must be registered with Lake Formation before creating the catalog:

```bash
aws lakeformation register-resource \
  --resource-arn "arn:aws:glue:REGION:ACCOUNT:connection/CONNECTION" \
  --role-arn "arn:aws:iam::ACCOUNT:role/ROLE" \
  --with-federation
```

### Athena returns "no accessible columns"

Grant Lake Formation permissions:

```bash
aws lakeformation grant-permissions \
  --principal '{"DataLakePrincipalIdentifier": "IAM_ALLOWED_PRINCIPALS"}' \
  --resource '{"Table": {"CatalogId": "ACCOUNT:CATALOG", "DatabaseName": "default", "TableWildcard": {}}}' \
  --permissions "ALL"
```

---

## FAQ

**Does this work with Delta tables?**

Yes, as long as UniForm (Iceberg) is enabled.

**Can I federate multiple catalogs?**

Yes. Run the script multiple times with different prefixes.

**What permissions does my Service Principal need?**

`USE CATALOG` and `SELECT` on the tables you want to query.

---

## License

MIT
