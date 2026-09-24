# INTERNSHIP WORK REPORT

## A Record of Projects, Technologies, and Learning

**Prepared by:** Kisha Agarwal  
**Department:** BFSI Innovation Centre  
**Duration:** 1st August – 30th September

---

## Introduction

This report documents the projects undertaken during the internship, including the technologies used, the architecture and workflow of each system, and the key concepts learned while implementing them. It is intended to serve as a running record that will be updated as additional projects are completed. Each project is documented in its own section below, following a consistent format: overview, technology stack, architecture, implementation details, and key learnings.

---

# Project 1: Image Matching System

## Overview

A Python-based system that checks whether a newly uploaded image matches any image in a pre-registered gallery. Azure AI Vision converts images (and text) into 1024-dim embedding vectors; Azure AI Search stores those vectors and performs nearest-neighbor similarity search. The frontend is a Streamlit app where a user uploads an image and instantly sees a MATCH / NOT MATCH result with a similarity score.

## Tech Stack

| Component | Technology |
|---|---|
| Frontend | Streamlit |
| Vectorization | Azure AI Vision (multimodal embeddings) |
| Vector Storage & Search | Azure AI Search |
| Language | Python |
| Hashing | hashlib (SHA-256) — change detection |
| HTTP Client | requests (Vision API has no SDK) |
| Azure SDK | azure-search-documents |

## Architecture & Workflow

- **Indexing (batch):** for each image in images/, hash it (SHA-256); if hash is unchanged → skip; if new/changed → get a vector from Azure Vision and upload {id, filename, file_hash, image_vector} to Azure AI Search.

- **Query (Streamlit UI):** user uploads an image → raw bytes sent straight to Vision API for a vector (no hashing at query time) → vector sent to Azure AI Search via VectorizedQuery(k=1) → returns nearest neighbor + cosine similarity score → score ≥ 0.80 = MATCH, else NOT MATCH.

- **Text-to-image search (utility):** text query → vectorized via retrieval:vectorizeText (same 1024-dim cross-modal space) → VectorizedQuery(k=5) → ranked list of matches.

- Similarity search runs server-side inside Azure AI Search (cosine similarity) — Python only sends the query vector.

## Key Learnings

- Multimodal embeddings: images and text mapped into the same vector space for cross-modal comparison.
- Vector databases and nearest-neighbor / cosine-similarity search vs. exact keyword matching.
- Hashing (exact-match, cheap, local) vs. embeddings (semantic similarity, costs an API call) — using both together to avoid redundant processing.
- End-to-end pipeline design: ingest → hash → vectorize → store → search → display.
- Using raw REST calls (requests) when no SDK exists, alongside an official SDK (azure-search-documents) where one does.

## Notes / Challenges

- 0.80 match threshold is hardcoded in app.py — may need tuning.
- No retry/backoff logic for transient Azure API failures.
- Filename-based document IDs could collide across subfolders.

---

# Project 2: RAG Document Assistant

## Overview

A Retrieval-Augmented Generation (RAG) web app that lets a user upload a PDF and ask questions about its contents. The system retrieves the most relevant sections of the document and uses Google Gemini to generate accurate, context-grounded answers, with support for multi-turn follow-up questions.

## Tech Stack

| Component | Technology |
|---|---|
| Frontend / UI | Streamlit |
| LLM | Google Gemini 3.6 Flash (gemini-3.6-flash) |
| Embeddings | Gemini Embedding 2 Preview (gemini-embedding-2-preview) |
| Orchestration | LangChain (v1.3.15) |
| PDF Parsing | PyPDF (PyPDFLoader) |
| Vector Store | FAISS (CPU, in-memory) |
| Text Splitting | RecursiveCharacterTextSplitter |
| Config | python-dotenv |

## Architecture & Workflow

- **Load & chunk:** PyPDFLoader extracts text page-by-page (with page metadata), then RecursiveCharacterTextSplitter splits it into 1000-char chunks with 200-char overlap to fit LLM context without losing boundary meaning.

- **Embed & store:** Gemini Embedding model vectorizes each chunk; FAISS.from_documents() builds an in-memory index for fast similarity search.

- **Retrieve:** a history-aware retriever rewrites follow-up questions into standalone queries, then fetches the top 8 most similar chunks from FAISS via cosine similarity.

- **Generate:** retrieved chunks + question + chat history go to Gemini 3.6 Flash (temperature=0) via a stuff-documents QA chain (no hallucination, page citations, concise answers); the Q&A pair is then appended to chat_history for multi-turn conversation.

## Key Learnings

- End-to-end RAG pipeline design: load → chunk → embed → store → retrieve → generate.
- Why chunking with overlap matters — balancing context-window limits against losing meaning at chunk boundaries.
- History-aware retrieval — rewriting ambiguous follow-ups into standalone queries so retrieval stays accurate across turns.
- Using LangChain to orchestrate multi-stage chains, and trade-offs in vector store choice (FAISS in-memory vs. managed stores like Pinecone/Chroma) for a simple, single-session use case.

## Notes / Challenges

- No persistence — in-memory FAISS means each session processes one document and a page reload resets everything.
- k=8 retrieval balances context recall against the LLM's context window (may need tuning for larger PDFs); prompts.py still holds unused reference templates not yet wired into the active chain.

---

# Project 3: Blockchain Payment PoC

## Overview

A proof-of-concept payment system built on a local Ethereum blockchain. Solidity smart contracts handle ETH deposits/payments/withdrawals and a custom ERC-20 token, while a Python + Streamlit UI lets a user trigger transactions and view on-chain transaction history in real time.

## Tech Stack

| Component | Technology |
|---|---|
| Smart Contracts | Solidity 0.8.34 |
| Dev Framework | Hardhat + Hardhat Ignition (deployment) |
| Local Blockchain | Hardhat Node (in-memory EVM, 20 funded test accounts) |
| Blockchain Client (Python) | web3.py (JSON-RPC over HTTP) |
| Frontend / UI | Streamlit |
| Token Standard | ERC-20 (custom Token.sol) |
| Language(s) | Solidity, Python, TypeScript (config/deploy scripts) |

## Architecture & Workflow

- **Compile:** npx hardhat compile turns Solidity (payment.sol, Token.sol) into ABI + bytecode artifacts — the ABI is the JSON blueprint Python needs since it can't read Solidity directly.

- **Deploy:** npx hardhat node starts a local EVM with 20 test accounts; Hardhat Ignition deploys Payment and Token contracts and saves their addresses to deployed_addresses.json.

- **Connect:** contract_utils.py wraps web3.py to connect to the node (HTTPProvider), load each contract's ABI, and build contract objects at their deployed addresses.

- **Transact:** a UI action (e.g. “Deposit ETH”) calls payment.functions.deposit().transact({from, value}) → web3.py signs it with that account's key → sent via eth_sendRawTransaction → Hardhat validates & the EVM executes the function, updating balances and emitting an event → mined into a block → receipt returned to Python → Streamlit reruns to show the new balance.

- **History:** reading past activity uses payment.events.Deposit.get_logs() (eth_getLogs) rather than re-scanning transactions manually, converting wei → ETH and block timestamps for display.

## Key Learnings

- How a dApp's stack fits together: Solidity contract → compiled ABI/bytecode → deployed address → called via a client library (web3.py) using that ABI.
- The role of a JSON-RPC node (Hardhat) as the single gateway between an app and the blockchain — every read/write is an RPC call (eth_sendRawTransaction, eth_getLogs, etc.).
- Transactions vs. calls: .transact() (signed, costs gas, changes state) vs. .call() (read-only, free, no signature).
- Events as the standard way to expose on-chain activity for an app to query and render, instead of parsing raw transaction data.
- Using Hardhat Ignition as a declarative, repeatable deployment script instead of manual deploy transactions.

## Notes / Challenges

- Entirely local/test setup — Hardhat's 20 accounts and their private keys are publicly known defaults, fine for a PoC but never for real funds.
- Contract addresses are network-specific (chain-31337 here) and deployed_addresses.json must stay in sync with app.py; no gas optimization or security audit yet (e.g. reentrancy checks on withdraw/pay) before any real deployment.

---

# Project 3 — Update: Live Testnet Deployment (Sepolia)

The PoC was extended from a local-only demo into a system that can run against a real public testnet (Sepolia), in addition to the local Hardhat node. A wallet was set up in MetaMask, Alchemy was used as the RPC provider to reach Sepolia, and a Sepolia faucet supplied test ETH to pay gas. The whole stack — contracts, deployment, Python backend, and the Streamlit UI — now switches behavior based on a single NETWORK setting in .env, instead of assuming a local chain.

## Before → Now

| Area | Before (Phase 1 — local only) | Now (Phase 2 — local + Sepolia) |
|---|---|---|
| Network | Only local Hardhat node | NETWORK=local or NETWORK=sepolia in .env |
| Wallet / signing | 20 auto-unlocked Hardhat test accounts | Real wallet via MetaMask; private key signs transactions locally in Python |
| RPC provider | Hardhat's own JSON-RPC (127.0.0.1:8545) | Alchemy Sepolia RPC URL (SEPOLIA_RPC_URL) |
| Test funds | 10,000 free ETH per account, instant | Real test ETH from a Sepolia faucet, limited amount |
| Deployed addresses | ignition/deployments/chain-31337/ | Also ignition/deployments/chain-11155111/ for Sepolia |
| Token constructor | ("PaymentToken", "PMT", 1000000) | ("Stable Coin", "STC", 1000000) |
| UI accounts | Sidebar dropdown of 20 named accounts | Local: same dropdown. Sepolia: single wallet, manual 0x address field |
| Verifying activity | Console / app only | Sepolia transactions also visible on sepolia.etherscan.io |

## New / Updated Components

- contract_utils.py gained network-aware helpers: get_network(), get_network_config(), get_account() / get_accounts(), and send_contract_tx() — the last one auto-picks node-signing (local) vs. build → sign with PRIVATE_KEY → send_raw_transaction (Sepolia).

- hardhat.config.ts now defines a sepolia network block (chainId 11155111) pulling SEPOLIA_RPC_URL and PRIVATE_KEY from .env via configVariable().

- app.py shows the active network in the title, gives network-specific connection error hints, and adapts the account picker and receiver input per network.

- New python/multinode_demo.py — a decentralization check that reads the same deployed contract's state (chain ID, block number, contract balance, token supply) from several independent public Sepolia RPC endpoints (PublicNode, Ankr, 1RPC, Llamarpc, plus Alchemy) and confirms they all agree.

## Key Learnings

- MetaMask + a faucet + Alchemy is the standard minimal setup to move from a simulated chain to a real public testnet: wallet for identity/signing, faucet for gas money, RPC provider for connectivity.
- The core signing difference between environments: on a local dev node the node itself can sign for its own unlocked test accounts, but on any real network the client (Python here) must sign with a private key before sending the raw transaction.
- Why deployed contract addresses are chain-specific — the same source code produces a different, independent contract instance on each network, so addresses (and ABIs read from them) must be looked up per chain.
- What “decentralized” concretely means at the RPC level: multiple independent nodes run by different providers all agreeing on the same chain state, which only a real shared network (not a single local node) can demonstrate.

## Notes / Challenges

- Sepolia test ETH from faucets is rate-limited, so testing has to be more deliberate than on the local network's unlimited funds.
- Secrets management matters more now — PRIVATE_KEY and SEPOLIA_RPC_URL live only in the git-ignored .env; committing them would expose a real (if test) wallet.
- Sepolia transactions take real block time to confirm, unlike the local node's near-instant mining, so the UI's wait-for-receipt step feels noticeably slower.
