+++
title = "Multi Level GUI Comparison Criteria"
date = 2022-08-07T22:12:43+08:00
math = true
tags = ['Software Analyze and Testing']
+++

Modern Android apps contain a number of dynamically constructed GUIs, which make accurate behavoir modeling more challenging. [Baek and Bae](https://dl.acm.org/doi/10.1145/2970276.2970313) proposed a set of multi-level GUI Comparison Criteria(GUICC) ,which provides multiple abstraction levels for GUI model generation.

A GUI model is a event-driven transitions beteen GUI states, naturally described in a graph formulation. We call a GUI modal as a “GUI graph”.

**Definition.** A GUI graph $G$ is a directed graph that consist of **distinct** GUI states as “***ScreenNodes”*** ($S$ for short), which distinuished by a specific GUI comparison criterion. Transactions between GUI states are triggerd by events as ***“EventEdges”*** ($E$ for short). Thus, $G$ can be dfined as $G=(S, E)$, where $S=\{s_1, s_2,\dots,s_n\}$ and $E=\{e_1,e_2,\dots,e_n\}$.

- A ScreenNode $S$ represents a GUI state that contains abstracted GUI information of an excution screen.  ScreenNodes are distinct fomr each other based on GUICC.
- A GUI Comparison Criterion (GUICC) represents a specific type of GUI information to determine equivalence or difference between GUI states.
- An EventEdge $E$ represents a single event that can be triggered both by a user (e.g. click event) and a system (e.g. phone calls). Each $E$ links between ScreenNodes.

A model-based testing tool repeatedly excutes test inputs on a running app, and analyzes the excution results to generate and update the GUI model. Specifically, After an event $e$ is executed on a certain screen $s_c$, the GUI graph should be updated based on the observed results. The updating process can be illustrated as following figure:

![](https://tva1.sinaimg.cn/large/e6c9d24ely1h4ykoz3ussj20ys0k441e.jpg)

Graph updating shows two cases:

1. The GUI state changes to an existing state or stay the same.
2. The GUI state changes to a new state that has a different GUI from other ScreenNodes in the GUI graph.

## Preparation

In order to determine whether a screen after an event execution is different from exsisting GUI state, the author proposed a set of Multi-Level GUI Comparison Criteria.

### Information collection

The key to representing a GUI state is GUI information, which can be collected via GUI layouts of a screen. The author used **UIAutomator** tool and generated **uidump.xml** for screen analyzing. Specifically, the author automatically parsing the **uidump.xml** files and analyzed the structure of widgets, with thier properties and values, through **dumpsys system diagnostics**.

### Information classification

After extracting information from apps, we could find **hierarchial relationships** among several types of GUI information. For example, a package includes multiple activities, and an activity includes multiple widget structures. 

On the other hand ,the author filtered out  redundant GUI information that highly depends on the device or environment (e.g. coordinates or screenshots). In addition, the author merged some GUI information. For instance, properties **\<clickable\>**, **\<long-clickable\>**, **\<checkable\>** and **\<scrollable\>** were meged into an **\<executable\>** property.

## GUICC

The comparison model has 5 comparison levels (marked as **C-Lv**), each level has 3 types of output which indicates the comparison result:

- T as terminated
- S as same
- N as new

As **C-Lv** increases from **C-Lv1** to **C-Lv5**, more fine-grained GUI information is used to distinguish GUI states. Users can set a maximum comparison level (**Max C-Lv**), and the whole comparison process would start from **C-Lv1** to **Max C-Lv**. We set **Max C-Lv** as **C-Lv5** in our example.

### C-Lv1: Compare Package Names

The author used **dumpsys** to dump the status of the Android system after event, and parsing **mFocusedApp** information to extracted the package name of the currently focused app. We mark the status under comparison as $s'$.

If the device does not focus on the target application for testing, the comparison engine would return a $T$. This case can be caused by transitions to another app, shutdown, or crashes.

### C-Lv2: Compare Activity Names

After the comparison in C-Lv1 is passed, the comparison engine performs the comparison of activity names.

If not all ScreenNodes in GUI Graph have the same activity name to $s'$, it would be marked as a newly discovered GUI state and insert into GUI graph. Otherwise the next level of comparison would be taken.

### C-Lv3, C-Lv4: Compare Widget Composition

Modern Android apps show dynamic and complicated user interfaces in a single activity, which provide better experience as well as good modularity. Therefore, the activity level comparison is inadequate for effective GUI model generation of real-world apps.

For instance, an activity consists of multiple widgets, shown as following:

![](https://tva1.sinaimg.cn/large/e6c9d24ely1h4ykpb133kj20xm0g4gnp.jpg)

In order to make comparisons on composition of widgets, the author used UIAutomator to construct a widget hierarchy tree (shown in figure below). 

![](https://tva1.sinaimg.cn/large/e6c9d24ely1h4ykpj22qyj212c0m4mzu.jpg)

The author uses a index to indicates the relationship between parent node and child node. For example, the first child node is marked as $0$, and the second as $1$, thus node $M$ in figure can be represented as $[0, 0, 2, 1]$.

The author then divided widget composition into two categories: ***layout widgets*** and ***executable widgets***. Layout widgets are the non-leaf widget nodes in widget tree, while executable widgets are the leaf widget nodes with at least one “true” value in event properties. A special case it that, if a layout widget has an executable property (e.g. ListView is clickable, while its children are not), its child leaf nodes are considered as exucutable widgets.

Each widget composition is represented as a ***cumulative index sequence*** (CIS), which is a part of information in ScreenNode. The following table shows the widget composition for figure 4.

| Widget type | Nodes | CIS |
| --- | --- | --- |
| Layout | A, B, C, F | [0]-[0,0]-[0,1]-[0,0,2] |
| Executable | D, G, L, M | [0,0,0]-[0,1,0]-[0,0,2,0]-[0,0,2,1] |

For **C-Lv3**, the comparison engine compares the difference of CIS between $s'$ and existing ScreenNodes.  If the CIS of $s'$ is not the same as any of existing ScreenNode, it would be regarded as a new GUI state, otherwise the **C-Lv4** would be performed.

**C-Lv4** further compares the event handler (i.e. property like \<clickable\>) in exucatable widget nodes. If there is no difference between executable widget nodes of $s'$ and existing ScreenNodes, **C-Lv5** would be performed.

### C-Lv5: Compare Contents

The author used UIAutomator again to extract contents values from related tags such as \<text\> and \<content-desc\>. For fast comparison, the abstraction representation of text values are simplified as **the length of text**. Specifically, the comparison engine sum up all length of text values in the current screen.

In addition, C-Lv5 also performs the comparison that examines the first item of ListView in GUIs, because events like scrolling iteratively changes the visible items of ListView.

Note that applications should carefully choose the proper GUICC level to avoid potential state explosion problem, i.e. a calculator app would change the text for every user input, which leads to a exponential states increasement if using C-Lv5.