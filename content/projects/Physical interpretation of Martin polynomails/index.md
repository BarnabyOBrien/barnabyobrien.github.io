---
title: "Physical interpretation of Martin polynomails"
date: 2026-07-13
summary: "I am currently working under Dr Paul Balduf, at the University of Oxford, on finding a physical interpretation for higher martin polynomails through Matrix Quantum Field Theory  "
status: "current"
cover:
  image: "2ndMartinInvaraint.jpeg"
  alt: "White board drawing illustrating steps to caculate the 2nd order Martin invariant"
  relative: true
  hiddenInSingle: true

---
<p align="center">
  <img src="2ndMartinInvaraint.jpeg" />
  <br/>
  <em>The potential in matrix field theory for a 2nd order Martin invariant. We are currently attempting to generalise this method to higher orders, as well as eventually caculate the asymtotics of these theories</em>
</p>


<!-- ![The potential in matrix field theory for a 2nd order Martin invariant](MartinInvariant.png "White board drawing illustrating steps to caculate the 2nd order Martin invariant") -->

<!-- **Field** Combinatorial Quantum Field Theory -->

**GitHub Repositories:** 
- [O(N) Polynomial for Feynman Graphs (Combinatorial Method)][def]
<!-- - [O(N) Polynomial for Feynman Graphs (Matrix QFT)](#) -->

<!-- <style>
.sq{display:inline-block;width:12px;height:12px;margin-right:8px}
.b{background:blue}
.r{background:red}
</style> -->

<style>
.sq{display:inline-block;width:12px;height:12px;margin-right:8px}
.b{background:blue}
.r{background:red}
.g{background:green}
ul{list-style: none; padding-left: 0;}
.key{margin-top:16px; font-size:0.9em}
.key-item{display:flex; align-items:center; margin-bottom:4px}
</style>

# Work in progress
I am currently still working on this project and thus have yet to provide a write-up outlining the topic and my contributions. Below I provide a current update on what we have already achieved, as well as current future aims. 

### <span style="color:blue">What we have achieved so far</span>
<!-- <div class="key">
  <div class="key-item"><span class="sq b"></span>Meaning of blue</div>
  <div class="key-item"><span class="sq r"></span>Meaning of red</div>
</div> -->

#### Initial Exploration
- <span class="sq b"></span>Explored different tensor field theories approaches such as, Edge Colored Graphs and Ribbon Graphs
- <span class="sq r"></span>Concluded **Ribbon Graphs** as the best approach
- <span class="sq b"></span>Explored the 7!! decompositions of a 2nd order Martin invariant vertex and ascertained potential interaction terms
- <span class="sq r"></span>Concluded we need a matrix <i>crossing propagator</i> which induced the matrices to be symmetric in nature


#### Calculating the 2nd Order Martin Invariant Potential


- <span class="sq b"></span>Weighted potential correctly in order to match to Martin invariant decompositions
- <span class="sq r"></span>Allowed exploration of simplifications of Lagrangian in order to gain more insight into the theory. Potentially could have links to a bosonic theory
- <span class="sq b"></span>Needed to be able to check our Lagrangian is correct
- <span class="sq r"></span>Created code to find the O(N) polynomial for a given Feynman graph ([code can be found here][def])

[def]: https://github.com/BarnabyOBrien/O-N--Polynomial-for-Feynman-Graphs--Combinatorial-Method-

### <span style="color:red">Current Future Aims
- <span class="sq g"></span> Create complimentary code for our constructed Lagrangian and compare to previous code 
- <span class="sq g"></span> Explore potential simplifications of Lagrangian which may make calculation of higher orders and asymptotics more feasible
- <span class="sq g"></span> If second approach fails continue on old approach for all higher orders 