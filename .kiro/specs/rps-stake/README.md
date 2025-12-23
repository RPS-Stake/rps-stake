# RPS-Stake Specification

## Overview

RPS-Stake is a skill-based Rock Paper Scissors betting platform with cryptocurrency integration and age verification. This project represents a fork of the existing RPS-Onchain game, adding monetization through staking mechanics and premium gameplay features.

## Project Structure

- **[Requirements](./requirements.md)** - Detailed functional and non-functional requirements
- **[Design](./design.md)** - System architecture, database schema, and technical design
- **[Tasks](./tasks.md)** - Phase-by-phase task breakdown and development timeline

## Quick Links

### Phase 1: Core Betting Platform (Crypto-Only)

- AI opponent gameplay with 75% house edge
- Multi-chain cryptocurrency payments (Base, Celo, Ethereum, Optimism, Arbitrum)
- Self Protocol age verification (18+)
- RPS-coin economy ($1 = 10 coins)
- Daily limits (50 coins wagered, 10 matches max)

### Future Phases

- **Phase 2:** DeFi staking layer (2% weekly APR)
- **Phase 3:** Social layer with friend battles and tournaments

## Business Model

### Revenue Streams

1. **AI Game House Edge:** 75% of losing stakes
2. **Coin Purchase Margins:** Revenue from RPS-coin sales
3. **Future:** Withdrawal fees, premium features, staking protocol fees

### Target Metrics

- **Conservative:** 100 DAU × $3 avg spend = $9,000/month
- **Optimistic:** 500 DAU × $5 avg spend = $75,000/month

## Technology Stack

### Frontend

- Next.js 14 (App Router)
- Tailwind CSS + shadcn/ui
- AppKit (Web3 integration)
- Zustand (State management)

### Backend

- Next.js API Routes
- PostgreSQL (Supabase)
- Redis (Caching)

### Blockchain

- Solidity ^0.8.19
- Foundry framework
- Chainlink oracles
- Multi-chain deployment

## Development Timeline

**Phase 1:** 12-16 weeks

- Weeks 1-2: Smart contracts & infrastructure
- Weeks 3-4: Backend & blockchain integration
- Weeks 5-6: AI engine development
- Weeks 7-8: Frontend core features
- Weeks 9-10: Payment & transaction systems
- Weeks 11-12: Game flow integration
- Weeks 13-14: Testing & security
- Weeks 15-16: Deployment & launch

## Success Criteria

### Technical

- 99.9% uptime
- <2s response times
- Zero critical vulnerabilities
- > 90% test coverage

### Business

- 100+ verified users in first month
- 70%+ user retention after first game
- $1,000+ revenue in first month
- 50+ DAU by end of Phase 1

## Risk Assessment

### Low Risk

- Technical implementation (building on proven RPS-Onchain foundation)
- Multi-chain deployment (established patterns)
- AI development (clear algorithms and patterns)

### Medium Risk

- User acquisition and retention
- Regulatory compliance (mitigated by crypto-only approach)
- Competition from established platforms

### High Risk

- Smart contract security (mitigated by audits and testing)
- Market volatility affecting token prices
- Scaling challenges if rapid growth occurs

## Next Steps

1. Review and approve requirements
2. Begin smart contract development
3. Set up development infrastructure
4. Start Phase 1 implementation

---

_This specification is part of the RPS-Onchain project evolution, transitioning from free-to-play to a premium skill-based betting platform._
