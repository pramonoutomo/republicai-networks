# Ubuntu 22.04 Compatible Node Installation Guide (GLIBC 2.38+ Fix)

## Problem
The current `republicd` binary requires GLIBC 2.38+, which is only available on Ubuntu 24.04. Many node operators still use Ubuntu 22.04 (GLIBC 2.35) and cannot run the binary without upgrading their OS.

## Solution
Using **patchelf** to make the binary use an isolated GLIBC 2.39 library without modifying the system GLIBC.
```
System GLIBC (untouched)         Isolated GLIBC 2.39
/lib/x86_64-linux-gnu/          /opt/glibc-2.39/lib/
├── libc.so.6 (2.35)            ├── libc.so.6 (2.39)
│                                │
│ ← All other apps use this      │ ← Only republicd uses this
```

### Benefits
- ✅ No system GLIBC upgrade required
- ✅ No risk to existing services/validators
- ✅ Works seamlessly with Cosmovisor
- ✅ Tested on production server with 7+ Cosmos validators

## Full Guide
📖 **[Republic AI Testnet Node Installation Guide - Ubuntu 22.04 Compatible](https://github.com/coinsspor/Republic-AI-Testnet-Node-Installation-Guide-Ubuntu-22.04-Compatible)**

## Public Endpoints by Coinsspor
- 🔗 RPC: https://rpc-republic-testnet.coinsspor.com
- 🔗 API: https://api-republic-testnet.coinsspor.com
- 🔗 EVM RPC: https://evm-rpc-republic-testnet.coinsspor.com
- 🔍 Explorer: https://explorer.coinsspor.com/republic-testnet

## Author
[@coinsspor](https://github.com/coinsspor)
