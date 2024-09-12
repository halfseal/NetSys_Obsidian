---
creator: 최민규
type: Glossary
created: 2022-05-10
---
<<[[NETSYS_Test/Undergrad Study I/NUMA|previous]]>>
## NUMA

CPU & memory → interconnected nodes

page mapping optimally?

## Problem

- Uniform-workers strategy
    - memory intensive → set of worker nodes → BW = bottleneck
    - recommended, default option for DB...
    - problem
        1. place pages in symmetric ratios
            
            BW normally asymmetric
            
        2. memory idle → BW decreases
            

## BWAP

: BW aware page placement

BW 비대칭 고려, “application-specific weighted interleaving“

Optimal placement: lower-BW memory → fewer pages

모든 메모리가 코어에 똑같은 BW 제공 (UMA일때)에 국한.

NUMA에선? thread 위치 따라 bw 제각각.

Contribution

- memory-intensive app에서 page placement 전략 평가
- analytic modelling + on-line iterative tuning

Discussion

What is hill climbing solution - scalar coefficient