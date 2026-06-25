---
layout: post
mermaid: true
---

## Insights into reproducing steelmaking scheduling literature models

In this first blog post, I explore a tricky aspect within the field operations research: reproducing the work of other researchers. For that, I will use a recent paper from steelmaking-continuous-casting (SCC) literature:

* Juntaek Hong, Kyungduk Moon, Kangbok Lee, Kwansoo Lee & Michael L. Pinedo (2022) *An iterated greedy matheuristic for scheduling in steelmaking-continuous casting process*, International Journal of Production Research, 60:2, 623-643, DOI: 10.1080/00207543.2021.1975839

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

### MIP implementation

Papers tipically start defining a mathematical description of the problem: sets, parameters, decision variables, constraints and objectives. Implementation is usually straightforward from description using any modeling language. I personally became quite comfortable with `pyomo`, since it does not bind me to a specific solver interface. Of course, it adds an overhead each time an instance is created and miss some advanced features usually available in the solver's native interface. 

#### Input data structure

Before implementing the mathematical model, I like to create a consistent data model that is easy to serialize in a text file (JSON, YAML, etc.) and represents the problem's structure. `pydantic` is very interesting for this step, especially because python does not strongly enforce types. 

Below is an example used for implementing the SCC problem. First, notice that there is a clear connection to the actual notation and parameters used in the manuscript. Moreover, the data transformation logic is contained within the data classes. Why is this important? This seggregates data pre-processing from modeling logic, making each part easier to test and maintain individually. This prevents some obscure data processing logic to be added when writing a mathematical expression, for example.  

```python
from pydantic import BaseModel

class Machine(BaseModel):
    id: int
    processing_time: int = 0
    transportation_time: dict[int, int] = {}

class Stage(BaseModel):
    id: int
    machines: list[Machine]

    @property
    def machine_ids(self) -> list[int]:
        """Return a list of machine IDs for this stage."""
        return [machine.id for machine in self.machines]

    @property
    def transportation_times(self) -> dict[tuple[int, int], int]:
        """Return a mapping of (from_machine_id, to_machine_id) to transportation time for this stage."""
        times = {}
        for machine in self.machines:
            for to_machine_id, time in machine.transportation_time.items():
                times[(machine.id, to_machine_id)] = time
        return times

class SCCData(BaseModel):
    casts: dict[int, Cast] = {}
    stages: dict[int, Stage] = {}
    penalty_tardiness: float
    penalty_earliness: float
    penalty_idle_time: float
    penalty_cast_break: float
    max_wait_time: int

    @property
    def charge_indexes(self) -> list[int]:
        """Return a list of all charge IDs."""
        return [charge.id for cast in self.casts.values() for charge in cast.charges]

    def charge_stages(self, charge_id: int) -> list[int]:
        """Return a list of stage IDs for the given charge ID."""
        for cast in self.casts.values():
            for charge in cast.charges:
                if charge.id == charge_id:
                    return [stage.id for stage in charge.steel_grade.route]
        raise ValueError(f"Charge ID {charge_id} not found in any cast.")
```


#### Working with pyomo Sets

A `pyomo` feature that I particularly enjoy is to define and use sets for creating the model. For me, the hardest part is actually understanding and getting familiar the mathematical notation. Using set notation is quite handy in translating a mathematical expession into python code. I will use Equation 4 from the paper:

$$X_{kk^′l} + X_{k^′kl} \leq 1 - (Y_{ikl} - Y_{ik^′l}), \:  \forall k, k^′ \in \Omega, k \neq k^′,  l \in \mathcal{S}_k \cap \mathcal{S}_{k′} , i \in M_l$$

It is hard to get lost in the notation, so in practice we can separate the sets from the mathematical expressions and keep the code cleaner. Moreover, it is also possible to reuse the same set for creating similar constraints (in this example, Equation 3):

```python
# define the set of machine indexes (M)
model.CHARGE_CHARGE_STAGE_INTERSECTION = pyo.Set(
            initialize=[
                (charge1, charge2, stage_id, machine_id)
                for charge1 in data.charge_indexes
                for charge2 in data.charge_indexes
                for stage_id in data.stages.keys()
                for machine_id in data.machine_indexes
                if stage_id in data.charge_stages(charge1)
                and stage_id in data.charge_stages(charge2)
                and machine_id in data.machine_indexes_per_stage[stage_id]
                and charge1 != charge2
            ]
        )

# add constraints (3) to the model
model.constraint3 = pyo.Constraint(
            model.CHARGE_CHARGE_STAGE_INTERSECTION,
            rule=lambda m, k1, k2, s, i: (
                (m.x[k1, k2, s] + m.x[k2, k1, s] >= m.y[i, k1, s] + m.y[i, k2, s] - 1)
                if k1 < k2
                else pyo.Constraint.Skip
            ),
        )

# add constraints (4) to the model
model.constraint4 = pyo.Constraint(
            model.CHARGE_CHARGE_STAGE_INTERSECTION,
            rule=lambda m, k1, k2, s, i: (
                m.x[k1, k2, s] + m.x[k2, k1, s] <= 1 - (m.y[i, k1, s] - m.y[i, k2, s])
            ),
        )

```

Of course, this is one approach and I am pretty sure there are more efficient ones. As a key takeaway, using a modeling framework that supports working with sets is very important in this task. 

### Validation

### Results

### Conclusions

Answering 

As a side note, something that I want to attempt in a few years is to reproduce one of my own papers. 