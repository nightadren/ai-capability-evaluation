# Bug Bounty System Architecture

**Version:** 9.0  
**Status:** Design Complete | Implementation Forthcoming

## Overview

A 7-layer pipeline for autonomous vulnerability discovery, designed using the META-BLUEPRINT methodology and validated against commercial benchmarks.

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        AI BUG BOUNTY SYSTEM v9.0                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   Layer 0: TARGET SELECTION                                                 │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  Thompson Sampling + Expected Value Scoring                         │   │
│   │  • Program intelligence (scope, payouts, response time)             │   │
│   │  • Historical success rates per vulnerability class                 │   │
│   │  • Competition analysis (hunter density)                            │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                              │                                              │
│                              ▼                                              │
│   Layer 1: COORDINATOR                                                      │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  LangGraph StateGraph + PostgreSQL Checkpointing                    │   │
│   │  • Mission state management                                         │   │
│   │  • Solver orchestration                                             │   │
│   │  • Early-stopping enforcement                                       │   │
│   │  • Budget/threshold monitoring                                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                              │                                              │
│                              ▼                                              │
│   Layer 2: DISCOVERY                                                        │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  Attack Surface Enumeration                                         │   │
│   │  • Crawl4AI (authenticated crawling)                                │   │
│   │  • Subfinder (subdomain enumeration)                                │   │
│   │  • ffuf + Kiterunner (endpoint discovery)                           │   │
│   │  • Scope enforcement via sidecar proxy                              │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                              │                                              │
│                              ▼                                              │
│   Layer 3: DETECTION                                                        │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  Vulnerability Identification                                       │   │
│   │  • Nuclei (template-based, zero false positives)                    │   │
│   │  • SQLMap (SQL injection)                                           │   │
│   │  • Dalfox (XSS)                                                     │   │
│   │  • WAF fingerprinting for payload adaptation                        │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                              │                                              │
│                              ▼                                              │
│   Layer 4: SOLVERS                                                          │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  Specialist Exploitation Agents (Docker-isolated)                   │   │
│   │  • XSS Solver                                                       │   │
│   │  • SQLi Solver                                                      │   │
│   │  • SSRF Solver (priority: high payout, deterministic validation)   │   │
│   │  • Attack Chain Reasoning (precondition-postcondition matrix)       │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                              │                                              │
│                              ▼                                              │
│   Layer 5: VALIDATION                                                       │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  Deterministic Proof Generation                                     │   │
│   │  • InteractSH (OOB callbacks for blind vulns)                       │   │
│   │  • Playwright (browser execution confirmation)                      │   │
│   │  • Sheriff (response diffing)                                       │   │
│   │  • NO LLM judgment - scripts verify                                 │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                              │                                              │
│                              ▼                                              │
│   Layer 6: OUTPUT                                                           │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  Report Generation & Submission                                     │   │
│   │  • SARIF format (industry standard)                                 │   │
│   │  • CVSS scoring                                                     │   │
│   │  • Platform API integration (HackerOne, Bugcrowd)                   │   │
│   │  • Human approval gate before submission                            │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                              │                                              │
│                              ▼                                              │
│   Layer 7: LEARNING                                                         │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  Feedback Integration                                               │   │
│   │  • Thompson Sampling (fast loop - program selection)                │   │
│   │  • PPO (slow loop - solver strategy optimization)                   │   │
│   │  • Outcome tracking (accepted/rejected/duplicate)                   │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Critical Thresholds

These thresholds are derived from MAPTA research and enforce early stopping to avoid diminishing returns:

| Metric | Limit | Rationale | Source |
|--------|-------|-----------|--------|
| Tool calls | 40 max | r = –0.661 correlation with failure | MAPTA |
| Cost per target | $0.30 max | r = –0.606 correlation | MAPTA |
| Time per target | 300s max | r = –0.557 correlation | MAPTA |
| Iterations | ~80 max | Cognitive degradation threshold | XBOW |
| OOB Confidence | 0.95 | Required for auto-submission | Research |
| Mission Budget | $5.00 | Hard stop for runaway costs | Operations |
| Consecutive Failures | 5 max | Early stopping trigger | MAPTA |

## Design Principles

### 1. Coordinator-Solver Pattern
Proven by XBOW (the #1 ranked program on HackerOne). The Coordinator maintains global state and delegates to specialized Solvers. Solvers are ephemeral and domain-specific.

### 2. Deterministic Validation
LLMs cannot be trusted to verify vulnerabilities. All validation uses deterministic scripts:
- OOB callbacks (did the server make an outbound request?)
- Browser execution (did JavaScript execute?)
- Response diffing (did the response change predictably?)

### 3. Early Stopping
Extended effort signals struggle, not capability. The system enforces hard limits based on the MAPTA correlation research.

### 4. Attack Chain Reasoning
Valuable vulnerabilities chain together. The system models vulnerabilities as state transitions:
```
SSRF → cloud metadata access → credential extraction → privilege escalation
```

A precondition-postcondition matrix tracks what each vulnerability enables.

### 5. Zero False Positives
Platform credibility is non-negotiable. A single false positive damages reputation. Nuclei templates (verified, community-maintained) provide the baseline.

## Key Research Sources

| Source | Contribution |
|--------|--------------|
| **XBOW** | Coordinator-Solver pattern, iteration limits |
| **MAPTA** | Efficiency correlation metrics, cost thresholds |
| **Horizon3.ai** | Attack graph reasoning |
| **Anthropic** | Agent GPA framework alignment |

## Implementation Status

| Component | Status |
|-----------|--------|
| Architecture design | ✅ Complete |
| Research synthesis | ✅ Complete |
| Docker infrastructure | 🔄 Forthcoming |
| Coordinator implementation | 🔄 Forthcoming |
| Solver development | 🔄 Forthcoming |
| Validation layer | 🔄 Forthcoming |
| Benchmark suite | 🔄 Forthcoming |

---

*For the complete 1,400+ line specification, see the full blueprint document.*
