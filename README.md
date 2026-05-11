# canton-demo-etf-daml

Production-grade Daml smart contract layer for a Canton Network ETF lifecycle management platform. Demonstrates LDAP-style enterprise identity and access control expressed as first-class ledger objects, governing real ETF operations — collateral, cap tables, rebalancing, and market data.

Built on Canton DevNet with SV sponsorship from the Canton Foundation.

---

## The Core Thesis

LDAP was built for a world where one institution owns the directory. Tokenized assets blow that perimeter apart. When a fund manager, custodian, compliance officer, and market maker are peers on a shared network, no single party owns identity.

This platform solves that by making authorization a property of the asset itself — enforced at the contract layer, not the application layer.

- `RoleMembership` existing on the ledger **is** the role grant
- Archiving it **is** the revocation
- No database flags. No soft deletes. No admin who can quietly flip a row.

---

## Contract Inventory

### IAM Layer
| Contract | Purpose |
|---|---|
| `DirectoryEntry` | Maps an LDAP DN to a Canton party. Identity anchor for all downstream contracts. |
| `RoleMembership` | One contract per (party, role) pair. Archiving = revocation. |
| `AccessEvent` | Immutable append-only audit log. Written on every auth decision, granted or denied. |

### ETF Structure
| Contract | Purpose |
|---|---|
| `ETFDefinition` | Master ETF record. Created by FundManager, observable by all parties. |
| `Constituent` | Single basket holding with weight. Supports UpdateWeight for rebalancing. |
| `CapTable` | Share issuance and redemption ledger. Joint signatory between FundManager and Custodian. |

### Pricing
| Contract | Purpose |
|---|---|
| `NAV` | Immutable daily NAV record posted by FundManager. Observable by Auditor. |
| `NBBOQuote` | Oracle-posted NBBO from Polygon.io. Consumed by the FIX execution engine. |

### Collateral Management
| Contract | Purpose |
|---|---|
| `CollateralAccount` | Custodian-held account tracking available collateral balance. |
| `CollateralLock` | Encumbered collateral pending release or liquidation. |
| `CollateralRelease` | Immutable audit record of a successful lock release. |
| `HaircutSchedule` | Compliance-managed table of haircut rates per asset class. |
| `MarginCall` | Issued by Custodian when collateral coverage drops below threshold. |
| `LiquidationOrder` | Terminal action when margin call goes unmet. |

### Rebalancing
| Contract | Purpose |
|---|---|
| `RebalanceProposal` | FundManager proposes new constituent weights. Requires ComplianceOfficer approval. |
| `RebalanceExecution` | Immutable record of an executed rebalance. Created only from an Approved proposal. |

---

## Roles

| Role | Responsibilities |
|---|---|
| `FundManager` | Creates ETFs, posts NAV, proposes and executes rebalances |
| `Custodian` | Issues/redeems shares, manages collateral accounts, issues margin calls |
| `ComplianceOfficer` | Approves/rejects rebalances, suspends ETFs, manages haircut schedule |
| `Auditor` | Read-only observer across all contracts. Receives all AccessEvents. |
| `MarketMaker` | Posts NBBO quotes, submits FIX orders |

---

## Regulatory Coverage

| Contract | Regulation | Requirement Satisfied |
|---|---|---|
| `AccessEvent` | FINRA Rule 3110 | Supervision and immutable audit trail |
| `RebalanceProposal` | SEC Rule 38a-1 | Compliance approval workflow |
| `NAV` | SEC Rule 22c-1 | Daily NAV pricing requirement |
| `CollateralLock` / `HaircutSchedule` | Basel III | Collateral management and haircut enforcement |
| `NBBOQuote` / `ExecutionReport` | Reg SHO | Best execution compliance |
| `AccessEvent` / `RebalanceExecution` | SEC Rule 17a-4 | Immutable records retention |

---

## Prerequisites

```bash
# Install Daml SDK 3.4.11
curl -sSL https://get.daml.com/ | sh -s 3.4.11

# Verify
daml version

# Java 21 required for Daml script service
java -version
```

---

## Build

```bash
daml build
# Output: .daml/dist/canton-demo-etf-0.1.0.dar
```

---

## Test

```bash
# Run all 61 Daml Script tests
daml test --all --no-legacy-assistant-warning
```

Expected output: 61 tests passing, 16/16 templates created.

---

## Deploy to Canton DevNet

```bash
# Upload DAR to DevNet participant node
daml ledger upload-dar \
  --host <devnet-participant-host> \
  --port 6865 \
  .daml/dist/canton-demo-etf-0.1.0.dar

# Run a script against DevNet
daml script \
  --ledger-host <devnet-participant-host> \
  --ledger-port 6865 \
  --dar .daml/dist/canton-demo-etf-0.1.0.dar \
  --script-name Canton.ETF.Test.IAM.DirectoryEntryTest:testFullLifecycle
```

---

## Java Codegen

Generates typed Java binding classes for use in the Spring Boot API layer:

```bash
daml codegen java \
  --input-file .daml/dist/canton-demo-etf-0.1.0.dar \
  --output-directory ../canton-demo-etf-api/src/main/java/generated
```

---

## Project Structure

```
daml/
└── Canton/ETF/
    ├── IAM/
    │   ├── DirectoryEntry.daml
    │   ├── RoleMembership.daml
    │   └── AccessEvent.daml
    ├── Fund/
    │   ├── ETFDefinition.daml
    │   ├── Constituent.daml
    │   └── CapTable.daml
    ├── Pricing/
    │   ├── NAV.daml
    │   └── NBBOQuote.daml
    ├── Collateral/
    │   ├── CollateralAccount.daml
    │   ├── CollateralLock.daml
    │   ├── CollateralRelease.daml
    │   ├── HaircutSchedule.daml
    │   ├── MarginCall.daml
    │   └── LiquidationOrder.daml
    └── Rebalance/
        ├── RebalanceProposal.daml
        └── RebalanceExecution.daml
```

---

## Related Repositories

| Repo | Purpose |
|---|---|
| `canton-demo-etf-api` | Java Spring Boot REST API gateway |
| `canton-demo-etf-iam` | LDAP → Auth0 → Canton IAM bridge |
| `canton-demo-etf-fix` | QuickFIX/J execution engine + Polygon.io NBBO oracle |
| `canton-demo-etf-infra` | GCP GKE Terraform + Helm charts |
| `canton-demo-etf-ui` | React frontend with compliance panel |

---

## Built By

Justin Atwell — Director-level solutions engineering and developer relations leader with production DLT experience at Hedera Hashgraph (Avery Dennison atma.io — billions of mainnet transactions) and traditional capital markets experience at Edward Jones (FIX connectivity, NBBO, T+1 settlement).

[LinkedIn](https://www.linkedin.com/in/justin-atwell)