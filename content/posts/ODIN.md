+++
title= "ODIN-Detecting Non-crashing Functional Bugs in Android Apps via Deep-State Differential Analysis"
date= 2022-10-14T17:03:56+08:00
math=true
+++

A note for *Detecting Non-crashing Functional Bugs in Android Apps via Deep-State Differential Analysis* by Jue Wang from Nanjing University.

# Introduction

Non-crashing functional bugs are ofen buried in rare program paths, which are difficult to detect but lead to servere consequences.

This paper proposed ODIN, a novel technique to detect non-crashing functional bugs via deep-state differential analysis.

There exists two observations of automatic generated test inputs:

1. Only a small portion of tests input would reach an errorneous app state.
2. Performing the same sequence of actions on similar GUI layouts would produce limited (similar) outcomes, which fits “least surprise” principle of Android app.

Based on observations abouve, **the key idea of this article is clustering GUI states, so that expected behaviors are of majority large clusters while anomalies are in small clusters**.

However, there are two key issues should be concerned here:

1. As exisiting coverage-directed test input generators tend to produce traces with erroneuous occurances, one should consider **how to ensure the sufficiency of expected behaviors, so that they account for the majority.**
2. As GUI layouts often contains rich but redundant, or even non-deterministic information, **one should find a way to abstract the GUI states, so that they can be correctly clustered.**

For the first issue, ODIN follows observation 2, and enrich test cases by appending the same events to them, and further propose a calibration procedure to ensure only small portion of them are anomalies.

For the second issue, ODIN performs a GUI-abstractoin-based hierarchical clustering by gradually merging similar clusters by refining GUI abstraction criteria, so that expected behaviors are in large clusters (the author named them *May Beliefs*), while anomalies are in small clusters. Note that each cluster starts with a transition between two layouts.

# Workflow

![https://tva1.sinaimg.cn/large/008vxvgGly1h72g9igm8tj31ww0gugqc.jpg](https://tva1.sinaimg.cn/large/008vxvgGly1h72g9igm8tj31ww0gugqc.jpg)

The input of ODIN is a set of execution traces produced by test case generators, while the output is bug reports. Besides, ODIN has four internal components:

1. Mining GUI model to build a graph, which consists of states (similar GUI layouts) and transitions (events).
2. Calibrating generated test inputs to promote the dominance of expected behaviors (i.e. to be majority).
3. Manifesting app behaviors by appending a single event to all inputs to yield new execution traces, which futher expand the state-transition space.
4. Mining may beliefs by hierachical clustering each abstracted state transition, and find defects in small clusters.

# Notations

Andriod apps are GUI-contered and event-driven. $P$ represents a android app, the GUI layout $l$ of specific app status $s$ of $P$ can be marked as $l=L(s)$.

Each layout $l$ is of a tree structure, in which each node $w\in l$ is a GUI widget (such as a button).

Each widget $w$ has mutiple attributes, such as widget types ($w.type)$, displayed text ($w.text$, and $w.text=\bot$ means no text displayed).

An GUI event $e=\langle t,r\rangle$ is consists of $e.t$ as event type and $e.r$ as event receiver. An event type can either be *click, long-click, swipe*, and the receiver $r(l)=w$ denotes widget $w$ in layout $l$ can be triggered by event $e$.

A test input is a event sequance $[e_1,e_2,\dots,e_n]$ which yields an execution trace $\tau=\langle s_0\xrightarrow{e_1}s_1\xrightarrow{e_2}\dots\xrightarrow{e_n}s_n\rangle$. Correspondingly, the GUI execution trace is $L=\langle l_0\xrightarrow{e_1}l_1\xrightarrow{e_2}\dots\xrightarrow{e_n}l_n\rangle$, where $l_i=L(s_i)$.

# GUI Model Mining

ODIN mines app’s GUI model from the massive and automatically generated test inputs by grouping similar GUI layouts as GUI model ***states***, and adding ***transitions*** based on the input event between GUI layouts pairs. Note that GUI model state $\sigma$ (groups similar layouts) is different from app status $s$ (represents specific app enviroment).

***Similarity Definition.*** Specifically, if two GUI layouts handle the same set of events, ODIN considers them as similar layouts and can be grouped into a specific state.

Therefore, based on a set of test input, ODIN can build a graph which consists of GUI states (each one with a set of layouts) and transitions (events).

Formulaically, given the execution trace set $T=\{\tau_1, \tau_2,\dots \tau_n\}$ whose corresponding GUI execution trace set $T_l=\{L_1,L_2,\dots, L_n\}$, its corresponding GUI model can be represented as $G(V,E,\delta)$. Concretely, $V$ is a set of all mined GUI model states ($\{\sigma |\sigma \in V\}$), which seperate all GUI layouts into individual parts. $E$ is the set of events. $\delta:V\times E \rightarrow V$ are transitions in $G$.

The mining procedure can be represented in the following Python pseudocode:

```python
def mine_model(layout_traces):
    """
    layout_traces: All GUI execution traces, consists of n GUI layout sequences.
    """
    v, e, delta = {}, {}, {}
    # initial
    for trace in layout_traces:
        v = set_union(v, {l for l in trace.layouts})
        e = set_union(e, {event for event in trace.events})
        delta = set_union(delta, {(s1, event, s2) for (s1, event, s2) in delta})

    # mine states
    for s1, s2 in combination(v, v):
        # try merging every combination of states in v
        if s1 == s2:
            continue
        new_v, new_delta = merge(s1, s2, v, delta)
        if new_v and new_delta:
            v, delta = new_v, new_delta
    
    return v, e, delta

def merge(s1, s2, v, delta):
    if all_layouts_similar(s1, s2):
        # remove s1 and s2, then add the union of s1 and s2
        new_v = set_remove(v, [s1, s2])
        new_s = set_union(s1, s2)
        new_v = set_union(v, new_s)
        # combine the transitions about s1 and s2 into a new_s
        new_delta = update_transitions(delta, [s1, s2], new_s)
        t = get_transitions(delta, new_s)
        # explore new states to be merged
        for (ns, event, s1), (ns, event, s2) in combination(t):
            if s1 == s2:
                continue
            # DFS
            new_v, new_delta = merge(s1, s2, new_v, new_delta)
            if not new_v or not new_delta:
                return None
        return new_v, new_delta
    return None
```

# Calibration

Those test cases are generated from exisiting automatic test generating techniques, which tends to yield trace that cause errors. To enrich the test cases with expected behaviors, ODIN apply a calibration procedure.

Given a GUI model state $\sigma$, ODIN simulates a random walk on the GUI model and find a random path terminating at $\sigma$. The random walk forces all outgoing transitions from the same state to be chosen in equal probability. 

For instance, if $\sigma$ has $m$ following states $\Sigma=\{\sigma_1,\dots,\sigma_m\}$, each following state has a probability of $\frac{1}{|\Sigma|+1}$ to be chosen, and so for $\sigma$ itself (a aditional a self-transition for calibration procedure).

By repeatedly selecting $\sigma$ that seldomly visited and applying random walk, ODIN generete sufficient and diverse test inputs for each GUI model state. The concrete procedure can be represented in the following Python pseudocode:

```python
def calibrate(traces):
    """
    traces: a set of state sequences based on test input and GUI model.
    """
    expanded_traces = {}
    start_state = traces[0][0]  # an arbitrary start state
    while len(expanded_traces) < SUFFICIENT_TRACES:
        # find state appears in least trace
        sigma = find_least_visited_state(traces)
        # apply random walk start from an arbitrary state
        # find a trace that can be reach sigma
        p = random_walk(p=[start_state], terminate=sigma)
        if p:
            # convert state-event sequence to layout-event sequence
            # ** this may generate layout trace that not avaliable in real app
            p_input = to_input(p)  
            expanded_traces = set_union(expanded_traces, p_input)

def random_walk(p, terminate):
    if len(p) >= LIMIT_TRACE_LEN:
        return p
    
    # find all avaliable next states of the last state in p
    dests = find_dest(p[-1])
    if p[-1] == terminate:
        dests = set_union(dests, {p[-1]})

    for d in dests:
        if d == p[-1] and p[-1] == terminate:
            new_p = p
        else:
            new_p = random_walk(p + {d}, terminate)
        if new_p:
            return new_p
    return None
```

## Feedback Loop

It is worth noting that, though we can losslessly get state-event sequences from layout-event sequences, we can not do it conversely. In other words, given a state-event sequance, we can randomly choose a layout for each state, but can not guarantee the event can be triggered between this layouts in real world app (as `to_input(p)` function in pseudocode).

Formulaically, ODIN obtaines transitional path $p=\langle \sigma_0,\sigma_1,\dots, \sigma_{|p|}\rangle$ from random walk, but in corresponding actual execution trace and its transitional path $p'=\langle \sigma_0',\sigma_1',\dots, \sigma_{|p|}'\rangle$, there exists $\sigma_i=\sigma_i'$ but $\sigma_{i+1}\ne\sigma_{i+1}'$.

To tackle this issue, ODIN record the actual GUI execution trace, adding it into test input and re-mine a new GUI model.

# Manifesting App Behaviors

Calibration produce a set of sufficient test input, and we can reasonably assume that only a small fraction of the inputs terminate with an errorneous internal state.

However, it’s difficult to directly using these internal state for clustering, due to the low-level representation of data within them (like serialized heap objects).

The auther observed that an internal state $s$ can be characterized by its future behaviors. They assume that for a state transition sequance $s_0\xrightarrow{e_1}s_1\xrightarrow{e_2}\dots\xrightarrow{e_{k-1}}s_{k-1}\xrightarrow{e_k}s_k$ and $s_k$ indicates a anomaly GUI layout, the calibrated inputs should contains sufficiently many test inputs that terminated with $s_{k-1}$. Therefore, ODIN append different single event $e_k$ to such inputs, **hoping it produce some more of expected behaviors and less of anomaly results.**

Concretely, given a GUI model state $\sigma$ which responds to $e$, ODIN finds all inputs trace that reach $\sigma$ (e.g. $[e_1,e_2,\dots ,e_n]$) and append $e$ to yield new execution trace

$$
\tau^+=\langle s_0\xrightarrow{e_1}s_1\xrightarrow{e_2}\dots\xrightarrow{e_n}s_n\xrightarrow{e}s_{n+1}\rangle
$$

The last state transition $\langle L(s_n),L(s_{n+1})\rangle$ represents the manifested app behaviour, which would be used for the following hierarchical behaviour clustering.


> One may consider that what is difference between calibration and manifesting app behaviors. **In my opinion**: 
> - calibration investigates possible execution trace in terms of **breadth**, which expand sufficient many test input.
> - While manifesting app behaviors focuses on exploring various behaviors on specific GUI model states, which reveal the expected layouts (which occupy the majority) and anomaly layouts (which occupy the minority).

# Mining May-beliefs

After manifesting with $e$, we can cluster the corresponding layouts of transitions $s\xrightarrow{e} s'$. However, GUI layouts often contrain rich but redundant information. It’s important to adopt proper similar criteria for clustering.

To this end, ODIN defines a set of abstraction rule and iteratively select one of them during hierarchical clustering. Furthermore, ODIN measures the similarity bwteen two clusters by comparing the fingerprints of the contained GUI layout pairs.

Given a set of layout pairs $B=\{\langle l_1,l_1'\rangle,\langle l_2,l_2'\rangle,\langle l_3,l_3'\rangle\}$, the clustering procedure can be described as:

1. Initialize all pairs into individual clusters.
2. Iteratively select one abstraction rule $R$ (that have not been chosen and leads to minimal cluster merging).
3. Apply to layout pairs (clusters). 
4. Get the ***fingerprint*** of each cluster with $R$.
5. Merge clusters with similar fingerprint.
6. Repeat 2 to 5 until all rules have been adopted, or only one cluster remains, or an anomaly is found.

Specifically, for step 2, there are three abstraction rules can be applied:

1. Set all *attr* of widget to be empty.
2. Remove widgets that do not receive any event in GUI layouts.
3. Remove duplicate sub-trees in each layout, e.g. remove duplicate items in ListView.

For step 4, to extract fingerprint of a cluster $C$, ODIN randomly select one layout pair ($\langle l_1, l_2\rangle \in C$), applying abstraction rules and calculate the differential between the anstracted layouts as the fingerprint. 

Specifically, for original pair $\langle l_1, l_2\rangle$ and abstracted $\langle l_1', l_2'\rangle$, ODIN seperately calculate **tree editing distance** $\Delta$ between $l_1'$ and $l_2'$. ODIN regard $\Delta$ as fingerprint of cluster. If two clusters have equal fingerprint, i.e. $\Delta_1=\Delta_2$, they should be merged.

The clustering procedure can be described as following pseudocode:

```python
def clustering(pairs):
    """
    pairs: layout pairs generated by manifesting app behaviors.
    """
    rules = {}
    clusters = {p for p in pairs}  # each pair is a individual cluster at first
    while len(clusters) > 1 and len(rules) < N_RULES:
        r = select_rule(clusters)
        rules = set_union(rules, r)
        for c1, c2 in combination(clusters):
            delta1 = fingerprint(c1, r)
            delta2 = fingerprint(c2, r)
            if delta1 == delta2:
                merged = set_union(c1, c2)
                clusters = set_remove(clusters, [c1, c2])
                clusters = set_union(clusters, merged)
						
						# mention in the next section
            anomaly = detect_anomaly(clusters)
            if anomaly:
                return anomaly
    return None

def fingerprint(c, r):
    l1, l2 = random_choice(c)
    ab1 = abstract(l1)
    ab2 = abstract(l2)
    delta = tree_edit_dist(ab1, ab2)
    return delta
```

# Detecting Anomaly

In the pseudocode of clustering, we have a `detect_anomaly` function. Given a set of clusters $\mathbb{C}=\{C_1,C_2,\dots,C_m\}$, ODIN conducts a z-score-based analysis to identify anomaly clusters.

Idealy, the z-score of a cluster $C\in \mathbb{C}$ is

$$
z_C=\frac{|C|-\mu_\mathbb{C}}{s_\mathbb{C}}
$$

where $\mu_C$ and $s_C$ represent the average size and standard deviation of all clusters respectively. However, anomaly clusters often has a small size which can largely affect $\mu_\mathbb{C}$ and $s_\mathbb{C}$. Therefore, ODIN select majorigy subset $\mathbb{C}'\in\mathbb{C}$ that occupiy the $\frac{3}{4}$ size of $\mathbb{C}$. One can sort clusters and select top-k of them that reach $\frac{3}{4}$ of all layouts. Then the $\mu_{\mathbb{C'}}$ and $s_{\mathbb{C'}}$ are used for calculating z-score.

Finally, ODIN considers clusters that $z_C\ge3$ and $|C|<\mu_{\mathbb{C}'}$ to be anomaly while others are may beliefs.