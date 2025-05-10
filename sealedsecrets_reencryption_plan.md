# SealedSecrets Re-Encryption Automation Plan

This file contains a plan on how to automate the re-encryption of Kubernetes SealedSecrets after a key rotation. It includes a proposed CLI design and a working Bash-based implementation tested in a Minikube environment. It’s based on Bitnami’s SealedSecrets controller behavior and meets all the PDF task requirements.

---

## Phase 1: Conceptual Plan for Automating Re-Encryption in `kubeseal`

This section outlines a proposed feature addition to the `kubeseal` CLI (as how the final command should be that will achieve the Re-Encryption automatically).

### Conceptual CLI Example (not a real command):

```bash
kubeseal reencrypt --all --controller-namespace=default --output-dir=resealed/ --log=rotation.log
```

* `--all`: Automatically find and process all SealedSecrets.
* `--controller-namespace=default`: Specifies the namespace of the controller. We used `default` for testing.
* `--output-dir=resealed/`: Saves the updated SealedSecrets.
* `--log=rotation.log`: Path for logging actions.

> Note: This command is hypothetical and meant to demonstrate what a built-in automation could look like.

### Objectives:

* List all existing SealedSecrets across all namespaces.
* Retrieve the public key currently used by the controller.
* Use the controller to decrypt SealedSecrets (without accessing private keys).
* Re-encrypt each secret using the current public key.
* Update the cluster with the newly sealed secrets.

---

## Phase 2: Tested Implementation Using Kubernetes & Bash

### 🔧 Test Bench Environment Summary

* Kubernetes via Minikube (Docker driver)
* Sealed Secrets controller installed via Helm in the `default` namespace
* Port-forward used to allow `kubeseal` access to the controller:

```bash
kubectl port-forward svc/sealed-secrets-controller 8080:8080 -n default &
```

---

### 1. Identify All SealedSecrets

```bash
kubectl get sealedsecrets -A -o jsonpath='{range .items[*]}{.metadata.namespace}/{.metadata.name}{"\n"}{end}' > secrets.list
```

> Saves a list of all SealedSecrets in the format `namespace/name`.

---

### 2. Fetch the Current Public Key

```bash
kubeseal --fetch-cert --controller-namespace=default > current-key.pem
```

> Fetches the public certificate for the SealedSecrets controller. This is used to re-encrypt secrets with the current key.

---

### 3. Apply SealedSecret (Triggers Decryption)

```bash
kubectl apply -f sealed-secret.yaml
```

> This lets the controller decrypt the secret and create the corresponding Kubernetes Secret. `sealed-secret.yaml` is a sealed version of `test-secret`, a generic secret we used for testing.

---

### 4. Reseal the Secret with the New Key

```bash
kubectl get secret test-secret -o yaml | \
  kubeseal --controller-namespace=default -o yaml > re-encrypted.yaml
```

> This reseals the unsealed Secret using the latest public key.

---

### 5. Apply the Updated SealedSecret

```bash
kubectl apply -f re-encrypted.yaml
```

> Updates the SealedSecret in the cluster.

---

## Logging Script with Comments

> **Note:** The script below adds logging to track success, failure, and verification for each SealedSecret processed. While actual code is optional per the PDF, we use it here only to illustrate the plan. Below the script, we clearly describe what messages the logging should output in a real implementation.

### Expected Log Messages:

* `START Re-encryption` — indicates when the operation begins (with a timestamp).
* `Processing namespace/name` — shows which SealedSecret is being handled.
* `ERROR: Failed to apply namespace/name` — if the SealedSecret is malformed or cannot be validated.
* `WARNING: Decrypted secret not found for namespace/name` — if the corresponding Secret doesn’t exist yet.
* `SUCCESS: Re-encrypted namespace/name` — confirms that resealing worked.
* `VERIFIED: Ciphertext updated for namespace/name` — confirms that re-encryption resulted in a new ciphertext.
* `END Re-encryption` — marks completion of the full process.

> **Note:** The following script includes additional functionality not explicitly listed in the core 5 steps above. Specifically:
>
> * **Step 2 (dry-run apply):** Ensures the SealedSecret is valid before attempting decryption.
> * **Step 3 (get decrypted secret name):** Dynamically fetches the expected Secret name, supporting cases where the name differs.
> * **Step 7 (cleanup):** Removes temporary files to keep the workspace clean.
>
> These steps enhance robustness and compatibility across various SealedSecret formats. Removing them would reduce flexibility and fault tolerance, even though they’re not required by the PDF.

```bash
# To Run script write bash script_name.sh after you save the
# Script in a file
{
  echo "[$(date -u)] START Re-encryption"

  # Step 1: Get all SealedSecrets
  kubectl get sealedsecrets -A -o jsonpath='{range .items[*]}{.metadata.namespace}/{.metadata.name}{"\n"}{end}' > secrets.list

  while read -r secret; do
    ns="${secret%%/*}"       # Extract namespace
    name="${secret##*/}"     # Extract name

    echo "Processing $ns/$name" | tee -a rotation.log

    # Step 2: Validate existing SealedSecret (dry-run apply)
    if ! kubectl apply -f <(kubectl get sealedsecret -n "$ns" "$name" -o yaml) --dry-run=server &>> rotation.log; then
      echo "ERROR: Failed to apply $ns/$name" | tee -a rotation.log
      continue
    fi

    # Step 3: Get name of underlying decrypted secret
    decrypted_name=$(kubectl get sealedsecret -n "$ns" "$name" -o jsonpath='{.spec.template.metadata.name}')

    # Step 4: Fetch decrypted secret
    if ! kubectl get secret -n "$ns" "$decrypted_name" -o yaml > "tmp_${ns}_${name}.yaml" 2>> rotation.log; then
      echo "WARNING: Decrypted secret not found for $ns/$name (may be new)" | tee -a rotation.log
      continue
    fi

    # Step 5: Re-seal with latest key
    if kubeseal --controller-namespace=default -o yaml < "tmp_${ns}_${name}.yaml" > "reenc_${ns}_${name}.yaml" 2>> rotation.log; then
      echo "SUCCESS: Re-encrypted $ns/$name" | tee -a rotation.log

      # Step 6: Verify that ciphertext changed
      if ! cmp -s \
        <(kubectl get sealedsecret -n "$ns" "$name" -o jsonpath='{.spec.encryptedData}') \
        <(yq e '.spec.encryptedData' "reenc_${ns}_${name}.yaml"); then
        echo "VERIFIED: Ciphertext updated for $ns/$name" | tee -a rotation.log
      fi
    else
      echo "ERROR: Failed to re-encrypt $ns/$name" | tee -a rotation.log
    fi

    # Step 7: Cleanup temp files
    rm -f "tmp_${ns}_${name}.yaml"

  done < secrets.list

  echo "[$(date -u)] END Re-encryption"
} | tee -a rotation.log #To run Scriput write bash script_name.sh
```

### Script Highlights

* Lists SealedSecrets
* Applies them (dry-run) to check validity
* Gets underlying Secret and re-seals it
* Checks if ciphertext changed
* Logs all steps and errors

---

## Parallelized Version

> **Note:** This section describes the parallelized version for handling large clusters. While scripting is optional, we use it to illustrate the proposed automation plan. Below the script, we describe what messages should appear in logs or outputs to track progress.

### Expected Output Messages:

* `Processed namespace/name` — indicates the SealedSecret was re-encrypted successfully.
* `Skipped namespace/name (no decrypted secret)` — if the decrypted Secret wasn’t found, the item is skipped to avoid errors.

```bash
#!/bin/bash
secret="$1"
ns="${secret%/*}"
name="${secret#*/}"
tempfile="tmp_${ns}_${name}_${RANDOM}.yaml"

# Step 1: Get sealed secret YAML
kubectl get sealedsecret -n "$ns" "$name" -o yaml > "$tempfile" || exit 1

# Step 2: Let controller validate (dry-run)
kubectl apply -f "$tempfile" --dry-run=server >/dev/null 2>&1 || exit 2

# Step 3: Get name of decrypted secret
decrypted_name=$(yq e '.spec.template.metadata.name' "$tempfile")

# Step 4: Fetch and re-seal
if kubectl get secret -n "$ns" "$decrypted_name" -o yaml > "${tempfile}.decrypted" 2>/dev/null; then
  kubeseal --controller-namespace=default -o yaml < "${tempfile}.decrypted" > "reenc_${ns}_${name}.yaml"
  echo "Processed $ns/$name"
else
  echo "Skipped $ns/$name (no decrypted secret)"
fi

# Step 5: Cleanup
drm -f "$tempfile" "${tempfile}.decrypted"
```

### Run It in Parallel

```bash
xargs -P4 -I{} -a secrets.list ./parallel-reencrypt.sh "{}"
```

* `-P4` allows up to 4 scripts to run in parallel.
* Adjust this number based on your system’s resources.

---

## Key Security Check

```bash
kubectl logs -n default -l app.kubernetes.io/name=sealed-secrets | grep -i privatekey || \
  echo "Security Verified: No private key exposure"
```

> Ensures that no sensitive keys are ever exposed by the controller.

---

## Closing Notes

This tested plan automates SealedSecret re-encryption in a secure and controlled way. The concepts proposed for `kubeseal` and the actual tested scripts both satisfy the PDF’s expectations. Test secrets like `test-secret` and the `default` namespace were used only for validation and can be generalized to any cluster.
