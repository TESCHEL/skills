# Agent Reputation

## Overview

The ERC-8004 Reputation Registry at 0x8004BAa17C55a88189AE136b182e5fdA19dE9b63 provides a trustless system for recording feedback and reputation scores for agents. Users and other agents can submit ratings, which are stored on-chain and aggregated into overall reputation metrics.

The reputation system enables:
- Recording interactions and feedback (ratings, comments, timestamps)
- Querying aggregate reputation scores for agents
- Filtering agents by minimum reputation thresholds
- Building trust networks based on historical performance

Reputation is linked to token IDs from the Identity Registry. Each feedback entry includes a rating (1-5 stars), optional comment, rater address, and timestamp. The registry calculates average ratings and total feedback counts.

## Reputation Registry Interface

The Reputation Registry implements these key methods:

```typescript
interface IAgentReputationRegistry {
  function recordFeedback(
    uint256 agentTokenId,
    uint8 rating,
    string calldata comment
  ) external;
  
  function getFeedbackCount(uint256 agentTokenId) external view returns (uint256);
  
  function getFeedback(
    uint256 agentTokenId,
    uint256 index
  ) external view returns (
    address rater,
    uint8 rating,
    string memory comment,
    uint256 timestamp
  );
  
  function getAverageRating(uint256 agentTokenId) external view returns (uint256);
  
  function getTotalRatings(uint256 agentTokenId) external view returns (uint256);
  
  function hasRated(uint256 agentTokenId, address rater) external view returns (bool);
}
```

## Recording Feedback

Submit feedback for an agent after interacting with its services:

```typescript
import { createPublicClient, createWalletClient, http } from 'viem';
import { privateKeyToAccount } from 'viem/accounts';
import { base } from 'viem/chains';

const REPUTATION_REGISTRY = '0x8004BAa17C55a88189AE136b182e5fdA19dE9b63';
const IDENTITY_REGISTRY = '0x8004A169FB4a3325136EB29fA0ceB6D2e539a432';

const account = privateKeyToAccount('0xYOUR_PRIVATE_KEY');

const publicClient = createPublicClient({
  chain: base,
  transport: http('https://mainnet.base.org')
});

const walletClient = createWalletClient({
  account,
  chain: base,
  transport: http('https://mainnet.base.org')
});

const reputationAbi = [
  {
    name: 'recordFeedback',
    type: 'function',
    stateMutability: 'nonpayable',
    inputs: [
      { name: 'agentTokenId', type: 'uint256' },
      { name: 'rating', type: 'uint8' },
      { name: 'comment', type: 'string' }
    ],
    outputs: []
  },
  {
    name: 'hasRated',
    type: 'function',
    stateMutability: 'view',
    inputs: [
      { name: 'agentTokenId', type: 'uint256' },
      { name: 'rater', type: 'address' }
    ],
    outputs: [{ name: '', type: 'bool' }]
  }
] as const;

async function recordFeedback(
  agentTokenId: bigint,
  rating: number,
  comment: string
) {
  if (rating < 1 || rating > 5) {
    throw new Error('Rating must be between 1 and 5');
  }
  
  const alreadyRated = await publicClient.readContract({
    address: REPUTATION_REGISTRY,
    abi: reputationAbi,
    functionName: 'hasRated',
    args: [agentTokenId, account.address]
  });
  
  if (alreadyRated) {
    throw new Error('You have already rated this agent');
  }
  
  const { request } = await publicClient.simulateContract({
    account,
    address: REPUTATION_REGISTRY,
    abi: reputationAbi,
    functionName: 'recordFeedback',
    args: [agentTokenId, rating, comment]
  });
  
  const hash = await walletClient.writeContract(request);
  
  console.log(`Feedback transaction: ${hash}`);
  
  const receipt = await publicClient.waitForTransactionReceipt({ hash });
  
  console.log(`Feedback recorded in block ${receipt.blockNumber}`);
  
  return receipt;
}

const agentTokenId = 42n;
await recordFeedback(
  agentTokenId,
  5,
  'Excellent service, fast execution, accurate results'
);
```

## Querying Reputation Scores

Retrieve aggregate reputation metrics for an agent:

```typescript
const scoreAbi = [
  {
    name: 'getAverageRating',
    type: 'function',
    stateMutability: 'view',
    inputs: [{ name: 'agentTokenId', type: 'uint256' }],
    outputs: [{ name: '', type: 'uint256' }]
  },
  {
    name: 'getTotalRatings',
    type: 'function',
    stateMutability: 'view',
    inputs: [{ name: 'agentTokenId', type: 'uint256' }],
    outputs: [{ name: '', type: 'uint256' }]
  },
  {
    name: 'getFeedbackCount',
    type: 'function',
    stateMutability: 'view',
    inputs: [{ name: 'agentTokenId', type: 'uint256' }],
    outputs: [{ name: '', type: 'uint256' }]
  }
] as const;

interface ReputationScore {
  tokenId: bigint;
  averageRating: number;
  totalRatings: bigint;
  feedbackCount: bigint;
}

async function getReputationScore(agentTokenId: bigint): Promise<ReputationScore> {
  const [averageRating, totalRatings, feedbackCount] = await Promise.all([
    publicClient.readContract({
      address: REPUTATION_REGISTRY,
      abi: scoreAbi,
      functionName: 'getAverageRating',
      args: [agentTokenId]
    }),
    publicClient.readContract({
      address: REPUTATION_REGISTRY,
      abi: scoreAbi,
      functionName: 'getTotalRatings',
      args: [agentTokenId]
    }),
    publicClient.readContract({
      address: REPUTATION_REGISTRY,
      abi: scoreAbi,
      functionName: 'getFeedbackCount',
      args: [agentTokenId]
    })
  ]);
  
  const avgRating = Number(averageRating) / 100;
  
  return {
    tokenId: agentTokenId,
    averageRating: avgRating,
    totalRatings,
    feedbackCount
  };
}

const score = await getReputationScore(42n);
console.log(`Agent #${score.tokenId}:`);
console.log(`  Average Rating: ${score.averageRating.toFixed(2)} / 5.00`);
console.log(`  Total Ratings: ${score.totalRatings}`);
console.log(`  Feedback Count: ${score.feedbackCount}`);
```

## Reading Individual Feedback

Retrieve specific feedback entries:

```typescript
const feedbackAbi = [
  {
    name: 'getFeedback',
    type: 'function',
    stateMutability: 'view',
    inputs: [
      { name: 'agentTokenId', type: 'uint256' },
      { name: 'index', type: 'uint256' }
    ],
    outputs: [
      { name: 'rater', type: 'address' },
      { name: 'rating', type: 'uint8' },
      { name: 'comment', type: 'string' },
      { name: 'timestamp', type: 'uint256' }
    ]
  }
] as const;

interface FeedbackEntry {
  rater: string;
  rating: number;
  comment: string;
  timestamp: bigint;
  date: Date;
}

async function getFeedback(
  agentTokenId: bigint,
  index: bigint
): Promise<FeedbackEntry> {
  const result = await publicClient.readContract({
    address: REPUTATION_REGISTRY,
    abi: feedbackAbi,
    functionName: 'getFeedback',
    args: [agentTokenId, index]
  });
  
  return {
    rater: result[0],
    rating: result[1],
    comment: result[2],
    timestamp: result[3],
    date: new Date(Number(result[3]) * 1000)
  };
}

async function getAllFeedback(agentTokenId: bigint): Promise<FeedbackEntry[]> {
  const count = await publicClient.readContract({
    address: REPUTATION_REGISTRY,
    abi: scoreAbi,
    functionName: 'getFeedbackCount',
    args: [agentTokenId]
  });
  
  const feedbackPromises = [];
  for (let i = 0n; i < count; i++) {
    feedbackPromises.push(getFeedback(agentTokenId, i));
  }
  
  return Promise.all(feedbackPromises);
}

const allFeedback = await getAllFeedback(42n);
allFeedback.forEach((fb, idx) => {
  console.log(`Feedback #${idx + 1}:`);
  console.log(`  Rater: ${fb.rater}`);
  console.log(`  Rating: ${fb.rating} / 5`);
  console.log(`  Comment: ${fb.comment}`);
  console.log(`  Date: ${fb.date.toISOString()}`);
});
```

## Common Patterns

**Pattern 1: Filter agents by minimum reputation**

```typescript
async function findAgentsWithMinRating(
  tokenIds: bigint[],
  minRating: number,
  minFeedbackCount: bigint
): Promise<bigint[]> {
  const scores = await Promise.all(
    tokenIds.map(id => getReputationScore(id))
  );
  
  return scores
    .filter(s => s.averageRating >= minRating && s.feedbackCount >= minFeedbackCount)
    .map(s => s.tokenId);
}

const allAgentIds = [1n, 2n, 3n, 42n, 100n];
const highlyRated = await findAgentsWithMinRating(allAgentIds, 4.0, 5n);
console.log('Highly rated agents:', highlyRated);
```

**Pattern 2: Reputation-weighted selection**

```typescript
function selectAgentByReputation(scores: ReputationScore[]): bigint {
  const weights = scores.map(s => {
    const ratingWeight = s.averageRating / 5.0;
    const countWeight = Math.min(Number(s.feedbackCount) / 20, 1.0);
    return ratingWeight * countWeight;
  });
  
  const totalWeight = weights.reduce((sum, w) => sum + w, 0);
  let random = Math.random() * totalWeight;
  
  for (let i = 0; i < scores.length; i++) {
    random -= weights[i];
    if (random <= 0) {
      return scores[i].tokenId;
    }
  }
  
  return scores[scores.length - 1].tokenId;
}

const candidateScores = await Promise.all(
  [1n, 2n, 3n].map(id => getReputationScore(id))
);
const selected = selectAgentByReputation(candidateScores);
console.log(`Selected agent #${selected}`);
```

**Pattern 3: Reputation decay over time**

```typescript
function calculateDecayedReputation(
  feedback: FeedbackEntry[],
  decayHalfLifeDays: number
): number {
  const now = Date.now() / 1000;
  const halfLife = decayHalfLifeDays * 24 * 60 * 60;
  
  let weightedSum = 0;
  let totalWeight = 0;
  
  feedback.forEach(fb => {
    const age = now - Number(fb.timestamp);
    const weight = Math.pow(0.5, age / halfLife);
    weightedSum += fb.rating * weight;
    totalWeight += weight;
  });
  
  return totalWeight > 0 ? weightedSum / totalWeight : 0;
}

const feedback = await getAllFeedback(42n);
const decayedRating = calculateDecayedReputation(feedback, 90);
console.log(`Decayed rating (90-day half-life): ${decayedRating.toFixed(2)}`);
```

**Pattern 4: Rater reputation weighting**

```typescript
interface WeightedFeedback extends FeedbackEntry {
  raterReputation: number;
  weight: number;
}

async function getWeightedReputation(
  agentTokenId: bigint,
  raterScores: Map<string, number>
): Promise<number> {
  const feedback = await getAllFeedback(agentTokenId);
  
  let weightedSum = 0;
  let totalWeight = 0;
  
  feedback.forEach(fb => {
    const raterScore = raterScores.get(fb.rater.toLowerCase()) || 1.0;
    weightedSum += fb.rating * raterScore;
    totalWeight += raterScore;
  });
  
  return totalWeight > 0 ? weightedSum / totalWeight : 0;
}

const raterScores = new Map([
  ['0x742d35cc6634c0532925a3b844bc9e7595f0beb', 1.5],
  ['0x123456789abcdef123456789abcdef123456789a', 0.8]
]);

const weightedRating = await getWeightedReputation(42n, raterScores);
console.log(`Weighted rating: ${weightedRating.toFixed(2)}`);
```

## Gotchas

- Each address can only rate an agent once. The contract enforces this with hasRated checks.
- Ratings must be between 1 and 5 (uint8). Sending 0 or values >5 will revert.
- The average rating is stored as an integer scaled by 100 (e.g., 425 = 4.25). Divide by 100 to get the decimal value.
- Comments are stored on-chain and contribute to gas costs. Keep them concise.
- There is no mechanism to edit or delete feedback once submitted. All ratings are permanent.
- The reputation system has no Sybil resistance built in. Multiple addresses from the same entity can submit ratings.
- Feedback count and total ratings may differ if the contract has been upgraded or has special logic.
- Reading all feedback for popular agents can be expensive. Consider pagination or off-chain indexing.
- The contract does not verify that the rater actually interacted with the agent. Trust is based on address reputation.
- Reputation scores do not decay automatically. Implement time-based decay in your application logic if needed.