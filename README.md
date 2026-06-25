# Privacy-Preserving AI Bounty Judge

A Ritual Chain workshop submission for a fair bounty judging flow. The core fix is simple: participants commit to hidden answers first, reveal only after the commit window closes, and only revealed + verified answers can be judged.

## Assignment Track

### Required: Commit-Reveal Bounty

The contract implements a commit-reveal flow that works on any EVM chain:

1. **Create bounty** with a reward, rubric, commit deadline, and reveal deadline.
2. **Commit phase**: participants submit only a hash commitment.
3. **Reveal phase**: participants reveal the plaintext answer and salt after the commit deadline.
4. **Verification**: the contract checks:

```solidity
keccak256(abi.encodePacked(answer, salt, msg.sender, bountyId)) == commitment
```

5. **AI judging**: the owner calls Ritual-backed judging for the batch of valid revealed answers.
6. **Finalize**: the owner selects the winner index and the contract pays the reward.

## Required Functions

```solidity
submitCommitment(uint256 bountyId, bytes32 commitment)
revealAnswer(uint256 bountyId, string calldata answer, bytes32 salt)
judgeAll(uint256 bountyId, bytes calldata llmInput)
finalizeWinner(uint256 bountyId, uint256 winnerIndex)
```

## Why Commit-Reveal

Plain on-chain submissions are public immediately, so later participants can copy earlier answers and improve them. Commit-reveal removes that advantage: everyone locks a hash first, then reveals later. A reveal is accepted only if the answer, salt, sender, and bounty ID reproduce the original hash.

## Advanced Ritual-Native Hidden Submissions

A stronger Ritual-native design would keep plaintext answers encrypted off-chain until judging. On-chain storage would contain commitments, metadata, deadlines, and encrypted references. Plaintext would exist only on the participant side before upload and inside Ritual's TEE-backed execution during batch judging. The LLM should receive all decrypted submissions in one batch, not one LLM call per answer, so judging is consistent and cheaper. The final public result should be only the AI review summary, the selected winner index, and enough proof to audit the lifecycle without exposing non-winning private drafts early.

## Test Plan

The Hardhat tests cover:

- bounty creation with commit/reveal deadlines
- commitment storage
- duplicate commitment rejection
- reveal after commit deadline
- wrong salt rejection
- wrong answer rejection
- double reveal rejection
- reveal-before-deadline rejection
- helper commitment hash parity

Run:

```bash
cd hardhat
pnpm install
pnpm test
```

## Reflection

What should be public in a bounty system is the bounty metadata, deadlines, reward amount, commitment hashes, valid reveal status, judging result, and final winner. What should stay hidden during the submission phase is each participant's actual answer and salt, because public answers create copycat incentives. After the reveal phase, answers can become public in the required commit-reveal track because every participant has already locked their commitment. In a stronger Ritual-native version, plaintext answers should remain private even longer and only be decrypted inside a TEE for batch judging. AI should decide ranking assistance, rubric comparison, and summarization, but a human or bounty owner should finalize the winner because reward decisions often require context and accountability. The contract should enforce timing, hash validity, duplicate prevention, and payout safety. The human should remain responsible for policy judgment and dispute resolution. This split keeps the system fair without pretending AI should own the entire bounty process.
