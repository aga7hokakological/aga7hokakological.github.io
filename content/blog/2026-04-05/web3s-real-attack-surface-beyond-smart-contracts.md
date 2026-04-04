+++
title = "Web3’s Real Attack Surface: Beyond Smart Contracts"
subtitle = "Smart contracts are not the weakest link anymore."
date = 2026-04-05
draft = false
tags = ["ethereum", "solana", "infrastructure"]
+++

{{< x user="ResolvLabs" id="2035830314799599616" >}}

Once again its proven by the black hats that they don't care how secure your codebase is or how many audits you have done.

There was no reentrancy. No oracle manipulation. No overflow hiding in an unchecked block.
The contracts behaved exactly as designed.
And yet, funds were drained.

This wasn’t a failure of code. It was a failure of control.

### Incident Without Failure
Across multiple recent incidents, a consistent pattern is emerging:

- No smart contract vulnerability
- No oracle desynchronization
- No rounding bug
- No economic exploit

From the smart contract's perspective, nothing went wrong. Every transaction was valid. Every state transition respected the rules. Every invariant encoded in smart contract held true.

And that is precisely the problem.

### What happened
The crypto industry witnesses over $3.4B in theft in 2025 alone. Majority of it coming from CEX giant [Bybit](https://www.bybit.com/en/). And guess what was the issue?
**A private key theft**. If that was not enough later that year around May, June another 2 major exchanges [Nobitex Exchange](https://www.chainalysis.com/blog/nobitex-iranian-exchange-exploit-june-2025/), [Phemex Exchange](https://www.bitdefender.com/en-us/blog/hotforsecurity/phemex-million-crypto-heist) got
hacked because for private key leak.

And now in 2026 only 1st quarter is completed and we have seen $169.4M lost across 55 incidents in 90 days. Within this 55 exploits not even a single vulnerability
was novel exploit or a zero day attack. Everything about the attacks is very well documented still we are seeing this many exploits. Guess what almost 40% of $169.4M 
lost is through private key leak. In 1st quarter [Step finance](http://halborn.com/blog/post/explained-the-step-finance-hack-january-2026) and recently late March [Resolv protocol](https://www.chainalysis.com/blog/lessons-from-the-resolv-hack/) were exploited due to leaked private key. 

And just recently on 1st April we saw another hack ie., Drift Protocol. Yes it was not an April fool.
{{< x user="milotruck" id="2039358746275598551" >}}
{{< x user="DriftProtocol" id="2039404931778535427" >}}

The protocol signer had full control over:
- market creation
- Oracle assignment
- withdrawal limits.

This many critical functionalities were in the hands of protocol signer but there was no timelock, no multisig, no delay. On top of that it was 2/5 multisig.
You can check complete postmortem of Drift Protocol [here](https://x.com/omeragoldberg/status/2039455314089746747?s=20)

### A Pattern, Not an Outlier
These are no longer isolated events.

They form a repeatable attack template:
![alt](/blog/2026-04-02/attack.png)

> Web3 has entered a post-exploit era — where attackers don’t break systems, they operate them.

Smart contract exploits are becoming the exception.

### The Misplaced Security Boundary
For years, Web3 security has been centered around one idea:

> If the smart contract is secure, then the protocol is secure.

This assumption is now outdated.

Modern protocols operate across multiple layers:

![services](/blog/2026-04-02/service.png)

Audits focus almost exclusively on the smart contract layer.

**Today attackers they do not.**

### The Illusion of “Audited = Safe”

Audits secure logic. Yes they're important. As more and more projects, web2 business make move to onchain.
The onchain liquidity will keep rising. With that comes the great responsibility of having safe and secure smart contracts.
But but but there is always but because:

Attackers target control.

Even if smart contracts are secure a fully audited protocol can still be drained if:

- Admin keys are compromised
- Upgrade authority is leaked
- Treasury signers are exposed

Security is not a property of code. It is a property of the entire system.

### Infrastructure is the New Attack Surface

#### Execution vs Control Plane

##### Execution Plane:
- Smart contract logic
- State transitions
- On-chain invariants

All of the above is part of the trigger which isn't in hand of the protocol team. They can do audits for smart contract logic.
State transition of the contract depends on how transactions are added to the block ie., MEV, transaction re-ordering is controlled by validators.
All invariants such as `deposits(vault) == balanceof(all users)` must hold true no matter what but 
it can be broken if there is any vulnerability in the code.

##### Control Plane:
- Private keys
- Admin roles
- Deployment pipelines
- Backend infrastructure
- Offchain computation

On the other hand private keys, admin roles, deployement, backend infrastructure, etc. are actually in the hands of the team.
If private keys are compromised then surely attacker can get hold of the whole TVL of the protocol. There is literally no way to stop the
bloodbath that happens after that if not acted quickly. Same with admin roles of the protocol.
You also need to keep in mind about deployment and backend infrastructure. If not then situation like [resolv protocol](https://www.chainalysis.com/blog/lessons-from-the-resolv-hack/#:~:text=The%20attacker%20compromised,operation%20they%20chose.) hack can happen.

![execution-control](/blog/2026-04-02/exe-control.png)

A protocol can be **perfectly secure** in the **execution plane** and still be **completely compromised** if the **control plane** is breached.

The attacker no longer needs to find a bug in solidity smart contracts. They can find one in your DevOps.

**Critical Risk Areas:**
- Key storage (hot wallets, env vars)
- Signing infrastructure
- Backend services
- Frontend / DNS
- CI/CD pipelines

Each of these can bypass the need for any on-chain exploit.

### Why These Attacks Succeed Instantly

Smart contracts got harder to crack, so attackers moved up the stack to the humans running them. That's not an excuse, it's where the attack surface shifted.
Key compromise is not inherently catastrophic. In web2 systems key compromises happens a lot. Some projects hardcode their keys in
github repo and forget about it. Some are targetted via web app vulnerabilities such as XSS, SSRF, etc.
But here in web3 the things are bit different.

What makes it catastrophic is instant, irreversible execution.

- No delay
- No visibility
- No secondary approval

Control compromise is not fatal. Instant execution is.

### Security Needs Friction
What do you mean by security needs friction? 

Smart contracts removed friction from execution. Transactions settle instantly. But in adversarial environments, speed benefits attackers more than defenders. 
So what I meant is we need to **increase the time and efforts required by the attacker to convert key compromise into irreversible loss.**

The goal is not to stop attackers from acting. I mean if we can stop attackers even before they act then it is the perfect setup but if not then we 
ensure that they cannot act instantly and invisibly.

We define a new security primitive:

Adversarial Latency: the delay between gaining control and causing irreversible damage.

Attack pattern:

- Gain access
- Act immediately
- Drain before detection

Security must break step 2.

### Current Web3 Infrastructure Security Stack
Don't we already have the solutions to solve this issue? We do.
Today’s security is mostly built around key protection + authorization, not execution control.

- **Key Management Solutions (Core Layer)**
- 1. MPC Wallets (dominant trend)
        Keys are split into multiple shares
        No single entity holds the full key
        Signing is collaborative

        - Why it exists: Eliminates single point of failure
        Prevents raw key exposure

        - Reality: If enough shares are compromised the attacker signs anyway

- 2. Multisig Wallets (e.g., Safe)
        M-of-N approvals required
        Enforced on-chain

        - Strength: Prevents unilateral control

        - Weakness: Keys often live in same infra (correlated risk)
        No delay by default

- 3. Hardware Security Modules (HSMs) / TEEs
        Keys stored in secure hardware
        Signing isolated from OS / memory

        - Strength: Prevents key extraction

        - Weakness: If attacker controls the signing request then signing still happens

- **Transaction-Level Security**
- 1. Transaction Simulation
        Simulate tx before execution
        Detect malicious outcomes

        Example: Systems like Hypernative simulate and block risky transactions

        - Strength: Detects bad intent

        - Weakness: Still relies on correct detection
        Can be bypassed if attacker mimics normal behavior

- **Infrastructure Security**
- 1. Zero Trust / Access Control
        Identity-based access
        Strict permissions

- 2. Secrets Management
        Vaults (AWS, Hashicorp, etc.)
        No plaintext keys

- 3. CI/CD Hardening
        Signed deployments
        Restricted pipelines

        - Strength: Reduces attack surface

        - Weakness: Still assumes prevention is enough

- **Human Layer Defenses**
 1. Hardware wallets

 2. MFA / biometrics

 3. Phishing protection


{{< x user="calilyliu" id="2039652201342050713" >}}

> Humans remain the weakest link in Web3

### Friction Invariants (New Layer)

Then am I suggesting anything new? Probably not. The core problem with current solution is that they are focused on preventing compromise.
But real-world data shows:

- Private key compromises still happen
- Infra breaches still happen
- Humans still get phished

And when they do:

There is nothing stopping instant execution.
The current solutions are Static, isolated, and optional. What we actually need is Dynamic, composable, and mandatory friction at the control boundary.

**What’s Broken Today (Even With Timelocks)**
1. Friction is Static

Example: “All upgrades have 24h delay”

Problem:
- Low-risk action -> unnecessarily slow
- High-risk action -> sometimes still insufficient
- No adaptability

2. Friction is Not Context-Aware

Current systems don’t ask:

- Is this abnormal behavior?
- Is this signer behaving differently?
- Is this transaction unusual in size or pattern?

Attacker can mimic “normal” flows

3. Friction is Optional (Critical flaw)
 - Many protocols:
 - Skip timelocks for “emergency”
 - Have privileged bypass roles
 - Use multisig but same infra

In real attacks friction paths are often bypassed entirely

4. No Coupling Between Detection and Execution

Today Monitoring (e.g. anomaly detection) and Execution (smart contracts) are separate systems.
Even if something suspicious is detected, it does NOT automatically slow/block execution

Friction defines how easily and how quickly can control be exercised?

Formalizing the Time Dimension

Let:
```
T_attack = time to drain funds
T_detect = time to detect anomaly
```
Current systems:
```
T_attack << T_detect -> catastrophic loss
```
With friction:
```
T_attack > T_detect -> intervention possible
```
Security now depends on time, not just correctness.

### Defining Friction Invariants
- Non-Instantaneity: Delays on critical actions
- Non-Unilaterality: Multisig / role separation
- Observability: Public execution queues
- Context Sensitivity: Higher risk -> higher friction

![evaluation](/blog/2026-04-02/evaluation.png)

If protocols were to implement time delays on smart contract upgrades, treasury withdrawal, governance then 
there is chance of detection of attack if it is happening then attackers might not be able drain the protocol.
Protocol can even implement multi-step execution rather than simple multisig. 
The `Drift protocol` situation occurred because of the concept of [durable nonces](https://solana.com/developers/guides/advanced/introduction-to-durable-nonces)
where signature can already be sent for execution. 
The security can be improved with context aware friction such as:
- If there is large transaction -> delay it
- If there is new operator -> can execute if allowed
- New contract -> requires multisig to deploy it or upgrade it

The most important thing to do will be to add circuit breaker which allows the protocol to pause the important functionality in emergency situations. 

Even if control is compromised damage must not be instantaneous.

This is the difference between:

Irrecoverable loss and containable incident

### Conclusion

Web3 security has spent years optimizing for correctness. We proved that contracts can enforce invariants. We built systems that execute exactly as specified.
And in doing so, we assumed that correctness was enough. It isn’t.

Recent incidents make one thing clear:

The system does not fail when invalid actions occur.
It fails when valid actions are executed by the wrong actor, at the wrong time, without resistance.

Today’s security stack focuses on preventing compromise:
- Protect the key
- Verify the transaction
- Execute instantly

But real-world attacks don’t need broken logic.
They need control and speed. This is the gap. The next evolution of Web3 security is not about new primitives.
It is about how those primitives are orchestrated under adversarial conditions.

Timelocks, multisigs, simulations they already exist.
But they are static, optional, and disconnected.

What’s missing is a system that:

- They adapts to risk in real time
- Couples detection with enforcement
- And ensures that high-impact actions are never instantaneous, silent, or unilateral

This is where friction becomes fundamental. Not as inconvenience, but as a security boundary.
A well-designed system does not assume safety.
It assumes compromise and ensures that compromise cannot immediately become catastrophe.
In this model, security is no longer binary.

A system is secure if it can detect and respond before irreversible damage occurs.
The future of Web3 security will not be defined by who writes the safest contracts.
It will be defined by who builds systems where even if an attacker gains control,
they cannot win fast enough.