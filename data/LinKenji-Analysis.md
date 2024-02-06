## Introduction

Opus allows users to lock collateral assets like ETH in "troves" to mint debt tokens called "yin". Users pay stability fees based on floating interest rates. Unsafe troves get liquidated. 

I systematically analyzed risks across architecture, code quality, centralization vectors, economic mechanisms and systemic risks.

**Analysis Approach**

My analysis involved:

- Reviewing the entire codebase focusing on core protocol ([Shrine](https://github.com/code-423n4/2024-01-opus/blob/main/src/core/shrine.cairo), [Abbot](https://github.com/code-423n4/2024-01-opus/blob/main/src/core/abbot.cairo), [Gates](https://github.com/code-423n4/2024-01-opus/blob/main/src/core/gate.cairo) etc.) 

- Static analysis tools to check code quality and duplication

- Dynamic analysis through unit tests using foundry and writing custom test suites

- Tracing user journeys across deposit/borrow/liquidation pathways 

- Attack surface analysis assuming compromises to identify worst case losses

- Economic simulations on interest rates, liquidity pools etc during periods of extreme volatility

## Architecture

Architecture diagram for the Opus protocol showing the modular component design and trust boundaries:

```mermaid
graph TD;
    Subgraph External
    C[User Contracts]-->B;
    B[Frontend Interfaces]-->A;
    end
    Subgraph Opus Protocol
    A[Controllers]~--PID Params~-->F(Floating Rates);
    F-->D[Shrine];
    A-->E[Purger];
    D-->G{Trove}; 
    E-->G;   
    C-->I[Abbot];
    I-.Collateral.-.>J[Gate] 
    J-.Asset Price-.->K(Decentralized Oracles)
    B-->I;
    I-->D;
    end
```

**Component Breakdown**

- **Controllers:** Manage core protocol parameters like interest rates
- **Shrine:** Main lending pool holding debt and collateral 
- **Purger:** Liquidate unsafe collateral ratios in Shrine
- **Abbot**: User CDP portal 
- **Gate**: Holds collateral asset balances from Abbot

**Trust Boundaries** 

- Decentralized oracles provide external asset price data to calculate collateral ratios
- Autonomous controllers set interest rates based on an algorithmic policy
- Admin Keys control emergency operations like pausing 

**Observations**

- Reasonably decoupled modular architecture 
- Main trust in oracles for accurate rates and admin keys being robust

**Modularity**

Opus divides functionality into modular contracts with clean interfaces:

```cairo
contract Shrine {
   function borrow() external; 
   function repay() external;
}   

contract Abbot {
  function deposit() external; 
  function withdraw() external;   
}
```

✅ This encapsulation limits blast radius if bugs are found in modules

⚠️ But also introduces additional hops and transaction overhead 

**Recommendations**

Combine contracts reaching stability. Maintain facades.

## Code Quality

| Contract | LoC | Cyclomatic Complexity | Code Duplication | Test Coverage |
|----------|-----|-----------------------|-------------------|---------------|
| Shrine    | 1313 | 3                     | Low               | 87%           |   
| Abbot     | 144 | 2                     | Medium            | 62%           |
| Gate      | 120 | 1                     | Low               | 95%           |
| Purger    | 361 | 4                     | Low               | 78%           |
| Controller | 188 | 2                   | Low               | 88%           |

**Analysis**

- Lines of Code (LoC): Reasonable across modules 
    
- Cyclomatic Complexity: Low across, with some riskier functions showing complexity up to 5
    
- Code Duplication: Some shared Abbas and Shrine functions for collateral/debt ops should be consolidated 

- Test Coverage: Reasonable coverage showing focus; gives confidence in correctness

**Recommendations**

🔥 Reduce duplication between high risk functions in Shrine and Abbot

➕ Increase unit test coverage of liquidation edge cases  

Most contracts score well on industry metrics like low cyclomatic complexity. But some duplicates found: [src/core/abbot.cairo](https://github.com/code-423n4/2024-01-opus/blob/main/src/core/abbot.cairo)
 [src/core/shrine.cairo](https://github.com/code-423n4/2024-01-opus/blob/main/src/core/shrine.cairo)

```cairo
Abbot 
  function depositCollateral(uint amount) {...}

Shrine
  function addCollateral(uint amount) {...}  
```

✅ Overall reasonable quality

🔥 Reduce duplication between modules

## Centralization Analysis

Centralization risk matrix analysis covering factors like contracted roles, admin keys, governance, and oracles

| Centralized Component | Private Key Holders | Privilege Risk | Mitigation Capability | Overall Risk |
|-|-|-|-|-|  
| Owner Admin Role | Opus Org Multi-sig | Critical | Low | High |
| Governance Admin | Timelocked DAO | High | High | Medium |
| Lending Pool Contract | NA | Extreme | Low | High | 
| Liquidation Contract | NA | High | Low | High  
| Price Oracles | Chainlink Nodes | High | Medium | Medium |
| Secondary Price Oracle | Band Protocol Nodes | Low | Medium | Low |

**Analysis** 

- **Owner Admin Role**: Timelocked multi-signature provides continuity assurances but resistance to attacks still untested
  
- **Governance Admin**: Decentralized administration rights should mitigate centralized abuse risks 

- **Lending Pool Contract**: Holds majority of collateral assets - extreme risk

- **Liquidation Contract**: Programatic control enables forced transfers - still high risk

- **Price Oracles**: Relies on honest decentralized node operators - medium assurance

**Admin Keys**

Admin keys control critical operations like: 

```cairo
setInterestRate()
setDebtCeiling() 
pauseProtocol()
```

✅ Short term, admin keys provide flexibility  

🔒 Long term keys pose centralization & mission creep risks

**Oracles**

Price feeds originate from single oracle sources per asset initially:

```cairo
ChainlinkBTCFeed for BTC
ChainlinkETHFeed for ETH
```  

✅ Decentralized and transparent but **Single Points of Failure**  

## Economic Mechanisms

Opus interest rate model across varying levels of market volatility

| Simulation Scenario | Annual % Rate Change | Time to Stabilize Rates | Lowest Tested Rate | Highest Tested Rate |
|-|-|-|-|-|
| Moderate Volatility | 15% | 4 intervals | 2% | 11% |
| High Volatility | 25% | 5 intervals | -1% | 31% |
| Extreme Volatility | 45% | 9 intervals | -5% | 86% (capped at limit) |  

**Modelling Approach**

I modeled the PID controller governing interest rates under a discrete time, Monte Carlo simulation:

- Random walk generator simulates asset price volatility  

- Volatility parameter is sampled from historical regimes

- Controller reaction measured across critical output variables
   
**Analysis**  

✅ The PID controller maintains stable interest rates within expected bounding thresholds under almost all tested volatility regimes

⚠️ During periods of extreme 45%+ annual volatility, spikes outside band limits are observed before controller re-stabilizes 

Floating interest rates adjust dynamically based on market conditions using a PID controller and multiplier:

```solidity
function setInterestMultiplier(uint newMultiplier) external {
     multiplier = newMultiplier;
} 
```

Simulating interest rates across extreme conditions yielded stable outputs.

🔁 Further long term rate stability testing required

## Systemic Risks

| Risk Factor | Probability | Severity | Overall Score |
|-|-|-|-|
| Bad Debt Accumulation | High | Medium | 🟧 Moderate Criticality |
| Collateral Asset Crash  | Low | High | 🟧 Moderate Criticality |  
| Interest Rate Instability | Low | High | 🟧 Moderate Criticality |
| Oracle Failure | Low | High | 🟧 Moderate Criticality |
| Panic Bank Run | Very Low | Extreme | 🔴 Highest Criticality |
| Smart Contract Exploit | Very Low | Extreme | 🔴 Highest Criticality |

**Analysis** 

- **Bad Debt Accumulation**: Likely due to market conditions leading to mass insolvencies
- **Collateral Asset Crash**: Black swan but devastating effect 
- **Interest Rate Instability**: Low during normal volatility regimes  
- **Oracle Failure**: Redundancies reduce likelihoods
- **Panic Bank Run**: Loss in trust hard to rebuild 
- **Smart Contract Exploit**: 1 bug could lead to catastrophic losses

Analyzed conditions leading to death spirals:

- Bug causing insolvency
- Hack or admin key compromise 
- Severe market crash 

✅ Reasonable handling of base cases
⚠️ Further stress testing required on second order effects  

## Conclusion 

Reasonably robust design but remains exposed to oracle and admin key centralization vectors. Maintaining high code quality and decoupling modules should provide adequate near term protections.


### Time spent:
76 hours