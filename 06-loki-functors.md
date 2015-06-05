---
layout: page
title: First steps in LHCb
subtitle: Fun with LoKi Functors
minutes: 15
---

> ## Learning Objectives {.objectives}
>
> * Understand what LoKi functors are
> * Use LoKi functors interactively
> * Be able to find functors that do what we want

LoKi functors are designed to flexibly compute and compare properties of the current decay, from simple quantities such as the transverse momentum of particle to complicated ones like helicity angles.
Internally, functors are implemented as C++ classes that take an object of type `TYPE1` and return another of `TYPE2`.
They can be used both in C++ and in python code, and can be combined with each other using logical operations.

According to `TYPE2` there are 3 types of functors:
 
 - *Functions*, which return `double`.
 - *Predicates*, which return a `bool`.
 - *Streamers*, which return a `std::vector` of some other type `TYPE3`.

When filling tuples, the most used functors are functions, while predicates are typically used for selections.

According to `TYPE1`, there are many types of functors, the most important of which are (you can find a full list in the [LoKi FAQ](https://twiki.cern.ch/twiki/bin/view/LHCb/FAQ/LoKiFAQ#How_to_code_own_LoKi_functor)):

 - *Particle functors*, which take `LHCb::Particle*` as input. 
 - *Vertex functors*, which take `LHCb::VertexBase*` as input.
 - *MC particle functors*, which take `LHCb::MCParticle*` as input. 
 - *MC vertex functors*, which take `LHCb::MCVertex*` as input.
 - *Array particle functors*, which take a `LoKi::Range_` (an array of particles) as input.
 - *Track functors*, which take `LHCb::Track` as input.

To understand what we can do with LoKi functors, we will start with exploring a [DST interactively](05-interactive-dst.html).
Open the DST and get the first candidate in the `D2hhCompleteEventPromptDst2D2RS` line:

```python
cands = evt['/Event/AllStreams/Phys/D2hhCompleteEventPromptDst2D2RSLine/Particles']
cand = cands[0]
```

We can now try to get very simple properties of the B candidate, such as its transverse momentum or measured mass.

```python
from LoKiPhys.decorators import PT, MM
print PT(cand)
print MM(cand)
```

You will see an error when loading the functors
```
LoKiSvc.REPORT      ERROR LoKi::AuxDesktopBase: 	loadDesktop(): unable to load IPhysDesktop! StatusCode=FAILURE
LoKiSvc.REPORT      ERROR The   ERROR message is suppressed : 'LoKi::AuxDesktopBase: 	loadDesktop(): unable to load IPhysDesktop!' StatusCode=FAILURE
```
This is related to the fact that some functors need to run in the `DaVinci` scope and they are all loaded in the `LoKiPhys.decorators` module, but it's harmless in the examples we will use.
Additionally, if the import is made *before* the instantiation of the `ApplicationMgr`, there will be no warnings.

Math operations are also allowed:
```python
from LoKiPhys.decorators import PX, PY, PZ
p_components_sum = PX + PY + PZ
p_components_sum(cand)
```

If we want to get the properties of the B vertex, for example its $\chi^2$, we need to pass the correct object to the LoKi functors

```python
from LoKiPhys.decorators import VCHI2
print VCHI2(cand.endVertex())
```

This is inconvenient when running from python options, since in that case we don't have any way of calling the `endVertex` method. 
In that case, we can use the `VFASPF` adaptor functor, which allows to use vertex functors as if they were particle functors (note how the functor is built by combining two functors)

```python
from LoKiPhys.decorators import VFASPF
VCHI2(cand.endVertex()) == VFASPF(VCHI2)(cand)
```

The calculation of some of the properties, such as the IP or DIRA, require the knowledge of the primary vertex associated to the candidate.
In `GaudPython`, we can get the PVs ourselves.

```python
pv_finder_tool = appMgr.toolsvc().create('GenericParticle2PVRelator<_p2PVWithIPChi2, OfflineDistanceCalculatorName>/P2PVWithIPChi2', interface='IRelatedPVFinder')
pvs = evt['/Event/AllStreams/Rec/Vertex/Primary']
best_pv = pv_finder_tool.relatedPV(cand, pvs)
from LoKiPhys.decorators import DIRA
print DIRA(best_pv)(cand)
```

Given that this is a very common operation, we have the possibility of using, in the context of a `DaVinci` application (Stripping, for example), a special set of functors, starting with the `BPV` (for Best PV) prefix, which get the PV for us.
Some functors also end with the suffix `DV`, which means they can only be used in the `DaVinci` context.

> ## Finding LoKi functors {.callout}
> The full list of defined LoKi functors can be found in the `LoKi::Cuts` namespace in the [doxygen](http://lhcb-release-area.web.cern.ch/LHCb-release-area/DOC/davinci/releases/latest/doxygen/d7/dae/namespace_lo_ki_1_1_cuts.html).
> They are quite well documented with examples on how to use them.
> The list can be overwhelming, so it's also worth checking a more curated selection of functors in the TWiki, [here](https://twiki.cern.ch/twiki/bin/view/LHCb/LoKiHybridFilters) and [here](https://twiki.cern.ch/twiki/bin/view/LHCb/LoKiParticleFunctions).

So far we've only looked at the properties of the head of the decay (that is, the D\*), but what if we want to get information of its daughters? As an example, let's take get the largest transverse momentum of the final state particles.
A simple solution would be to navigate the tree and calculate the maximum `PT`

```python
def find_tracks(particle):
    tracks = []
    if particle.isBasicParticle():
        proto = particle.proto()
        if proto:
            track = proto.track()
            if track:
                tracks.append(particle.data())
    else:
        for child in particle.daughters():
            tracks.extend(find_tracks(child))
    return tracks

max_pt = max([PT(child) for child in find_tracks(cand)])
```

However, LoKi offers functions for performing such operations, namely `MAXTREE` and `MINTREE`, which get as parameters the selection criteria, the functor to calculate and a default value.
In our example,

```python
from LoKiPhys.decorators import MAXTREE, ISBASIC, HASTRACK
MAXTREE(ISBASIC & HASTRACK, PT, -1)(cand) == max_pt
```

In this example, we have used two selection functors, `ISBASIC` and `HASTRACK`, which return true if the particle doesn't have children and is made up by a track, respectively.
We can see that they do the same thing as `particle.isBasicParticle()` and `particle.proto().track()` in a more compact way.

Similarly, the `SUMTREE` functor allows us to accumulate quantities for those children that pass a certain selection:
```python
from LoKiPhys.decorators import SUMTREE, ABSID
print SUMTREE(211==ABSID, PT)(cand)
print SUMTREE('pi+'==ABSID, PT)(cand)
```
In this case, we have summed the transverse momentum of the pions in the tree.
Note the usage of the `ABSID` functor, which allows to select particles of a given ID using either their PID or their name.

Another very useful LoKi functor is `CHILD`, which allows to access a property of a single children of the particle.
To specify which child we want, its order is used, so we need to know how the candidate was built.
For example, from
```output
In [10]: cand.daughtersVector()
Out[10]:

 0 |->D0                           M/PT/E/PX/PY/PZ: 1.8624/ 6.4521/ 47.44/-4.939/-4.152/ 46.96 [GeV]  #  0
                                       EndVertex  X/Y/Z:0.2911/-0.2378/-14.38 [mm]  Chi2/nDoF 0.4039/1 #  0
 1    |->K-                        M/PT/E/PX/PY/PZ: 0.4937/ 2.8013/ 25.45/-1.799/-2.147/ 25.29 [GeV]  # 19
 1    |->pi+                       M/PT/E/PX/PY/PZ: 0.1396/ 3.7258/ 21.99/-3.141/-2.004/ 21.67 [GeV]  # 22
 0 |->pi+                          M/PT/E/PX/PY/PZ: 0.1396/ 0.3701/ 2.678/-0.2873/-0.2333/ 2.649 [GeV]  # 10
```
we know that `D0` is the first child and `pi+` is the second.
Therefore, to access the `PT` of the `D0` we have 2 options
```python
from LoKiPhys.decorators import CHILD
mass = MM(cand.daughtersVector()[0])
mass_child = CHILD(MM, 1)(cand)
mass == mass_child
```

The usage of LoKi functors extends much further than in the interactive `GaudiPython` world.

They constitute the basis of particle filtering in the *selection framework*, discussed in the [Building your own decay chain](06-building-decays.html) lesson.
Selecting particles means using LoKi *predicates*, ie, functors that give a `bool` output, which we have not discussed so far.
Amongst these, a key expression is `in_range`, which returns `True` if the value of the given *function* functor (that is, the functor that returns a `double`) is within the given lower and upper limit.
It helps writing CPU-efficient functors and thus is very important when building time-critical software like trigger or stripping lines.

```python
from LoKiCore.functions import in_range
in_range(2000, MM, 2014)(cand)
in_range(1860, CHILD(MM, 1), 1870)(cand)
```

Additionally, LoKi functors can be used directly inside our `DaVinci` jobs to store specific bits of information in our ntuples without the need for a complicated C++-based algorithms;
this second option will be discussed in the [TupleTools and branches lesson](12-add-tupletools.html).

> ## Debugging LoKi functors {.callout}
> If you write complicated LoKi functors, typically in the context of selections, you need functions for debugging when things go wrong.
> LoKi provides wrapper functors that evaluate a functor (or functor expression), print debugging information and return the result;
> the most important of these are:
>
>  - `dump1`, which prints the input object and returns the calculated functor value,
    ```python
    from LoKiCore.functions import dump1
    debug_p_components_sum = dump1(p_components_sum)
    debug_p_components_sum(cand)
    ```
>  - `monitor` which prints the input the functor string and returns the calculated functor value,
    ```python
    from LoKiCore.functions import monitor
    monitor_p_components_sum = monitor(p_components_sum)
    monitor_p_components_sum(cand)
    ```


