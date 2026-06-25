---
layout: post
mermaid: true
---

## Insights into reproducing steelmaking scheduling literature models

In this first blog post, I explore a tricky aspect within the field operations research: reproducing the work of other researchers. For that, I will use a recent paper from steelmaking-continuous-casting (SCC) literature:

* Juntaek Hong, Kyungduk Moon, Kangbok Lee, Kwansoo Lee & Michael L. Pinedo (2022) An iterated greedy matheuristic for scheduling in steelmaking-continuous casting process, International Journal of Production Research, 60:2, 623-643, DOI: 10.1080/00207543.2021.1975839

My goal is not to reproduce their results exactly or verify their clams, rather illustrate the effort and pitfalls of reproducing literature results. In the future, I want to integrate such model with ladle dispatching decisions -- as a way to answer some questions from my research that I still want to investigate. Moreover, I am restricting myself to use open-source solvers, also as a way to understand how competitive they are for the SCC problem.

---

### Problem description

First things first, I will provide a brief introduction to the problem at hand. A more complete description can be found in the paper used throughout this post.

Steel is produced in two main routes: the integrated and electric routes. In the integrated route, hot metal is charged into a BOF (Basyc Oxygen Furnace) along with a small percentage of scrap to produce crude steel heat. Alternatively, scrap can also be molten into an EAF (Electric Arc Furnace) with the same end goal -- this is the electric route. 

The crude steel is then tapped into a steel ladle and proceeds to the secondary refining stages. It can include a stirring station, ladle furnace and vaccum degasser, for example. A steel grade defines the temperature and quality specifications, which in turn defines which routes (sequence of stages) a heat must undergo until casting. Finally, the continuous casting process is reached, where the molten steel is loaded into a tundish and then solidified into a water-cooled mold to produce slabs. 

<div class="mermaid">
---
title: The steel processing stages
config:
  layout: dagre
---
%%{init: {'theme': 'forest'}}%%

flowchart LR
 subgraph Steelmaking["Primary Refining"]
    direction LR
        BOF["BOF"]
        EAF["EAF"]
        CrudeSteel["Crude Steel"]
  end
    Ladle["Ladle"]
 subgraph Refining["Secondary Refining"]
    direction TB
        Stirring["Stirring Station"]
        LF["Ladle Furnace"]
        VD["Vacuum Degasser"]
    RefinedSteel["Steel"]
  end
 subgraph Casting["Continuous Casting"]
    direction LR
        CastingProcess["Caster"]
        Slabs["Slabs"]
  end
    BOF --> CrudeSteel
    EAF --> CrudeSteel
    CrudeSteel --> Ladle
    Ladle --> Stirring
    Ladle --> LF
    Ladle --> VD
    Stirring --> RefinedSteel
    LF --> RefinedSteel
    VD --> RefinedSteel
    RefinedSteel --> CastingProcess --> Slabs

     CrudeSteel:::output
     RefinedSteel:::output
     Slabs:::output
     
    classDef output fill:#f7fee7,stroke:#a3e635,color:#000,font-weight:bold
    style Steelmaking fill:#f0fdfa,stroke:#2dd4bf
    style Refining fill:#f5f3ff,stroke:#a78bfa
    style Casting fill:#f0fdf4,stroke:#4ade80
    style Ladle fill:#eef2ff,stroke:#818cf8
</div>

A particularity of the SCC scheduling problem is that heats in a cast must be processed continuously -- which makes it more complicated than typical flow-shop scheduling formulation. Several formulations of this problem have been proposed in the literature. Heuristics are mostly employed to obtain high-quality solutions for practical instances. 

### Step 1: the main MIP problem

### Step 2: heuristics

### Conclusions

Answering 

As a side note, something that I want to attempt in a few years is to reproduce one of my own papers. 