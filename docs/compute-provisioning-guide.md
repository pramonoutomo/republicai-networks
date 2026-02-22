# Republic Protocol: Compute Provisioning Guide

Welcome to the comprehensive guide on providing compute power to the Republic Protocol. This document explains the architecture, concepts, and operational steps required to function as a **Compute Provider** and a **Committee Verifier** for the Republic network.


## 1. Core Concepts

### What is Republic?
Republic is a decentralized compute network that allows users to request computational effort (AI inference, data processing, cryptographic operations, etc.) to be performed off-chain while maintaining on-chain security and settlement. 

Traditional smart contracts are execution-limited: every node in the network must run the same code for consensus. This makes AI model inference or complex physics simulations impossible on-chain. Republic solves this by moving the heavy lifting to specialized nodes (compute providers) and using the blockchain as a coordination and verification layer.

### The Problem: Trusting Off-Chain Work
Since computation happens in a black-box environment on a miner's local server, the blockchain cannot natively verify if the result is correct. We solve this using **Optimistic Execution with Committee Verification**:
1. **Selection**: Every job is assigned to a particular validator, who has a trusted benchmark score and a reputation.
2. **Commitment**: The miner broadcasts their result hash to the network.
3. **Verification**: A randomly selected Committee of other validators downloads the result and runs a lightweight verification script.
4. **Consensus**: If the committee reaches a majority quorum (consensus), the result is finalized and the state is updated. 

This model ensures that even if a miner tries to cheat by submitting fake data, the committee will detect the mismatch, reject the work, and ensure the requester is not charged.

## 2. System Roles

The protocol defines three primary actors for every compute job, each with distinct incentives and responsibilities.

### Job Requester
The entity that needs computation.
- **Responsibilities**: Submits job metadata, Docker images (execution and optional verification), and pays the job fee in the native protocol token.
- **Incentive**: Access to scalable, verifiable compute power without managing infrastructure.

### Miner (Target Validator)
The "worker" node assigned to a specific job.
- **Responsibilities**: Monitors the chain for assigned jobs, pulls Docker images, executes the code, generates a `result.bin` file, uploads it to the provided storage endpoint, and submits a completion transaction.
- **Incentive**: Receives 100% of the Job Fee (minus protocol burns) upon successful committee validation.

### Committee Member
A validator selected to check the Miner's work.
- **Responsibilities**: Downloads the miner's result, verifies the SHA-256 hash, runs the verification Docker image, and submits a `true`/`false` vote to the network.
- **Incentive**: Earns a portion of the block rewards and avoids slashing. Failure to participate in committee duties when selected can lead to reputation decay or stake slashing.

## 3. The Compute Job Lifecycle

The lifecycle of a job is a carefully orchestrated dance between on-chain state transitions and off-chain execution.

### Phase 1: Submission (On-Chain)
A requester broadcasts a `MsgSubmitJob`. The blockchain validates that the requester has sufficient funds for the fee and that the targeted miner is active.
- **Job ID**: A unique incremental identifier assigned by the module.
- **Fee Escrow**: The job fee is locked in a module-controlled account, ensuring the miner is guaranteed payment if they succeed.

### Phase 2: Discovery & Execution (Off-Chain)
The Miner's `job-sidecar` daemon polls the blockchain (via RCP). 
1. **Trigger**: The sidecar identifies a job in `PendingExecution` status where the `TargetValidator` matches the miner's address.
2. **Environment**: The sidecar pulls the execution container and maps a high-speed local NVMe path to the container's `/output` directory.
3. **Execution**: The container runs. It must produce a binary output at `/output/result.bin`. The system is agnostic to the internal logic (Python, C++, Rust), as long as it respects this output contract.

### Phase 3: Reporting & Commitment (Off-Chain/On-Chain)
1. **Upload**: The sidecar streams `result.bin` to the `ResultUploadEndpoint` (e.g., an S3 bucket or a peer-to-peer file server).
2. **Hashing**: The sidecar computes a SHA-256 hash of the result. This hash is the "commitment" to the work.
3. **Transition**: The miner submits `MsgSubmitJobResult`. On-chain, the job status moves from `PendingExecution` to `PendingValidation`.

### Phase 4: Verification (Off-Chain)
Validators selected for the committee detect the state change.
1. **Fetching**: Each member downloads the `result.bin` from the `ResultFetchEndpoint`.
2. **Integrity Check**: They verify the downloaded file's hash matches the miner's reported hash. If it doesn't, they immediately vote `false`.
3. **Verification Container**: If hashes match, they run the `VerificationImage`. The downloaded file is mounted at `/input` (read-only).
4. **The "True" Signal**: The verifier sidecar captures the container's stdout. If the container emits the exact string `True`, the sidecar submits a `MsgJobValidation` with a `true` payload.

### Phase 5: Settlement & Finalization (On-Chain)
The module tracks votes. Once the number of votes exceeds the `CommitteeSize` quorum (usually >50% of selected members):
- **Succeeded**: If `true` votes win, the fee is released to the miner. 
- **Failed**: If `false` votes win (or the miner fails to submit in time), the fee is refunded to the requester.
- **Cleanup**: The job is archived, and the committee members are released from duty.

## 4. Miner Operational Guide

Operating as a miner requires a robust, performance-oriented stack.

### Hardware & Software Examples
Republic Protocol is highly scalable, but the following configurations are examples of professional miner setups:

**Standard Miner (Consumer Grade):**
- **CPU**: AMD Ryzen 9 or Intel i9 (12+ Cores)
- **GPU**: NVIDIA RTX 4090 (24GB VRAM)
- **RAM**: 64GB DDR5
- **Storage**: 1TB NVMe Gen4 SSD

**Enterprise Miner (Datacenter Grade):**
- **CPU**: AMD EPYC or Intel Xeon (32+ Cores)
- **GPU**: 1-4x NVIDIA H100 or A100 (80GB VRAM)
- **RAM**: 256GB+ ECC RAM
- **Storage**: 4TB+ NVMe Enterprise RAID

**Software Requirements:**
- **OS**: Ubuntu 22.04 LTS or higher (Recommended).
- **Docker Engine**: Version 20.10+ is required. The sidecar uses the Docker Socket (`/var/run/docker.sock`) to manage containers.
- **NVIDIA GPU & CUDA**: To accept AI/ML jobs with CUDA, you must have an NVIDIA GPU with CUDA 11.8+ drivers. Ensure `nvidia-container-toolkit` is installed so Docker can access the GPU.
- **Republic Core Utils**: This Python library handles the math and formatting for capacity benchmarks.
  ```bash
  pip install republic-core-utils
  ```
- **Network**: 1Gbps symmetrical connection is ideal. Slow uploads will delay validation and may lead to missed deadlines.

We are currently working to make CUDA setup easier for Republic validators.

### Binary Installation
You only need the `republicd` binary. You can download the latest release from our official channels or build it from the repository if you are an advanced user.

```bash
# Check your installation
republicd version --long
```

## 5. Network & RPC Configuration

As a miner, you must stay synced with the network state. While you can run a local full node, most miners connect to a high-availability **RPC Endpoint**.

**Official Public RPC: `rpc.republicai.io`**

### Security Note
Always use HTTPS when connecting to the RPC.

### Example: Manual Interaction
You can query the current list of jobs directly:
```bash
republicd query computevalidation list-job --node https://rpc.republicai.io
```

If you need to manually submit a result (e.g., if a sidecar crash occurs):
```bash
republicd tx computevalidation submit-job-result \
  [job-id] \
  "https://storage.provider.com/result_123.bin" \
  "docker.io/my-verification-image:v1" \
  "HASH_OF_FILE" \
  --from <my-key> \
  --node https://rpc.republicai.io \
  --chain-id republic-1 \
  --gas-prices 0.1stake
```

## 6. Running the Sidecar Daemon

The `job-sidecar` is the heart of your operation. It automates monitoring, execution, and voting.

### Production Execution
Run the sidecar as a background service (e.g., using `systemd` or `pm2`).

```bash
republicd tx computevalidation job-sidecar \
  --from <your_validator_key> \
  --work-dir /var/lib/republic/jobs \
  --poll-interval 10s \
  --node https://rpc.republicai.io \
  --chain-id republic-1 \
  --gas auto \
  --gas-adjustment 1.5
```

### Critical Flags
- `--work-dir`: This is where containers mount results. If this path is slow (e.g., a HDD), your throughput will suffer.
- `--poll-interval`: Setting this too high (e.g., >60s) might cause you to miss jobs in a high-traffic environment.
- `--node`: Ensure this points to a low-latency RPC.

## 7. File Server Protocol & Storage

Republic Protocol does not store large result files (BLOBs) directly on the blockchain because it would lead to unsustainable state bloat. Instead, we use "Off-Chain Storage / On-Chain Commitment."

### The Protocol Interface
Any storage provider can be used as long as it supports these two operations:
1. **Upload**: A standard `POST` request where the body is the raw binary data. 
   - URL: `{ResultUploadEndpoint}/{job_id}`
2. **Fetch**: A standard `GET` request that returns the binary data.
   - URL: `{ResultFetchEndpoint}/{job_id}`

### Recommended Providers
- **S3 / Cloudfront**: Best for high-reliability commercial miners.
- **IPFS / Filecoin**: Best for decentralization purists.
- **Private File Server**: If you are a vertically integrated miner/requester.

## 8. Capacity Benchmarking

The network uses capacity benchmarks to rank miners. A higher benchmark score increases your probability of being assigned premium "Targeted" jobs with higher fees.

### How it Works
1. **Random Seeding**: When a benchmark starts, the network generates a `seed`. This seed is combined with your validator address to create a unique task that cannot be shared or pre-computed.
2. **Standardized Tasks**:
   - **Inference**: Measures how many tokens/frames a miner can process per second.
   - **Throughput**: Measures raw floating-point operations (FLOPS).
3. **Running the Bench**:
   ```bash
   republicd tx computevalidation run-benchmark \
     <benchmark-id> \
     <seed> \
     '{"n": 1024, "precision": "fp16"}' \
     inference
   ```
4. **Verification**: Other validators verify your benchmark result using the same committee rules as a standard job.

## 9. Troubleshooting & Best Practices

### "Image not found"
- **Cause**: The `execution_image` is in a private registry or hasn't been pushed yet.
- **Fix**: The miner sidecar will retry for a few intervals. Ensure you can manually `docker pull` the image on the server.

### "Verification Mismatch (Hash Error)"
- **Cause**: The file was corrupted during upload or the miner's hashing algorithm is incorrect.
- **Fix**: Check `work-dir` logs. The sidecar logs the hash it computed vs what it saw on-chain.

### "Non-Deterministic Verification"
- **Cause**: Your verification image uses `time.now()`, random seeds, or network calls.
- **Fix**: **IMPORTANT**. Verification images MUST be deterministic. If two validators run the same verification on the same `result.bin`, they must get the exact same answer. If your container logic varies, you will be rejected by the committee.

### "Out of Gas"
- **Cause**: Your transactions are failing because the network is busy and your gas settings are too low.
- **Fix**: Always use `--gas auto --gas-adjustment 1.5` in your sidecar flags.

## 10. Security Best Practices

Securing your miner is essential to protecting your validator stake and providing reliable compute.

### Containerization
As a best practice, you should run the `republicd` sidecar and all associated processes within a **containerized environment**. This provides several benefits:
- **Isolation**: Prevents compute jobs from accessing the host file system or network beyond specified volumes.
- **Reproducibility**: Ensures your environment matches the expected configuration regardless of the host OS.
- **Resource Control**: Allows you to set hard limits on CPU and Memory usage to prevent system crashes.

### Upcoming Security Updates
We are committed to the security of the Republic network. **Significant security updates are currently in development** and will be released in the coming cycles. These updates will focus on:
- Improved sandboxing for execution containers.
- Enhanced authentication for storage endpoints.
- Hardened P2P communication protocols.
- Native TEE (Trusted Execution Environment) support.

Stay tuned for official announcements and move to the latest binary versions immediately upon release.