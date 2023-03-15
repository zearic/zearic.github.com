---
title: bash script to convert Openshift Secrets to hashicorp Vault
tags: [openshift, hashicorp vault]
---

Create a bash script to convert Openshift Secrets `my-secrets` to Hashicorp Vault, with the following requirements:
- Use a temp file to save all the secrets before putting them into Vault.
- Add a flag to not run the Vault command.

Here's an example bash script that can help you achieve the conversion of Openshift secrets to Hashicorp Vault. It uses a temp file to store the secrets before putting them into Vault, and has a flag to not run the Vault command if desired. Please note that this is just an example and you may need to modify it to suit your specific needs.

```bash
#!/bin/bash

# Set the name of the secrets to be converted
SECRETS_NAME=my-secrets

# Set the path to the temp file where the secrets will be stored
TEMP_FILE=/tmp/secrets.txt

# Set the path to the Vault server and token
VAULT_SERVER=http://localhost:8200
VAULT_TOKEN=mytoken

# Parse command-line arguments
RUN_VAULT_COMMAND=true
while getopts ":n:t:f:h" opt; do
  case ${opt} in
    n ) SECRETS_NAME=$OPTARG;;
    t ) VAULT_TOKEN=$OPTARG;;
    f ) TEMP_FILE=$OPTARG;;
    h ) echo "Usage: $0 [-n SECRETS_NAME] [-t VAULT_TOKEN] [-f TEMP_FILE] [-h] [--no-vault]
  -n SECRETS_NAME: The name of the secrets to be converted (default: my-secrets)
  -t VAULT_TOKEN: The token to use to access the Vault server (default: mytoken)
  -f TEMP_FILE: The path to the temp file where the secrets will be stored (default: /tmp/secrets.txt)
  -h: Show this help message
  --no-vault: Do not run the Vault command"
        exit 0;;
    - ) case "${OPTARG}" in
          no-vault) RUN_VAULT_COMMAND=false;;
          *) echo "Invalid argument: -${OPTARG}" >&2; exit 1;;
        esac;;
    \? ) echo "Invalid option: -$OPTARG" >&2; exit 1;;
  esac
done

# Get the secrets from Openshift and store them in the temp file
oc get secret $SECRETS_NAME -o json | jq -r '.data[] | @base64d' > $TEMP_FILE

# If the --no-vault flag was provided, exit without running the Vault command
if [ "$RUN_VAULT_COMMAND" = false ]; then
  echo "Secrets stored in $TEMP_FILE"
  exit 0
fi

# Loop through the secrets in the temp file and store them in Vault
while read line; do
  echo $line | vault kv put secret/$SECRETS_NAME -
done < $TEMP_FILE

# Clean up the temp file
rm $TEMP_FILE

echo "Secrets converted and stored in Vault"
```

To use this script, save it as a bash file (e.g., `convert-secrets.sh`), make it executable (`chmod +x convert-secrets.sh`), and run it with the desired flags (e.g., `./convert-secrets.sh -n my-secrets -t mytoken -f /tmp/secrets.txt --no-vault`).
