# RESEARCH_OVERVIEW.md

# CPO-MCGS: Counterexample-Guided, Certificate-Carrying Proof-Obligation Monte Carlo Graph Search for sLLM Inequality Proving

작성일: 2026-05-06  
연구 목표 분야: sLLM 기반 수학 증명, 부등식 증명, Search + Verifier, Neuro-symbolic reasoning

---

## 1. Executive Summary

본 연구는 **소형 언어모델(sLLM)의 부등식 증명 능력**을 개선하기 위한 Search + Verifier 시스템을 제안한다. 핵심 아이디어는 sLLM을 완전한 풀이 생성기가 아니라 **검증 가능한 proof action policy**로 사용하고, 각 action을 **반례 탐색(counterexample search)** 과 **symbolic certificate verifier**로 검증한 뒤, **PUCT/MCTS 기반 Monte Carlo Graph Search**로 증명 경로를 탐색하는 것이다.

최종 제안 시스템의 이름은 다음과 같다.

> **CPO-MCGS: Counterexample-Guided, Certificate-Carrying Proof-Obligation Monte Carlo Graph Search**

이 연구는 다음 문제의식에서 출발한다.

- 일반 LLM은 부등식 문제의 최종 답을 맞히더라도, 중간 증명 step에는 논리적 비약이 자주 발생한다.
- IneqMath는 top model조차 step-wise scrutiny 기준 overall accuracy가 10% 미만으로 떨어질 수 있음을 보고했다.
- rStar-Math는 sLLM도 MCTS와 process verifier를 결합하면 수학 추론 성능이 크게 향상될 수 있음을 보여주었다.
- AlphaProof, DeepSeek-Prover, AlphaGeometry2는 강한 verifier와 search가 결합될 때 수학 증명에서 큰 성과가 가능함을 보여주었다.
- 그러나 Lean 전체 formalization은 비용이 높고, 자연어 PRM만으로는 부등식 증명의 universal correctness를 보장하기 어렵다.

따라서 본 연구는 부등식 증명을 다음과 같이 재정의한다.

> 자연어 풀이 생성 문제가 아니라, **증명 의무(proof obligation)를 줄여 나가는 certificate-carrying graph search 문제**로 본다.

초기 검증 단계부터 search core를 약화하지 않는다. 즉, length-normalized best-first search 같은 단순화된 search를 주력 방법으로 사용하지 않는다. 해당 방식은 baseline 또는 ablation으로만 사용한다. 연구 가설을 제대로 검증하기 위해, 처음부터 다음 핵심 요소를 포함한 full-strength architecture를 사용한다.

- PUCT/MCTS 기반 Monte Carlo Graph Search
- proof-obligation state
- graph transposition / shared facts
- counterexample-guided rejection
- symbolic certificate gating
- verifier-generated positive/negative traces
- 최소 1회 LoRA-SFT 또는 DPO 기반 policy improvement

줄이는 것은 search architecture가 아니라 **문제 범위, theorem library, rollout budget, self-evolution round 수**이다.

---

## 2. Background and Motivation

### 2.1 왜 부등식 증명인가

부등식 증명은 일반적인 산술·대수 문제보다 더 엄격한 추론 능력을 요구한다. 단순히 정답을 계산하는 것이 아니라 다음을 모두 만족해야 한다.

1. 제안한 부등식이 모든 정의역에서 성립함을 보여야 한다.
2. 최적 상수 문제의 경우, 상수가 실제로 sharp함을 보여야 한다.
3. AM-GM, Cauchy-Schwarz, Jensen, Schur, uvw, SOS decomposition 등 정리 적용 조건을 정확히 확인해야 한다.
4. 등호 조건, 극한 반례, case split coverage를 빠뜨리지 않아야 한다.
5. 특정 toy case나 symmetric slice에서만 확인하고 일반 결론을 내려서는 안 된다.

IneqMath는 이러한 문제의식을 반영하여 Olympiad-level inequality proving을 bound estimation과 relation prediction이라는 비교적 검증 가능한 subtask로 재구성했다. 또한 최종 답 judge뿐 아니라 Toy Case, Logical Gap, Numerical Approximation, Numerical Computation 관련 step-wise judge를 포함한다.

### 2.2 왜 sLLM인가

sLLM은 다음 이유로 연구 가치가 크다.

- 실험 비용이 낮다.
- open-weight 모델을 사용할 수 있다.
- 내부 architecture를 수정하지 않고도 search/verifier sidecar를 붙일 수 있다.
- verifier-generated trace를 LoRA-SFT/DPO로 빠르게 반영할 수 있다.
- rStar-Math 계열 결과는 sLLM도 search + verifier + self-evolution 구조에서는 강한 수학 추론 능력을 보일 수 있음을 시사한다.

본 연구는 sLLM이 “혼자서 완전한 증명을 작성하는 능력”보다, **검증 가능한 action 후보를 생성하는 policy 능력**을 활용한다.

---

## 3. Related Work Analysis

### 3.1 rStar-Math

rStar-Math는 sLLM이 MCTS와 process reward model을 결합하면 강한 수학 reasoning 성능을 낼 수 있음을 보여준 대표 연구다. math policy SLM이 test-time MCTS를 수행하고, SLM-based process reward model이 탐색을 guide한다. 또한 MCTS rollout으로 step-by-step verified reasoning trajectories를 만들고, policy SLM과 process preference model을 self-evolve시킨다.

본 연구가 가져오는 교훈은 다음이다.

- sLLM은 단발성 CoT 생성기보다 search policy로 쓸 때 더 강하다.
- MCTS는 초기부터 core method로 들어가야 한다.
- verifier trace를 학습 신호로 되돌리는 self-improvement loop가 중요하다.

한계는 다음이다.

- Python/code execution과 PRM은 계산 오류에는 강하지만, 부등식의 universal proof correctness를 완전히 보장하지 않는다.
- 자연어 CoT의 logical gap을 hard하게 차단하기 어렵다.

본 연구는 rStar-Math의 MCTS + sLLM policy 구조를 유지하되, PRM 중심 검증 대신 **counterexample rejection + symbolic certificate gating**을 도입한다.

### 3.2 IneqMath

IneqMath는 부등식 증명에서 LLM의 “최종 답 정확도”와 “전체 증명 정확도” 사이의 큰 간극을 보여준다. 특히 top model도 step-wise scrutiny를 적용하면 overall accuracy가 10% 미만으로 떨어질 수 있다고 보고했다.

본 연구가 직접 겨냥하는 IneqMath failure modes는 다음이다.

- Toy Case Error: 몇 개의 특수 사례만 보고 일반 결론을 내림.
- Logical Gap Error: WLOG, 자명하다, 알려진 정리에 의해 등의 표현으로 미검증 step을 숨김.
- Numerical Approximation Error: 근삿값으로 부등식 방향이나 최적 상수를 부정확하게 판단함.
- Numerical Computation Error: 계산 실수로 잘못된 결론에 도달함.

본 연구에서는 위 failure mode를 search transition 단계에서 직접 막는다.

### 3.3 AlphaProof and DeepSeek-Prover

AlphaProof와 DeepSeek-Prover 계열은 formal verifier가 있는 환경에서 search/RL이 강력함을 보여준다. AlphaProof는 Lean environment와 AlphaZero-inspired RL을 결합해 IMO 2024에서 silver-medal 수준 결과를 보고했다. DeepSeek-Prover-V1.5는 Lean 4 proof assistant feedback, RLPAF, RMaxTS라는 MCTS 변형을 결합했다.

본 연구가 가져오는 교훈은 다음이다.

- verifier는 judge가 아니라 environment처럼 작동해야 한다.
- search transition은 검증 가능한 상태 변화를 만들어야 한다.
- proof assistant 수준의 hard feedback은 강력하지만, 모든 자연어 부등식을 Lean으로 formalize하는 것은 초기 연구 범위로는 무겁다.

본 연구는 전체 Lean formalization 대신, 부등식에 특화된 **symbolic certificate**를 사용한다. Lean은 optional local certificate checker로만 둔다.

### 3.4 AlphaGeometry2

AlphaGeometry2는 LLM이 auxiliary construction을 제안하고, symbolic engine이 검증하는 neuro-symbolic 구조로 Olympiad geometry에서 높은 성과를 보였다. 특히 Shared Knowledge Ensemble of Search Trees(SKEST)를 통해 여러 search tree가 증명된 facts를 공유한다.

본 연구가 가져오는 교훈은 다음이다.

- domain-specific symbolic verifier는 매우 강력하다.
- LLM은 창의적 후보 생성기로 쓰고, verifier가 허용한 것만 state에 반영해야 한다.
- 여러 경로에서 얻은 proven facts, failed facts, residual states를 공유하는 graph/transposition 구조가 중요하다.

본 연구는 이를 부등식에 맞게 변환한다.

- geometry auxiliary construction → inequality theorem application, decomposition, substitution, case split
- DDAR symbolic engine → inequality certificate verifier, counterexample search, SOS/identity checker
- SKEST fact sharing → proof-obligation graph transposition and shared facts

### 3.5 LIPS and IneqSearch

LIPS와 IneqSearch는 부등식 증명에서 LLM과 symbolic reasoning의 결합이 유망함을 보여준다. LIPS는 symbolic methods와 LLM을 결합하여 proof search를 수행하고, IneqSearch는 theorem combination을 찾아 non-negative component decomposition을 구성하는 structured search로 부등식 증명을 재구성했다.

본 연구와의 차별점은 다음이다.

- 단순히 theorem combination을 찾는 데서 그치지 않고, 각 transition을 certificate-carrying edge로 만든다.
- 잘못된 proof action은 counterexample, failed precondition, uncovered case split과 함께 negative trace로 저장한다.
- sLLM policy를 verifier-generated trace로 LoRA-SFT/DPO 개선한다.
- IneqMath failure taxonomy를 metric과 training signal로 직접 연결한다.

### 3.6 BFS-Prover

BFS-Prover는 Lean proof search에서 length normalization과 DPO를 결합한 best-first search가 강력할 수 있음을 보여준다. 그러나 본 연구의 초기 가설검증에서 BFS를 주력 설계로 쓰는 것은 위험하다.

이유는 다음이다.

- 우리의 핵심 가설은 MCTS/PUCT 기반 search + verifier가 proof correctness를 높인다는 것이다.
- BFS로 축소하면, 실패 시 “아이디어가 틀렸다”가 아니라 “search core를 약화해서 실패했다”는 ambiguity가 생긴다.
- 부등식 자연어 증명은 Lean tactic search보다 state transition의 검증성이 낮기 때문에, 장기 reward backup과 exploration이 더 중요하다.

따라서 BFS류 방법은 baseline 또는 ablation으로만 사용한다.

---

## 4. Research Problem

### 4.1 Problem Statement

입력은 Olympiad-level inequality problem이다.

예시:

```text
For all positive real numbers a, b, c, find the largest constant C such that F(a,b,c) ≥ C.
```

기존 LLM 접근은 다음과 같다.

```text
problem → LLM generates full natural-language proof → final answer / verifier reranking
```

본 연구는 이를 다음과 같이 바꾼다.

```text
problem → proof-obligation graph → sLLM proposes proof actions → verifier accepts/rejects transitions → MCGS explores certified proof paths
```

### 4.2 Core Research Question

> sLLM을 proof action policy로 사용하고, counterexample-guided rejection과 symbolic certificate gating을 hard transition rule로 둔 PUCT/MCTS 기반 Monte Carlo Graph Search가 부등식 증명의 overall proof correctness를 유의미하게 개선할 수 있는가?

### 4.3 Main Hypothesis

> 부등식 증명을 proof-obligation graph로 표현하고, sLLM이 제안한 proof action을 counterexample search와 symbolic certificate로 검증한 뒤, PUCT/MCTS 기반 Monte Carlo Graph Search로 확장하면, sLLM의 final-answer accuracy뿐 아니라 step-wise proof correctness가 개선된다.

### 4.4 Secondary Hypotheses

1. Counterexample-guided rejection은 Toy Case Error와 false lemma expansion을 크게 줄인다.
2. Symbolic certificate gating은 Logical Gap Error를 크게 줄인다.
3. Graph transposition/shared facts는 동일 residual inequality와 proven lemma의 반복 탐색을 줄인다.
4. Verifier-generated positive/negative traces를 이용한 LoRA-SFT/DPO는 sLLM policy의 action quality를 개선한다.
5. 단순 theorem-hint prompting은 theorem misapplication을 유발할 수 있으나, certificate-gated theorem action은 이 문제를 줄인다.

---

## 5. Proposed Method: CPO-MCGS

## 5.1 System Overview

```text
                    ┌──────────────────────┐
                    │   sLLM Policy Model   │
                    │  propose proof action │
                    └──────────┬───────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────┐
│          PUCT / Monte Carlo Graph Search          │
│                                                  │
│  state = obligations + facts + failures + certs  │
│  edge  = proof action                            │
│  node  = canonical residual proof state          │
└──────────┬───────────────────────┬───────────────┘
           │                       │
           ▼                       ▼
┌──────────────────────┐   ┌────────────────────────┐
│ Counterexample Search │   │ Symbolic Cert Verifier │
│ reject false actions  │   │ accept only cert edges │
└──────────┬───────────┘   └──────────┬─────────────┘
           │                          │
           ▼                          ▼
   negative traces              verified transitions
           │                          │
           └──────────┬───────────────┘
                      ▼
          ┌────────────────────────┐
          │ LoRA-SFT / DPO update  │
          │ from positive/negative │
          │ verifier traces        │
          └────────────────────────┘
```

---

## 5.2 Proof-Obligation State

각 graph node는 자연어 문장이 아니라 증명 의무 상태다.

```python
State = {
    "domain": D,
    "obligations": [O1, O2, ...],
    "proven_facts": [F1, F2, ...],
    "failed_facts": [N1, N2, ...],
    "certificates": [C1, C2, ...],
    "equality_candidates": [...],
    "case_coverage": [...]
}
```

### Bound estimation problem

```text
Find largest C such that F(a,b,c) ≥ C for all a,b,c > 0.
```

이를 다음 두 obligation으로 분해한다.

```text
O1: Prove F(a,b,c) ≥ C under a,b,c>0.
O2: Prove sharpness: no larger C is valid.
```

### Relation prediction problem

```text
Determine whether LHS ≤ RHS, LHS ≥ RHS, LHS = RHS, or no fixed relation.
```

이를 다음 obligation으로 바꾼다.

```text
Candidate relation ≥:
  O1: Prove LHS - RHS ≥ 0.

Candidate relation ≤:
  O1: Prove RHS - LHS ≥ 0.

No fixed relation:
  O1: Find counterexample where LHS > RHS.
  O2: Find counterexample where LHS < RHS.
```

---

## 5.3 Action Space

sLLM은 전체 풀이를 자유문장으로 생성하지 않고, 구조화된 proof action을 제안한다.

예시:

```json
{
  "action_type": "apply_theorem",
  "theorem": "Cauchy_Engel",
  "target_obligation": "O3",
  "claim": "sum(a_i^2/x_i) >= (sum a_i)^2 / sum x_i",
  "required_conditions": ["x_i > 0"],
  "expected_residual_obligations": []
}
```

주요 action type:

| Action type | 예시 | verifier requirement |
|---|---|---|
| normalize | homogeneity로 `a+b+c=1` 설정 | scaling invariance certificate |
| apply_theorem | AM-GM, Cauchy, Schur | theorem schema + precondition check |
| decompose | `E = Σ P_i^2 Q_i + R` | algebraic identity + nonnegativity obligations |
| substitute | `x=a/b`, `a=x^2` | domain preservation |
| case_split | `a≥b≥c`, `x≤1/x` | coverage checker |
| introduce_lemma | sublemma 생성 | counterexample + certificate |
| prove_sharpness | equality case, limiting sequence | exact substitution / limit checker |
| disprove_relation | counterexample 제시 | exact/numerical verification |

---

## 5.4 Verifier Stack

Verifier는 단일 LLM judge가 아니라 계층형 hard environment다.

```text
V0: Schema verifier
V1: Counterexample verifier
V2: Symbolic certificate verifier
V3: Obligation update verifier
V4: Final proof verifier
```

### V0. Schema verifier

- action JSON schema 확인
- target obligation 존재 여부 확인
- variable/domain mismatch 확인

### V1. Counterexample verifier

목표는 action claim이 false인지 빠르게 찾는 것이다.

사용 방법:

- random sampling
- boundary sampling
- symmetry-breaking sampling
- positive variable transform: `a = exp(x)`
- constrained numerical optimization
- equality candidate perturbation

중요한 원칙:

```text
counterexample found  → reject action
counterexample absent → do not accept action
counterexample absent → move to certificate verifier
```

Counterexample verifier는 accept verifier가 아니라 **reject verifier**다.

### V2. Symbolic certificate verifier

반례가 없더라도 proof graph에 들어가려면 certificate가 필요하다.

예: AM-GM certificate

```python
Certificate = {
    "type": "theorem_application",
    "theorem": "AM-GM",
    "terms_nonnegative": True,
    "weights_positive": True,
    "weights_sum_to_one": True,
    "before": "...",
    "after": "...",
    "residual_obligations": []
}
```

예: SOS/decomposition certificate

```python
Certificate = {
    "type": "decomposition",
    "identity": "E = (a-b)^2*P + (b-c)^2*Q + R",
    "identity_verified": True,
    "nonnegative_factors": ["(a-b)^2", "(b-c)^2"],
    "residual_obligations": ["P>=0", "Q>=0", "R>=0"]
}
```

핵심 규칙:

```text
LLM: P ≥ 0 is obvious.
Verifier: certificate 없음.
System: prove P ≥ 0를 새 obligation으로 추가.
```

즉 미검증 claim은 폐기되거나 새 obligation으로 남는다.

### V3. Obligation update verifier

검증된 action이 실제로 proof state를 올바르게 갱신하는지 확인한다.

- solved obligation 제거
- residual obligation 추가
- case coverage 갱신
- proven fact 등록
- failed fact 등록
- equality/sharpness condition 갱신

### V4. Final proof verifier

최종 proof accept 조건:

```text
all obligations solved
all transitions have certificate
sharpness obligation solved if bound estimation
no unresolved residual obligations
no action accepted only by numerical evidence
```

---

## 5.5 Monte Carlo Graph Search

### Node and Edge

```text
node = canonical proof-obligation state
edge = verified or partially verified proof action
terminal = all obligations solved with certificates
```

### Selection: PUCT

```text
score(s,a)
= Q(s,a)
+ c_puct * Pθ(a|s) * sqrt(N(s)) / (1 + N(s,a))
+ c_cert * U_cert(s,a)
+ c_novelty * Novelty(s,a)
```

의미:

- `Q(s,a)`: 과거 rollout에서 해당 action이 성공으로 이어진 정도
- `Pθ(a|s)`: sLLM policy가 action을 제안한 확률 또는 rank score
- `U_cert(s,a)`: verifier가 불확실하지만 유망하다고 판단한 action의 exploration bonus
- `Novelty(s,a)`: 같은 theorem/action 반복 방지

### Expansion

sLLM이 현재 state를 입력받아 `K`개의 action 후보를 생성한다.

입력 prompt에는 다음이 포함된다.

- 원문 문제
- 현재 obligations
- proven facts
- failed facts
- equality candidates
- rejected action summaries
- allowed theorem/action schema

### Transition

각 action은 verifier stack을 통과해야 한다.

```text
action invalid schema → reject
counterexample found → reject + negative trace
certificate failed → reject or residual obligation
certificate passed → state update
partial certificate → residual obligation 생성
```

### Backup

terminal success와 verifier-shaped reward를 함께 사용한다.

```text
terminal success:
  all obligations certified → +1

hard failure:
  counterexample found / precondition fail → 0 or -1

partial progress:
  obligation 감소
  residual expression 단순화
  certificate 추가
  false lemma 제거
```

중요한 안전장치:

> shaped reward는 search priority와 value backup에만 사용하고, final proof accept에는 사용하지 않는다. 최종 accept는 certificate 기준이다.

---

## 5.6 Graph Transposition and Shared Facts

순수 tree search는 같은 residual inequality를 여러 번 반복 탐색할 수 있다. 따라서 graph transposition을 사용한다.

동일 node로 merge할 수 있는 예:

```text
E(a,b,c) ≥ 0
expand(E) ≥ 0
factor(E) ≥ 0
λE(a,b,c) ≥ 0, λ>0
E(b,c,a) ≥ 0 under symmetry
```

canonical key:

```python
node_key = canonicalize(
    expression=normalized_expression,
    domain=normalized_constraints,
    symmetry_group=detected_symmetry,
    obligation_type=proof_or_sharpness
)
```

공유되는 정보:

- proven lemmas
- failed lemmas
- counterexamples
- successful decompositions
- theorem applications with certificate
- equality candidates

---

## 5.7 Verifier-Generated Training

Search에서 나온 trace를 버리지 않고 policy improvement에 사용한다.

### Positive traces

- certificate를 통과한 action
- obligation을 실제로 줄인 action
- terminal proof path에 포함된 action

### Negative traces

- counterexample가 발견된 false lemma
- theorem precondition이 실패한 action
- case coverage가 불완전한 split
- sharpness를 증명하지 못한 bound answer
- approximation-only conclusion
- algebraic identity mismatch

### Training method

```text
LoRA-SFT:
  successful action sequence를 supervised fine-tuning

DPO:
  같은 state에서 valid action > invalid action preference 학습
```

초기 검증부터 최소 1회 LoRA-SFT 또는 DPO를 포함한다. 그래야 rStar-Math식 self-evolution 가설과 부합한다.

---

## 6. Research Design Principle

### 6.1 줄이면 안 되는 것

초기 가설검증 단계부터 반드시 포함한다.

1. PUCT/MCTS search
2. proof-obligation state
3. graph transposition / shared facts
4. counterexample-guided rejection
5. symbolic certificate gating
6. verifier-generated negative traces
7. 최소 1회 LoRA-SFT 또는 DPO policy improvement

### 6.2 줄여도 되는 것

초기 연구 범위를 현실화하기 위해 줄일 수 있다.

1. theorem library 크기
2. benchmark 문제 수
3. rollout budget
4. self-evolution round 수
5. Lean/local formal verification 범위
6. non-polynomial inequality coverage

### 6.3 왜 search core를 단순화하지 않는가

연구 가설은 “sLLM + hard verifier + MCTS/graph search가 proof correctness를 개선한다”이다. 만약 초기 검증에서 BFS나 heuristic best-first search로 대체하면, 실패했을 때 다음 ambiguity가 생긴다.

```text
실패 원인 A: 제안 아이디어 자체가 틀림
실패 원인 B: search core를 약화했기 때문에 실패
실패 원인 C: verifier는 맞지만 long-horizon backup이 부족해서 실패
```

따라서 초기 실험은 작은 범위의 full system이어야 한다.

```text
bad initial validation:
  sLLM + simple best-first search + heuristic score

good initial validation:
  sLLM + PUCT/MCGS + counterexample verifier + symbolic certificate verifier
  with limited theorem library and limited benchmark subset
```

---

## 7. Implementation Plan

## 7.1 Backbone Models

초기 대상 모델:

- Qwen2.5-Math-1.5B
- Qwen2.5-Math-7B
- Phi-3-mini 또는 유사 sLLM math-tuned model
- optional: Llama/Mistral 7B math-tuned model

모델 내부 구조는 수정하지 않는다.

```text
Generator: AutoModelForCausalLM
Verifier: symbolic tools + optional PRM/critic
Search controller: pure Python
Training: LoRA-SFT / DPO
```

## 7.2 Minimal Theorem/Action Library

초기 theorem library는 작게 유지하되, search architecture는 full-strength로 유지한다.

초기 포함:

- homogeneity normalization
- AM-GM
- Cauchy-Schwarz / Engel form
- basic Schur
- simple rearrangement
- elementary SOS-like decomposition
- case split with coverage
- equality/sharpness check
- counterexample generation

초기 제외 또는 optional:

- advanced uvw
- Jensen general form
- full CAD
- full Lean formalization
- high-dimensional SDP/SOS

Jensen과 uvw는 theorem misapplication 위험이 크므로 certificate 조건이 충분히 구현된 뒤 추가한다.

## 7.3 Symbolic Verifier Components

사용 가능한 도구:

- SymPy expand/factor/simplify
- polynomial/rational identity check
- random sampling and constrained numerical search
- interval/boundary sampling
- equality candidate perturbation
- simple SOS/decomposition checker
- theorem schema checker
- case coverage checker

## 7.4 Prompt Format

sLLM에게 자유문장 풀이 대신 structured action을 요구한다.

예시 prompt output:

```json
{
  "action_type": "apply_theorem",
  "theorem": "AM-GM",
  "target_obligation": "O2",
  "target_terms": ["a^2/b", "b^2/c", "c^2/a"],
  "claim": "a^2/b + b^2/c + c^2/a >= a + b + c",
  "required_conditions": ["a>0", "b>0", "c>0"],
  "expected_residual_obligations": []
}
```

## 7.5 Pseudocode

```python
def solve(problem, policy_model, value_model, budget):
    root = parse_to_proof_obligations(problem)
    graph = ProofGraph(root)

    positive_edges = []
    negative_edges = []

    while budget.remaining():
        path = select_path_with_puct(graph)
        state = path.leaf

        actions = policy_model.propose_actions(
            problem=problem,
            state=state.to_prompt(),
            k=K,
        )

        for action in actions:
            schema_result = schema_verify(action, state)
            if schema_result.failed:
                negative_edges.append((state, action, schema_result.reason))
                continue

            ce_result = search_counterexample(state, action)
            if ce_result.found:
                negative_edges.append((state, action, ce_result.counterexample))
                backup_failure(graph, path, action)
                continue

            cert_result = verify_certificate_or_generate_obligations(state, action)
            if cert_result.failed:
                negative_edges.append((state, action, cert_result.reason))
                backup_failure(graph, path, action)
                continue

            new_state = update_obligations(state, action, cert_result)
            new_state = canonicalize_and_merge(graph, new_state)
            positive_edges.append((state, action, cert_result))

            reward = compute_progress_reward(state, new_state, cert_result)
            backup_success_or_progress(graph, path, action, reward)

            if new_state.is_terminal_certified():
                proof = assemble_proof(new_state)
                if final_verify(proof, new_state):
                    return proof, positive_edges, negative_edges

    return None, positive_edges, negative_edges
```

---

## 8. Experimental Plan

## 8.1 Datasets

Primary:

- IneqMath dev/test

Secondary:

- LIPS benchmark problems, if accessible
- IneqSearch benchmark problems, if accessible
- curated Olympiad inequality subset
- synthetic polynomial/rational inequality subset for controlled ablation

## 8.2 Baselines

Main baselines:

```text
B0. Direct CoT
B1. Best-of-N / self-consistency
B2. Best-of-N + PRM or LLM verifier reranking
B3. rStar-style MCTS without symbolic certificate gating
B4. theorem-hint prompting
B5. LIPS/IneqSearch, if reproducible or reported comparison is possible
```

Search baseline / ablation:

```text
A1. length-normalized best-first search instead of PUCT/MCGS
```

BFS는 주력 방법이 아니라 ablation이다.

## 8.3 Main Method Variants

```text
M1. CPO-MCGS without policy training
M2. CPO-MCGS + LoRA-SFT from positive traces
M3. CPO-MCGS + DPO from positive/negative action pairs
M4. CPO-MCGS + LoRA-SFT + DPO
```

## 8.4 Ablation Studies

| Ablation | Purpose |
|---|---|
| replace PUCT/MCGS with length-normalized BFS | MCTS/graph search core의 필요성 확인 |
| remove counterexample verifier | false lemma / toy-case error 감소 효과 확인 |
| remove certificate verifier | logical gap 방지 효과 확인 |
| remove graph transposition | shared facts / residual merge 효과 확인 |
| remove negative trace training | verifier-generated preference learning 효과 확인 |
| remove sharpness obligation | bound estimation에서 최적성 증명 중요성 확인 |
| theorem hints only | theorem hint와 certificate-gated theorem action 차이 확인 |
| no case coverage checker | WLOG/case split 오류 증가 확인 |

## 8.5 Metrics

최종 답 정확도만으로는 부족하다.

Primary metrics:

- Answer Accuracy
- Overall Proof Correctness
- False Acceptance Rate
- Certificate Coverage

Failure-mode metrics:

- Toy Case Error Rate
- Logical Gap Error Rate
- Numerical Approximation Error Rate
- Numerical Computation Error Rate
- Theorem Misapplication Rate
- Unresolved Obligation Rate

Search efficiency metrics:

- success per rollout budget
- LLM calls per solved proof
- verifier calls per solved proof
- graph merge ratio
- positive/negative trace ratio
- wall-clock time

Training metrics:

- action validity rate before/after LoRA-SFT
- DPO win-rate on held-out action pairs
- reduction in repeated false action proposals

---

## 9. Expected Results

예상 결과 패턴은 다음이다.

### Final answer accuracy

```text
Direct CoT < Best-of-N < PRM rerank ≈ CPO-MCGS
```

최종 답만 보면 Best-of-N 또는 PRM rerank가 CPO-MCGS와 비슷할 수 있다. 그러나 본 연구의 핵심은 답만 맞히는 것이 아니다.

### Overall proof correctness

```text
Direct CoT << Best-of-N < PRM rerank < rStar-style MCTS < CPO-MCGS
```

### Failure mode reduction

```text
Toy Case Error: 크게 감소
Logical Gap Error: 크게 감소
Numerical Approximation Error: 중간~크게 감소
Numerical Computation Error: 중간 감소
Theorem Misapplication: certificate 구현 범위 내에서 크게 감소
```

### Search/training 효과

```text
CPO-MCGS without training < CPO-MCGS + LoRA-SFT/DPO
```

특히 negative trace training 후에는 다음이 줄어들 것으로 기대한다.

- 잘못된 WLOG
- false lemma 재제안
- theorem precondition 누락
- sharpness obligation 누락
- approximation-only conclusion

---

## 10. Success Criteria

본 연구의 성공 기준은 다음이다.

### Minimum success

- Direct CoT 및 Best-of-N 대비 Overall Proof Correctness 개선
- False Acceptance Rate 감소
- Toy Case Error 또는 Logical Gap Error 중 최소 하나에서 유의미한 감소
- certificate를 가진 proof path를 일정 비율 이상 생성

### Strong success

- rStar-style MCTS without symbolic certificate 대비 Overall Proof Correctness 개선
- theorem-hint prompting 대비 theorem misapplication 감소
- LoRA-SFT/DPO 후 action validity rate 개선
- IneqMath step-wise judge 기준 유의미한 개선

### Paper-worthy success

- sLLM에서 final answer accuracy뿐 아니라 proof correctness가 개선됨을 보임
- counterexample-guided negative trace가 policy improvement에 효과적임을 보임
- certificate-carrying proof-obligation graph가 natural-language CoT의 logical gap을 줄임을 보임
- BFS ablation 대비 PUCT/MCGS의 필요성 또는 장점을 실험적으로 보임

---

## 11. Risks and Mitigations

## 11.1 Symbolic verifier coverage가 좁다

Risk:

- 모든 Olympiad inequality를 AM-GM/Cauchy/SOS schema로 처리할 수 없다.

Mitigation:

- 초기 scope를 polynomial/rational inequality 중심으로 제한한다.
- unsupported theorem은 accept하지 않고 residual obligation으로 남긴다.
- coverage별 subset 성능을 분리 보고한다.

## 11.2 Counterexample search가 반례를 못 찾을 수 있다

Risk:

- 반례가 없다고 참인 것은 아니다.

Mitigation:

- counterexample search는 reject-only verifier로 사용한다.
- accept는 symbolic certificate가 있을 때만 허용한다.

## 11.3 Legitimate symmetry/uvw reduction을 over-reject할 수 있다

Risk:

- 예를 들어 uvw에서 특정 extremal reduction이 정당하지만 certificate가 없어 reject될 수 있다.

Mitigation:

- 초기에는 reject가 아니라 residual obligation으로 전환한다.
- uvw certificate checker는 후속 확장으로 추가한다.

## 11.4 MCTS compute cost가 높다

Risk:

- rollout budget이 커지면 실험 비용이 증가한다.

Mitigation:

- theorem library와 benchmark subset을 줄인다.
- graph transposition과 failed fact cache를 사용한다.
- action generation batch를 최적화한다.

## 11.5 Natural-language proof assembly가 약할 수 있다

Risk:

- certificate graph는 맞지만 최종 사람이 읽는 풀이가 어색할 수 있다.

Mitigation:

- proof correctness metric과 readability metric을 분리한다.
- certificate graph에서 template 기반으로 proof를 assemble한다.

## 11.6 LIPS/IneqSearch와 novelty가 약해 보일 수 있다

Risk:

- “LLM + symbolic inequality prover”라는 큰 틀에서 유사해 보일 수 있다.

Mitigation:

- novelty를 다음 세 가지로 명확히 한다.
  - counterexample-guided negative trace generation
  - certificate-carrying proof-obligation graph transition
  - MCGS + verifier-generated LoRA-SFT/DPO for sLLM policy improvement

---

## 12. Milestones

## Phase 1: Full-architecture MVP

Goal:

- 축소된 theorem library로 CPO-MCGS 전체 구조 구현

Deliverables:

- problem parser
- proof-obligation state
- structured action prompt
- counterexample verifier
- basic symbolic certificate verifier
- PUCT/MCGS controller
- graph transposition
- trace logger

Scope:

- AM-GM, Cauchy, homogeneity, simple decomposition 중심
- IneqMath dev subset

## Phase 2: Verifier-generated training

Goal:

- positive/negative trace로 sLLM action policy 개선

Deliverables:

- positive trace SFT dataset
- negative pair DPO dataset
- LoRA-SFT script
- DPO script
- pre/post action validity evaluation

## Phase 3: Main evaluation

Goal:

- IneqMath dev/test 및 curated subset에서 성능 비교

Deliverables:

- Direct CoT, Best-of-N, PRM rerank, rStar-style MCTS baseline
- CPO-MCGS main results
- ablation results
- failure-mode analysis

## Phase 4: Paper drafting

Goal:

- 논문 초안 작성

Deliverables:

- method section
- experiment section
- ablation analysis
- qualitative proof examples
- limitation section

---

## 13. Repository Structure Proposal

```text
cpomcgs/
  README.md
  configs/
    model.yaml
    search.yaml
    verifier.yaml
  data/
    ineqmath/
    curated/
  src/
    parser/
      problem_parser.py
      obligation_builder.py
    policy/
      action_prompt.py
      generation.py
    verifier/
      schema.py
      counterexample.py
      certificates.py
      theorem_schemas.py
      sharpness.py
    search/
      mcgs.py
      puct.py
      graph.py
      canonicalize.py
    training/
      build_sft_data.py
      build_dpo_pairs.py
      train_lora.py
      train_dpo.py
    eval/
      metrics.py
      failure_modes.py
      run_eval.py
  experiments/
    baselines/
    ablations/
    main/
  outputs/
    traces/
    proofs/
    reports/
```

---

## 14. Paper Framing

### 14.1 Candidate titles

1. **Small Language Models Prove Inequalities with Counterexample-Guided Monte Carlo Graph Search**
2. **CPO-MCGS: Certificate-Carrying Proof-Obligation Search for sLLM Inequality Proving**
3. **From Plausible Solutions to Verified Inequality Proofs: Counterexample-Guided Search with Small Language Models**

### 14.2 Main contribution statements

Contribution 1:

> We formulate informal Olympiad inequality proving as certificate-carrying proof-obligation graph search, where each LLM-proposed action must reduce an explicit obligation or produce a verifiable counterexample.

Contribution 2:

> We introduce a counterexample-guided verifier that turns failed proof actions into structured negative training traces, targeting common inequality-proof failures such as toy-case generalization, unjustified reductions, and false lemmas.

Contribution 3:

> We propose a PUCT-based Monte Carlo Graph Search framework in which sLLMs serve as proof-action policies and symbolic verifiers serve as hard transition gates.

Contribution 4:

> We show that verifier-generated LoRA-SFT/DPO improves sLLM action quality and reduces repeated theorem misapplications.

Contribution 5:

> We evaluate not only final-answer accuracy but also overall proof correctness and failure-mode-specific error rates on IneqMath-style benchmarks.

---

## 15. Key Experimental Claims to Validate

본 연구에서 반드시 검증해야 하는 claim은 다음이다.

1. **MCTS/MCGS necessity**
   - CPO-MCGS가 length-normalized BFS ablation보다 높은 proof correctness 또는 better search efficiency를 보이는가?

2. **Hard certificate necessity**
   - certificate verifier 제거 시 logical gap과 false acceptance가 증가하는가?

3. **Counterexample guidance necessity**
   - counterexample verifier 제거 시 toy-case error와 false lemma expansion이 증가하는가?

4. **Graph transposition necessity**
   - graph merge/shared facts 제거 시 동일 residual obligation 반복과 search cost가 증가하는가?

5. **Training loop necessity**
   - verifier-generated LoRA-SFT/DPO 후 action validity와 proof success가 개선되는가?

6. **sLLM relevance**
   - 큰 모델 없이도 sLLM이 search policy로 충분한 성능 개선을 보이는가?

---

## 16. Final Research Position

본 연구의 핵심 입장은 다음이다.

> 부등식 증명에서 LLM의 한계는 “정답 생성 능력”보다 “미검증 중간 step을 증명처럼 포장하는 경향”에 있다. 따라서 모델을 더 크게 만들거나 CoT를 더 길게 생성하는 것만으로는 충분하지 않다. sLLM을 proof action generator로 제한하고, verifier가 허용한 transition만 graph에 반영하며, MCTS/MCGS가 장기 증명 경로를 탐색하도록 해야 한다.

최종적으로 본 연구는 **부등식용 AlphaGeometry**에 가까운 시스템을 지향한다.

```text
AlphaGeometry:
  LLM proposes auxiliary constructions.
  Symbolic geometry engine verifies facts.
  Search explores construction space.

CPO-MCGS:
  sLLM proposes theorem/decomposition/case-split actions.
  Counterexample + symbolic certificate verifier validates transitions.
  MCGS explores proof-obligation graph.
```

단, 본 연구의 대상은 formal geometry가 아니라 informal Olympiad inequality이다. 따라서 Lean 전체 formalization 대신, 부등식 전용 certificate와 residual obligation graph를 사용한다.

---

## 17. References

1. Guan et al., **rStar-Math: Small LLMs Can Master Math Reasoning with Self-Evolved Deep Thinking**, arXiv:2501.04519.  
   https://arxiv.org/abs/2501.04519

2. Sheng et al., **Solving Inequality Proofs with Large Language Models / IneqMath**, arXiv:2506.07927.  
   https://arxiv.org/abs/2506.07927

3. Hubert et al., **Olympiad-level formal mathematical reasoning with AlphaProof and AlphaGeometry 2**, Nature, 2025.  
   https://www.nature.com/articles/s41586-025-09833-y

4. Chervonyi et al., **Gold-medalist Performance in Solving Olympiad Geometry with AlphaGeometry2**, arXiv:2502.03544.  
   https://arxiv.org/abs/2502.03544

5. Xin et al., **DeepSeek-Prover-V1.5: Harnessing Proof Assistant Feedback for Reinforcement Learning and Monte-Carlo Tree Search**, arXiv:2408.08152.  
   https://arxiv.org/abs/2408.08152

6. Xin et al., **BFS-Prover: Scalable Best-First Tree Search for LLM-based Automatic Theorem Proving**, ACL 2025 / arXiv:2502.03438.  
   https://arxiv.org/abs/2502.03438

7. Li et al., **Proving Olympiad Inequalities by Synergizing LLMs and Symbolic Reasoning / LIPS**, arXiv:2502.13834.  
   https://arxiv.org/abs/2502.13834

8. Ye et al., **Hybrid Reasoning for Olympiad Inequality Proofs / IneqSearch**, OpenReview, 2025.  
   https://openreview.net/forum?id=ft971e7HH5
