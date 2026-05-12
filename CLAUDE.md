# CLAUDE.md

## Project Context: E-VaultX

**Description:** Personal Finance & Multi-Currency Wallet Platform with Double-Entry Ledger.
**Architecture:** Multi-Protocol Microservices.
- **External API:** GraphQL Gateway (BFF).
- **Internal Service-to-Service:** gRPC (Protocol Buffers).
- **Edge Cases:** REST API (OAuth2, File Uploads).
- **Asynchronous:** Apache Kafka (KRaft mode), Outbox Pattern.

**Tech Stack:**
- **Backend:** Java 21, Spring Boot 3, Spring Data JPA, Spring GraphQL, Spring Security, gRPC.
- **FX Rate Engine:** C++17, Boost.Asio, librdkafka.
- **Infrastructure:** PostgreSQL 15, Redis 7, Kafka 3.x (KRaft).
- **Frontend:** React 18.

**Key Constraints & Rules:**
- **Ledger:** Strict Double-Entry Ledger. Every transaction must have a DEBIT and CREDIT. Do not update balances directly outside of ledger entry flows.
- **Database:** Strict 3NF. All primary keys are UUID v4. Use Flyway for schema migrations (no `ddl-auto=update`).
- **Mapping:** Use MapStruct for DTO ↔ Entity mapping. No reflection-based mappers like ModelMapper.
- **Locking:** Use JPA `@Lock(PESSIMISTIC_WRITE)` and Redisson for distributed locks during balance mutations.
- **Idempotency:** Required for all transfer mutations.
- **FX Rate Cache:** Redis key `fx:rate:{from}:{to}`, TTL **10 giây**. Luôn đọc từ cache, không hardcode tỷ giá.

**Common Commands:**
- Build Protobuf: `mvn clean install -pl proto-module`
- Build All: `mvn clean install`
- Start Infra: `docker-compose up -d`
- Start API Gateway: `cd services/api-gateway && mvn spring-boot:run`

---

## 1. Think Before Coding

**Don't assume. Don't hide confusion. Surface tradeoffs.**

Before implementing:
- State your assumptions explicitly. If uncertain, ask.
- If multiple interpretations exist, present them - don't pick silently.
- If a simpler approach exists, say so. Push back when warranted.
- If something is unclear, stop. Name what's confusing. Ask.

## 2. Simplicity First

**Minimum code that solves the problem. Nothing speculative.**

- No features beyond what was asked.
- No abstractions for single-use code.
- No "flexibility" or "configurability" that wasn't requested.
- No error handling for impossible scenarios.
- If you write 200 lines and it could be 50, rewrite it.

Ask yourself: "Would a senior engineer say this is overcomplicated?" If yes, simplify.

## 3. Surgical Changes

**Touch only what you must. Clean up only your own mess.**

When editing existing code:
- Don't "improve" adjacent code, comments, or formatting.
- Don't refactor things that aren't broken.
- Match existing style, even if you'd do it differently.
- If you notice unrelated dead code, mention it - don't delete it.

When your changes create orphans:
- Remove imports/variables/functions that YOUR changes made unused.
- Don't remove pre-existing dead code unless asked.

The test: Every changed line should trace directly to the user's request.

## 4. Goal-Driven Execution

**Define success criteria. Loop until verified.**

Transform tasks into verifiable goals:
- "Add validation" → "Write tests for invalid inputs, then make them pass"
- "Fix the bug" → "Write a test that reproduces it, then make it pass"
- "Refactor X" → "Ensure tests pass before and after"

For multi-step tasks, state a brief plan:
```
1. [Step] → verify: [check]
2. [Step] → verify: [check]
3. [Step] → verify: [check]
```

Strong success criteria let you loop independently. Weak criteria ("make it work") require constant clarification.

---

**These guidelines are working if:** fewer unnecessary changes in diffs, fewer rewrites due to overcomplication, and clarifying questions come before implementation rather than after mistakes.