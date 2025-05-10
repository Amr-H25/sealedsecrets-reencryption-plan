# Sealed Secrets Re-encryption Plan - Infrastructure Internship Task 2025

This repository contains the documentation for a plan to re-encrypt existing Sealed Secrets in a Kubernetes cluster using the latest public key of the Sealed Secrets controller. This work was completed as part of the Infrastructure Internship Task 2025 at Instabug.

## Overview

The document outlines a strategy to:

* Identify all Sealed Secrets within a Kubernetes cluster.
* Fetch the active public key(s) of the Sealed Secrets controller.
* Decrypt existing Sealed Secrets using the current private key.
* Re-encrypt these secrets using the fetched public key(s).
* Update the SealedSecret objects in the cluster with the re-encrypted data.

The plan also considers the following bonus requirements:

* **Logging:** A logging mechanism is included to track the re-encryption process and any potential errors.
* **Scalability:** Considerations for efficiently handling a large number of Sealed Secrets are discussed.
* **Security:** The plan emphasizes maintaining the security of the private keys throughout the re-encryption process.

## Contents

* `sealedsecrets_reencryption_plan.md`: This file contains the detailed plan of action in MarkDown format, addressing all the core and bonus requirements outlined in the internship task.
