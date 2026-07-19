---
title: "Symbolic Regression Neural Networks"
date: 2026-07-14
summary: "How can we fix the black box natures of neural networks? Is there a way to turn one into an equation we can understand? Yes there is, the solution is Symbolic Regression Neural Networks!"
math: true
cover:
  image: "CoverPhoto.png"
  alt: "The EQL network architecture: a neural network whose activation functions are mathematical primitives"
  relative: true
  hiddenInSingle: true
---

*Based on two talks I gave on symbolic regression neural networks. The neural network crash course follows Michael Nielsen's [Neural Networks and Deep Learning](http://neuralnetworksanddeeplearning.com/chap1.html); the Equation Learner material follows Kim et al. (2021), and the final section follows the AI Feynman 2.0 paper by Udrescu et al. (2020). Full references at the [bottom](#references).*

## Contents

1. [Fitting equations, not curves](#introduction)
2. [A crash course in neural networks](#crash-course)
3. [Interactive: backpropagation, demon's-eye view](#demo-backprop)
4. [Turning a network into an equation](#eql)
5. [Interactive: sparsity discovers the formula](#demo-sparsity)
6. [The EQL in action: rediscovering mechanics](#action)
7. [AI Feynman 2.0](#ai-feynman)
8. [What's next?](#conclusion)
9. [References](#references)

## Fitting equations, not curves {#introduction}

**Symbolic regression (SR) fits an equation to data.** Not a smoothed curve, not a million tuned weights — an actual formula you can write on a blackboard.

Physicists have been doing this by hand for centuries. Kepler stared at Tycho Brahe's tables until he deduced that orbits are ellipses (1601). Coulomb extracted his inverse-square law from torsion-balance data (1785). The gas laws came out of decades of careful measurement (1662–1811), and Planck's radiation law (1900) was famously a formula fitted to blackbody data *before* he understood why it worked. Every one of these was symbolic regression, performed by a human, and every one was a breakthrough.

The dream is to automate that. The difficulty is that the space of possible formulas is combinatorially enormous — you can't just gradient-descend through "all equations". This post covers one family of answers: **make the neural network itself the formula**, then squeeze it until only the formula is left. To see how, we need to know how an ordinary network works first.

## A crash course in neural networks {#crash-course}

The classic entry point (this section follows Nielsen's book) is teaching a network to read handwritten digits from the MNIST database:

<figure style="margin:0;">
  <img src="mnist.png" alt="Sample handwritten digits from the MNIST database" style="width:60%;">
  <figcaption><strong>Fig. 1</strong> — MNIST digits. Trivial for you; surprisingly deep for a machine.</figcaption>
</figure>

A network is layers of *neurons* connected by weighted edges. Each neuron takes the outputs of the previous layer, forms a weighted sum, adds a bias, and squashes the result through an **activation function** \(\sigma\) — typically a sigmoid or \(\tanh\). Intuitively each neuron "weighs up the evidence" arriving on its inputs and reports a confidence between 0 and 1:

$$
\vec{a}^{\,l} = \sigma\!\left(W^l \vec{a}^{\,l-1} + \vec{b}^{\,l}\right)
$$

Chain the layers together and the whole network is one big nested function mapping pixels to predictions:

<figure style="margin:0;">
  <img src="nn-weights.png" alt="A fully connected neural network; edge colours indicate the sign and size of each weight" style="width:70%;">
  <figcaption><strong>Fig. 2</strong> — a fully connected network. The lines carry each neuron's output; their colours are the weights applied to it.</figcaption>
</figure>

At initialisation it's a *random* function. Training means measuring how wrong it is — the **cost function**, just a mean squared error (MSE),

$$
C = \frac{1}{2n}\sum_{x} \left\lVert \vec{y}(x) - \vec{a}^{\,L}(x) \right\rVert^2
$$

— and then adjusting weights and biases to reduce it. If \(C\) had two parameters you could picture a ball rolling downhill. Our little digit-reader has **25,574 parameters**, so we need the hill-rolling to be organised: *stochastic gradient descent*, with the gradients delivered efficiently by **backpropagation**.

Backprop assigns each neuron an "error" \(\delta\) — how much the cost would change if that neuron's input were nudged — computed backwards from the output layer:

$$
\delta^{L} = (\vec{a}^{L} - \vec{y}) \odot \sigma'(\vec{z}^{L}), \qquad
\delta^{l} = \left( (W^{l+1})^{T} \delta^{l+1} \right) \odot \sigma'(\vec{z}^{l})
$$

Once you know every \(\delta\), every gradient is free: \(\partial C / \partial w^l_{jk} = a^{l-1}_k \delta^l_j\).

## Interactive: backpropagation, demon's-eye view {#demo-backprop}

In the talk this was an animation: Nielsen's picture of a little demon sitting inside each neuron, twiddling its input and reporting how much the cost complains. Step through it below — forward pass first, then watch the demons carry the error backwards.

<div id="bp-widget" style="border:1px solid #8884; border-radius:8px; padding:1rem; margin:1rem 0;">
<svg id="bp-svg" viewBox="0 0 560 260" style="width:100%; max-width:560px; display:block; margin:0 auto;" aria-label="Backpropagation demo network"></svg>
<div id="bp-caption" style="text-align:center; min-height:3.2em; font-size:0.95em; margin-top:0.4rem;"></div>
<div style="text-align:center; margin-top:0.4rem;">
<button id="bp-back" style="padding:0.35rem 0.9rem; margin:0 0.2rem; cursor:pointer;">◀ Back</button>
<button id="bp-step" style="padding:0.35rem 0.9rem; margin:0 0.2rem; cursor:pointer; font-weight:bold;">Step ▶</button>
<button id="bp-reset" style="padding:0.35rem 0.9rem; margin:0 0.2rem; cursor:pointer;">Reset</button>
</div>
</div>
<script>
(function(){
  const layers = [[60,3],[210,4],[360,4],[510,2]];
  const pos = [];
  layers.forEach(function(L,li){
    const arr=[]; const n=L[1]; const x=L[0];
    for(let i=0;i<n;i++){ arr.push([x, 130 + (i-(n-1)/2)*52]); }
    pos.push(arr);
  });
  const svg = document.getElementById('bp-svg');
  const NS='http://www.w3.org/2000/svg';
  let edges=[], nodes=[], demons=[];
  for(let l=0;l<3;l++){
    pos[l].forEach(function(p){ pos[l+1].forEach(function(q){
      const e=document.createElementNS(NS,'line');
      e.setAttribute('x1',p[0]);e.setAttribute('y1',p[1]);e.setAttribute('x2',q[0]);e.setAttribute('y2',q[1]);
      e.setAttribute('stroke','#999');e.setAttribute('stroke-width','1');e.setAttribute('opacity','0.5');
      e.dataset.layer=l; svg.appendChild(e); edges.push(e);
    });});
  }
  pos.forEach(function(L,li){ L.forEach(function(p){
    const c=document.createElementNS(NS,'circle');
    c.setAttribute('cx',p[0]);c.setAttribute('cy',p[1]);c.setAttribute('r','15');
    c.setAttribute('fill', li===0 ? '#cde8cd' : (li===3 ? '#f3cdcd' : '#cdd5f3'));
    c.setAttribute('stroke','#555'); c.dataset.layer=li;
    svg.appendChild(c); nodes.push(c);
    const t=document.createElementNS(NS,'text');
    t.setAttribute('x',p[0]);t.setAttribute('y',p[1]+6);t.setAttribute('text-anchor','middle');
    t.setAttribute('font-size','16'); t.textContent='😈'; t.setAttribute('opacity','0');
    t.dataset.layer=li; svg.appendChild(t); demons.push(t);
  });});
  const labels=[['input',60],['hidden 1',210],['hidden 2',360],['output',510]];
  labels.forEach(function(L){
    const t=document.createElementNS(NS,'text');
    t.setAttribute('x',L[1]);t.setAttribute('y',20);t.setAttribute('text-anchor','middle');
    t.setAttribute('font-size','12');t.setAttribute('fill','#888');t.textContent=L[0];
    svg.appendChild(t);
  });
  const captions=[
    'A fresh network: random weights, no idea what it’s doing. Press Step to feed an input forward.',
    'Forward pass, layer 1: each hidden neuron computes σ(w·a + b) from the inputs.',
    'Forward pass, layer 2: activations ripple onward through the next weighted layer.',
    'The output appears — and it’s wrong. The cost C measures how wrong (mean squared error).',
    'Demons arrive at the output: δᴸ = (aᴸ − y) ⊙ σ′(zᴸ). Each one reports how much its neuron’s input moves the cost.',
    'The demons hop backwards: δˡ = ((Wˡ⁺¹)ᵀδˡ⁺¹) ⊙ σ′(zˡ) — the error one layer back is a weighted mix of the errors ahead of it.',
    'One more hop and every neuron has its demon — every error δ is known in a single backward sweep.',
    'Gradients come for free: ∂C/∂w = a·δ. Nudge every weight downhill, repeat over many mini-batches — that’s training.'
  ];
  let step=0;
  function setEdges(l,color,w){ edges.forEach(function(e){ if(+e.dataset.layer===l){e.setAttribute('stroke',color);e.setAttribute('stroke-width',w);e.setAttribute('opacity','0.9');} }); }
  function resetEdges(){ edges.forEach(function(e){e.setAttribute('stroke','#999');e.setAttribute('stroke-width','1');e.setAttribute('opacity','0.5');}); }
  function setNodes(l,fill){ nodes.forEach(function(c){ if(+c.dataset.layer===l){c.setAttribute('fill',fill);} }); }
  function resetNodes(){ nodes.forEach(function(c){ const li=+c.dataset.layer; c.setAttribute('fill', li===0 ? '#cde8cd' : (li===3 ? '#f3cdcd' : '#cdd5f3')); }); }
  function showDemons(l,on){ demons.forEach(function(t){ if(+t.dataset.layer===l){t.setAttribute('opacity', on?'1':'0');} }); }
  function render(){
    resetEdges(); resetNodes(); [0,1,2,3].forEach(function(l){showDemons(l,false);});
    if(step>=1){ setEdges(0,'#2e7d32',2); setNodes(1,'#a5d6a7'); }
    if(step>=2){ setEdges(1,'#2e7d32',2); setNodes(2,'#a5d6a7'); }
    if(step>=3){ setEdges(2,'#2e7d32',2); setNodes(3,'#ef9a9a'); }
    if(step>=4){ showDemons(3,true); }
    if(step>=5){ showDemons(2,true); setEdges(2,'#c62828',2.5); }
    if(step>=6){ showDemons(1,true); setEdges(1,'#c62828',2.5); }
    if(step>=7){ setEdges(0,'#c62828',2.5); }
    document.getElementById('bp-caption').textContent = captions[step];
    document.getElementById('bp-step').disabled = (step===captions.length-1);
    document.getElementById('bp-back').disabled = (step===0);
  }
  document.getElementById('bp-step').addEventListener('click',function(){ if(step<captions.length-1){step++; render();} });
  document.getElementById('bp-back').addEventListener('click',function(){ if(step>0){step--; render();} });
  document.getElementById('bp-reset').addEventListener('click',function(){ step=0; render(); });
  render();
})();
</script>

Repeat forward-pass → backprop → nudge over many mini-batches (an *epoch* is one pass through all the data) and the cost falls until the network reads digits it has never seen. Trained — but *opaque*. 25,574 numbers is not an explanation of anything.

## Turning a network into an equation {#eql}

Here is the key move of a symbolic regression neural network, usually called an **Equation Learner (EQL) network** (Kim et al. 2021). Take the network above and make one change: **replace the uniform activation function with a different mathematical primitive at each neuron** — identity, \((\cdot)^2\), \(\sin(\cdot)\), multiplication of pairs, and so on:

<figure style="margin:0;">
  <img src="eql-network.png" alt="EQL network architecture: each neuron applies a different primitive function such as identity, square, sine or product" style="width:70%;">
  <figcaption><strong>Fig. 3</strong> — the EQL architecture from Kim et al. Weights form linear combinations exactly as before, but each neuron applies a different primitive function.</figcaption>
</figure>

Nothing about training changes — it's still differentiable end-to-end, so backprop works untouched. But now the network *is* a formula: composing the layers writes out some nested algebraic expression. The catch is that a freshly trained EQL is a **very complicated but correct** formula — hundreds of terms, all overlapping. To perform symbolic regression we must make it *sparse*: kill off as many weights as possible so a short expression remains.

Sparsity comes from adding a penalty to the cost function,

$$
L_q = \frac{1}{N}\sum_{i}\left\lVert \hat{y}_i - y_i \right\rVert^2 + \lambda \sum_{j} \lvert w_j \rvert^{q}
$$

The harshest complexity penalty would be \(q = 0\) (count the nonzero weights), but that's an NP-hard combinatorial problem. \(q = 1\) (LASSO) is friendly but under-penalises. The sweet spot used here is \(q = 0.5\) — except \(L_{0.5}\) has an infinite gradient at \(w = 0\), which stalls training. The fix is a *smoothed* \(L_{0.5}^{*}\) that swaps in a well-behaved polynomial below a threshold \(a\):

<div style="display:flex; gap:1rem; flex-wrap:wrap; align-items:flex-start;">
  <figure style="flex:1; min-width:260px; margin:0;">
    <img src="l05.png" alt="L0.5 regularizer with a cusp at zero" style="width:100%;">
    <figcaption><strong>Fig. 4a</strong> — the raw L₀.₅ penalty: the gradient blows up at w = 0.</figcaption>
  </figure>
  <figure style="flex:1; min-width:260px; margin:0;">
    <img src="l05-smoothed.png" alt="Smoothed L0.5 regularizer" style="width:100%;">
    <figcaption><strong>Fig. 4b</strong> — the smoothed L₀.₅* used in practice: same tails, finite gradient at the origin.</figcaption>
  </figure>
</div>

## Interactive: sparsity discovers the formula {#demo-sparsity}

Drag the regularisation strength and watch a tiny EQL network prune itself. Every edge that survives contributes a term; the formula underneath is what the network *is* at that sparsity. Too little pressure and the truth hides inside junk terms; too much and the model breaks. (True function: \(y = 0.5x^2 + \sin x\).)

<div id="sp-widget" style="border:1px solid #8884; border-radius:8px; padding:1rem; margin:1rem 0;">
<svg id="sp-svg" viewBox="0 0 560 220" style="width:100%; max-width:560px; display:block; margin:0 auto;" aria-label="EQL sparsity demo"></svg>
<div style="text-align:center; margin-top:0.5rem;">
<label for="sp-slider" style="font-size:0.95em;">regularisation strength λ:</label>
<input id="sp-slider" type="range" min="0" max="4" value="0" step="1" style="width:60%; vertical-align:middle;">
</div>
<div id="sp-formula" style="text-align:center; font-family:monospace; font-size:1.02em; min-height:2.4em; margin-top:0.5rem;"></div>
<div id="sp-stats" style="text-align:center; font-size:0.9em; opacity:0.8;"></div>
</div>
<script>
(function(){
  const NS='http://www.w3.org/2000/svg';
  const svg=document.getElementById('sp-svg');
  const inX=[70,110];
  const prims=[[280,40,'id'],[280,100,'(·)²'],[280,160,'sin'],[280,200,'×']];
  const out=[490,110];
  const edgeDefs=[
    {from:inX,to:prims[0],min:4},
    {from:inX,to:prims[1],min:99},
    {from:inX,to:prims[2],min:99},
    {from:inX,to:prims[3],min:2},
    {from:prims[0],to:out,min:4},
    {from:prims[1],to:out,min:99},
    {from:prims[2],to:out,min:3},
    {from:prims[3],to:out,min:2}
  ];
  // min = lambda value at which this edge dies (99 = never dies)
  let edgeEls=[];
  edgeDefs.forEach(function(d){
    const e=document.createElementNS(NS,'line');
    e.setAttribute('x1',d.from[0]+22);e.setAttribute('y1',d.from[1]);
    e.setAttribute('x2',d.to[0]-22);e.setAttribute('y2',d.to[1]);
    e.setAttribute('stroke','#1565c0');e.setAttribute('stroke-width','2');
    svg.appendChild(e); edgeEls.push(e);
  });
  function node(x,y,label,fill){
    const c=document.createElementNS(NS,'circle');
    c.setAttribute('cx',x);c.setAttribute('cy',y);c.setAttribute('r','20');
    c.setAttribute('fill',fill);c.setAttribute('stroke','#555');
    svg.appendChild(c);
    const t=document.createElementNS(NS,'text');
    t.setAttribute('x',x);t.setAttribute('y',y+5);t.setAttribute('text-anchor','middle');
    t.setAttribute('font-size','13'); t.textContent=label; svg.appendChild(t);
    return c;
  }
  node(inX[0],inX[1],'x','#cde8cd');
  const primEls = prims.map(function(p){ return node(p[0],p[1],p[2],'#cdd5f3'); });
  node(out[0],out[1],'ŷ','#f3cdcd');
  const formulas=[
    'ŷ = 0.43x² + 1.02·sin x − 0.11x + 0.07x·sin x − 0.03',
    'ŷ = 0.46x² + 1.01·sin x − 0.09x + 0.02',
    'ŷ = 0.48x² + 0.99·sin x',
    'ŷ = 0.5x² + sin x   ✓ recovered!',
    'ŷ = 0.5x²   (too sparse — sin term lost)'
  ];
  const stats=[
    'test error: 0.011 | active weights: 8 — accurate but unreadable',
    'test error: 0.012 | active weights: 6',
    'test error: 0.013 | active weights: 4',
    'test error: 0.014 | active weights: 4 (snapped to round values) — the Pareto sweet spot',
    'test error: 0.31 | active weights: 2 — the fit has broken'
  ];
  const slider=document.getElementById('sp-slider');
  function render(){
    const lam=+slider.value;
    edgeDefs.forEach(function(d,i){
      const dead = lam>=d.min;
      edgeEls[i].setAttribute('opacity', dead?'0.12':'1');
      edgeEls[i].setAttribute('stroke-dasharray', dead?'4 4':'none');
      edgeEls[i].setAttribute('stroke-width', dead?'1':String(2.6-0.35*lam));
    });
    // grey out primitives with no live output edge
    const liveOut=[4,5,6,7].map(function(k){return lam<edgeDefs[k].min;});
    primEls.forEach(function(c,i){ c.setAttribute('opacity', liveOut[i]?'1':'0.25'); });
    document.getElementById('sp-formula').textContent=formulas[lam];
    document.getElementById('sp-stats').textContent=stats[lam];
  }
  slider.addEventListener('input',render);
  render();
})();
</script>

This is the whole philosophy in one slider: **accuracy and simplicity trade off**, and somewhere on that trade-off curve sits a formula a physicist would recognise. Hold that thought — it returns as a *Pareto frontier* in the final section.

## The EQL in action: rediscovering mechanics {#action}

Kim et al. wired an EQL into a larger architecture and pointed it at physics. Take a simple harmonic oscillator — a mass on a spring, \(\ddot{u} = -\omega^2 u\) — and generate trajectories for many random spring constants:

<figure style="margin:0;">
  <img src="sho-spring.png" alt="Mass on a spring: the simple harmonic oscillator setup" style="width:30%;">
  <figcaption><strong>Fig. 5</strong> — the test problem: a mass on a spring.</figcaption>
</figure>

The clever part is the surrounding structure: a *dynamics encoder* compresses each observed trajectory into a latent parameter \(z\) (which training reveals to be essentially \(\omega^2\) — the network discovers the relevant physical variable on its own), and a *propagating decoder* unrolls an EQL cell repeatedly to predict the trajectory, exactly like a numerical integrator stepping through time:

<figure style="margin:0;">
  <img src="eql-pipeline.png" alt="Dynamics encoder and propagating decoder architecture with an unrolled EQL cell" style="width:85%;">
  <figcaption><strong>Fig. 6</strong> — architecture from Kim et al.: an encoder finds the physical parameter, an unrolled EQL cell learns the update rule.</figcaption>
</figure>

What update rule does the EQL learn? Read the extracted equations against the ground truth:

<figure style="margin:0;">
  <img src="sho-table.png" alt="Table of expected and extracted equations for the simple harmonic oscillator" style="width:75%;">
  <figcaption><strong>Fig. 7</strong> — extracted update rules (Kim et al., Table IV). The EQL's equation matches the Euler integrator — plus correction terms.</figcaption>
</figure>

The EQL rediscovers the **Euler method**, \(u_{i+1} = u_i + v_i\,\Delta t,\; v_{i+1} = v_i - \omega^2 u_i\,\Delta t\) — and then does slightly better, picking up higher-order correction terms (the authors note a larger EQL could plausibly reach Runge–Kutta). And because it learned an *equation* rather than a black-box map, it extrapolates beyond its training window far better than a standard ReLU network, which falls apart the moment it leaves familiar territory:

<figure style="margin:0;">
  <img src="sho-extrapolation.png" alt="Position and velocity predictions: the EQL extrapolates well beyond training, the ReLU network does not" style="width:85%;">
  <figcaption><strong>Fig. 8</strong> — extrapolation on the oscillator (Kim et al.). "True" vs EQL vs a conventional ReLU network vs Euler integration. The black-box network's extrapolation is visibly bad; the EQL tracks the physics.</figcaption>
</figure>

That is the payoff of symbolic regression in one picture: **equations generalise; black boxes interpolate.**

## AI Feynman 2.0 {#ai-feynman}

The EQL bakes the formula into the network. **AI Feynman 2.0** (Udrescu et al., 2020) flips the relationship: train an ordinary black-box network \(f_{\mathrm{NN}}\) to fit the data, then *interrogate it* to reverse-engineer the formula's structure. The name comes from its benchmark — 100 equations from the Feynman Lectures on Physics, which the method solves in full.

The central idea is **graph modularity**. Almost every physics formula, drawn as a computational graph, decomposes into modules with fewer inputs — e.g. \(f(x,y,z) = g\!\left[h(x,y), z\right]\) where \(h\) is simpler than \(f\). Because the trained network is differentiable, its *gradients* betray these decompositions: compositionality, symmetry and generalized additivity each leave a distinct fingerprint in \(\nabla f\) (for instance, if \(f(\mathbf{x}) = g(h(\mathbf{x}'))\,\) only through \(h\), the normalised gradient with respect to \(\mathbf{x}'\) is independent of the remaining variables — a smoking gun). Detect the fingerprint, split the problem, recurse on the smaller pieces until brute-force search or polynomial fitting can finish the job:

<figure style="margin:0;">
  <img src="aifeynman-flowchart.png" alt="AI Feynman 2.0 algorithm flowchart: recursively decompose the problem via symmetries and separability discovered from a neural network fit" style="width:45%;">
  <figcaption><strong>Fig. 9</strong> — the recursive AI Feynman pipeline: fit a network, hunt for modularity in its gradients, decompose, recurse.</figcaption>
</figure>

Three more ideas make it robust. First, instead of accepting one "best" formula, it maintains the whole **Pareto frontier** of accuracy versus description-length complexity — our slider's trade-off, formalised in bits. Convex corners of the frontier are where the interesting physics lives:

<figure style="margin:0;">
  <img src="pareto.png" alt="Pareto frontier of formula inaccuracy versus complexity for the relativistic kinetic energy data" style="width:70%;">
  <figcaption><strong>Fig. 10</strong> — Pareto frontier for kinetic energy data (Udrescu et al., Fig. 1). The corners are ½mv² (the classical approximation) and Einstein's exact formula.</figcaption>
</figure>

Feed it data on how kinetic energy depends on mass, velocity and the speed of light, and it doesn't just return the exact relativistic formula — it *also* hands you \(\tfrac{1}{2}mv^2\) as the best simple approximation. The Pareto frontier contains the physics *and* the physicist's intuition about when the simple version suffices.

Second, it swaps MSE for a **mean error description length**, which ignores outliers instead of being dragged by them, and it rejects candidate formulas by statistical hypothesis testing rather than a single bad data point. Third, using **normalizing flows**, it extends symbolic regression to probability distributions known only through samples — recovering, for example, hydrogen orbital densities.

The results: all 100 Feynman-Lecture equations solved; on noisy data it stays correct at noise levels typically **one to three orders of magnitude higher** than the original AI Feynman, solving 73 of 100 baseline problems even with noise at the \(10^{-1}\) level; and it cracked all 17 "mystery" equations that the previous state of the art had failed on.

## What's next? {#conclusion}

Symbolic regression neural networks close a loop that physics opened four centuries ago: Kepler fitted an ellipse by hand; an EQL fits an integrator by gradient descent; AI Feynman recovers a century of formulas from raw tables. The tools are genuinely usable today (`pip install aifeynman`), and the mainstream view the AI Feynman authors offer is worth repeating — *all* known physics formulas are Pareto-optimal approximations, accurate **and** simple, and we live in a golden age of datasets waiting to be compressed into equations.

What I'd like to explore next: SR on systems where we *don't* know the answer (that's the point, after all), physics-informed architectures that encode symmetries and conservation laws from the start, and — more modestly — whether an EQL can solve my Statistical Mechanics II problem sheet.

## References {#references}

1. <span id="ref-1"></span>S. Kim, P. Lu, S. Mukherjee, M. A. Gilbert, J. Li, V. Čeperić and M. Soljačić, "Integration of Neural Network-Based Symbolic Regression in Deep Learning for Scientific Discovery", *IEEE Transactions on Neural Networks and Learning Systems* **32**(9), 4166–4177 (2021). [doi:10.1109/tnnls.2020.3017010](https://doi.org/10.1109/tnnls.2020.3017010)
2. <span id="ref-2"></span>M. A. Nielsen, *Neural Networks and Deep Learning*, Determination Press (2015). [neuralnetworksanddeeplearning.com](http://neuralnetworksanddeeplearning.com/)
3. <span id="ref-3"></span>S.-M. Udrescu, A. Tan, J. Feng, O. Neto, T. Wu and M. Tegmark, "AI Feynman 2.0: Pareto-optimal symbolic regression exploiting graph modularity", *NeurIPS 2020*. [arXiv:2006.10782](https://arxiv.org/abs/2006.10782)
4. <span id="ref-4"></span>S.-M. Udrescu and M. Tegmark, "AI Feynman: A physics-inspired method for symbolic regression", *Science Advances* **6**(16) (2020). [doi:10.1126/sciadv.aay2631](https://doi.org/10.1126/sciadv.aay2631)
5. <span id="ref-5"></span>B. Moseley, "Physics-informed machine learning: from concepts to real-world applications", DPhil thesis, University of Oxford (2022). [ORA](https://ora.ox.ac.uk/objects/uuid:b790477c-771f-4926-99c6-d2f9d248cb23)

*Figures 1–8 are reproduced from Kim et al. [1] and Nielsen [2]; Figures 9–10 from Udrescu et al. [3].*
