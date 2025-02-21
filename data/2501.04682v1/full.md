# Towards System 2 Reasoning in LLMs: Learning How to Think With Meta Chain-of-Thought  

Violet Xiang2, Charlie Snell3, Kanishk Gandhi2, Alon Albalak1, Anikait Singh2, Chase Blagden1, Duy Phung1, Rafael Rafailov2,1, Nathan Lile1, Dakota Mahan1, Louis Castricato1, Jan-Philipp Fränken2, Nick Haber2 and Chelsea Finn2  

1SynthLabs.ai, 2Stanford University, $^3\mathrm{UC}$ Berkeley  

We propose a novel framework, Meta Chain-of-Thought (Meta-CoT), which extends traditional Chain-ofThought (CoT) by explicitly modeling the underlying reasoning required to arrive at a particular CoT. We present empirical evidence from state-of-the-art models exhibiting behaviors consistent with in-context search, and explore methods for producing Meta-CoT via process supervision, synthetic data generation, and search algorithms. Finally, we outline a concrete pipeline for training a model to produce Meta-CoTs, incorporating instruction tuning with linearized search traces and reinforcement learning post-training. Finally, we discuss open research questions, including scaling laws, verifer roles, and the potential for discovering novel reasoning algorithms. This work provides a theoretical and practical roadmap to enable Meta-CoT in LLMs, paving the way for more powerful and human-like reasoning in artifcial intelligence.  

Give a man a fsh and you feed him for a day; teach a man to fsh and you feed him for a lifetime. -Proverb  

# Contents  

1 Introduction 4   
1.1 Motivation 4   
1.2 Outline 6   
2 Meta Chain-Of-Thought 6   
2.1 Deriving The Meta-CoT Process 6   
2.2 Why Does (Classical) CoT Fail? 9   
3 Towards Deliberate Reasoning With Language Models - Search 9   
3.1 Inference-Time Compute: Search 10   
3.2 Inference-Time Compute: Verifcation . 11   
Authors are ordered randomly. Correspondence to team@synthlabs.ai.   
3.3 From Best-of-N To General Search 12   
3.4 Is Search (Inference Time Compute) A Fundamental Capability Shift? . . 14  

# 4 Towards Meta-CoT Reasoning 15  

4.1 Bootstrapping Meta-CoT 15   
4.1.1 Self-Taught Reasoner 15   
4.1.2 Meta-STaR 16   
4.2 Empirical Examples Of Internalizing Search 16   
4.2.1 Small-Scale Empirical Results on Internalizing Search 16   
4.2.2 In-context Exploration For LLMs . 19   
4.2.3 Using variable Compute 19   
4.2.4 Backtracking in LLMs 20   
4.3 Synthetic Meta-CoT Via Search 21   
4.3.1 Monte-Carlo Tree Search . 22   
4.3.2 $\mathrm{A^{*}}$ search 22   
4.4 Do Advanced Reasoning Systems Implement In-Context Search? 23  

# 5 Process Supervision 26  

5.1 Learning Process Reward Models 26   
5.2 PRM Quality And Its Efect On Search 27   
5.3 Verifable Versus Open-Ended Problems 27   
6.1 Meta-RL In Small Domains 31   
6.2 Meta-RL In Language Model Reasoning 32   
6.3 Efciency Or Super-Intelligence? 33   
6.4 Can System 2 Reasoning Emerge From Pure RL? . 35   
6.4.1 Inducing Meta-Reasoning In LLMs 35  

# 7 Putting It All Together - A Pipeline for System 2 Reasoning 39  

7.1 Instruction Tuning 39   
7.2 Post-Training With RL 40   
7.2.1 $\mathrm{{Q^{*}}}$ or q-STaR? 40   
7.2.2 Discount Rates 42  

# 8 Going Forward 43  

8.1 The "Big MATH" Project 44   
8.1.1 Data Sourcing . 45   
8.1.2 Data Filtering 45   
8.2 Infrastructure 46   
8.3 Open Research Questions 47   
8.3.1 Open-Ended Verifcation And CoT Faithfulness 47   
8.3.2 Process Guidance And The Verifer Gap 48   
8.3.3 Scaling Laws For Reasoning And Search 49   
8.3.4 Meta-Search/Search 2 50   
8.3.5 Reasoning with External Tools . 50  

64  

# 1. Introduction  

# 1.1. Motivation  

A key aspect of the current era of Large Language Models has been the foundational principle of next-token prediction (Elman, 1990; Jordan, 1997). That is, tokenizing text (or other continuous modalities) into a discrete sequence in the following way:  

where $\mathbf{y}_{i}$ are elements of some fnite vocabulary and, subsequently, train a large parameterized neural network $p_{\theta}$ (transformer) model with the following maximum likelihood objective:  

$$
\mathcal{L}_{\theta}=\mathbb{E}_{\mathcal{D}_{\mathrm{train}}}\left[-\sum_{t}\log p_{\theta}(\mathbf{y}_{t+1}\vert\mathbf{y}_{\le t})\right].
$$  

Behind this approach is a simple principle often abbreviated as "compression is intelligence", or the model must approximate the distribution of data and perform implicit reasoning in its activations in order to predict the next token (see Solomonof Induction; Solomonof 1964). That is, language models learn the implicit meaning in text, as opposed to the early belief some researchers held that sequence-to-sequence models (including transformers) simply ft correlations between sequential words.  

A fundamental question, however, is: What is the relationship between the complexity of the underlying data stream and the model’s capability to learn the data-generating algorithm? While natural language has some irreducible entropy, this question holds even in deterministic scenarios (as demonstrated below). Mathematical reasoning is a good domain to demonstrate and evaluate these questions. One can ask an LLM to generate a response to questions like:  

“What is $1\!+\!2^{\prime}$ ?”  

to which most immediately answer $^{\ast}3^{\prime\prime}$ . We can increase the complexity of the question by asking, for example:  

<html><body><table><tr><td colspan="3">"What is the value of</td></tr><tr><td colspan="3">(α² - 1)(x + 1) 1</td></tr><tr><td colspan="3">c3 -C</td></tr><tr><td colspan="3"></td></tr><tr><td colspan="3">evaluated dat=π?"</td></tr></table></body></html>  

This is actually a simple question since the expression is canonically equal to 1, yet even powerful LLMs such as GPT-4o and Claude fail to answer it correctly even once. That is, the true conditional generative process $p(\mathbf{y}_{t+1}|\mathbf{y}_{t})$ could have arbitrarily high computational complexity even when it is deterministic as in the example above. In contrast, if we instruct models to “think step by step” and produce a chain-of-thought we can signifcantly improve their capabilities producing traces like the following:  

We are tasked with evaluating  

$$
\frac{(x^{2}-1)(x+1)}{x^{3}-x}-\frac{1}{x}
$$  

$x=\pi$ . Let us simplify the expression before substituting  

Step 1: Simplify $(x^{2}-1)(x+1)$  

The numerator of the frst term can be rewritten:  

$$
x^{2}-1=(x-1)(x+1),
$$  

so  

$$
(x^{2}-1)(x+1)=(x-1)(x+1)^{2}.
$$  

Step 2: Simplify $x^{3}-x$ in the denominator  

Factorize $x^{3}-x$ :  

$$
x^{3}-x=x(x^{2}-1)=x(x-1)(x+1).
$$  

Thus, the frst term becomes:  

$$
{\frac{(x^{2}-1)(x+1)}{x^{3}-x}}={\frac{(x-1)(x+1)^{2}}{x(x-1)(x+1)}}.
$$  

Cancel $(x-1)$ and one factor of $(x+1)$ :  

$$
{\frac{(x^{2}-1)(x+1)}{x^{3}-x}}={\frac{x+1}{x}}.
$$  

Step 3: Subtract $\textstyle{\frac{1}{x}}$  

Now substitute this simplifed result into the full expression:  

$$
{\frac{x+1}{x}}-{\frac{1}{x}}.
$$  

Combine the terms over the common denominator $x$ :  

$$
{\frac{x+1}{x}}-{\frac{1}{x}}={\frac{x+1-1}{x}}={\frac{x}{x}}=1.
$$  

Step 4: Evaluate at $x=\pi$ The expression simplifes to 1, which is independent of $x$ . Thus, the value at $x=\pi$ is:  

$$
\boxed{1}.
$$  

There have been multiple explanations for this phenomenon (Merrill & Sabharwal, 2023; Prystawski et al., 2024; Ye et al., 2024a; Dutta et al., 2024), however we focus on the complexity hypothesis (Merrill & Sabharwal, 2023). Specifcally, a CoT expansion allows for (in-theory) arbitrarily large amounts of compute to be deployed towards the prediction of answer tokens. A large theoretical literature has been developed arguing for the representational complexity of transformers with and without CoT (Merrill & Sabharwal, 2023; Nowak et al., 2024; Li et al., 2024). The literature says that endowing LLMs with CoT allows them to represent new orders of complexity and (with assumptions, such as infnite memory) may even make them Turing complete. However, in practice, today’s LLMs can only reliably solve problems of limited complexity (Snell et al., 2024).  

# 1.2. Outline  

In this paper, we investigate the limitations of current LLMs in handling complex reasoning tasks and propose a novel framework, Meta Chain-of-Thought (Meta-CoT), to address these shortcomings. We argue that traditional Chain-of-Thought (CoT) methods, while efective for simpler problems, fail to capture the true data-generating process of complex reasoning which often involves a non-linear, iterative, and latent process of exploration and verifcation. Meta-CoT extends CoT by explicitly modeling this latent “thinking” process, which we hypothesize is essential for solving problems that require advanced reasoning capabilities.  

We draw inspiration from Cognitive Science’s dual-process theory, framing Meta-CoT as a form of System 2 reasoning. We establish the theoretical foundations of Meta-CoT, demonstrating how it can be realized through systematic search processes, and how these processes can be internalized within a single auto-regressive model. We then present empirical evidence supporting our claims, including analyses on state-of-the-art models like OpenAI’s o1 (OpenAI, 2024) and DeepSeek-R1 (DeepSeek, 2024), which exhibit behaviors consistent with internalized (in-context) search. We further explore methods for training models on Meta-CoT through process supervision, and synthetic data generation via search algorithms like Monte Carlo Tree Search (MCTS) and $\mathrm{A^{*}}$ .  

Finally, we outline a concrete pipeline for achieving Meta-CoT in a single end-to-end system, incorporating instruction tuning with linearized search traces and reinforcement learning (RL) posttraining. We discuss open research questions, including the scaling laws of reasoning and search, the role of verifers, and the potential for discovering novel reasoning algorithms through meta-RL. We also present the “Big MATH” project, an efort to aggregate over 1,000,000 high-quality, verifable math problems to facilitate further research in this area. Our work provides both theoretical insights and a practical road map to enable Meta-CoT in LLMs, paving the way for more powerful and human-like reasoning in artifcial intelligence.  

# 2. Meta Chain-Of-Thought  

In this section, we frst formulate the meta chain-of-thought process and discuss how it can describe the problem solving process for complex reasoning problems. Then, we describe and demonstrate why classical chain-of-thought fails under certain circumstances.  

# 2.1. Deriving The Meta-CoT Process  

A question to ask ourselves is: Should language models with Chain-Of-Thought prompting really be able to express any function, and thus solve arbitrarily complex problems, which was the theoretical point of the previous section? We will stick with the mathematical reasoning domain for the purpose of the discussion. Today, the capabilities of frontier models are enough for a large class of mathematical reasoning problems. Current state-of-the art systems such as GPT-4o and Claude largely solve the Hendrycks MATH Levels 1-3 Benchmark (Hendrycks et al., 2021), however, they still struggle with advanced problems such as those in Levels 4 and 5, HARP (Yue et al., 2024) and Omni-MATH (Gao et al., 2024) (as well as other advanced reasoning tasks). We put forward the following theory to explain these empirical observations.  

Reasoning data present in pre-training corpuses does not represent the true data generation process, especially for complex problems, which is a product of extensive latent reasoning. Moreover, this process generally does not occur in a left-to-right, auto-regressive, fashion.  

In more details, the CoT reasoning data prevalent in the pre-training corpus and post-training instruction tuning follows the true data-generating process for solutions of simple problems such as algebraic computations, counting, basic geometry etc.. That is, for example, the textbook solutions for solving high-school algebra present the general process of generating those solutions. If we follow some set of steps or approaches present in existing textbooks, we can eventually arrive at the solution. Hence, these are learnable with a constant-depth transformers that can express the complexity of each individual step in the process. In contrast, complex reasoning problems do not follow that pattern. We may have a set of triples $(\mathbf{q},\mathbf{S},\mathbf{a})$ of questions $\mathbf{q},$ solution steps $\mathbf{S}=(\mathbf{s}_{1},\ldots,\mathbf{s}_{n})$ and (optionally) answers a, but the true data generation process is not auto-regressive:  

$$
{\bf q}\rightarrow{\bf z}_{1}\rightarrow...\rightarrow{\bf z}_{K}\rightarrow({\bf s_{1}},...\,,{\bf s}_{n},{\bf a}),
$$  

where $\mathbf{z}_{i}$ are the latent "thoughts" left out of the solutions steps, which can be represented fully with left-to-right generation, while the dataset solution $\mathbf{S}\,=\,(\mathbf{s}_{1},\hdots,\mathbf{s}_{n})$ is generated jointly. Let us illustrate this with an example from the International Mathematics Olympiad 2011. This is the (in)famous "windmill" problem:  

“Let $\mathcal{S}$ be a fnite set of at least two points in the plane. Assume that no three points of $\mathcal{S}$ are collinear. A windmill is a process that starts with a line $\ell$ going through a single point $P\in S$ . The line rotates clockwise about the pivot $P$ until the frst time that the line meets some other point belonging to $\mathcal{S}$ . This point, $Q$ , takes over as the new pivot, and the line now rotates clockwise about $Q$ , until it next meets a point of $\mathcal{S}$ . This process continues indefnitely. Can we choose a point $P$ in $\mathcal{S}$ and a line $\ell$ going through $P$ such that the resulting windmill uses each point of $\mathcal{S}$ as a pivot infnitely many times.”  

which has the following solution:  

“Let $\vert S\vert=n$ . Now consider an arbitrary point $P$ in $S$ and choose a line $l$ through $P$ which splits the points in the plane into roughly equal chunks. Next notice that as the line rotates it will sweep a full $2\pi$ angle against some fxed reference frame. Now take another random point $P^{\prime}$ and similarly constructed stationary line $l^{\prime}$ . At some point in the windmill process we will have $l||l^{\prime}$ . However notice that $l$ and $l^{\prime}$ split the points into the same two sets and are parallel. Therefore we must have that $l\equiv l^{\prime}$ and thus $l$ passes through $P^{\prime}$ . This of course holds for any $P^{\prime}\in S$ . Applying the same argument recursively yields the fnal proof that it is in fact possible to make such a construction for any set $S$ with these properties.”  

The solution above does not use any prior knowledge and fts within a few sentences. Yet, this problem was considered among the most difcult in the competition (there were only a handful of solutions among the $600+$ participants). What makes the problem difcult is the highly non-linear structure of the solution. Most participants would follow the standard "generative" solution process and investigate approaches based on the convex hull construction or use tools from Hamiltonian graph theory, none of these leading to a solution. Instead, participants who solved the problem followed an experimental approach with a lot of geometric exploration and inductive reasoning. Moreover, the solution itself is not linear. It’s hard to see the utility of the proposed construction in the beginning without the analysis of the dynamics of $l$ . Essentially, to start generating the solution requires that we already know the full approach. The underlying generative process of the solution is not auto-regressive from left-to-right.  

We can formalize this argument through the interpretation of reasoning as a latent variable process (Phan et al., 2023). In particular, classical CoT can be viewed as  

![](images/c9d400a79bd62951a363cac2aabd041f71ae1d695b6a98d1affe998739c86cb1.jpg)  
Figure 1: Top: Performance of current frontier models by size on the HARP mathematics benchmark (Yue et al., 2024) by difculty level and topic. The OpenAI O1 series signifcantly out-performs prior generation models across the board. Source: Figure 3 in (Yue et al., 2024). Bottom Average number of tokens generated by each model grouped by difculty level, as well as average number of tokens in human-generated solutions (using GPT4 tokenizer). Source: Figure 4 in (Yue et al., 2024).  

$$
p_{\mathrm{data}}(\mathbf{a}|\mathbf{q})\propto\int\underbrace{p_{\mathrm{data}}(\mathbf{a}|\mathbf{s}_{1},\dots,\mathbf{s}_{n},\mathbf{q})}_{\mathrm{Answer\;Generation}}\prod_{\epsilon=1}^{n}p_{\mathrm{data}}(\mathbf{s}_{t}|\mathbf{s}_{<t},\mathbf{q})\,d\mathbf{S},
$$  

i.e., the probability of the fnal answer being produced by a marginalization over latent reasoning chains. We claim that for complex problems, the true solution generating process should be viewed as  

$$
p_{\mathrm{data}}(\mathbf{a},\mathbf{s}_{1},\ldots,\mathbf{s}_{n}|\mathbf{q})\propto\int\underbrace{p_{\mathrm{data}}(\mathbf{a},\mathbf{s}_{1},\ldots,\mathbf{s}_{n}|\mathbf{z}_{1},\ldots,\mathbf{z}_{k},\mathbf{q})}_{\mathrm{Joint~Answer+CoT}}\prod_{\ell=1}^{K}p_{\mathrm{data}}(\mathbf{z}_{t}|\mathbf{z}_{<t},\mathbf{q})\,d\mathbf{Z},
$$  

i.e., the joint probability distribution of the solution $(\mathbf{a},\mathbf{s}_{1},\ldots,\mathbf{s}_{n})$ is conditioned on the latent generative process. Notice that this argument is a meta-generalization of the prior CoT argument, hence why we will refer to the process ${\bf q}\rightarrow{\bf z}_{1}\rightarrow...\rightarrow{\bf z}_{K}$ as Meta-CoT.  

# 2.2. Why Does (Classical) CoT Fail?  

Based on the previous discussion, a natural question follows: Why do LLMs fail at these advanced reasoning tasks? Above we proposed that the pre-training and instruction-tuning corpora consist of data of the type $(\mathbf{q},\mathbf{s}_{1},\ldots,\mathbf{s}_{\mathbf{n}},\mathbf{a})$ , which do not contain the true data generating process as shown in Equation 1. Indeed, the solution to the windmill problem above is widely available on the internet, but there is little to no discussion about the ways in which commonly used convex hull or planar graph arguments fail. This is true in general - textbooks contain advanced proofs but not the full thought process of deriving these proofs. We can then apply the same general meta-argument of why CoT is necessary to the Meta-CoT case: simply because the conditional solution-level distribution $p_{\mathrm{data}}(\mathbf{a},\mathbf{s}_{1},\ldots,\mathbf{s}_{n}|\mathbf{q})$ (without the intermediate Meta-CoT) on hard reasoning questions can have arbitrarily high complexity in the same way that $p_{\mathrm{data}}(\mathbf{a}|\mathbf{q})$ can have arbitrarily high complexity in the standard CoT setting. We will examine some empirical evidence for our stance in the following sections.  

We will argue in the following chapters that the OpenAI o1 model series performs full Meta-CoT reasoning in an auto-regressive fashion at inference time. A useful analysis is presented in a new mathematics benchmark with challenging high-school Olympiad-level problems (Yue et al., 2024). Figure 1 sourced from that work shows the relevant results. First, we see that the o1 family of models signifcantly outperforms “standard” reasoning models across the board. However, the gap between o1 and other models’ performance increases on higher difculty problems (with the interesting exception of the LLaMa 3.1 models), that is, problems which have higher solution complexity.  

Furthermore, the bottom half of Figure 1 shows the average number of tokens generated grouped by problem difculty level. First, we see that outside of the o1 series of models, LLMs generate solutions of comparable lengths to humans. While this may initially appear quite intriguing, suggesting that models are learning to approximate or replicate human reasoning, the simple explanation is that models are learning solutions to match the training data - i.e. $p_{\mathrm{data}}(\mathbf{a},\mathbf{s}_{1},\ldots,\mathbf{s}_{n}|\mathbf{q})$ . Much more intriguingly, the o1 series of models exhibits signifcantly diferent token behavior. We see that:  

1. On level 1 problems the o1 series generates a comparable number of tokens to human-written solutions. These are the types of problems where the training solutions likely match the true data generation process and each individual logical step can be internalized in a constant-depth transformer.   
2. At higher difculty, the o1 series of models generates signifcantly more tokens per problem and also widens the performance gap over the classical reasoning models. In fact the gap between the inference compute used by the o1 model and prior series of models seems to scale with the complexity of the problems. We hypothesize that in those more challenging problems the solutions do NOT in fact represent the true data generative process, which is instead better approximated by the more extensive Meta-CoT generated by the o1 family of models.  

Of course, in practice the distinction between these two is not so clear cut, and in fact the constantdepth transformer can likely internalize part of the Meta-CoT generative process as evidenced by the gradation of (Meta-)CoT lengths from Levels 2-6 in Figure 1. In the next chapter we will discuss in greater detail what the Meta-CoT process actually represents.  

# 3. Towards Deliberate Reasoning With Language Models - Search  

In the previous section we introduced the Meta-CoT process and argued that LLMs fail on advanced reasoning tasks because the training data does not adequately represent the true data generation process, i.e. text corpora do not include (or only include limited amounts of) Meta-CoT data. So the remaining question is: what does the true data generating process look like?  

![](images/d40fc4fd7b425ad8d47d6f47c4d5457474a7de7326182e213bed2827516f8caf.jpg)  
Figure 2: Train-time scaling curves for increasing quantities of training data during supervised fne-tuning of Llama3.1 8B, when evaluating for accuracy on the MATH test set. Left: A comparison of training on on the original data $\bf{(x)}$ and base fltered data (o) and evaluated using either greedy or majority vote decoding. For all settings, the fltered dataset shows improved scaling, and has not plateaued. Right: A comparison of pass $@_{\mathrm{k}}$ for increasing $k$ with a model trained on the base fltered dataset. The chart shows how increasing $k$ leads to a much greater probability of at least 1 correct solution, even for a relatively small model. Additionally, the performance improvement from $k=32$ to 64 does not show plateauing, suggesting that $k$ can further be increased for improved performance.  

1. First, we argue that for many advanced reasoning or goal-oriented problems there exist meaningful gaps between the complexity of generation and verifcation. This is of course one of the fundamental open problems of theoretical computer science and any attempt to prove this is signifcantly beyond the scope of the current writing, but we will review what we believe to be compelling empirical evidence from the literature.   
2. Second, assuming a non-trivial generator-verifer gap, we argue that the solutions to challenging problems present in text corpora are the outcomes of an extended search process, which itself is not represented in the data.  

We will dive deeper into these two points in the remainder of this section.  

# 3.1. Inference-Time Compute: Search  

The frst point above (generation-verifcation gap) has recently become a popular research and discussion direction under the framework of “deploying inference-time compute” and we explore this in our frst experiment. We start with a LLaMa 3.1 8B base model (Dubey et al., 2024) and carry out extensive supervised fne-tuning on the Numina MATH dataset (LI et al., 2024). Refer to Figure 2 for results and Section 8.1 for dataset details. For each intermediate checkpoint we evaluate performance on the Hendrycks MATH (Hendrycks et al., 2021) 500 problems evaluation dataset (Lightman et al., 2023). Based on the results, we make a few observations here:  

1. We evaluate pass $@k$ (i.e. using an oracle verifer) on intermediate checkpoints and see a signifcant jump in performance for increasing $k$ . While zero-shot performance with greedy  

![](images/8cc13ba40376bdede242a8bbb1e64189ca212f1f0fff2e17d16625a0f5926730.jpg)  
Figure 3: Scaling trends for verifer models on algorithmic reasoning, grade-school math (GSM8k), and transfer from GSM8k to MATH. The performance of all verifers improves in the best-of-N setting, as N increases. Figure sourced from (Zhang et al., 2024a).  

decoding improves from about $20\%$ to $40\%$ (see the base flter on the left side of Figure 2), even the frst model checkpoint outperforms these results at pass $@4$ (right side of Figure 2). Moreover, the pass $@64$ for the fnal checkpoint of an 8B model achieves accuracy close to $85\%$ , outperforming the zero-shot performance of many current frontier models. 2. We also evaluate performance under majority voting with $k=8$ and $k=64$ . There is continuous improvement for both increased training and samples, with maj $@64$ outperforming the greedy model performance with only $\mathbf{15\%}$ of the training compute without access to a ground-truth verifer.  

These results demonstrate that even as we directly optimize for answer generation ability by fnetuning on increasing amounts of SFT data, there remains a consistent verifer-generator gap, as evidenced by the improved performance in botht eh pass $@\mathbf{k}$ and majority voting settings. Recent literature has observed similar results on post-training sampling (Lightman et al., 2023; Brown et al., 2024; Snell et al., 2024). However, most of these studies do not systematically evaluate the efects of varying amounts of training data, compute, and model size which we believe is a fruitful direction for additional empirical work. These questions are important as the observed gains from additional inference might disappear at larger scales and training - i.e. the model may be able to fully internalize the reasoning process. This defnitely seems to be the case for advanced models and simpler benchmarks like GSM8k (Cobbe et al., 2021). While we observe the opposite result in our experiments, we admit that our results are the outcomes of preliminary study and additional work is required, but we will argue from a theoretical point in Section 6 that a persistent search gap remains in domains with high enough epistemic uncertainty. Besides this point, the question remains whether the improvement from increased inference can be efectively achieved without oracle verifers or environment feedback. In theory, it is possible to generate correct solutions under an increased inference budget, but we may not be able to verify them efectively, as verifcation complexity may be just as high as, or even higher than, generation complexity. We will address this issue next.  

# 3.2. Inference-Time Compute: Verifcation  

Several works focus on training verifer models, which explicitly evaluate the correctness of reasoning steps and solutions. Verifers can be trained either using explicit binary classifcation (Cobbe et al., 2021; Lightman et al., 2023; Snell et al., 2024; Anonymous, 2024; Setlur et al., 2024b) or modeling evaluation directly in natural language, using the LLM-as-a-judge prior (Zhang et al., 2024a; Mahan et al., 2024). The unifying formulation of these approaches is the model $v_{\theta}$ which evaluates a reasoning process $v_{\theta}(\mathbf{q},\mathbf{S})\rightarrow[0,1]$ . Under this framework, $K$ candidate solutions $(\mathbf{S}^{1},\ldots,\mathbf{S}^{K})$ can be generated from a fxed generator $\pi_{\boldsymbol{\theta}}(\cdot|\mathbf{q})$ and ranked based on their evaluation score.  

![](images/c326b3bd84e1cf9d2ef3343e155abe44eda84f56eb4d7ca8b8da7f955c06e9b0.jpg)  
Figure 4: Reasoning via Planning (RAP) demonstrates the search procedure described here. If we have access to a state evaluator, we can truncate branches with low values and backtrack to promising nodes, without resampling the same steps again. Source: Figure 2 in (Hao et al., 2023).  

$$
\mathbf{S}^{*}=\arg\operatorname*{max}\{v_{\boldsymbol{\theta}}(\mathbf{q},\mathbf{S}^{1}),\dots,v_{\boldsymbol{\theta}}(\mathbf{q},\mathbf{S}^{K})\}
$$  

For empirical results, we refer the reader to Figure 3 sourced from (Zhang et al., 2024a) which evaluates a number of verifer models $v_{\theta}$ . Regardless of the efciency of the verifer, there is a signifcant improvement in performance with additional online sampling. Moreover using explicitly trained verifer models outperforms naive inference-compute scaling strategies such as self-consistency or majority voting.  

A question remains regarding the efect of using a fxed generation model (policy): Could this model be under-trained, and if it were further trained, could its zero-shot performance improve to the point where additional online search no longer provides meaningful improvement? We will address this in Section 3.4.  

# 3.3. From Best-of-N To General Search  

So far, we empirically explored best-of-N approaches, generating multiple full solutions independently and selecting the most promising one based on scores. However, this approach is inefcient because it requires exploring full solution paths, even if a mistake occurs early on, and may repeatedly sample the same correct steps. Instead, we can model reasoning as a Markov Decision Process (MDP), defned by the tuple $\mathcal{M}=(\mathcal{S},\mathcal{A},P,R,\gamma)$ , where:  

• $\mathcal{S}$ : the set of states, where each state $\mathbf{S}\in{\mathcal{S}}$ , consists of the prompt and generations so far, i.e. $\mathbf{S}_{t}=(\mathbf{q},\mathbf{s}_{1},\ldots,\mathbf{s}_{t})$ .   
• $\boldsymbol{\mathcal{A}}$ : the set of actions, where each action $\mathbf{a}\in A$ will be represented as the next reasoning step $\mathbf{a}_{t+1}=\mathbf{s}_{t+1}$ .   
• $P(\mathbf{s}^{\prime}\mid\mathbf{s},\mathbf{a})$ : the transition probability function, representing the probability of transitioning to state ${\bf s}^{\prime}$ when taking action a in state s. For simplicity, we will mostly consider the deter  

ministic transition function $P(\cdot|\mathbf{s}_{t+1},(\mathbf{q},\mathbf{s}_{1},\ldots,\mathbf{s}_{t}))\to(\mathbf{q},\mathbf{s}_{1},\ldots,\mathbf{s}_{t},\mathbf{s}_{t+1})$ that appends the next reasoning step to the context. In general, the environment dynamics can be more complex. For example, models with tool access have to call the actual tool and receive the environment feedback in context or even modify their environment such as the cases of SWE and Web agents. • $R(\mathbf{s},\mathbf{a})$ : the reward function, which provides a scalar reward for taking action a in state s. We will assume zero intermediate rewards and fnal reward of 1 for a correct solution and zero otherwise, although this is not strictly necessary in the presence of a good process reward model (Setlur et al., 2024c). • $\gamma\in[0,1]$ : the discount factor, balancing the trade-of between further computation and rewards.  

We refer to the LLM generating the reasoning steps as the policy $\mathbf{s}_{t+1}\sim\pi_{\theta}(\cdot|\mathbf{S}_{t})$ . In addition we refer to a solution starting from $\mathbf{s}_{0}=\mathbf{q}$ as an episode or a trajectory. We will also use the notation $\mathbf{z}_{t}$ to represent individual reasoning steps that are part of the Meta-CoT and correspondingly denote $\mathbf Z_{t}=(\mathbf q,\mathbf z_{1},\ldots,\mathbf z_{t})$ .  

In the prior section we considered generating and ranking full solutions, which may be inefcient. We can extend the concept of a solution verifer from the prior section, to estimating the probability that a particular intermediate state will lead to a solution: $v_{\theta}(\mathbf{q},\mathbf{S}_{t})\rightarrow[0,1]$ . These models have become more widely known as Process Reward Models (PRMs) (Lightman et al., 2023). If we have access to such a model, we can improve the efciency of the search process with the following steps:  

1. Terminate a solution attempt that is not making progress, or is incorrect prior to reaching the fnal answer.   
2. Reset the agent to any intermediate, previously visited, state that has a high likelihood of success.  

Notice that with these two operations, and the general structure of language, we can implement any tree search procedure. This is the premise of several approaches (Yao et al., 2023; Hao et al., 2023; Zhou et al., 2024a) with the RAP method (Hao et al., 2023) illustrated in Figure 4.  

These approaches use difering search strategies (DFS/BFS vs. MCTS) and process guidance evaluation (generative self-evaluation vs. Monte-Carlo rollouts), but they all share the same core idea: formulate the reasoning problem as tree search guided by an intermediate heuristic function. As noted above, in theory, tree search does not induce a fundamental capability shift over parallel sampling, however, it may induce signifcant efciency gains as demonstrated by Yao et al. (2023). In particular, Figure 5 (sourced from Yao et al. (2023)) shows nearly 4 times increased efciency, in terms of inference budget, on a toy reasoning problem (Game of 24) when using a tree-structured search approach compared to parallel sampling. While these earlier works focus on zero-shot (or close to zero-shot) performance on simple reasoning tasks, it is important to note that tree-search methods have been successfully scaled and deployed to a number of realistic agentic applications (Koh et al., 2024; Putta et al., 2024; Brown et al., 2024; Yu et al., 2024).  

![](images/a7d537ec55e701ed6a16e857a93d3c9ea9b0f1f7bf425e6221cc62ec231c7b2a.jpg)  
Figure 5: ToT efciency on the game of 24 shown as accuracy (y-axis) vs. # nodes visited (x-axis). Source: Figure 3 in Yao et al. (2023).  

![](images/f2d6bfb7b0ec94e2d7a3d9de3796a7190126c223c0c4cd4a739f099ebd45a182.jpg)  
Figure 6: Scaling trends for MCTS at training and inference-time on board games. Left: Elo scores of models trained with diferent architectures (depth $\times$ width) where each point represents the Elo score of that model evaluated with the labeled tree size (between 1 to 512 nodes). The curves demonstrate that the performance of each model snapshot follows a sigmoid pattern with respect to the test-time compute budget. Source: Figure 8 in (Jones, 2021). Right: The trade-of between train-time and test-time compute, with progressively improving Elo (from bottom-left to top-right). Source: Figure 9 in (Jones, 2021).  

# 3.4. Is Search (Inference Time Compute) A Fundamental Capability Shift?  

As pointed out earlier, the question remains whether inference-time search is a fundamental new capability or whether it is accessible with additional training. Results from classical RLHF tuning (Dubois et al., 2024) suggest that this is a learnable capability, where zero-shot performance of post-trained models matches or outperforms the best-of-N paradigm.  

We stipulate that performance on complex reasoning tasks is governed by a scaling law, which involves model size, training data (compute) and inference time compute.  

This is indeed consistent with the theoretical results of Li et al. (2024) and the intuition presented in Section 2. Larger models are more capable of internalizing the Meta-CoT process in their activations, and are also capable of using longer inference-time Meta-CoT to approximate solutions with signifcantly higher computational complexity. Empirically, we have limited (but promising) evidence towards this hypothesis. A major prior work to study these questions is Jones (2021) which carries  

out studies using the AlphaZero algorithm (Silver et al., 2018) on board games. This approach fts our desiderata very well as the underlying MCTS algorithm jointly scales the policy and value (verifer) models’ training in conjunction with search. Moreover, this family of board games have a clear generator-verifer gap as generating optimal strategies at intermediate steps can be quite computationally complex, while verifying a winning condition is trivial. The major empirical results on scaling are shown in Figure 6. On the right side we see that performance increases both with increased training compute and model size, as outlined earlier. Interestingly, on the left we see the performance  

<html><body><table><tr><td>Method</td><td>Policy</td><td>Value</td><td>Accuracy(%)</td></tr><tr><td>Greedy</td><td>T00</td><td></td><td>41.4 47.9</td></tr><tr><td rowspan="4">MCTS-Q</td><td>T01 T00</td><td>{v,r}po</td><td>51.9</td></tr><tr><td>T00</td><td>{v,r}Φ1</td><td>53.2</td></tr><tr><td>T01</td><td>{u,r}Φo</td><td>54.1</td></tr><tr><td>T01</td><td>{u,r}p1</td><td>56.5</td></tr></table></body></html>  

Table 1: Iterative update results on GSM8k. $\theta_{0},\phi_{0}$ are the old parameters while $\theta_{1},\phi_{1}$ are the new ones. TS-LLM can boost performance by training LLM policy, value, or both. Source: Table 4 in (Feng et al., 2024).  

of using diferent quantities of compute (i.e., search with a value function) during inference. There is also a clear scaling trend, showing improved performance with additional online search at each intermediate checkpoint of training. In fact, the results in this domain indicate there is a clear log-log scaling trade-of between train-time and test-time compute deployment. Currently, we have limited evidence of similar scaling laws in LLMs because such a training pipeline requires signifcant resources. One major work towards that goal is Feng et al. (2024) which carries out two iterations of MCTS fne-tuning using a LLaMa 7B on the GSM8k dataset (Cobbe et al., 2021). They show improved performance in zero-shot evaluations of the policy, as well as signifcant gains from using additional inference-time search, at both iterations 1 and 2 (full results are shown in Table 1). However, their work does not ablate the model size, data scaling, or inference-time search scaling, which remain under-explored in the literature for LLM reasoning.  

# 4. Towards Meta-CoT Reasoning  

In prior sections we: introduced the concept of Meta-CoT and argued that it is necessary for advanced reasoning, discussed the generator-verifer gap as a fundamental limitation, argued for search as a fundamental building block of the Meta-CoT, and discussed the utility of approaches integrating generator, verifer, and search components. However, the question remains on how to integrate these into a model to perform Meta-CoT or “System 2” reasoning. The frst question we need to answer is: why do we actually need to internalize deliberate reasoning inside a single model? We propose two main reasons:  

1. Efciency: By incorporating search within the context of an auto-regressive model, exploration can be done efciently since the model has access to all previously visited nodes, in context. Unique to the case of reasoning in natural language, many branches may contain semantically similar content, unlike other domains (e.g., board games), motivating the need for improved efciency. In fact, even advanced reasoning models carry out many repeated steps of semantically identical reasoning as we show in Figure 14 and 15.   
2. Super-Intelligence: If an auto-regressive model can learn to implement search algorithms in-context, then additional RL training may enable the model to discover novel reasoning approaches. Essentially, we propose that training a model capable of internal System 2 reasoning (e.g. Meta-CoT) and search is an optimization over algorithms rather than specifc outputs, possibly yielding novel modes of problem solving. This will potentially allow the model to solve classes of problems previously unsolvable under symbolic-bases tree-search approaches as we’ve outlined in Sections 3.3 and 3.4.  

In the remainder of this section, we explore how to train a model to internalize such a reasoning system.  

# 4.1. Bootstrapping Meta-CoT  

In this subsection, we overview the core idea behind the Self-Taught Reasoner (STaR) approach (Zelikman et al., 2022; Singh et al., 2024; Yuan et al., 2023) to bootstrapping intermediate CoT steps and how to use a similar concept to generalize to meta-reasoning strategies.  

# 4.1.1. Self-Taught Reasoner  

The STaR method introduces an iterative bootstrapping approach designed to improve the reasoning capability of LLMs (Zelikman et al., 2022). STaR focuses on training models to generate and refne rationales, particularly for tasks requiring complex reasoning in a reinforcement learning-based manner. In this formulation we assume we have access to a dataset $\mathcal{D}=\{\mathbf{q}^{(i)},\mathbf{a}^{(i)}\}_{i=1}^{N}$ of questions $\mathbf{q}$ that require reasoning along with corresponding answers a. Notice that we do not require access to ground-truth rationales for these problems. We begin by prompting a model $\hat{\mathbf{a}}^{(i)},\hat{\mathbf{S}}^{(i)^{}}\sim\pi(\mathbf{a},\mathbf{S}|\mathbf{q}^{(i)})$ to provide CoT rationale $\hat{\mathbf{S}}^{(i)}=\mathbf{s}_{1}^{(i)},\ldots,\mathbf{s}_{N_{i}}^{(i)}$ and fnal answer $\hat{\mathbf{a}}^{(i)}$ . We then flter the generated data, keeping only rationales that lead to a correct fnal answer (i.e., $\hat{\mathbf{a}}^{(i)}=\mathbf{a}^{(i)})$ to create a dataset of questions, (bootstrapped) rationales and answers $\mathcal{D}_{\mathtt{S T a R}}=\{\mathbf{q}^{(i)},\hat{\mathbf{S}}^{(i)},\mathbf{a}^{(i)}\}_{i=1}^{N}$ . $\mathcal{D}_{\mathrm{STaR}}$ is then used to train a model with the standard supervised fne-tuning objective:  

$$
\begin{array}{r}{\mathcal{L}_{\mathrm{STaR}}(\pi_{\phi})=-\mathbb{E}_{(\mathbf{q},\hat{\mathbf{S}},\mathbf{a})\sim\mathcal{D}_{\mathrm{STaR}}}\left[-\log\pi_{\phi}(\mathbf{a},\hat{\mathbf{S}}|\mathbf{q})\right].}\end{array}
$$  

The above procedure is repeated over several iterations. The core idea behind STaR is to generate a training dataset of synthetic rationales through sampling and verifcation. We will extend that idea to the the concept of Meta-CoT below.  

# 4.1.2. Meta-STaR  

We can generalize the above idea to Meta-CoT in a straightforward way. Consider a base policy $\pi_{\theta}$ combined with some general search procedure over intermediate steps. Given a question q we perform the search procedure repeatedly to generate search traces $\hat{\mathbf{z}}_{1},\hdots,\hat{\mathbf{z}}_{K}$ until we fnd a fnal solution $\left(\mathbf{s}_{1},\ldots,\mathbf{s}_{n}\right)$ . If we can verify the fnal produced solution $v(\mathbf{S})\to\{0,1\}$ , for example by using a formalization and verifcation approach (as in AlphaProof1) or some other outcome verifcation, we can then apply a similar approach to STaR. For example, we can construct a dataset $\mathcal{D}_{\mathrm{STaR}}=$ $\{\mathbf{q}^{(i)},\hat{\mathbf{Z}}^{(i)},\hat{\mathbf{S}}^{(i)}\}_{i=1}^{\bar{N}}$ and use a similar training objective as before:  

$$
\mathcal{L}_{\mathrm{Meta-STaR}}(\pi_{\phi})=-\mathbb{E}_{(\mathbf{q},\hat{\mathbf{Z}},\hat{\mathbf{S}})\sim\mathcal{D}_{\mathrm{STaR}}}\left[-\log\pi_{\phi}(\hat{\mathbf{S}},\hat{\mathbf{Z}}|\mathbf{q})\right].
$$  

Essentially, we can use a base policy and search procedure to generate synthetic search data and then train the model to implement these in-context through the Meta-CoT concept. We are efectively proposing to linearize the search approaches described in Section 3 and teach an auto-regressive model to run them sequentially. So far we have deliberately been vague about how these search procedures and datasets look. We will now provide examples and proof of concept from the literature on practical approaches to this problem as well as synthetic examples of realistic training data.  

# 4.2. Empirical Examples Of Internalizing Search  

When we formulate search in a sequential fashion we can explicitly parameterize each component in language, or choose leave it implicit (Gandhi et al., 2024). Note that models trained with standard next token prediction still need to implicitly internalize all of these components anyway in order to accurately model the search sequence, even if they are not explicitly verbalized. However, allowing the model to vocalize it’s certainty or estimated progress could allow for additional modeling capacity or be useful for interpretability purposes. We will present some examples of auto-regressive search procedures from the literature in the following section.  

# 4.2.1. Small-Scale Empirical Results on Internalizing Search  

Two particular prior works that explore the idea of in-context search are Yang et al. (2022) and Lehnert et al. (2024) which focus on mazes and other classical RL environments. The formulation from Lehnert et al. (2024) is shown in Figure 7, which illustrates linearizing $\mathrm{A^{*}}$ search. In our framework the “Trace” corresponds to the Meta-CoT Z, and the “Plan” is the CoT output S. In this setting the search procedure is stated explicitly as it shows node states, actions, costs and heuristic values. In this “stream” format we can then use standard auto-regressive language models with a next token-prediction objective to train a model to internalize the search process. Evaluation results are shown in Figure 8 sourced from the same paper. We observe empirical efects consistent with the scaling law hypothesis presented in 3.4; there is consistent improvement with additional training data and model size (train-time compute) across the board. A particularly interesting observation is the complexity scaling relationship in part (c) of the fgure. At smaller mazes (lower complexity) the model directly producing the Plan (CoT) and performs comparably to smaller search (Meta-CoT) augmented models, however as maze size (complexity) increases we see a widening gap in performance between the search-augmented and zero-shot models. This is essentially identical to the results shown in Figure 1 on the challenging HARP benchmark (Yue et al., 2024) between the prior frontier models and the o1 series. These empirical observations are well aligned with the intuition we presented in Section 2. For small mazes (low complexity problems) models are capable of internalizing the reasoning process, but as problem complexity (maze size) increases this becomes more challenging and model performance falls of compared to models which explicitly carries out a search procedure. Unfortunately, Lehnert et al. (2024) did not publish inference compute scaling laws, but given the algorithmic structure of the training data we can presume that inference-time tokens scale with the same complexity as the $\mathrm{A^{*}}$ search algorithm, which can be exponential in the branching factor, while the plan length is linear in $n$ . These results would also be consistent with the inference costs on advanced math reasoning tasks reported in Figure 1.  

![](images/810be9ce2ce3da21395da14d1e32ed9a21fc0081f6cfa661c477651e93d09b9a.jpg)  
Figure $7\colon\mathbf{A}^{*}$ planning algorithm outline for a simple maze navigation task, along with a state and action tokenization scheme. The search representation explicitly models nodes and queue state, the search procedure and the cost and heuristic evaluation. Source: Figure 1 in (Lehnert et al., 2024).  

![](images/7e06cda57bc5cfda0dac62b602ad8a114e57ef9d479d08bf611db4c2385e1ec3.jpg)  
Figure 8: Model performance vs. training compute when using the $\mathsf{A}^{*}$ planning algorithm (Search Augmented) vs. no search (Solution Only). We see that the search augmented models perform much better across all training scales (charts a and b). In particular performance is consistent with the search formulation of Section 3.4. Figure c) shows performance in terms of task complexity as maze size increases. Results are consistent with the Meta-CoT complexity argument presented in Section 2 and results on the HARP benchmark in Figure 1. Source: Figure 2 in (Lehnert et al., 2024).  

![](images/1b26d4f624508324ccfb855c6718b15c6633e921b1cac96e6dd6aec78f538530.jpg)  
Figure 9: Inference compute scaling relationships for the o1 model (Left, sourced from (OpenAI, 2024) on AIME, Stream-of-Search on the Game of 24 (Middle) and MAV-MCTS on Chess (Right, sourced from (Schultz et al., 2024)). These fgures show performance of a single model under diferent token sampling budgets.  

Gandhi et al. (2024) extend the linearized search idea to a more realistic reasoning task - the Countdown game - which requires the model to predict a sequence of mathematical operations on a given set of numbers to match a target value. While Gandhi et al. (2024) use a fxed 250M parameter transformer model and do not explore or discuss the role of model size, training data, and complexity in terms of scaling performance, we obtain additional results in terms of inference-time scaling, shown in Figure 9. Our fndings demonstrate a consistent log-linear relationship between tokens spent and success rate. Similar results were also observed in recent work by Schultz et al. (2024), who train language models on linearized search traces obtained from MCTS on board game environments. Similar to the work of Gandhi et al. (2024), they fnd consistent improvements in performance as the model is given additional search budget at test-time (Figure 9 right). Note that these models demonstrate an inference-time scaling law with the same functional form as the o1 model on difcult  

mathematics problems (OpenAI, 2024).  

# 4.2.2. In-context Exploration For LLMs  

While the prior section showed promise in teaching auto-regressive language models to internalize complex search strategies involving exploration and backtracking, it remains unclear whether these results can generalize to realistic language domains. In this section we will overview several recent works, which show promise in internalizing episode-level search. Both Qu et al. (2024) and Snell et al. (2024) evaluate results using open-source LLMs in the 7B and larger range on problems from the MATH dataset (Hendrycks et al., 2021). They pose the problem as sequential sampling - i.e. given a problem $\mathbf{q}_{\mathrm{:}}$ , generating full solutions from the same model auto-regressively as  

$$
\mathbf{S}^{j}\sim\pi_{\boldsymbol{\theta}}(\cdot|\mathbf{S}^{j-1},\ldots,\mathbf{S}^{1},\mathbf{q})
$$  

where $\mathbf{S}^{i}$ are full solutions to the problem q. Both works formulate the problem as self-correction, or revisions, during training. The approach generates training data by concatenating a number of incorrect solutions with the correct revision and training on a linearized sequence (although the exact training objective use a particular weighting grounded in RL (Peng et al., 2019)). The general objective follows the form  

$$
\operatorname*{min}_{\theta}\ \mathbb{E}_{\mathbf{S}^{i}\sim\pi_{\mathrm{ref}}(\cdot|\mathbf{q}),\mathbf{q}\sim\mathcal{D}_{\mathrm{train}}}\left[-\log\pi_{\theta}(\mathbf{S}^{*}|\mathbf{S}^{j-1},\cdot\cdot\cdot,\mathbf{S}^{1},\mathbf{q})\right]
$$  

where $j$ is a fxed number of in-context exploration episodes sampled from a fxed distribution $\pi_{\mathrm{ref}}$ (i.e. $\pi_{0}]$ ) and $\mathbf{S}^{\ast}$ is some optimal solution. Essentially, this can be considered a linearization of the Best-Of-N search strategy presented in Section 3.1 with rejection sampling. In this setting, the Meta-CoT represents search in full episodes $\mathbf{Z}=\mathbf{S}^{1},\ldots,\mathbf{S}^{j-1}$ and $\mathbf{S}=\mathbf{S}^{j}$ . At test time we can further control the quantity of compute by iteratively sampling from  

$$
\mathbf{S}^{i}\sim\pi_{\boldsymbol{\theta}}(\cdot|\mathbf{S}^{i-1},\ldots,\mathbf{S}^{i-j},\mathbf{q}).
$$  

Representative results for this approach are are shown in Figure 10, sourced from (Snell et al., 2024). We see clear improvement in the pass $@1$ metric with additional amounts of in-context exploration episodes with nearly $6.7\%$ gain from zero-shot to the level of saturation. At the same time, autoregressive generation shows clearly better scaling properties than independent parallel sampling (Figure 10 right). These results indicate that the model learns some degree of in-context exploration and self-correction.  

# 4.2.3. Using variable Compute  

While the above approaches demonstrate promise for the model’s capability to carry-out in-context search, they are trained with a fxed number of revisions and use a pre-determined number of revisions at test time. This is not ideal, as ideally the model would be able to use arbitrary amounts of compute until it arrives at a solution with high enough confdence. We repeat the above experiment using a uniform number of incontext solutions during training (ranging between 0-7), allowing the model to generate up to 8 solutions at inference time by optimizing  

![](images/805e720727980c6cc00a3be98e55c700b962aaa02ef9d53cf4dd18d8f8c3a15c.jpg)  
Figure 11: Number of in-context revisions the model attempts grouped by difculty level.  

![](images/bd1ea8c51567adfe748321d6ca8f2cc393788189ae2c97d7aadb74dc5083a0b1.jpg)  
Figure 10: Left: $\operatorname{Pass}(\mathcal{O}1$ accuracy of a revision model after the specifed number of generations (revisions). Right: Scaling performance of the best-of-N strategy under parallel and auto-regressive (in-context) sampling. The performance gap indicates that the model learns some degree of in-context exploration and self-correction. Source: Figure 6 from (Snell et al., 2024).  

$$
\operatorname*{min}_{\theta}\ \mathbb{E}_{{\mathbf{S}}^{i}\sim\pi_{\mathrm{ref}}(\cdot|{\mathbf{q}}),{\mathbf{q}}\sim\mathcal{D}_{\mathrm{rain}}}\left[-\log\pi_{\theta}({\mathbf{S}}^{*},{\mathbf{EOS}}|{\mathbf{S}}^{j-1},\ldots,{\mathbf{S}}^{1},{\mathbf{q}})\right],j\sim\mathrm{Unif}(1,8)
$$  

This formulation encourages the model to continue revising it’s solution until it reaches a solution with high confdence of correctness. Interestingly, our model generates an increasing number of solutions based on question difculty. Summary statistics by problem difculty are shown in Figure 11 (right), where the model generates an average of 2.45 solutions for Level 1 problems and an average of 5.84 for Level 5 problems, consistent with the behavior shown in Figures 1 and 8 (c). Specifcally, this demonstrates that the model has internalized the need for extended exploration on complex reasoning tasks.  

Our best performing run achieved an improvement of $2\%$ over the LLaMa 3.1 8B Instruct model that we initialize our models from. We attribute this to a mismatch between the instruct model’s RL post-training, the of-policy SFT fne-tuning we utilize, and the limited amount of training data in the MATH train dataset. Indeed, any regular SFT training we performed on the RL post-trained model actually worsened performance. We are currently exploring post-training on pre-RL base models with extended datasets.  

# 4.2.4. Backtracking in LLMs  

In the prior sections, we reviewed evidence that auto-regressive models can internalize complex search strategies in simple domains. We also showed that LLMs can learn in-context exploration at the episode-level. However, whether models can implement complex search strategies (e.g. those outlined in Section 3) auto-regressively remains an open question in public research. Specifcally, we refer to the ability to terminate a reasoning chain prior to completion, and the ability to reset (semantically) to an arbitrary previously visited state in-context. These two steps can be unifed under the concept of backtracking. Here we will review some recent works demonstrating that LLMs can learn to backtrack.  

Recent works have demonstrated that training on data with backtracking can improve language models on simple reasoning tasks (Ye et al., 2024b; Anonymous, 2024) fnd that language models can sometimes “recognize” their errors internally, but do not have the required mechanisms to self-correct.  

Similar to our motivation with Meta-CoT, their aim is for a single model to both recognize errors and self-correct in-context. In their approach they create training data with incorrect steps directly followed by the correction. The backtracking is signifed by a special token, [BACK], at the end of an incorrect step to allow the model to explicitly state it’s belief that an error has occurred. That is: given a dataset $\mathcal{D}_{\mathrm{train}}$ of questions $\mathbf{q}$ and correct reasoning CoT solutions $\mathbf{S}=\mathbf{s}_{1},\ldots,\mathbf{s}_{n}$ the training objective becomes  

$$
\mathcal{L}_{\mathrm{backtrack}}(\theta)=-\mathbb{E}_{\mathbf{s}_{1},\dots,\mathbf{s}_{n}\sim\mathcal{D}_{\mathrm{train}},t\sim\mathrm{Unif}(1,n)}\left[\log\pi_{\theta}(\mathbf{s}_{1},\dots,\mathbf{s}_{t}^{-},[\mathrm{BACK}],\mathbf{s}_{t},\dots,\mathbf{s}_{n}|\mathbf{q})\right]
$$  

where $t$ is a randomly sampled time step in the solution and $\mathbf{s}_{t}^{-}$ is a single incorrect reasoning step. This is in contrast to the standard approach, which only trains on the correct solution chains:  

$$
\mathcal{L}_{\mathrm{standard}}(\theta)=-\mathbb{E}_{{\mathbf{S}}\sim\mathcal{D}_{\mathrm{train}}}\left[\log\pi_{\theta}({\mathbf{S}}|{\mathbf{q}})\right].
$$  

Ye et al. (2024b) explore inserting incorrect steps at varying rates (between $1\%$ and $50\%$ ) and fnd that high rates of incorrect steps actually leads to improved downstream performance. In particular, they fnd that a $50\%$ rate of incorrect steps (objective in Equation 8) leads to an increase from $78\%$ to $94\%$ accuracy on hard math problems as compared to training on only correct solutions (Equation 9, CoT). While promising, these results are only verifed on small models (124M parameters).  

In contrast, Zhang et al. (2024b) teach LLMs to backtrack based on safety considerations using the larger Gemma 2B and LLaMa 3 8B models. In particular, following the above notation, given a prompt q and two possible answers - a safe option $\mathbf{S}^{+}=\mathbf{s}_{1}^{+},\ldots,\mathbf{s}_{n}^{+}$ and an unsafe option $\mathbf S^{-}=\mathbf s_{1}^{-},\ldots,\mathbf s_{n^{\prime}}^{-}$ , where s here represent individual tokens (unlike before where they stood for logical steps), they optimize the objective:  

$$
\begin{array}{r}{\begin{array}{c c}{\mathcal{L}(\boldsymbol{\theta})=-\mathbb{E}_{(\mathbf{q},\mathbf{S}^{+},\mathbf{S}^{-})\sim\mathcal{D}_{\operatorname{train}},t\sim\mathrm{Unif}(1,n^{\prime})}[\log\pi_{\boldsymbol{\theta}}([\mathrm{BACK}],\mathbf{S}^{+}|\mathbf{S}_{t}^{-},\mathbf{q})}&{+}\\ {\log\pi_{\boldsymbol{\theta}}(\mathbf{S}^{+}|\mathbf{q})].}&{}\end{array}}\end{array}
$$  

That is a combination of the Meta-CoT and regular CoT objectives as outlined above. Additionally, notice that this objective masks out the unsafe completion, while the prior work trains on all tokens including the incorrect logical steps. While the approach of Ye et al. (2024b) backtracks for a single logical step (correction) this work always resets the agent to the initial state. SFT training is successful in teaching the model to backtrack and improves the safety characteristics over supervised fne-tuning on just the safe answer (only the second term of Equation 10). However, these efects appear weak in regular SFT models, but are signifcantly improved through further downstream RL training, which we will discuss later on.  

# 4.3. Synthetic Meta-CoT Via Search  

In the prior sections we argued for an approach to reasoning that teaches an LLM to internalize an auto-regressive search procedure in-context. We also reviewed several recent works showing that small auto-regressive models can carry out in-context exploration at the episode level, and larger models can learn individual step backtracking. In this section, we explore how to construct synthetic data for realistic Meta-CoT that involves full-scale in-context tree search.  

Setup. For demonstrative purposes, we use the math problem presented by OpenAI (2024) as our benchmark task, where Gemini 1.5 Pro (Reid et al., 2024) achieves a Pass $@128$ score of $6.25\%$ (8/128 correct) – notably being the only frontier model (without advanced reasoning) to demonstrate non-zero performance at the time of our experiments. We use the same RL formulations for state and actions as presented in 3.3. We explore two principal search algorithms for generating synthetic training data: Monte Carlo Tree Search (MCTS) and $\mathbf{A}^{*}$ variants. Both approaches necessitate a heuristic state estimation function, for which we employ pure Monte-Carlo rollouts following the methodology of Silver et al. (2018). Specifcally, we estimate the value of a partial solution trajectory as  

$$
v(\mathbf{S}_{t},\mathbf{q})=\mathbf{E}_{\mathbf{S}_{\geq t+1}^{j}\sim\pi_{\theta}(\mathbf{S}_{\geq t+1}\mid\mathbf{S}_{t},\mathbf{q})}\frac{1}{K}\sum_{j=1}^{K}r^{\ast}([\mathbf{S}_{\geq t+1}^{j},\mathbf{S}_{t}],\mathbf{q})
$$  

where $r^{*}$ is the verifable ground-truth outcome reward. In our experiments, we sample 128 completions from the partial solution and evaluate the mean success rate under ground-truth outcome supervision. In Appendix E, the numerical values of the states are listed after each step.  

# 4.3.1. Monte-Carlo Tree Search  

We conduct an example based on Monte-Carlo Tree Search (MCTS), which seeks to balance exploration and exploitation. The MCTS implementation of Silver et al. (2018) has been widely applied to the reasoning domain (Tian et al., 2024; Feng et al., 2024), and we mostly follow their implementation with some modifcations to account for the structure of our search problem (see Appendix D).  

We present the search trace for our example problem - all the actions taken during the search (i.e., the Meta-CoT in a linear format) - in Appendix E. The numbers following each reasoning step represent the value estimates. In our initial MCTS attempt we obtained a trace with an excessive number of backtracks and repetitions, including from high-value states (as  

![](images/1e822d7633271074830fd80949ae87d3d0bd4b10929bade8c4fe03d6772874f9.jpg)  
Figure 12: MCTS tree for the math problem presented by OpenAI (2024). The red node indicates the solution.  

high as 1.0) with the resulting exploration tree is shown in Figure 12. We believe these efects are due to the exploration bonus in MCTS search. We did not carry out extensive ablations on the search parameters due to speed and costs. Since we use pure MC rollouts ("simulations") for state value estimation, a single tree uses up to 20 million tokens inference (a cost of ${\sim}\mathfrak{S}100$ per tree). Moreover the process can take up to half an hour due to API limits. Because of these issues we also evaluate a more efcient best-frst exploration strategy, which we present below.  

# 4.3.2. $A^{*}$ search  

We begin with an exploration of a type of best-frst search based on the work by Koh et al. (2024), which itself loosely follows an $\mathbf{A}^{*}$ approach. The search procedure maintains a frontier $\mathcal{F}$ of states, which is implemented as a max priority queue. Similarly to the MCTS approach, each state $\mathbf{S}_{t}$ consists of the question q and a partial solution consisting of generated reasoning steps $\left(\mathbf{s}_{1},\ldots,\mathbf{s}_{t}\right)$ . At each iteration, the state $\mathbf{S}_{p}\gets\mathsf{p o p}(\mathcal{F})$ with the highest value $v_{p}=v(\mathbf{S}_{p},\mathbf{q})$ is selected, where $v_{p}\in[0,1]$ is the value of the partial solution $\mathbf{S}_{p}$ including current and previous reasoning steps. At each node the policy $\pi_{\phi}$ proposes $b$ candidate next steps, each of which is evaluated by $v$ and added to $\mathcal{F}$ if the depth of the tree $\left|(\mathbf{s}_{0},\ldots,\mathbf{s}_{p})\right|$ has not reached the maximum depth search limit $d$ . For the purpose of generating synthetic data, we run the search until we fnd a solution that is correct using the ground-truth verifer. The resulting tree is shown in Figure 13. It shows more consistent fow of the  

reasoning steps, with less backtracking concentrated around a few key steps.  

# 4.4. Do Advanced Reasoning Systems Implement In-Context Search?  

In this section we will investigate whether advanced reasoning systems, such as OpenAI’s O1 (OpenAI, 2024), DeepSeek R1 (DeepSeek, 2024) and Gemini 2.0 Flash Thinking Mode 2 and the Qwen QwQ Team (2024) implement in-context search. We provide successful reasoning traces for the same math problem in Appendix E.  

Starting with OpenAI’s o1 model, by carefully examining the provided mathematical reasoning trace, we observe:  

1. Inconsistent fow of thought - consecutive steps do not logically continue the prior state.   
2. Backtracking - the model carries out "semantic backtracking" - frequently returning to the same logical points.   
3. Repetition - the model often repeats logical steps.  

![](images/98af3e8774697add630cde871085b466600f00539cd676efc56ec7b6e7631f9c.jpg)  
Figure 13: Resulting $\mathbf{A}^{*}$ search tree on the math problem from OpenAI (2024). This trace presents more of a best-frst approach with fewer backtracks, concentrated around key steps, as compared to the one produced by MCTS in Figure 12.  

The qualitative behaviors observed in o1 (Figure 14 left) are similar to those in the example synthetic trace (Figure 15) generated by Gemini 1.5 with and MCTS-like search processes. In particular, there are abrupt changes in logical fow of the (Meta) CoT, which is natural as the model backtracks between branches of the tree. Moreover, the model may explore multiple child nodes of the same parent which are diferent strings, but can also be very semantically similar leading to repetitive logic. This is clear in the provided trace, as the model repeats logical statement and goes over the same derivations multiple times. Note also that we do not claim the model is implementing tree search at test time, but rather that as much as the model’s output are expected to resemble it’s training data, we hypothesize that examples of search were used during training (likely model initialization). We will specifcally address the need and efects of RL training in Section 6.  

The DeepSeek R1 model DeepSeek (2024) also exhibits similar behaviors, as shown in Figure 14, however, it also carries out a signifcant amount of self-evaluation steps. This could be achieved by integrating a form of self-criticism (Madaan et al., 2023; Shinn et al., 2023) or a generative verifer (Zhang et al., 2024a) in the search trace. The LATS framework (Zhou et al., 2024a) uses a similar approach, combining MCTS search with self-criticism and shows empirical improvements from self-refection. Another alternative for synthetic data generation is the "Iteration-Of-Thought" approach Radha et al. (2024) which also interleaves generation with inner dialogue. This would explain the rather smooth logical fow of the R1 model, which does not exhibit as much abrupt back-tracking, as compared to O1. As mentioned earlier, in order to adequately model the search  

![](images/ff95784bc2a2a536b113a34224efb0e90314567a8549af5571381c10aa596622.jpg)  

Figure 14: Examples of intermediate traces from o1 (OpenAI, 2024), DeepSeek-R1 (DeepSeek, 2024), and Gemini 2.0 Flash Thinking Mode. We highlight two types of steps: Backtracking, where the model visits a bad state and returns to a previously visited step, and Verifcation, where the model assesses the correctness of the previous output. Inconsistent logical fow and repetition are present in all three traces. DeepSeek-R1 and Gemini 2.0 Flash Thinking Mode both exhibit generative verifcation before reaching an answer, while Gemini makes an incorrect verifcation and returns to the initial state. Full search traces can be found in Appendix E.  

![](images/e5dc627fcbffd91c3d9ceac4c13fb791c5efac88f3fd3fba4cd7d16c15b20d3a.jpg)  
Figure 15: Example synthetic search traces from Gemini 1.5 with MCTS and $A^{*}$ . We highlight backtracking steps, where the model visits a bad state and returns to a previously visited step. Inconsistent logical fow and repetition are present in both traces, but stands out particularly in MCTS trace. Full search traces can be found in Appendix E.  

process the model must internalize an evaluation mechanism. However, providing an explicit CoT verifcation may be able to expand the model computational capacity and improve self-verifcation. This is an empirical question, which is currently unclear in open research.  

Gemini 2.0 Flash Thinking Mode appears to implement a somewhat diferent structure. Specifcally, the fow of reasoning qualitatively appears smoother with fewer logically inconsistent steps. Moreover, it backtracks less frequently and often returns to the initial state. In fact in the provided example the model solves the problem correctly and then fully re-generates a new solution from scratch (backtracks from the fnal state to the initial one). It’s behavior seems to be to attempt a full solution, which may be terminated early based on some search heuristic. In cases where the solution attempt is unsuccessful, the model attempts a diferent solution approach, rather than branch at the step-level in a tree search structure. This seems more consistent with a revision-based strategy as refected in past works (Qu et al., 2024; Anonymous, 2024; Kumar et al., 2024). The Qwen QwQ model Team (2024) shows similar behavior, generating multiple solutions in-context, as also pointed out by Chen et al. (2024).  

# 5. Process Supervision  

A key component of the search approaches presented in prior sections is the evaluation function $v(\mathbf q,\mathbf S_{t})$ , which scores intermediate states in a reasoning chain. These evaluation functions have become widely known as Process Reward Models (PRM). By incorporating process supervision, the search mechanism gains the fexibility to backtrack to earlier promising states when suboptimal paths are encountered, thereby enabling more efective exploration. However, the question of how to efciently access such capabilities remains an open question. In Section 4.3 we showed examples of using outcome-based verifcation with MCTS in combination with Monte-Carlo rollouts. However, this approach can only be used during training due to the necessity for ground-truth answers, and moreover it is extremely computationally inefcient. As mentioned earlier, a single training example requires up to 20 million inference tokens, costing up to hundreds of dollars. It is signifcantly more efcient to amortize the evaluation procedure into a single parameterized model, and we will outline strategies for building such process guidance models below.  

# 5.1. Learning Process Reward Models  

Parameterized PRMs are built on top of pre-trained models, either using a linear head or the logits of specifc tokens. The model takes the question $\mathbf{q}$ and a partial solution $\mathbf{S}_{t}$ as input and outputs a single scalar value $v_{\theta}(\mathbf{q},\mathbf{S}_{t})\rightarrow[0,1]$ . Given a dataset $\mathcal{D}_{\mathrm{train}}$ of partial solutions $\mathbf{S}_{t}$ and corresponding value targets $y{\bf s}_{t}$ the model is generally optimized with a standard cross-entropy classifcation loss. A central question for training PRMs is: where do the supervision labels $y{\bf s}_{t}$ come from? One approach is to have human annotators provide step-by-step level evaluation of reasoning problems, as done by Lightman et al. (2023). While their work showed promise in terms of empirical results, this method is challenging to scale due to the high annotation time and cost, especially as evaluating hard reasoning problems requires high-caliber experts. An alternative approach presented by Wang et al. (2024) only relies on access to outcome verifcation - i.e. problems with a ground truth answer. The proposed approach is to amortize the Monte Carlo state-value estimation into a parameterized function. Essentially, this method fts an empirical value function of the reference rollout policy where the targets $y{\bf s}_{t}$ are represented by Equation 11. This idea has been widely adopted in follow-up works (Snell et al., 2024; Anonymous, 2024) and further extended (Setlur et al., 2024c).  

# 5.2. PRM Quality And Its Efect On Search  

The performance and efciency of search at test-time depends on the quality of the PRM (Setlur et al., 2024b; Anonymous, 2024). Setlur et al. (2024b) demonstrate efective scaling (in both training data size and label quality) of a specifc variant of PRMs that estimate values based on the improvement in likelihood of the correct answer after a step. The accuracy of test-time search improves log-linearly with training data size, and the quality of learned value labels improve with more Monte Carlo estimates. Anonymous (2024) show that oracle verifer-enabled search is orders of magnitude more efcient than a learned PRM with noisy value estimates.  

In this section we conduct an experiment demonstrating the scaling characteristics of a PRM. To train our PRM, we frst need to generate diverse solution trajectories where each solution step is annotated with a ground truth value. To do so, we use the method from Wang et al. (2024) to obtain ground truth values, performing 16 Monte Carlo (MC) rollouts for each step of a seed solution. We generate the seed solutions and step-level MC rollouts from a supervised fnetuned (SFT) Llama3.1-8B using the PRM800K (Lightman et al., 2023) dataset. The PRM training data uses 7,086 unique questions - each with seed solutions - and after removing duplicate seed solutions results in 97,000 trajectories in the training data. To evaluate the scaling performance with increasing data, we split the small set of data into three subsets: one with 500 unique questions, one with 3,000 unique questions, and one with all 7,086 unique questions. We create an evaluation set using the MATH-500 dataset (Hendrycks et al., 2021; Lightman et al., 2023) by generating step-by-step solutions from the SFTed model and step-level ground truth values from 128 MC rollouts.  

With this trained PRM, we fnd a reduction in the absolute error of predicted values when comparing PRMs that are trained across datasets of diferent sizes, as well as a selection of intermediate checkpoints in Figure 16. We observe that: 1) the prediction error decreases as the size of the training data increases, and 2) when the size of the dataset is small, improvement converges early during training (around $30\%$ of an epoch for $Q s\!=\!500$ and $\scriptstyle\mathrm{Qs}=3000\mathrm{)}$ ). Although these fndings are based on small-scale experiments, we anticipate continued improvement in prediction errors with larger datasets and more extensive training, suggesting signifcant potential in further refning and scaling PRMs. Additionally, we evaluate the performance of the three fully-trained PRMs as outcome verifers when performing a Best-of-N search during inference time. Figure 17 left shows that the PRM’s ability to verify full solutions improves as they are trained with more data, yet there exists a remarkable gap between the trained PRMs and an oracle PRM. Additionally, we observe that the PRM’s ability to guide the search process towards the correct answer with a more efcient path also improves as the increased accuracy and reduced number of tokens used in the search process are both observed in Figure 17 right. One interesting remaining question is: what is the scaling law for these process supervision models?  

# 5.3. Verifable Versus Open-Ended Problems  

Training a value function with MC rollouts is scalable with infrastructure and inference, but is fundamentally limited to problems with verifable solutions. This excludes proof problems and scientifc derivations which are often more important than the numerical answer itself. While automated proof assistance is an established area of research in mathematics (mathlib Community, 2020), this is rather limiting. First of all, these methods are largely limited to math and do not transfer to other domains such as science or more general problem-solving scenarios. In those domains, training a PRM based on human evaluations of valid reasoning steps could yield a general verifer, which can be used for assuring the validity of the proof/solution chain. This would explain the need for human annotators and verifcation.  

![](images/25849b8301f4031a563455674d47641a82c15c8f777cd96bd4d89617ec6e829b.jpg)  
Figure 16: Distribution of a learned PRM’s predicted values for each state vs. ground truth (computed using $128\,\mathrm{MC}$ samples) as unique training questions increase. Mean absolute error (MAE) decreases as the PRM is trained with more questions. When the quantity of training data is small the performance on the test set converges early $\sqrt[\object Object]{0}$ of an epoch in training).  

# 6. Meta Reinforcement Learning - Learning How To Think  

In this section we will build out an interpretation of the reasoning problem and Meta-CoT from the perspectives of meta-learning and meta-RL. In Section 4.2.1 we motivated the need for in-context search through the paradigm of computational complexity and the generator-verifer gap. In this section, we build an alternative formulation which will help us formalize empirical results of RL training. In particular, we consider the search problem in the deterministic MDP formulation from Section 3.3, however, in this section we assume the reward function $r(\mathbf{S},\mathbf{q})\rightarrow\{0,1\}$ is a deterministic (but a-priori unknown) function of the prompt q, which accepts only a particular set of solutions. At test time under a new prompt, this creates epistemic uncertainty of the reward function i.e. a-priori we do not know the full set of accepted or rejected solutions for this task (prompt question). This process turns the MDP formulation we previously outlined in 3.3 into a Partially Observable MDP (POMDP), a view formalized in classical RL by Ghosh et al. (2021). Their work proves the following general remark:  

![](images/8c989d4b2b9ea1eb55971eb6812c88935953c1ef2e8d66451b9faf96f5219c79.jpg)  
Figure 17: Left: Scaling curves for Best-of-N (BoN) using PRMs trained with diferent number of questions with oracle and majority vote. Right: Beam search $\mathrm{N}{=}5$ , beam width $=4$ ) accuracy and number of tokens used during search with the same PRMs. With more training data, the PRM’s ability to verify at outcome-level and process-level improves.  

![](images/db339debdfc97ab41f05895f5a5093b132eb93d6b04afad4db49e8f58de5adae.jpg)  
Figure 18: Right: The paradigm of the $\mathrm{{RL^{2}}}$ formulation of meta-reinforcement learning. For each new task (prompt), the agent explores its environment over multiple episodes, keeping all the experience in context and maximizes rewards over the entire interaction. Source: Figure 1 in (Duan et al., 2016).  

Remark 5.2 from Ghosh et al. (2021): The expected test-time return of policies that are learned by maximizing reward in any MDP from the posterior, as standard RL methods do, may be arbitrarily low compared to that of the Bayes-optimal behavior.  

In other words, regular RL-trained policies can, in theory, have arbitrarily bad performance on new reasoning domains Setlur et al. (2025). This statement is expanded on and made mathematically precise by Ghosh et al. (2021) and follow-up works, however that formalization is beyond the scope of the current report. In this work, we stipulate that the reasoning problem, as a POMDP, is better suited to a meta-RL framework (Humplik et al., 2019; Rakelly et al., 2019) rather than the regular RL setting, as described above. In the meta-learning setting we are given a distribution of tasks, which in our case would be individual reasoning problems $\mathbf{q}\sim\mathcal{D}_{\mathrm{train}}$ . The meta-learning objective seeks a policy $\pi_{\theta}$ and an adaptation procedure $U$ such that, for a sampled task $\mathbf{q},$ the adapted policy $\pi_{U(\theta)}$ performs well with minimal additional samples. The overall optimization objective is  

$$
\operatorname*{min}_{\theta}\mathbb{E}_{{\mathbf{q}}\sim\mathcal{D}_{\operatorname{train}}}\mathbb{E}_{\pi_{U(\theta)}}\left[L_{{\mathbf{q}}}(\theta)\right],
$$  

where $L_{\mathbf{q}}$ represents the loss associated with task q. For example, in the revision formulation of Section 4.2.2 the objective is  

$$
\operatorname*{min}_{\theta}\mathbb{E}_{\mathbf{q}\sim\mathcal{D}_{\mathrm{train}}}\mathbb{E}_{\mathbf{S}^{i}\sim\pi_{\mathrm{ref}}(\cdot|\mathbf{q})}\left[-\log\pi_{\theta}(\mathbf{S}^{\ast}|\mathbf{S}^{j},\cdot\,.\,.\,,\mathbf{S}^{1},\mathbf{q})\right],
$$  

where $\mathbf{S}^{\ast}$ is the optimal solution to the problem $\mathbf{q}$ and $\mathbf{S}^{i},i\,=\,1,\dots,j$ are solutions provided by some reference policy (usually $\pi_{\theta_{0}}$ ). Here the adaptation procedure is represented by the operator $\pi_{U(\boldsymbol{\theta})}(\cdot|\mathbf{q})\to\pi_{\boldsymbol{\theta}}(\cdot|\mathbf{S}^{j},\cdot\cdot\cdot,\mathbf{S}^{1},\mathbf{q}),$ , which is reminiscent of meta-learning with memory networks Santoro et al. (2016). The issue with this approach is that at inference time, given a new test problem $\mathbf{q},$ we sample solutions auto-regressively from the current iteration of the model $\pi_{\boldsymbol{\theta}}(\cdot|\mathbf{q})$ rather than $\pi_{\theta_{0}}$ which generated our training data, which generates a train-test distribution shift. Indeed, Kumar et al. (2024) noted a continuous shift during training even with $j=1$ where the model successfully corrects wrong solutions from the reference training data, but it’s capability to self-correct (correct wrong solutions sampled from the current policy) diminishes - refer to Figure 19 (sourced from Kumar et al. (2024)) for  

empirical results. If the reference model $\pi_{\mathrm{ref}}$ does not generate data with sufciently high coverage, then this distribution shift fundamentally necessitates the use of on-policy reinforcement learning approaches. Unlike conventional reinforcement learning, where the objective is to optimize for immediate rewards, meta-RL emphasizes training agents to quickly explore a new environment and adapt to the task at hand. This requires optimizing the sampling process during meta-training to ensure the adaptation process $U$ maximizes the agent’s fnal performance.  

If we modify the meta-learning objective in Equation 12 with on-policy sampling, we can essentially recover a formulation of the $\mathrm{RL^{2}}$ (Duan et al., 2016) approach, which has strong synergies with LLMs due to their in-context learning  

![](images/7cb1c0f0e51fd0b799d60a208f4408fcbbadad9eab690648c95e0062846e2ccc.jpg)  
Figure 19: Self-correction performance from selfgenerated vs. fxed frst steps. As training progresses (from left to right) the model becomes more capable of correcting errors in solutions from the reference distribution, but less capable of correcting its own induced errors. Source: Figure 5 from (Kumar et al., 2024).  

capabilities. In this setting the agent, represented as a recurrent policy, encounters a series of tasks $\mathbf{q}$ and interacts with them for several episodes with persistent intra-episode memory for the particular task. Here, the goal of the agent is to maximize the accumulated reward over $K$ episodes:  

$$
\operatorname*{max}_{\boldsymbol{\pi}_{\boldsymbol{\theta}}}\mathbb{E}_{\mathbf{q}\sim\mathcal{D}_{\mathrm{train}}}\mathbb{E}_{\mathbf{S}^{j}\sim\pi_{\boldsymbol{\theta}}(\cdot|\mathbf{S}^{j-1},\ldots,\mathbf{S}^{1},\mathbf{q})}\,\left[\sum_{j=1}^{K}r(\mathbf{S}^{j},\mathbf{q})\right]
$$  

across the distribution of tasks, also notice that here the expectation is taken over the current policy iterate $\pi_{\theta}$ , removing the issue of distribution shift. We will draw some additional connections between Equation 12 and Equation 13 in the next section. This objective be optimized through standard reinforcement learning algorithms such as REINFORCE (Williams, 1992) and PPO (Schulman et al., 2017). While successful in classical meta-RL tasks, this approach can lead to policy collapse on locally greedy behavior - i.e. it may not be able to fully explore new environments but instead collapse to common behaviors. Addressing this issue, Stadie et al. (2019) propose a simple modifcation, E-RL2, which considers the objective  

$$
\operatorname*{max}_{\boldsymbol{\pi}_{\boldsymbol{\theta}}}\mathbb{E}_{\mathbf{q}\sim\mathcal{D}_{\mathrm{train}}}\mathbb{E}_{\mathbf{S}^{j}\sim\pi_{\boldsymbol{\theta}}(\cdot|\mathbf{S}^{j-1},\dots,\mathbf{S}^{1},\mathbf{q})}\left[r(\mathbf{S}^{K},\mathbf{q})\right]
$$  

aiming to maximize the return over the fnal episode only. This allows the policy to explore without reward supervision for $K-1$ episodes, providing wider coverage of the environment which allows the policy to maximize rewards in the fnal episode. This slight modifcation mostly prevents the collapse to greedy behaviors seen in the standard $\mathrm{{RL^{2}}}$ algorithm. Since these early works there has been signifcant follow-up literature and we refer the reader to the survey of Beck et al. (2024) for an overview.  

![](images/6c9177669d0ef88466bdb872144cb9a411626a12e1f505dcd53cfed30fa735cc.jpg)  
Figure 20: The benefts of reinforcement learning for langauge model reasoning. When comparing Expert Iteration $(\mathrm{SoS}\!+\!\mathrm{STaR})$ vs. the RL-based $\mathrm{SoS+APA}_{\mathrm{}}$ , we see that the use of RL leads to improved policy performance (left), with fewer arithmetic errors (center), and improved efciency (right). Source: (left to right) Figures 4a, 6a, and 6b from (Gandhi et al., 2024).  

The above discussion uses the standard $\mathrm{{RL^{2}}}$ episodic formulation as it was studied in classical MDPs, however, this is not strictly necessary in the language setting outlined in Section 4. If we equip the agent with the capability to  

1. terminate an episode early - i.e. achieve the information objective before it reaches the fnal solution 2. reset to an arbitrary state in context rather than restart the solution from the frst step  

then the meta-RL formulation remains valid for any in-context exploration strategy, including general tree search approaches. In purely language domains, such as mathematical reasoning, this is feasible and we can directly apply the E-RL2 algorithm outlined above to models such as those by Yang et al. (2022) , Lehnert et al. (2024), or Gandhi et al. (2024). However, in domains where the policy does not have full control of the environment, such as code or agentic formulations, diferent search structures may be required.  

# 6.1. Meta-RL In Small Domains  

In the prior section we argued for on-policy RL in reasoning systems to handle train-test distribution shifts, but whether pure instruction-tuning (without RL) is enough to induce capabilities in the model is still debated. In general, the use of RL in post-training improves model performance. Indeed, in Section 4 we outlined the capability to discover improved exploration (adaptation $U$ ) algorithms through RL as a major advantage, but the degree to which this occurs remains unclear. Specifcally, we want to answer the question: can an in-context search algorithm, post-trained with reinforcement learning, outperform a modular search system? One such example exists where, as shown in Figure 20, RL post-training improves overall performance in terms of accuracy, reduces the number of logical mistakes, and makes the search more efcient. However, the overall performance is still only comparable to the modular (symbolic) search paradigm - i.e. while RL signifcantly improves performance over the pure SFT (instruction-tuned) model, RL is not currently able to discover new search algorithms. Whether this is a fundamental limitation of the environment, pre-training data, or scaling is currently an open question in the literature.  

![](images/6a39a587fa7886e943f5ea863b19fbaba78aed8e33ad3c92f1a043a18e4269e6.jpg)  
Figure 21: Overview of Reinforcement Learning with Execution Feedback. This training routine directly maps to the E-RL2 framework (Stadie et al., 2019). Source: Figure 2 in (Gehring et al., 2024).  

# 6.2. Meta-RL In Language Model Reasoning  

Two works that have scaled the multi-turn formulation of meta-RL for reasoning tasks are Kumar et al. (2024) and Gehring et al. (2024). While Reinforcement Learning with Execution Feedback (RLEF) (Gehring et al., 2024) does not use the meta-RL or search formulations, they do frame their approach as "learning from feedback", and it fts the Meta-CoT framework. Specifcally, RLEF implements the $\mathrm{E-RL^{2}}$ objective from equation 14 with an additional distributional constraint following standard RLHF practices (Stiennon et al., 2022; Ouyang et al., 2022). First, the model runs several iterations of exploration (episodes, which represent a full code solution), receiving compiler feedback from public test cases until it passes or reaches an exploration limit. Next, the model proposes  

<html><body><table><tr><td>Model</td><td>Training Method</td><td>Valid</td><td>Test</td></tr><tr><td rowspan="3">Llama 3.1 8BInstruct</td><td>一</td><td>8.9</td><td>10.5</td></tr><tr><td>Few-Shot</td><td>8.5</td><td>8.5</td></tr><tr><td>SFT</td><td>10.3 17.2</td><td>10.0 16.0</td></tr><tr><td rowspan="3">Llama 3.1</td><td>RLEF</td><td></td><td></td></tr><tr><td>一 Few-Shot</td><td>25.9 22.5</td><td>27.5 20.3</td></tr><tr><td></td><td></td><td></td></tr><tr><td rowspan="2">70B Instruct</td><td>SFT</td><td>27.7</td><td>27.2</td></tr><tr><td>RLEF</td><td>37.5</td><td>40.1</td></tr></table></body></html>  

Table 2: Comparison of diferent training methods for 8B and 70B Instruct models on validation and test datasets. “–” directly evaluates the instruct model. Source: Table 3 in (Gehring et al., 2024).  

a fnal solution (evaluation episode) and receives a reward based on hidden private test cases, which is used for RL training. See Figure 21 for an overview of the process, along with an example of model outputs. (Gehring et al., 2024) shows a number of interesting empirical fndings. The frst fnding is the relative efect of SFT versus RL training (results shown in Table 2). The clear trend is that SFT training does not induce any in-context exploration capability and the does not lead to improved performance. This fnding is consistent with the results from Kumar et al. (2024), which carry out a similar analysis over multi-turn revisions of math problem solutions.  

![](images/611e5d526b2d832a8cf0d8a5762f16cb5baba521e529f9ff53b140024c141596.jpg)  
Figure 22: Scaling results for Reinforcement Learning with Execution Feedback. Left: Pass $@1$ and pass $@10$ for 8 and 70B models when given either ground truth feedback or random execution feedback. Right: Model solve rates at various turn limits (1, 3, 5, and 10) and sample budgets. Source: Figure 4 in (Gehring et al., 2024).  

Next, as shown in Figure 22, it is clear from the pass $@1$ results that the model is able to explore and internalize environment feedback to refne the code over multiple turns. However, more interestingly, it is able to do so without reliable feedback. In particular the "random feedback" results replaces the compiler output with output from an unrelated problem. This likely creates signifcant issues for the model, since the feedback could throw of the LLM’s grounding (Mirzadeh et al., 2024). However, we see that with RL tuning performance continues to improve with additional test-time revisions, although a gap with ground-truth feedback still exists. This is consistent with the fndings from Kumar et al. (2024), which demonstrate the capability to self-correct without external feedback in the mathematical reasoning domain. Furthermore, these results are also consistent with our formulation of exploration in the epistemic POMDP from Section 6.  

# 6.3. Efciency Or Super-Intelligence?  

As outlined in the beginning of Section 4 two main reasons to internalize a search procedure within a single auto-regressive model are: (1) improved efciency of the search procedure, and (2) the emergence of “super”-intelligence.  

Results in Figure 20 show signifcant improvement from RL post-training in the limited domain of the Countdown game, specifcally, success rates improve while also using smaller search budgets. However, even after RL post-training the SoS model does not out-perform the success rate of the symbolic approach which generated the instruction-tuning data. Similar fndings have also been reported by Lehnert et al. (2024), where the model achieves signifcant improvement in token efciency, but not in success rates based on some simple post-training procedure. Currently, it is unclear whether continued RL training can lead to a stronger model that substantially out-performs the modular search approach. An interesting analysis by Gandhi et al. (2024) (Figure 23) shows the performance of various methods on difcult problems (i.e., problems that no modular symbolic search approach solves). The base SFT-tuned SoS model solves about $1\%$ of these problems, with RL-tuned models reaching up to $4\%$ (STaR in particular).  

![](images/43facbcce2527548b4ad9b61663ea3bba3fdac06642f1889d2287cb0202d21f6.jpg)  
Figure 23: Left: Scaling laws of the R1 model. Red: diferent training checkpoints. Blue: inference time scaling curves from each checkpoint. We see that the model uses inference budget more efciently than parallel sampling with majority vote, but does not outperform large-scale parallel sampling at higher token budgets. Source: (DeepSeek, 2024). Right: The percentage of problems which the SoS model solves on Countdown, but symbolic strategies do not. The biggest gain comes from RL tuning via STaR. Source: Figure 5c in (Gandhi et al., 2024).  

In Section 6 we outlined the Meta-RL formulation, which stipulates that through RL post-training, we’re essentially searching over algorithms, i.e. in-context adaptation procedures $U(\theta)$ , rather than pure policies. In-theory, this process could discover novel reasoning strategies that unlock capabilities beyond the training data or manually designed reasoning approaches. Within the simple domain of Countdown, this does allow us to solve some complex problems that standard search methods do not, but this efect appears weak, even in this limited setting. As outlined earlier, it is unclear whether further RL training will allow us to discover novel reasoning algorithms that allow us to solve new classes of problems. At larger scales, similar results seem to hold in the math (Snell et al., 2024) and code-generation domains (with ground-truth environment feedback) (Gehring et al., 2024). As shown in Figures 10 and 22 the E-RL2 training clearly improves the search performance in terms of pass rate versus search budgets. However, it is unclear whether the RL post-trained model actually solves classes of problems that are unsolved by the base model under increased search budget. Indeed, for code generation, at the 70B scale, with $k=100$ the performance of the RL-tuned model is only about $5\%$ above the base model. At the same time, performance of the RL model appears to be saturating, while the base model demonstrates continued scaling. One potential counterpoint is the scaling law presented by DeepSeek (2024), as shown in Figure 23 (left). In particular, we see that at low token limits the model demonstrates signifcantly better scaling with additional in-context search rather than parallel majority voting. On the other hand, at the higher end of token limits, majority voting can out-perform the model with in-context search, but at a higher inference cost. We should note that this approach uses solution-level majority voting, rather than a separate verifer, which has proven to be a much stronger search strategy (Cobbe et al., 2021; Lightman et al., 2023).  

In conclusion, based on public evidence, the proposed framework of in-context search can signifcantly improve the efciency of the search procedure over base models and "symbolic" approaches or search strategies. However, there is only weak evidence that these models can discover novel reasoning methods that can solve classes of problems that were not solvable previously under some search budget.  

That is, under current empirical evidence the benefts outlined in point (1) above appear clear. However, evidence of emergent "super"-reasoning under point (2) appears weak.  

# 6.4. Can System 2 Reasoning Emerge From Pure RL?  

There is an opinion in the open-research community that the current generation of advanced reasoning models are artifacts of continual reinforcement learning 3. For both theoretical and practical reasons, we believe this is unlikely with the current generation of language models. First, outside of the OpenAI o1, DeepSeek R1, and Qwen QwQ models, which have undisclosed training routines, “standard” models do not exhibit such strong reasoning behaviors, despite the fact that they have already been extensively post-trained with instruction-tuning and reinforcement learning on reasoning tasks. From a more theoretical point-of-view, meta-RL does not arise from standard RL. Indeed, this is the main point of Ghosh et al. (2021) and the remark in Section 6 - models trained with RL on standard CoT formulation can exhibit arbitrarily bad performance on new problems.  

We see this empirically in Figure 22 (right). Note that setting the turn limit to 1 (solid blue line) corresponds to standard RL training (no in-context exploration episodes). In this setting the RL post-trained model performs worse than the base model (dotted blue line) at higher levels of inference compute (Sample Budget) on a held-out test set. Furthermore, models trained with even a turn limit of 3 show signifcantly better inference scaling over the base model, continuing well beyond their training budget. Similar results were obtained by Kumar et al. (2024), showing that a naive application of the $\mathrm{{RL^{2}}}$ objective in Equation 13 leads the model to collapse onto a greedy policy which does not perform in-context exploration (consistent with the fndings of the original work (Stadie et al., 2019)). On the frst step, the two-episode formulation of SCoRe (method proposed by Kumar et al. (2024)) is identical to standard RL, which prevents the model from meaningful in-context exploration. To alleviate this issue they include an additional training stage using the E-RL2 objective, similar to Gehring et al. (2024), before annealing into an $\mathrm{RL^{2}}$ approach. These empirical observations are in line with prior results from meta-RL theory. However, in the pure language setting, such as mathematical reasoning, the model uses an auto-regressive architecture with memory. Moreover, it essentially has full control of the environment and, in theory, can induce meta-behaviors such as backtracking and branching. In practical terms, there is no theoretical reason that the LLM cannot produce a complex sequence of tokens, such as the Meta-CoTs described earlier. We will investigate the presence of such artifacts next.  

# 6.4.1. Inducing Meta-Reasoning In LLMs  

Prior works have shown that complex exploration and reasoning behaviors can be induced in LLMs through in-context demonstrations (Sel et al., 2024; Gandhi et al., 2023; Nie et al., 2024). However, the degree to which such prompting can induce genuine meta-reasoning capabilities, particularly for complex reasoning, remains an open question. In this section, we investigate meta-reasoning induction through carefully constructed prompting strategies. We evaluate model performance on the MATH test set (Hendrycks et al., 2021). Our analysis examines three key dimensions: token generation patterns, expressions of meta-cognitive behaviors (such as explicit error recognition), and mathematical problem-solving accuracy. We evaluate fve prompting strategies with progressively increasing complexity:  

![](images/f0c683af93c1832f8383060364e80d5fd98385c18bf11306bae1a58f496073d8.jpg)  
Figure 24: Per-problem, token-ordered attempts $(\mathtt{n}{=}5)$ ) analysis of solution accuracy and selfcorrection behavior. Lines indicate accuracy trajectories; bars represent frequency of explicit error recognition.  

![](images/8661de755d52e6184def47aa4fae0a70128e3ddcaebcbf11dbfcf61dbbdb3637.jpg)  
Figure 25: Boxplots of completion tokens generated and scatter plots of accuracy across MATH difculty levels, broken down by model and prompting strategy. Higher difculty problems generally elicit longer generations. Box plots represent token distributions while dashed lines track accuracy. Although complex strategies (e.g. Think & Verify) spend the most total tokens, the baseline prompt consistently scales at faster percentage gains from level-to-level while accuracy lines follow similar curves.  

1. The Baseline strategy implements minimal instruction, directing the model to assume mathematical expertise while providing basic formatting requirements.   
2. The CoT strategy extends the Baseline prompt by requesting explicit step-by-step reasoning and chain of thought verbalization.   
3. The Think strategy implements a distinctive approach to mathematical reasoning by requesting stream-of-consciousness meta-cognition within explicit structural constraints. This strategy aims to elicit authentic search by placing an inner monologue inside of a private “thinking” header, using natural language patterns $\mathsf{\binom{\omega}{H m m}}^{\mathfrak{N}}$ , “Let me see...”, “Because of this...”) that request self-verifcation and can demand explicit backtracking on identifed errors. These design choices mimic the non-linear reasoning patterns of mathematicians approaching new problems. The three-shot variant (Think 3-shot) augments this with examples, using a static set of in-context demonstrations to illustrate desired behavior patterns.   
4. The Think & Verify prompt introduces structured iteration bounds and verifcation requirements through dual constraints. The prompt mandates a second-pass verifcation for confdent solutions while maintaining a soft maximum of six distinct solution attempts. Upon reaching a high-confdence solution, the protocol demands one additional verifcation attempt. Successful verifcation permits termination, though the model retains autonomy to continue exploration within the maximum bound should it self-report uncertainty. This adaptive termination criterion functions as an in-context best-of-N approach with self-consistency based early-stopping, optimistically balancing comprehensive solution space exploration with self-verifcation.  

We analyze expressions of regret and self-correction across models. We defne regret as explicit acknowledgments of errors or uncertainty through phrases like "I made a mistake", "oops", "let me reconsider", or similar language. A detailed breakdown of regret statistics across diferent models and prompting strategies is provided in Table 5 in Appendix B.  

The regret analysis reveals a clear distinction between basic prompting approaches and those explicitly designed to encourage meta-cognitive behaviors. Under baseline and standard CoT prompting, which provide no explicit instruction for self-verifcation, models rarely express regret or acknowledge errors $_{<0.5\%}$ of solutions across all models), showing that behaviors like self-verifcation and backtracking rarely emerge natively with standard prompting. In contrast, the Think strategy, which explicitly instructs models that they can "think, refect, revise, backtrack, and verify during responding when it considers doing so would lead to a better solution," produces substantially higher rates of error recognition and correction, particularly in larger models like Llama 3.1 70B $(12.65\%)$ . This efect is further amplifed when combining such instruction with structured examples (Think 3-shot) and explicit verifcation requirements (Think & Verify), leading to signifcantly increased rates of regret expression across most models, peaking at $25.67\%$ for Llama 3.1 70B under the Think & Verify strategy.  

Interestingly, model scale correlates with willingness to express regret, but not monotonically. While larger Llama models show high rates of regret expression ( $15{-}25\%$ under Think & Verify), more advanced models like GPT-4o and Claude 3.5 Sonnet exhibit notably lower rates $(1\!-\!4\%)$ .  

Our main set of results is shown in Figure 25. First, to further understand how model behavior varies across difculty levels and prompting strategies, we analyze the relationship between token usage and accuracy. The box plots show token distributions while overlaid dashed lines track accuracy across difculty levels. Notably, more complex prompting strategies incur higher token overhead but do not consistently translate this additional computation into proportional accuracy gains, particularly in higher difculty regimes. For smaller models like Llama 3.1 8B, we observe signifcantly increased token usage in higher difculty levels that corresponds with a marked decline in accuracy. In contrast, larger models maintain more consistent token usage patterns across difculty levels. The Think & Verify approach consistently produces longer solutions across all models, refecting the inherent overhead of verifcation steps. However, this increased verbosity does not translate to improved accuracy—particularly for smaller models where longer solutions may indicate joint struggles with problem-solving and verifcation. Moreover, we see signifcant increases in verbosity on all problem difculties. Of particular interest, we fnd that the Think & Verify strategy leads to equivalent accuracies on the lowest difculty problems (for all models), while often requiring more than double the token budget. This observation indicates that models seek to match the reasoning "style" rather than substance and may be even faking mistakes to match the desired in-context behaviors (Gudibande et al., 2023).  

In conclusion, while sophisticated prompts successfully elicit reasoning-like behaviors absent from baseline approaches, these behaviors - including recognizing mistakes and backtracking - do not consistently yield performance benefts. In fact, when models engage in self-correction and backtracking, the fnal answers are more likely to be incorrect. This pattern, combined with the observation that token generation increases with MATH difculty level across all strategies while accuracy trajectories remain similar, suggests fundamental limitations in using explicit meta-cognitive instruction to induce robust reasoning capabilities.  

# 7. Putting It All Together - A Pipeline for System 2 Reasoning  

So far, we have presented a theory of advanced reasoning capabilities based around search, as well as some early empirical fndings. In this section we suggest an overall approach to training advanced reasoning models. Our proposal follows the overall structure of modern post-training, consisting of instruction-tuning and RL training (Stiennon et al., 2022; Ouyang et al., 2022).  

# 7.1. Instruction Tuning  

In Section 6.4.1, we presented evidence that the current generation of models cannot induce efective meta-reasoning through in-context prompting, and may even exhibit misleading behaviors. Instead, we propose to begin the process through instruction-tuning with synthetic in-context search data, in the manner outlined in Section 4.3. We showed in Section 6.4.1 that even advanced models rarely demonstrate meta-reasoning capabilities, such as expressing regret or backtracking. Hence, we believe an instruction-tuning stage is critical to endow the model with such backtracking and branching capabilities, which are not frequently present in the pre-training corpus or general purpose instructiontuning data. Following the synthetic data approach in Section 4.3, we construct a training dataset $\mathcal{D}_{\mathrm{train}}=\{\mathbf{q}^{(i)},\mathbf{Z}^{(i)},\mathbf{S}^{(i)}\}_{i=1}^{N}$ which represents the Meta-CoT ${\bf Z}={\bf z}_{1},\ldots,{\bf z}_{K}$ , followed by verifable solution $\mathbf{S}=\mathbf{s},\ldots,\mathbf{s}_{n}$ . For this stage of training, multiple training objectives can be considered, which we outline in Appendix C. To what degree each of these diferent objectives (and combinations thereof) yield qualitatively diferent behaviors is an open empirical question. However, based on previous results at the scale of modern LLMs, we hypothesize that the exact format of the pre-training stage is not crucial, rather the key performance contributor is RL post-training (Ye et al., 2024b; Kumar et al., 2024; Gehring et al., 2024).  

# 7.2. Post-Training With RL  

Prior works have shown strong empirical results with multi-turn RL training (Kumar et al., 2024; Gehring et al., 2024). We propose using a similar objective based on the $\mathrm{E-RL^{2}}$ approach outlined in Equation 14 with the standard additional distributional constraints:  

$$
\operatorname*{max}_{\theta}\mathbb{E}_{\mathbf{S},\mathbf{Z}\sim\pi_{\theta}(\cdot|\mathbf{q}),\mathbf{q}\sim\mathcal{D}_{\mathrm{train}}}\left[r^{*}(\mathbf{S},\mathbf{q})-\beta\sum_{t}\mathbb{D}_{K L}[\pi_{\theta}(\mathbf{z}_{t+1}|\mathbf{Z}_{t},\mathbf{q})||\pi_{\mathrm{ref}}(\mathbf{z}_{t+1}|\mathbf{Z}_{t},\mathbf{q})]\right]
$$  

where $r^{*}$ is the verifable reward from the solution and $\pi_{\mathrm{ref}}$ is a reference policy (usually the instructiontuned model). Similar objectives have been considered in agentic applications where the "Meta-CoT" represents an actual search over a web interface (Nakano et al., 2022; Putta et al., 2024). Note that the reference constraint here is not strictly necessary, but likely required to keep the chain stable and interpretable. Keeping the RL process stable over long horizons and learning robust credit assignment is likely a signifcant challenge. There are a number of possible options for dealing with these challenges:  

1. The step-wise branching structure of MCTS training presents one possible resolution. For example, by using an objective similar to that proposed by Feng et al. (2024), who perform an SFT policy distillation approach using MCTS. Although in general SFT-based policy optimization tends to be inefcient in language model settings (Tajwar et al., 2024).   
2. A number of step-level DPO objectives have been proposed, specifcally for reasoning applications, which can be combined with tree-search based exploration (Xie et al., 2024; Setlur et al., 2024a; Lai et al., 2024). We believe this might be a promising direction, as DPO-based approaches also tend to be more robust to of-policy data (Noukhovitch et al., 2024) than comparable policy-gradient based approaches (Shao et al., 2024), which allows for massive asynchronous RL scaling.   
3. A branching version of on-policy methods could also be used, such as a step-level version of RLOO (Ahmadian et al., 2024) or VinePPO (Havrilla et al., 2024; Kazemnejad et al., 2024), although these methods might not be as robust to of-policy data as prior options and hence less scalable (Noukhovitch et al., 2024).  

The exact correlations between instruction-tuning, pre-training, and RL are unknown in the public literature. Smaller scale studies have found SFT tuning to be efective in endowing transformers with meta-reasoning behaviors (Yang et al., 2022; Lehnert et al., 2024; Gandhi et al., 2024; Ye et al., 2024b), while larger scale studies indicate that purely training with next-token prediction on synthetic traces might not be enough due to train-test distribution shifts and fnd that on-policy RL is the key component for improving performance (Zhang et al., 2024b; Kumar et al., 2024; Gehring et al., 2024).  

# 7.2.1. $\mathbf{Q}^{*}$ or $\pmb q$ -STaR?  

In the prior section, we introduced the RL training objective in Equation 15 using a verifer reward $r^{*}(\mathbf{S},\mathbf{q})$ . Here we will derive another RL formulation with a self-training type of RL objective without the need for an external verifer.  

Under the (Meta-)CoT formulation of Section 2, we can consider the Meta-CoT process $\mathbf{Z}$ as a general latent variable model for the whole problem solution S. We can then apply a widely known  

![](images/2331e2cd16d5a1e75049427dd173b2020e1c0ff7d38f1d8e61a3d2c4e136ddf3.jpg)  
Figure 26: Left: The training loss of ftting only the oracle answer or a sequence of answers with the revision objective in Equation 5. The model that uses in-context exploration is better able to model the correct solution than directly outputting it after the question, consistent with our complexity considerations in Section 2. Source: Figure 5 in Qu et al. (2024). Right: An example in which the DeepSeek R1 model DeepSeek (2024) reaches the correct fnal solution (448) in its Meta-CoT ${\bf Z}\,\sim\,\pi_{\theta}({\bf\cdot}|{\bf q})$ , but the summary model ${\bf S}\sim\pi_{\theta}(\cdot|{\bf Z},{\bf q})$ fails and produces the wrong fnal solution (1792).  

formulation of the STaR algorithm as latent inference Singh et al. (2024):  

$$
\begin{array}{r}{\log\pi_{\mathrm{data}}(\mathbf{S}|\mathbf{q})=\log\int\pi(\mathbf{S}|\mathbf{Z},\mathbf{q})\pi(\mathbf{Z}|\mathbf{q})d\mathbf{Z}\geq}\\ {\displaystyle\operatorname*{max}_{q(\mathbf{Z}|\mathbf{q})}\mathbb{E}_{q(\mathbf{Z}|\mathbf{q})}[\log\pi(\mathbf{S}|\mathbf{Z},\mathbf{q})]+\mathbb{D}_{K L}[q(\mathbf{Z}|\mathbf{q})||\pi(\mathbf{Z}|\mathbf{q})]}\end{array}
$$  

where $q(\mathbf{Z}|\mathbf{q})$ is a variational inference function Kingma $\&$ Welling (2013), or in our case a latent reasoner. Since all components here are auto-regressive transformers (LLMs), we can amortize them inside a single model. Now, if we set $\pi(\mathbf{Z}|\mathbf{q})$ to be the prior model $\pi_{\mathrm{ref:}}$ , which is initialized as the instruction-tuned base model $(\pi_{\theta_{0}})$ from Section 7.1 and amortize the reasoning inference model $q(\mathbf{Z}|\mathbf{q})$ and the decoder model $\pi(\mathbf{S}|\mathbf{Z},\mathbf{q})$ into a single LLM $\pi_{\theta}$ we get the objective:  

$$
\operatorname*{max}_{\theta}\mathbb{E}_{\mathbf{Z}\sim\pi_{\theta}(\cdot|\mathbf{q}),\mathbf{S},\mathbf{q}\sim\mathcal{D}_{\mathrm{train}}}\left[\log\pi_{\theta}(\mathbf{S}|\mathbf{Z},\mathbf{q})-\beta\mathbb{D}_{K L}[\pi_{\theta}(\mathbf{Z},|\mathbf{q})||\pi_{\mathrm{ref}}(\mathbf{Z}|\mathbf{q})]\right]
$$  

where we used the $\beta{-}\mathrm{V}\!\mathsf{A}\mathsf{E}$ formulation Higgins et al. (2017). Unlike standard VAEs though, the parametric models here are auto-regressive transformers, which sample discrete tokens, hence we cannot use the reparameterization trick to compute gradients of the above distribution and must result to RL optimization. This objective indeed looks similar to the main RL objective from the previous section, as formulated in Equation 15, but with a few key diferences. First, the reward function is represented as  

$$
r(\mathbf{S},\mathbf{q})=\log\pi_{\boldsymbol{\theta}}(\mathbf{S}|\mathbf{Z},\mathbf{q})
$$  

where the solution S is no longer sampled on-policy from the model, but from a pre-existing questionsolution training dataset. Hence the sampling expectation from the policy is only over the latent Meta-CoT. Notice also that the reward itself is a function of the model parameters $\theta$ , which then requires a modifcation to the standard policy gradient approach. With some simple diferentiationby-parts calculus, the above objective can be represented as:  

$$
\begin{array}{r l}&{\displaystyle\operatorname*{max}_{\theta}\mathbb{E}_{\mathbf{Z}\sim\pi_{\theta}(\cdot|\mathbf{q}),\mathbf{S},\mathbf{q}\sim\mathcal{D}_{\mathrm{train}}}\left[s\mathbf{g}(\log\pi_{\theta}(\mathbf{S}|\mathbf{Z},\mathbf{q}))-\beta\mathbb{D}_{K L}[\pi_{\theta}(\mathbf{Z},|\mathbf{q})||\pi_{\mathrm{ref}}(\mathbf{Z}|\mathbf{q})]\right]+}\\ &{\displaystyle\operatorname*{max}_{\theta}\mathbb{E}_{\mathbf{Z}\sim s\mathbf{g}(\pi_{\theta}(\cdot|\mathbf{q})),\mathbf{S},\mathbf{q}\sim\mathcal{D}_{\mathrm{train}}}[\log\pi_{\theta}(\mathbf{S}|\mathbf{Z},\mathbf{q})]}\end{array}
$$  

Here the frst equation is now a standard RL problem with reward as defned in Equation 19 which can be optimized with the standard methods, outlined in Section 7.2. The second part is a maximum likelihood training of the Meta-CoT "summarization" model on the ground-truth solution. Essentially the goal of this training objective is to make the conditional model $\pi_{\boldsymbol{\theta}}(\mathbf{S}|\mathbf{Z},\mathbf{q})$ more expressive than the standard training objective $\pi_{\boldsymbol{\theta}}(\mathbf{S}|\mathbf{q})$ . Some empirical evidence from Qu et al. (2024) supports this hypothesis as shown in Figure 26 (left) which shows training curves for the revision objective in Equation 5. Indeed, even with standard SFT using in-context exploration the model achieves signifcantly lower loss than directly predicting the correct answer. This is similar in spirit to the goal of Quiet-STaR Zelikman et al. (2024), with the diference being that we optimize and marginalize over the entire latent meta-reasoning process and fnal solution with arbitrary complexity. In comparison Quiet-STaR applies the above objective at a token-level with fxed token budget for the latent process, thus essentially making the transformer computation graph deeper, similarly to Universal Transformer Dehghani et al. (2019), rather than optimizing meaningful semantic reasoning. This line of reasoning is also consistent with the complexity arguments outlined in Section 2.  

Finally, we should note that while this approach does not utilize verifers for RL training, it still needs to bootstrap the latent reasoning processes $\mathbf{Z}$ , which still requires verifable outcomes to generate high-quality synthetic training data.  

The objective in Equation 20 has one major advantage that we no longer require verifcation of the fnal solution, since the training objective only requires a dataset of question-solution pairs. This allows us to train on open-ended and hard to verify problems, such as proofs and general reasoning. The downside is now that the model $\pi_{\boldsymbol{\theta}}(\mathbf{S}|\mathbf{Z},\mathbf{q})$ is only essentially trained with supervised fne-tuning and does not get on-policy reward feedback, which may be sub-optimal. If the Meta-CoT process can fnd a solution with high certainty, then perhaps an SFT training objective for the summarization model is sufcient, which is an empirical question. Anecdotally, we have observed instances of the DeepSeek R1 model DeepSeek (2024) fnds the correct answer in its Meta-CoT but actually outputs the wrong fnal solution as shown in Figure 26 (right).  

# 7.2.2. Discount Rates  

While standard RLHF pipelines have not used discount rates in the past, they may be required in reasoning applications. Recent works have discovered that small biases in preferences for longer answers are routinely exploited by reward models in RL pipelines, which has yielded signifcantly more verbose models (Singhal et al., 2023; Park et al., 2024). These issues are present even in strong systems, such as GPT-4 (OpenAI, 2023) and require explicit regularization. As demonstrated in prior chapters, when increased sampling correlates with higher accuracy then, without explicit regularization, the model can choose to continue generating/searching for solutions or collapse on some majority voting approach with potentially unlimited inference targets. Indeed, recent work Chen et al. (2024) found that advanced reasoning models can generate signifcantly longer sequences even for simple problems ("What is $2+3="$ ) as shown in Figure 27. The Qwen QwQ model (Team, 2024) generates up to 13 solutions in context before providing a fnal answer. To mitigate this issue, we might require a modifcation of the RL objective from Equation 15, to use a discounted objective  

$$
\operatorname*{max}_{\theta}\mathbb{E}_{\mathbf{S},\mathbf{Z}\sim\pi_{\theta}(\cdot|\mathbf{q}),\mathbf{q}\sim\mathcal{D}_{\mathrm{train}}}\left[\gamma^{|\mathbf{Z}|}{_{T}}^{*}(\mathbf{S},\mathbf{q})-\beta\sum_{t}\mathbb{D}_{K L}[\pi_{\theta}(\mathbf{z}_{t+1}|\mathbf{Z}_{t},\mathbf{q})||\pi_{\mathrm{ref}}(\mathbf{z}_{t+1}|\mathbf{Z}_{t},\mathbf{q})]\right]
$$  

![](images/5af633bb612d339f1c058775c7ce3ee564fe37249a46439936248ba2d880ae3f.jpg)  
Figure 27: Left: Figure 17 with discounted objective. The undiscounted version of Best-Of-N sees continued improvement with additional sampling, while the discounted objective exhibits a humpshaped frontier. Right: Token lengths for answers of "what is $2\!+\!3\!=\!?"$ by diferent models. Source: Figure 1 in Chen et al. (2024).  

fwhere $\gamma<1$ . This objective would (in theory) force the model to trade-of compute versus certainty in its response. This efect is demonstrated in Figure 27 (left), which shows a discounted version of the best-of-N objective. Without a particular penalty on the generation budget, performance increases steadily with additional compute, which may push the model to continuously increase the inference budget (as compared to base models) as shown on the right. If we consider the discounted reward (accuracy) instead, we see a hump-shaped objective as there is a stronger trade-of between verifer confdence and the generation budget.  

This also raises an interesting trade-of on controlling model behaviors in terms of inference/accuracy. We might be interested in forcing a particular trade-of depending on problem difculty. Consider then a distribution over discount rates $\gamma_{1}<...<\gamma_{m}$ . We can also associated an individual language prompts $\mathbf{p}_{\gamma}$ with each discount rate such as "Work as quickly as possible" for high discount, to "Take as much time as needed" for a low discount rate. Then, at train time we can optimize the objective  

$$
:\mathbb{E}_{\mathbf{S},\mathbf{Z}\sim\pi_{\theta}(\cdot|\mathbf{q},\mathbf{p}_{\gamma_{i}}),\mathbf{q}\sim\mathcal{D}_{\mathrm{train}},i\sim1;m}\left[\gamma_{i}^{|\mathbf{Z}|}r^{*}(\mathbf{S},\mathbf{q})-\beta\sum_{t}\mathbb{D}_{K L}\big[\pi_{\theta}(\mathbf{z}_{t+1}|\mathbf{Z}_{t},\mathbf{q},\mathbf{p}_{\gamma_{i}})||\pi_{\mathrm{ref}}(\mathbf{z}_{t+1}|\mathbf{Z}_{t},\mathbf{q})\big]\right]
$$  

Then, at inference time we can control the qualitative model behavior through conditioning on the prompt to trade-of inference compute versus accuracy. Schultz et al. (2024) showed that a if we train a model on MCTS search traces with diferent parameters, at inference time we can condition the model on a particular parameter confguration and recreate the qualitative behaviors(as shown in Figure 9). As discussed earlier, simple supervised fne-tuning may not be sufcient to induce advanced reasoning capabilities (and likely even more difcult to do so in a controllable way), however, these results demonstrate promise in inducing controllable behaviors into the model through the conditional prompting objective above.  

# 8. Going Forward  

Public open-research/source progress on reasoning models is currently bottle-necked by three main issues:  

1. Access to resources in terms of both data and compute as these algorithms require signifcant amount of both.   
2. Open-source infrastructure for large scale inference and training is currently lackluster.   
3. Algorithmic exploration - we have a lot of avenues to explore and only limited people and resources actively working on the right directions.  

We expand on these issues in this section.  

# 8.1. The "Big MATH" Project  

While compute bottlenecks are a persistent issue in open research, we found the lack of open datasets with verifable reasoning problems to be an even bigger challenge. Prior works have used the GSM8k (Cobbe et al., 2021) and MATH (Hendrycks et al., 2021) datasets, but the frst has largely been saturated by the current generation of models and the second is quite limited, with only 12,000 problems. To overcome these challenges, we have put signifcant work into the "Big MATH" project - an efort to aggregate over 1,000,000 high-quality, high-confdence, and diverse verifable math problems. We combine existing datasets with signifcant post-processing, as well as eforts to acquire additional data from novel sources. We outline this below.  

Our proposed training pipeline requires a large-scale corpus of challenging prompts with verifable answers, but developing such datasets presents fundamental constraints in automated verifcation and assessment of reasoning capabilities. In many domains, solutions are too unstructured or nuanced for automated verifcation: a single objective answer may not exist, correct solutions can appear in equally valid but textually distinct forms, and certain tasks (e.g. complex proofs) resist reduction to deterministically verifable outputs with a single canonical representation. These issues make automated grading infeasible. Even in contexts where problem-answer pairs can be constructed and easily verifed, these formats may reduce to factual retrieval rather than exercises that require reasoning capabilities.  

As evidenced by the wide array of literature discussed in previous sections, mathematics ofers a stable ground truth for correctness, naturally programmatic answer grading, and existing public datasets in problem-solution-answer format. While we recognize signifcant potential in many domains (e.g., programming, diagnostics, fnance, analysis) where structured reasoning paths can be validated, the substantial resources required to construct and verify such datasets, particularly without access to proprietary data, makes them impractical for our immediate objectives. While the availability of existing datasets ultimately drove our domain selection, even these openly available resources are constrained in both scale and distribution. We face the critical challenge of distinguishing between computational and pattern profciency and conceptual understanding when assessing model capabilities - a distinction that fundamentally shapes our dataset requirements and evaluation metrics.  

To guide our data construction, we defne three core criteria:  

1. the existence of uniquely verifable solutions, meaning that problems must admit a single correct answer that can be reliably verifed;   
2. open-ended problem formulations, ensuring that tasks cannot be easily solved by guessing (as might occur in multiple-choice formats) and instead require nontrivial reasoning steps; and   
3. closed-form solutions, such that the fnal answer must be expressible in a closed form (e.g., a scalar or formula, not a proof), thereby enabling automated evaluation.  

These criteria reveal signifcant limitations in existing datasets. Those that meet our structural requirements remain severely limited in scale, containing orders of magnitude fewer examples than we estimate necessary for model training. Within the structured collections, we observe a concerning scarcity of non-trivial problems that efectively challenge reasoning capabilities, with many examples simply testing computational abilities or following predictable patterns. Perhaps most problematic is the non-negligible proportion of incorrect problem-solution labels, even in widely-used datasets, introducing substantial complexity to data cleaning.  

Preliminary analysis of one of the largest available datasets, NuminaMath (LI et al., 2024), exemplifes these issues. Basic verifcation shows that out of roughly 860,000 entries, more than 42,500 are duplicates ( $.{\sim}5\%$ of the dataset). Further, while $89.7\%$ of entries contain exactly one clearly boxed solution suitable for automated verifcation, $2.6\%$ contain no boxed solution, and $7.7\%$ include multiple boxed solutions. Problems like this underscore the state of open and publicly available data and suggest deeper quality concerns.  

# 8.1.1. Data Sourcing  

We consider multiple established mathematical problem datasets that are commonly used in the literature (Table 3). First, we include the Human Annotated Reasoning Problems (HARP) dataset (Yue et al., 2024), containing nearly 4,800 competition-level short answer problems with programmaticallycheckable answers. Next, we use the NuminaMath (LI et al., 2024) dataset, which is composed of roughly 860,000 problems from a variety of benchmarks and sources: Chinese high school math exercises, math olympiad-style competition problems, the art of problem solving forum, MATH (Hendrycks et al., 2021), and GSM8k (Cobbe et al., 2021). NuminaMath further incorporates synthetic data from the synthetically generated dataset Orca-Math (Mitra et al., 2024) as well as further generating a signifcant amount of synthetic data based on the MATH dataset and a subset of the math competition problems. Additionally, while NuminaMath uses the original split of the MATH dataset (7,000 training problems, 5,500 test problems), we choose to use version with 12,000 training problems and 500 test problems, as originally proposed by (Lightman et al., 2023). We incorporate the Omni-MATH dataset (Gao et al., 2024), contributing almost 4,500 olympiad-level problems curated from 39 diferent competition websites. Notably, this dataset employs professional annotators and verifers to maintain solution-answer quality. Finally, we include OpenMathInstruct-2 (Toshniwal et al., 2024), composed of about 607,000 synthetically generated problems. Toshniwal et al. (2024) use Llama3.1- 405B (Dubey et al., 2024) in a multi-step data augmentation pipeline which they seed using the GSM8k (Cobbe et al., 2021) and MATH (Hendrycks et al., 2021) datasets.  

Beyond drawing from these established datasets, we maintain an active data collection initiative that extends our corpus through manual curation of competition-level problems. This efort has already contributed thousands of additional olympiad-style problems and continues to grow. Our manual collection process focuses on acquiring mathematically rigorous content from competition archives, olympiad materials, and historical sources not present in existing datasets.  

# 8.1.2. Data Filtering  

To create datasets with which we can train a model in our training pipeline, we next clean and flter the data from each source using a combination of bespoke and common strategies (Albalak et al., 2024). Specifcally for our two-stage pipeline, we create two versions of the dataset. First, we have the base flter, where the data will be used during SFT for the model to learn (1) an approximation of the distribution of math problems and (2) to follow the desired chain-of-thought format. Next, the strict flter is intended to be used for the Meta-RL training stage. Thus, the strictly fltered data should contain only problems that closely follow our three desired properties: open-ended, verifable, closed-form problem-solution pairs.  

Table 3: Comparison of Problems by Data Source and Filter Type   


<html><body><table><tr><td>DataSource</td><td>Original</td><td>Base Filter</td><td>StrictFilter</td></tr><tr><td>HARP Yue et al.,2 2024</td><td>4,780</td><td>3,691</td><td>2,996</td></tr><tr><td>NuminaMath (LI et al., 2 2024</td><td>859,608</td><td>452,820</td><td>231,887</td></tr><tr><td>Omni-MATH (Gao et al.,2 2024</td><td>4,428</td><td>3,660</td><td>2,478</td></tr><tr><td>OpenMathInstruct-2 (Toshniwal 1 et al.,2024)</td><td>607,324</td><td>600,191</td><td>496,331</td></tr><tr><td>Total</td><td>1,476,140</td><td>1,060,362</td><td>733,692</td></tr></table></body></html>  

The Base Filter performs some operations that are unique to each subset, and some operations that are shared across the full collection of data. First, in the HARP dataset (Yue et al., 2024), we fnd many problems that contain fgures in the Asymptote4 vector graphics language, which we flter out of the dataset. Next, we flter the NuminaMath dataset (LI et al., 2024). This dataset contains a signifcant amount of synthetic data, which is often difcult to verify for accuracy and correctness. However, the Orca-Math (Mitra et al., 2024) subset has empirically been proven to improve performance on supervised fne-tuning, and for this reason, we elect to maintain only the Orca-Math data, while discarding the other synthetic subsets. Next, NuminaMath does not explicitly contain answers to each problem, so we extract answers to problems by searching for boxed solutions (“\boxed{}” in LaTeX). Any problems whose solution does not contain exactly 1 boxed answer is fltered out. When exploring Omni-MATH (Gao et al., 2024), we found many problems containing author attributions (e.g. a person’s name in parenthesis) and removed the attributions from the problem. OpenMathInstruct-2 originally contains multiple solutions and fnal answers to each problem (Toshniwal et al., 2024). The frst step we take is to group all matching problems together, removing those which have mismatched expected answers. Next, we found that the data still contains evidence of the synthetic data generation process, such as asking “do you want to solve it?” or “here’s the new problem” and remove any problems with similar phrases.  

Finally, after running each of the described flters over the individual subsets, we perform four fltering operations across the full collection. First, we use exact matching to fnd and remove duplicate problems. Then, we use a FastText language identifer (Joulin et al., 2016b,a; Grave et al., 2018) and remove any problems where English is not the primary language. Next, we remove problems containing a hyperlink, as this suggests that a model may not have the full resources required to solve the problem. Lastly, we decontaminate the dataset by removing any examples of the MATH500 test set (Lightman et al., 2023).  

The Strict Filter further reduces the base fltered data to problem-answer pairs which are more likely to be open-ended, verifable, and closed-form. The strict flters are applied equally to all subsets of the dataset through a mix of rule-based and model-based fltering. First, we remove questions containing multiple parts as these can be challenging to evaluate. For a similar reason, we also remove questions that ask for a proof as these problems are difcult to evaluate. Next, we choose to remove multiple choice problems as models have a high probability of selecting the correct answer without producing an accurate reasoning chain. For the same reason, we also remove Yes/No and True/False problems, as these may give a poor learning signal during Meta-RL training. Finally, we use the SemDeDup algorithm (Abbas et al., 2023) with the model at sentence-transformers/all-MiniLM-L6-v2 and remove problems with a cosine similarity over 0.5.  

# 8.2. Infrastructure  

For the proposed family of algorithms we need RL training infrastructure that can:  

1. Scale to multiple nodes in a straightforward way   
2. Allow for high-performance inference throughput   
3. Allow for interleaving inference and training efciently for online RL algorithms  

Here we outline our recent progress on these issues in the open-source GPT-NeoX framework (Andonian et al., 2023). We leverage CUDA IPC handles to enable true asynchronous RLHF training by sharing GPU memory directly between training and inference processes. By allocating model weights  

in the training framework and sharing CUDA memory pointers with the inference framework, both processes maintain access to the same physical memory throughout training. When the training process updates weights, these updates are immediately visible to the inference process through its mapped pointers without requiring any explicit synchronization. This shared memory architecture enables fully parallel execution - the inference process can continuously generate tokens while the training process updates weights, with neither process blocking the other. Although early portions of generated rollouts may be of-policy, the fnal steps will generally be only one training step of-policy at most, as the inference process immediately sees weight updates through the shared memory. Unlike previous asynchronous approaches (Noukhovitch et al., 2024) that require distinct generation and training phases,  

![](images/7c0f2024a2b467b8e152492363c942f5349eceb4058d13a782f95d7db70069b3.jpg)  
Figure 28: Async training versus slightly of policy methods that require dedicated weight synchronization  

our system maintains constant GPU utilization across both processes. Direct memory sharing provides signifcantly higher throughput by eliminating synchronization overhead between training and inference. Our initial experiments show $40\%$ improved throughput compared to 3-step asynchronous training that requires explicit weight synchronization, as shown in Figure 28. However, this approach comes with important trade-ofs in memory utilization and parallelism. Sharing GPUs between processes restricts us to suboptimal tensor parallelism (TP) confgurations compared to dedicated training and inference setups, resulting in lower theoretical peak throughput for both processes. The impact is particularly pronounced for inference, where reduced memory for KV cache and suboptimal TP confgurations signifcantly constrain the generation speed. In scenarios where inference compute signifcantly exceeds training compute (e.g. MCTS, where most of the search tree is discarded after each rollout), using separate dedicated GPU clusters with optimized TP confgurations for each process can achieve higher overall throughput despite the synchronization overhead.  

# 8.3. Open Research Questions  

A number of open research questions remain which are currently not answered in the literature.  

# 8.3.1. Open-Ended Verifcation And CoT Faithfulness  

In this report we outline a pipeline for training advanced reasoning models on verifable questions. These types of questions largely consist of things like math, scientifc questions with symbolic or numerical answers, or code problems with unit tests. However, we believe it is still an open question on how to ensure that the resulting CoTs are faithful and in fact provide valid reasoning. In many applications, such as science for example, the reasoning chain can be even more important than the fnal answer. Qualitatively, current models with “advanced” reasoning still struggle and produce many CoTs which are inconsistent or unfaithful, even when they obtain the correct fnal answer. Interestingly, even the single math example provided along with the announcement of the O1 model has an incomplete solution as it makes an unsubstantiated assumption on the form of the polynomial $h(x)=x^{2}-c$ without proving that the coefcient of $x$ is in fact zero (OpenAI, 2024). Under a competitive setting this would likely receive 5/7 points, even though the fnal answer is correct. It is worth noting that the model actually verifes and proves this assumption in it’s Meta-CoT but the fnal output provided does not include those steps. We believe this might be one type of artifact that arises from training with fnal answer verifcation only. How to provide rewards on full CoTs and open-ended problems such as proofs remains an open question. Under the assumption of a wide enough generation-verifcation gap, we believe an RLAIF approach could be promising. In particular, foregoing concepts such as formal verifcation, one avenue to explore is a “generative verifer” similar to Zhang et al. (2024a) or Mahan et al. (2024). In this setting the evaluation model can be provided with one or several reference solutions/proofs, or other relevant material such as textbook chapters, tools like Wolfram or Mathematica, and specifcally fne-tuned to provide evaluations to proposed solutions, rather than fnal answers alone. As outlined in Section 3.2, we found advanced models to already posses some capabilities in that vein. Unfortunately, a major bottleneck in that line of research is the need for a dataset of open-ended reasoning problems (such as proofs), which would likely require signifcant human annotations.  

![](images/f15c355785d35e274b8b11fcc79be943a9782d7872a0c82ff9a865cadcb12fad.jpg)  
Figure 29: Under a fxed generator model and a variable verifer performance seems to also obey a form of a scaling law. Right: Accuracy seems to improve in a log-linear fashion with amount of training data under a standard discriminative PRM. Source: Figure 6 in (Setlur et al., 2024c). Left: Performance of a Generative Reward Model (with CoT) on reasoning tasks with a a reference using variable amount of CoTs and majority voting. Increasing the amount of inference-time sampling improves accuracy. Source: Figure 5 in (Mahan et al., 2024).  

# 8.3.2. Process Guidance And The Verifer Gap  

The current work brings to light some remaining questions on the approaches to process guidance and PRMS. In Section 5 we outlined results showing that pre-trained PRMs still signifcantly lack behind pure Monte Carlo approaches in terms of search efciency. Moreover, with few exceptions (Setlur et al., 2024c), we believe algorithmic approaches for training these models remain understudied. In particular, if we view a PRM as a value function, then many approaches from ofine RL (Levine et al., 2020) become applicable with some already showing promise in agentic domains (Zhou et al., 2024b). In the same vein, Silver et al. (2016) introduced a separate value function over the standard MCTS approach using roll-outs for efciency purposes. However, we believe there may be a fundamental verifer gap beyond just the efciency question. As shown in Figure 3 there remains a signifcant gap between the "best-of-N" verifer driven approach and the oracle pass $\textcircled{a}\Nu$ performance. A similar gap seems to exist in the multi-turn code generation domain, as shown in Figure 22 (Gehring et al., 2024). Moreover, under a fxed generator, results from Setlur et al. (2024c) and our own empirical fndings in Section 5 indicate that verifer performance may also be driven by a scaling law as well. In Figure 29 (left) we see clear log-linear scaling in performance in terms of training data. One hypothesis is that verifcation is also a matter of computational complexity, which is why allowing verifers to use CoT improves performance as shown in Figure 3 (Zhang et al., 2024a; Mahan et al., 2024). In addition, Mahan et al. (2024) show further inference-time scaling for verifers, as using additional CoTs with majority vote improves reward accuracy on reasoning problems well beyond the performance of the standard discriminative verifer. This raises questions for the fundamental learnability of in-context search traces. In essence, if a search trace was generated using ground-truth verifcation or Monte-Carlo roll-outs, the resulting sequence may have signifcant implicit complexity, which would be challenging for the model to learn. This would also motivate incorporating further self-evaluation or refection into reasoning chains. The associated verifcation scaling laws and design choices remain largely unexplored in the literature and present an important research direction.  

# 8.3.3. Scaling Laws For Reasoning And Search  

While recent model releases have demonstrated strong inference scaling performance, we still lack a thorough scientifc exploration of scaling laws in public research, which is a key piece of the recently emerging model paradigm.  

1. The scaling laws of search presented by Jones (2021) have not been publicly evaluated on realistically challenging reasoning applications with LLMs. While the results from Feng et al. (2024) show promise, they are still limited and partial. Obtaining a more thorough evaluation on joint policy and verifer scaling will provide clarity to current research questions. So far, unfortunately, the research community has been limited by a lack of data and scalable infrastructure.   
2. Exploration of search approaches remains a very under-studied topic. As discussed in Section 4.4, it appears that current reasoning models all implement diferent approaches to search, which leads to qualitatively diferent behaviors based on the search strategies presented in Section 4.3. Given a strong trained verifer, what are the performance efects of diferent search strategies such as BFS/DFS-V, $\mathrm{A^{*}}$ , MCTS, etc.? Furthermore, the relationship between exploration strategies at training and test time is an important direction of future study.   
3. The trade-ofs between instruction-tuning and RL remain unclear. As discussed in Section 6, it appears that standard instruction tuning can endow models with meta-reasoning capabilities in small/simple domains, but these results do not scale, and realistic applications require signifcant on-policy RL. Is this a fundamental issue with distribution shift, or is performance driven by another relationship, similar to the trade-ofs discovered by Setlur et al. (2024a)?   
4. The fundamental missing piece of the current inference time scaling law discussion is the performance of the proposed in-context search strategy compared with an explicit search-based method. In theory, the post-training approach outlined in Section 7 can discover novel reasoning approaches (algorithms) that solve fundamentally new classes of problems unsolvable under any search budget by a standard search approach. While there is clear evidence of the efciency of in-context search (fewer tokens per interaction), it is unclear whether the current generation of models have any emergent capabilities yet. In simplest terms, do strong reasoning models shift the compute-accuracy curve to the left or up?. As outlined in Section 6.4, current evidence in the open literature for emergent capabilities remains weak.  

# 8.3.4. Meta-Search/Search 2  

In this report we argued that advanced reasoning should incorporate in-context search, which can yield higher efciency and potentially more advanced capabilities in reasoning tasks. However, such an approach also has potential downsides. In particular, we are limited by the model’s context length, which induces fundamental limits on the search com  

plexity. Moreover, the sequential nature of in-context search can make the process slow, limiting the power of these models. At the same time neither of these are issues with classical search approaches such as MCTS, which can be parallelized Liu et al. (2020). A natural question is: can we can build an additional search procedure on top of an advanced reasoning model (a process which we call Meta-Search or Search2 following the naming convention of Duan et al. (2016))? In recent literature, Anonymous (2024) trained a value function (PRM) with a “multi-turn” approach, conditioning the value function on all prior explored solutions. Rather than the traditional value function, they instead train a function $v_{\theta}(\mathbf{Z_{t}},\mathbf{S}_{t},\mathbf{q})$ , where the MetaCoT $\mathbf{Z}_{t}$ consists of prior solution attempts. The empirical performance of their method compared with regular PRM training is shown in Figure 30, demonstrating improved scaling over traditional value function training. This result suggests that we may be able to increase the search efciency using a meta-critic, however this remains an open empirical question. If this is indeed the case, it would allow us to also massively scale online search with reasoning models.  

![](images/e24751f6f00bfa82d148768c6f13a5519952aa57d460034ef8176cd5c9adfbd9.jpg)  
Figure 30: Scaling performance of search with "meta"-value function and regular (single turn) value function. Conditioning the model on prior explored paths improves search effciency. Source: Figure 5 in Anonymous (2024).  

# 8.3.5. Reasoning with External Tools  

Building on our discussion of scaling laws and search approaches, we investigate a critical direction: augmenting model reasoning with external computational tools. Just as humans leverage calculators, spreadsheets, and specialized software to solve problems more efciently, LLMs can potentially achieve better scaling properties by ofoading the compute burden to external tools - requiring less training data during SFT and fewer samples during test-time search to reach the high level of performance. For instance, while pure CoT reasoning requires models to perform all calculations internally to solve math problems, these computations can be ofoaded to a Python interpreter. Prior works demonstrate that such tool-integrated reasoning (TIR) improves performance on mathematical questions (LI et al., 2024; Yin et al., 2024; Chen et al., 2022). However, the scaling properties of  

![](images/704f605065a9e215cdcd8a0a45d2bd4a70eca89a18085d882b1e2938d813ead6.jpg)  
Figure 31: Scaling curves for a Tool Integrated Reasoning (TIR) model, trained on 100K problems, and a Chain-of-Thought (CoT) model trained on 400K problems on the First AIMO Prize, using an oracle verifer.  

TIR remain unexplored.  

Figure 31, using models trained by LI et al. (2024), provides initial evidence for the efciency of TIR models. Despite being fne-tuned on $25\%$ as much data, the TIR model demonstrates superior scaling properties compared to the CoT model. This occurs across all sample sizes, with both majority vote and Best-of-N strategies consistently outperforming their CoT counterparts. The efciency gain is particularly evident in the low-sample regime ( $2^{0}$ to $2^{3}$ samples), where TIR achieves nearly double the accuracy of CoT methods. This suggests that ofoading computations to external tools signifcantly improves the model’s ability to solve problems even with limited attempts, and TIR is more efcient at both training and inference time.  

Based on these promising initial results, we identify several critical directions for future investigation:  

1. Scaling laws and search strategies: We need to systematically study the scaling properties of diferent search methods (e.g., BFS, DFS, $A^{*}~^{5}$ , MCTS) in Tool-Integrated Reasoning models.   
2. Verifcation scaling: The role and scaling properties of the verifer in TIR settings remains under-explored - what is the relation between training data for the verifer, verifer accuracy, and policy accuracy?   
3. Internal reasoning vs reasoning with external tools: While our initial results show TIR’s efciency advantages, we need to better understand the fundamental trade-ofs between these approaches. When does ofoading computation to external tools provide the most beneft compared to pure language reasoning? How do these trade-ofs change with model scale and problem complexity?  

# 9. Conclusion  

In this position paper, we have introduced Meta Chain-of-Thought (Meta-CoT) as a framework for understanding and enhancing the reasoning capabilities of Large Language Models (LLMs). We have argued that traditional Chain-of-Thought does not fully represent the underlying data generative process on reasoning problems. By incorporating the concepts of search, verifcation, and iterative refnement, Meta-CoT provides a more complete model of the cognitive processes required for advanced problem-solving.  

We believe that Meta-CoT represents a promising path towards more robust and generalizable reasoning in LLMs. The observed behaviors of state-of-the-art models, along with our experiments on in-context exploration and backtracking, lend support to the hypothesis that internal search processes are crucial for performance on complex tasks. Furthermore, the proposed training pipeline presents a concrete approach for developing LLMs with enhanced Meta-CoT capabilities. Future work should validate the efcacy of our proposed pipeline.  

Beyond our proposed method, numerous open questions and challenges remain. Further research is needed to determine the optimal scaling laws for reasoning and search, to develop more efective process supervision and verifcation techniques, and to clarify the interplay between instruction tuning and reinforcement learning for meta chain of thought reasoning. The "Big MATH" dataset we have introduced aims to support this research by providing a large-scale resource for training reasoning models. Furthermore, the possibility of meta-RL paired with Meta-CoT automatically discovering novel search algorithms is a particularly intriguing open research question for future work to explore.  

# 10. Acknowledgments  

We would like to thank Aviral Kumar, Benjamin Eysenbach, Nathan Lambert, Rishabh Agarwal, Sasha Rush and Noah Goodman for the fruitful discussions and feedback on this report.  

# References  

Amro Abbas, Kushal Tirumala, Dániel Simig, Surya Ganguli, and Ari S. Morcos. Semdedup: Dataefcient learning at web-scale through semantic deduplication, 2023. URL https://arxiv.org/ abs/2303.09540.  

Arash Ahmadian, Chris Cremer, Matthias Gallé, Marzieh Fadaee, Julia Kreutzer, Ahmet Üstün, and Sara Hooker. Back to basics: Revisiting reinforce style optimization for learning from human feedback in llms. arXiv preprint arXiv:2402.14740, 2024.  

Alon Albalak, Yanai Elazar, Sang Michael Xie, Shayne Longpre, Nathan Lambert, Xinyi Wang, Niklas Muennighof, Bairu Hou, Liangming Pan, Haewon Jeong, Colin Rafel, Shiyu Chang, Tatsunori Hashimoto, and William Yang Wang. A survey on data selection for language models, 2024. URL https://arxiv.org/abs/2402.16827.  

Alex Andonian, Quentin Anthony, Stella Biderman, Sid Black, Preetham Gali, Leo Gao, Eric Hallahan, Josh Levy-Kramer, Connor Leahy, Lucas Nestler, Kip Parker, Michael Pieler, Jason Phang, Shivanshu Purohit, Hailey Schoelkopf, Dashiell Stander, Tri Songz, Curt Tigges, Benjamin Thérien, Phil Wang, and Samuel Weinbach. GPT-NeoX: Large Scale Autoregressive Language Modeling in PyTorch, 2023. URL https://www.github.com/eleutherai/gpt-neox.  

Anonymous. Improving the efciency of test-time search in LLMs with backtracking. In Submitted to The Thirteenth International Conference on Learning Representations, 2024. URL https:// openreview.net/forum?id=hJ2BCYGvFg. under review.  

Jacob Beck, Risto Vuorio, Evan Zheran Liu, Zheng Xiong, Luisa Zintgraf, Chelsea Finn, and Shimon Whiteson. A survey of meta-reinforcement learning, 2024. URL https://arxiv.org/abs/2301. 08028.  

Bradley Brown, Jordan Juravsky, Ryan Ehrlich, Ronald Clark, Quoc V Le, Christopher Ré, and Azalia Mirhoseini. Large language monkeys: Scaling inference compute with repeated sampling. arXiv preprint arXiv:2407.21787, 2024.  

Wenhu Chen, Xueguang Ma, Xinyi Wang, and William W Cohen. Program of thoughts prompting: Disentangling computation from reasoning for numerical reasoning tasks. arXiv preprint arXiv:2211.12588, 2022.  

Xingyu Chen, Jiahao Xu, Tian Liang, Zhiwei He, Jianhui Pang, Dian Yu, Linfeng Song, Qiuzhi Liu, Mengfei Zhou, Zhuosheng Zhang, Rui Wang, Zhaopeng Tu, Haitao Mi, and Dong Yu. Do not think that much for $2\!+\!3\!=\!?$ on the overthinking of o1-like llms, 2024. URL https://arxiv.org/abs/ 2412.21187.  

Karl Cobbe, Vineet Kosaraju, Mohammad Bavarian, Mark Chen, Heewoo Jun, Lukasz Kaiser, Matthias Plappert, Jerry Tworek, Jacob Hilton, Reiichiro Nakano, Christopher Hesse, and John Schulman. Training verifers to solve math word problems, 2021. URL https://arxiv.org/abs/2110. 14168.  

DeepSeek. Deepseek-r1-lite-preview is now live: unleashing supercharged reasoning power!, 11 2024. URL https://x.com/deepseek_ai/status/1859200141355536422. Posted on X (formerly Twitter).  

Mostafa Dehghani, Stephan Gouws, Oriol Vinyals, Jakob Uszkoreit, and Łukasz Kaiser. Universal transformers, 2019. URL https://arxiv.org/abs/1807.03819.  

Yan Duan, John Schulman, Xi Chen, Peter L. Bartlett, Ilya Sutskever, and Pieter Abbeel. $\mathrm{Rl^{2}}$ : Fast reinforcement learning via slow reinforcement learning, 2016. URL https://arxiv.org/abs/ 1611.02779.  

Abhimanyu Dubey, Abhinav Jauhri, Abhinav Pandey, Abhishek Kadian, Ahmad Al-Dahle, Aiesha Letman, Akhil Mathur, Alan Schelten, Amy Yang, Angela Fan, Anirudh Goyal, Anthony Hartshorn, Aobo Yang, Archi Mitra, Archie Sravankumar, Artem Korenev, Arthur Hinsvark, Arun Rao, Aston Zhang, Aurelien Rodriguez, Austen Gregerson, Ava Spataru, Baptiste Roziere, Bethany Biron, Binh Tang, Bobbie Chern, Charlotte Caucheteux, Chaya Nayak, Chloe Bi, Chris Marra, Chris McConnell, Christian Keller, Christophe Touret, Chunyang Wu, Corinne Wong, Cristian Canton Ferrer, Cyrus Nikolaidis, Damien Allonsius, Daniel Song, Danielle Pintz, Danny Livshits, David Esiobu, Dhruv Choudhary, Dhruv Mahajan, Diego Garcia-Olano, Diego Perino, Dieuwke Hupkes, Egor Lakomkin, Ehab AlBadawy, Elina Lobanova, Emily Dinan, Eric Michael Smith, Filip Radenovic, Frank Zhang, Gabriel Synnaeve, Gabrielle Lee, Georgia Lewis Anderson, Graeme Nail, Gregoire Mialon, Guan Pang, Guillem Cucurell, Hailey Nguyen, Hannah Korevaar, Hu Xu, Hugo Touvron, Iliyan Zarov, Imanol Arrieta Ibarra, Isabel Kloumann, Ishan Misra, Ivan Evtimov, Jade Copet, Jaewon Lee, Jan Gefert, Jana Vranes, Jason Park, Jay Mahadeokar, Jeet Shah, Jelmer van der Linde, Jennifer Billock, Jenny Hong, Jenya Lee, Jeremy Fu, Jianfeng Chi, Jianyu Huang, Jiawen Liu, Jie Wang, Jiecao Yu, Joanna Bitton, Joe Spisak, Jongsoo Park, Joseph Rocca, Joshua Johnstun, Joshua Saxe, Junteng Jia, Kalyan Vasuden Alwala, Kartikeya Upasani, Kate Plawiak, Ke Li, Kenneth Heafeld, Kevin Stone, Khalid El-Arini, Krithika Iyer, Kshitiz Malik, Kuenley Chiu, Kunal Bhalla, Lauren Rantala-Yeary, Laurens van der Maaten, Lawrence Chen, Liang Tan, Liz Jenkins, Louis Martin, Lovish Madaan, Lubo Malo, Lukas Blecher, Lukas Landzaat, Luke de Oliveira, Madeline Muzzi, Mahesh Pasupuleti, Mannat Singh, Manohar Paluri, Marcin Kardas, Mathew Oldham, Mathieu Rita, Maya Pavlova, Melanie Kambadur, Mike Lewis, Min Si, Mitesh Kumar Singh, Mona Hassan, Naman Goyal, Narjes Torabi, Nikolay Bashlykov, Nikolay Bogoychev, Niladri Chatterji, Olivier Duchenne, Onur Çelebi, Patrick Alrassy, Pengchuan Zhang, Pengwei Li, Petar Vasic, Peter Weng, Prajjwal Bhargava, Pratik Dubal, Praveen Krishnan, Punit Singh Koura, Puxin Xu, Qing He, Qingxiao Dong, Ragavan Srinivasan, Raj Ganapathy, Ramon Calderer, Ricardo Silveira Cabral, Robert Stojnic, Roberta Raileanu, Rohit Girdhar, Rohit Patel, Romain Sauvestre, Ronnie Polidoro, Roshan Sumbaly, Ross Taylor, Ruan Silva, Rui Hou, Rui Wang, Saghar Hosseini, Sahana Chennabasappa, Sanjay Singh, Sean Bell, Seohyun Sonia Kim, Sergey Edunov, Shaoliang Nie, Sharan Narang, Sharath Raparthy, Sheng Shen, Shengye Wan, Shruti Bhosale, Shun Zhang, Simon Vandenhende, Soumya Batra, Spencer Whitman, Sten Sootla, Stephane Collot, Suchin Gururangan, Sydney Borodinsky, Tamar Herman, Tara Fowler, Tarek Sheasha, Thomas Georgiou, Thomas Scialom, Tobias Speckbacher, Todor Mihaylov, Tong Xiao, Ujjwal Karn, Vedanuj Goswami, Vibhor Gupta, Vignesh Ramanathan, Viktor Kerkez, Vincent Gonguet, Virginie Do, Vish Vogeti, Vladan Petrovic, Weiwei Chu, Wenhan Xiong, Wenyin Fu, Whitney Meers, Xavier Martinet, Xiaodong Wang, Xiaoqing Ellen Tan, Xinfeng Xie, Xuchao Jia, Xuewei Wang, Yaelle Goldschlag, Yashesh Gaur, Yasmine Babaei, Yi Wen, Yiwen Song, Yuchen Zhang, Yue Li, Yuning Mao, Zacharie Delpierre Coudert, Zheng Yan, Zhengxing Chen, Zoe Papakipos, Aaditya Singh, Aaron Grattafori, Abha Jain, Adam Kelsey, Adam Shajnfeld, Adithya Gangidi, Adolfo Victoria, Ahuva Goldstand, Ajay Menon, Ajay Sharma, Alex Boesenberg, Alex Vaughan, Alexei Baevski, Allie Feinstein, Amanda Kallet, Amit Sangani, Anam Yunus, Andrei Lupu, Andres Alvarado, Andrew Caples, Andrew Gu, Andrew Ho, Andrew Poulton, Andrew Ryan, Ankit Ramchandani, Annie Franco, Aparajita Saraf, Arkabandhu Chowdhury, Ashley Gabriel, Ashwin Bharambe, Assaf Eisenman, Azadeh Yazdan, Beau James, Ben Maurer, Benjamin Leonhardi, Bernie Huang, Beth Loyd, Beto De Paola, Bhargavi Paranjape, Bing Liu, Bo Wu, Boyu Ni, Braden Hancock, Bram Wasti, Brandon Spence, Brani Stojkovic, Brian Gamido, Britt Montalvo, Carl Parker, Carly Burton, Catalina Mejia, Changhan Wang, Changkyu Kim, Chao Zhou, Chester Hu, Ching-Hsiang Chu, Chris Cai, Chris Tindal, Christoph Feichtenhofer, Damon Civin, Dana Beaty, Daniel Kreymer,  

Daniel Li, Danny Wyatt, David Adkins, David Xu, Davide Testuggine, Delia David, Devi Parikh, Diana Liskovich, Didem Foss, Dingkang Wang, Duc Le, Dustin Holland, Edward Dowling, Eissa Jamil, Elaine Montgomery, Eleonora Presani, Emily Hahn, Emily Wood, Erik Brinkman, Esteban Arcaute, Evan Dunbar, Evan Smothers, Fei Sun, Felix Kreuk, Feng Tian, Firat Ozgenel, Francesco Caggioni, Francisco Guzmán, Frank Kanayet, Frank Seide, Gabriela Medina Florez, Gabriella Schwarz, Gada Badeer, Georgia Swee, Gil Halpern, Govind Thattai, Grant Herman, Grigory Sizov, Guangyi, Zhang, Guna Lakshminarayanan, Hamid Shojanazeri, Han Zou, Hannah Wang, Hanwen Zha, Haroun Habeeb, Harrison Rudolph, Helen Suk, Henry Aspegren, Hunter Goldman, Ibrahim Damlaj, Igor Molybog, Igor Tufanov, Irina-Elena Veliche, Itai Gat, Jake Weissman, James Geboski, James Kohli, Japhet Asher, Jean-Baptiste Gaya, Jef Marcus, Jef Tang, Jennifer Chan, Jenny Zhen, Jeremy Reizenstein, Jeremy Teboul, Jessica Zhong, Jian Jin, Jingyi Yang, Joe Cummings, Jon Carvill, Jon Shepard, Jonathan McPhie, Jonathan Torres, Josh Ginsburg, Junjie Wang, Kai Wu, Kam Hou U, Karan Saxena, Karthik Prasad, Kartikay Khandelwal, Katayoun Zand, Kathy Matosich, Kaushik Veeraraghavan, Kelly Michelena, Keqian Li, Kun Huang, Kunal Chawla, Kushal Lakhotia, Kyle Huang, Lailin Chen, Lakshya Garg, Lavender A, Leandro Silva, Lee Bell, Lei Zhang, Liangpeng Guo, Licheng Yu, Liron Moshkovich, Luca Wehrstedt, Madian Khabsa, Manav Avalani, Manish Bhatt, Maria Tsimpoukelli, Martynas Mankus, Matan Hasson, Matthew Lennie, Matthias Reso, Maxim Groshev, Maxim Naumov, Maya Lathi, Meghan Keneally, Michael L. Seltzer, Michal Valko, Michelle Restrepo, Mihir Patel, Mik Vyatskov, Mikayel Samvelyan, Mike Clark, Mike Macey, Mike Wang, Miquel Jubert Hermoso, Mo Metanat, Mohammad Rastegari, Munish Bansal, Nandhini Santhanam, Natascha Parks, Natasha White, Navyata Bawa, Nayan Singhal, Nick Egebo, Nicolas Usunier, Nikolay Pavlovich Laptev, Ning Dong, Ning Zhang, Norman Cheng, Oleg Chernoguz, Olivia Hart, Omkar Salpekar, Ozlem Kalinli, Parkin Kent, Parth Parekh, Paul Saab, Pavan Balaji, Pedro Rittner, Philip Bontrager, Pierre Roux, Piotr Dollar, Polina Zvyagina, Prashant Ratanchandani, Pritish Yuvraj, Qian Liang, Rachad Alao, Rachel Rodriguez, Raf Ayub, Raghotham Murthy, Raghu Nayani, Rahul Mitra, Raymond Li, Rebekkah Hogan, Robin Battey, Rocky Wang, Rohan Maheswari, Russ Howes, Ruty Rinott, Sai Jayesh Bondu, Samyak Datta, Sara Chugh, Sara Hunt, Sargun Dhillon, Sasha Sidorov, Satadru Pan, Saurabh Verma, Seiji Yamamoto, Sharadh Ramaswamy, Shaun Lindsay, Shaun Lindsay, Sheng Feng, Shenghao Lin, Shengxin Cindy Zha, Shiva Shankar, Shuqiang Zhang, Shuqiang Zhang, Sinong Wang, Sneha Agarwal, Soji Sajuyigbe, Soumith Chintala, Stephanie Max, Stephen Chen, Steve Kehoe, Steve Satterfeld, Sudarshan Govindaprasad, Sumit Gupta, Sungmin Cho, Sunny Virk, Suraj Subramanian, Sy Choudhury, Sydney Goldman, Tal Remez, Tamar Glaser, Tamara Best, Thilo Kohler, Thomas Robinson, Tianhe Li, Tianjun Zhang, Tim Matthews, Timothy Chou, Tzook Shaked, Varun Vontimitta, Victoria Ajayi, Victoria Montanez, Vijai Mohan, Vinay Satish Kumar, Vishal Mangla, Vítor Albiero, Vlad Ionescu, Vlad Poenaru, Vlad Tiberiu Mihailescu, Vladimir Ivanov, Wei Li, Wenchen Wang, Wenwen Jiang, Wes Bouaziz, Will Constable, Xiaocheng Tang, Xiaofang Wang, Xiaojian Wu, Xiaolan Wang, Xide Xia, Xilun Wu, Xinbo Gao, Yanjun Chen, Ye Hu, Ye Jia, Ye Qi, Yenda Li, Yilin Zhang, Ying Zhang, Yossi Adi, Youngjin Nam, Yu, Wang, Yuchen Hao, Yundi Qian, Yuzi He, Zach Rait, Zachary DeVito, Zef Rosnbrick, Zhaoduo Wen, Zhenyu Yang, and Zhiwei Zhao. The llama 3 herd of models, 2024. URL https://arxiv.org/abs/2407.21783.  

Yann Dubois, Xuechen Li, Rohan Taori, Tianyi Zhang, Ishaan Gulrajani, Jimmy Ba, Carlos Guestrin, Percy Liang, and Tatsunori B. Hashimoto. Alpacafarm: A simulation framework for methods that learn from human feedback, 2024.  

Subhabrata Dutta, Joykirat Singh, Soumen Chakrabarti, and Tanmoy Chakraborty. How to think step-by-step: A mechanistic understanding of chain-of-thought reasoning. ArXiv, abs/2402.18312, 2024. URL https://api.semanticscholar.org/CorpusID:268041831.  

Jefrey L Elman. Finding structure in time. Cognitive science, 14(2):179–211, 1990.  

Xidong Feng, Ziyu Wan, Muning Wen, Stephen Marcus McAleer, Ying Wen, Weinan Zhang, and Jun Wang. Alphazero-like tree-search can guide large language model decoding and training, 2024.   
Kanishk Gandhi, Dorsa Sadigh, and Noah D. Goodman. Strategic reasoning with language models, 2023. URL https://arxiv.org/abs/2305.19165.   
Kanishk Gandhi, Denise Lee, Gabriel Grand, Muxin Liu, Winson Cheng, Archit Sharma, and Noah D Goodman. Stream of search (sos): Learning to search in language. arXiv preprint arXiv:2404.03683, 2024.   
Bofei Gao, Feifan Song, Zhe Yang, Zefan Cai, Yibo Miao, Qingxiu Dong, Lei Li, Chenghao Ma, Liang Chen, Runxin Xu, Zhengyang Tang, Benyou Wang, Daoguang Zan, Shanghaoran Quan, Ge Zhang, Lei Sha, Yichang Zhang, Xuancheng Ren, Tianyu Liu, and Baobao Chang. Omnimath: A universal olympiad level mathematic benchmark for large language models, 2024. URL https://arxiv.org/abs/2410.07985.   
Jonas Gehring, Kunhao Zheng, Jade Copet, Vegard Mella, Taco Cohen, and Gabriel Synnaeve. Rlef: Grounding code llms in execution feedback with reinforcement learning, 2024. URL https: //arxiv.org/abs/2410.02089.   
Dibya Ghosh, Jad Rahme, Aviral Kumar, Amy Zhang, Ryan P. Adams, and Sergey Levine. Why generalization in rl is difcult: Epistemic pomdps and implicit partial observability, 2021. URL https://arxiv.org/abs/2107.06277.   
Edouard Grave, Piotr Bojanowski, Prakhar Gupta, Armand Joulin, and Tomas Mikolov. Learning word vectors for 157 languages. In Proceedings of the International Conference on Language Resources and Evaluation (LREC 2018), 2018.   
Arnav Gudibande, Eric Wallace, Charlie Snell, Xinyang Geng, Hao Liu, Pieter Abbeel, Sergey Levine, and Dawn Song. The false promise of imitating proprietary llms, 2023. URL https://arxiv. org/abs/2305.15717.   
Shibo Hao, Yi Gu, Haodi Ma, Joshua Jiahua Hong, Zhen Wang, Daisy Zhe Wang, and Zhiting Hu. Reasoning with language model is planning with world model, 2023. URL https://arxiv.org/ abs/2305.14992.   
Alex Havrilla, Yuqing Du, Sharath Chandra Raparthy, Christoforos Nalmpantis, Jane Dwivedi-Yu, Maksym Zhuravinskyi, Eric Hambro, Sainbayar Sukhbaatar, and Roberta Raileanu. Teaching large language models to reason with reinforcement learning, 2024. URL https://arxiv.org/abs/ 2403.04642.   
Dan Hendrycks, Collin Burns, Saurav Kadavath, Akul Arora, Steven Basart, Eric Tang, Dawn Song, and Jacob Steinhardt. Measuring mathematical problem solving with the math dataset, 2021. URL https://arxiv.org/abs/2103.03874.   
Irina Higgins, Loic Matthey, Arka Pal, Christopher Burgess, Xavier Glorot, Matthew Botvinick, Shakir Mohamed, and Alexander Lerchner. B-vae: Learning basic visual concepts with a constrained variational framework. In International Conference on Learning Representations (ICLR), 2017. URL https://openreview.net/forum?id $\equiv$ Sy2fzU9gl.   
Jan Humplik, Alexandre Galashov, Leonard Hasenclever, Pedro A. Ortega, Yee Whye Teh, and Nicolas Heess. Meta reinforcement learning as task inference, 2019. URL https://arxiv.org/abs/ 1905.06424.   
Andy L. Jones. Scaling scaling laws with board games, 2021. URL https://arxiv.org/abs/2104. 03113.   
Michael I Jordan. Serial order: A parallel distributed processing approach. In Advances in psychology, volume 121, pp. 471–495. Elsevier, 1997.   
Armand Joulin, Edouard Grave, Piotr Bojanowski, Matthijs Douze, H’erve J’egou, and Tomas Mikolov. Fasttext.zip: Compressing text classifcation models. arXiv preprint arXiv:1612.03651, 2016a.   
Armand Joulin, Edouard Grave, Piotr Bojanowski, and Tomas Mikolov. Bag of tricks for efcient text classifcation. arXiv preprint arXiv:1607.01759, 2016b.   
Amirhossein Kazemnejad, Milad Aghajohari, Eva Portelance, Alessandro Sordoni, Siva Reddy, Aaron Courville, and Nicolas Le Roux. Vineppo: Unlocking rl potential for llm reasoning through refned credit assignment, 2024. URL https://arxiv.org/abs/2410.01679.   
Diederik P Kingma and Max Welling. Auto-encoding variational bayes. arXiv preprint arXiv:1312.6114, 2013. URL https://arxiv.org/abs/1312.6114.   
Levente Kocsis and Csaba Szepesvári. Bandit based Monte-Carlo planning. In Johannes Fürnkranz, Tobias Schefer, and Myra Spiliopoulou (eds.), Machine Learning: ECML 2006, volume 4212 of Lecture Notes in Computer Science, Berlin, Heidelberg, 2006. Springer. doi: 10.1007/11871842_29.   
Jing Yu Koh, Stephen McAleer, Daniel Fried, and Ruslan Salakhutdinov. Tree search for language model agents. arXiv preprint arXiv:2407.01476, 2024.   
Aviral Kumar, Vincent Zhuang, Rishabh Agarwal, Yi Su, John D Co-Reyes, Avi Singh, Kate Baumli, Shariq Iqbal, Colton Bishop, Rebecca Roelofs, Lei M Zhang, Kay McKinney, Disha Shrivastava, Cosmin Paduraru, George Tucker, Doina Precup, Feryal Behbahani, and Aleksandra Faust. Training language models to self-correct via reinforcement learning, 2024. URL https://arxiv.org/ abs/2409.12917.   
Xin Lai, Zhuotao Tian, Yukang Chen, Senqiao Yang, Xiangru Peng, and Jiaya Jia. Step-dpo: Step-wise preference optimization for long-chain reasoning of llms, 2024. URL https://arxiv.org/abs/ 2406.18629.   
Lucas Lehnert, Sainbayar Sukhbaatar, DiJia Su, Qinqing Zheng, Paul Mcvay, Michael Rabbat, and Yuandong Tian. Beyond a\*: Better planning with transformers via search dynamics bootstrapping, 2024. URL https://arxiv.org/abs/2402.14083.   
Sergey Levine, Aviral Kumar, George Tucker, and Justin Fu. Ofine reinforcement learning: Tutorial, review, and perspectives on open problems, 2020.   
Jia LI, Edward Beeching, Lewis Tunstall, Ben Lipkin, Roman Soletskyi, Shengyi Costa Huang, Kashif Rasul, Longhui Yu, Albert Jiang, Ziju Shen, Zihan Qin, Bin Dong, Li Zhou, Yann Fleureau, Guillaume Lample, and Stanislas Polu. Numinamath. [https: //huggingface.co/AI-MO/NuminaMath-CoT](https://github.com/project-numina/ aimo-progress-prize/blob/main/report/numina_dataset.pdf), 2024.   
Zhiyuan Li, Hong Liu, Denny Zhou, and Tengyu Ma. Chain of thought empowers transformers to solve inherently serial problems, 2024. URL https://arxiv.org/abs/2402.12875.   
Hunter Lightman, Vineet Kosaraju, Yura Burda, Harri Edwards, Bowen Baker, Teddy Lee, Jan Leike, John Schulman, Ilya Sutskever, and Karl Cobbe. Let’s verify step by step. arXiv preprint arXiv:2305.20050, 2023.   
Anji Liu, Yitao Liang, Ji Liu, Guy Van den Broeck, and Jianshu Chen. On efective parallelization of monte carlo tree search, 2020. URL https://arxiv.org/abs/2006.08785.   
Aman Madaan, Niket Tandon, Prakhar Gupta, Skyler Hallinan, Luyu Gao, Sarah Wiegrefe, Uri Alon, Nouha Dziri, Shrimai Prabhumoye, Yiming Yang, Shashank Gupta, Bodhisattwa Prasad Majumder, Katherine Hermann, Sean Welleck, Amir Yazdanbakhsh, and Peter Clark. Self-refne: Iterative refnement with self-feedback, 2023. URL https://arxiv.org/abs/2303.17651.   
Dakota Mahan, Duy Van Phung, Rafael Rafailov, Chase Blagden, Nathan Lile, Louis Castricato, JanPhilipp Fränken, Chelsea Finn, and Alon Albalak. Generative reward models. arXiv preprint arXiv:2410.12832, 2024.   
The mathlib Community. The lean mathematical library. In Proceedings of the 9th ACM SIGPLAN International Conference on Certifed Programs and Proofs, POPL ’20. ACM, January 2020. doi: 10.1145/3372885.3373824. URL http://dx.doi.org/10.1145/3372885.3373824.   
William Merrill and Ashish Sabharwal. The expresssive power of transformers with chain of thought. arXiv preprint arXiv:2310.07923, 2023.   
Iman Mirzadeh, Keivan Alizadeh, Hooman Shahrokhi, Oncel Tuzel, Samy Bengio, and Mehrdad Farajtabar. Gsm-symbolic: Understanding the limitations of mathematical reasoning in large language models, 2024. URL https://arxiv.org/abs/2410.05229.   
Arindam Mitra, Hamed Khanpour, Corby Rosset, and Ahmed Awadallah. Orca-math: Unlocking the potential of slms in grade school math, 2024. URL https://arxiv.org/abs/2402.14830.   
Reiichiro Nakano, Jacob Hilton, Suchir Balaji, Jef Wu, Long Ouyang, Christina Kim, Christopher Hesse, Shantanu Jain, Vineet Kosaraju, William Saunders, Xu Jiang, Karl Cobbe, Tyna Eloundou, Gretchen Krueger, Kevin Button, Matthew Knight, Benjamin Chess, and John Schulman. Webgpt: Browser-assisted question-answering with human feedback, 2022.   
Allen Nie, Yi Su, Bo Chang, Jonathan N. Lee, Ed H. Chi, Quoc V. Le, and Minmin Chen. Evolve: Evaluating and optimizing llms for exploration, 2024. URL https://arxiv.org/abs/2410. 06238.   
Michael Noukhovitch, Shengyi Huang, Sophie Xhonneux, Arian Hosseini, Rishabh Agarwal, and Aaron Courville. Asynchronous rlhf: Faster and more efcient of-policy rl for language models, 2024. URL https://arxiv.org/abs/2410.18252.   
Franz Nowak, Anej Svete, Alexandra Butoi, and Ryan Cotterell. On the representational capacity of neural language models with chain-of-thought reasoning. arXiv preprint arXiv:2406.14197, 2024.   
OpenAI. Gpt-4 technical report. arXiv preprint, 2023. https://arxiv.org/abs/2303.08774.   
OpenAI. Learning to reason with llms. https://openai.com/index/ learning-to-reason-with-llms/, 2024. Accessed: 2024-12-20.   
Long Ouyang, Jefrey Wu, Xu Jiang, Diogo Almeida, Carroll Wainwright, Pamela Mishkin, Chong Zhang, Sandhini Agarwal, Katarina Slama, Alex Ray, John Schulman, Jacob Hilton, Fraser Kelton, Luke Miller, Maddie Simens, Amanda Askell, Peter Welinder, Paul F Christiano, Jan Leike, and Ryan Lowe. Training language models to follow instructions with human feedback. In S. Koyejo, S. Mohamed, A. Agarwal, D. Belgrave, K. Cho, and A. Oh (eds.), Advances in Neural Information Processing Systems, volume 35, pp. 27730–27744. Curran Associates, Inc., 2022. URL https://proceedings.neurips.cc/paper_files/paper/2022/ file/b1efde53be364a73914f58805a001731-Paper-Conference.pdf.  

Ryan Park, Rafael Rafailov, Stefano Ermon, and Chelsea Finn. Disentangling length from quality in direct preference optimization, 2024.  

Xue Bin Peng, Aviral Kumar, Grace Zhang, and Sergey Levine. Advantage-weighted regression: Simple and scalable of-policy reinforcement learning. arXiv preprint arXiv:1910.00177, 2019.  

Du Phan, Matthew D. Hofman, David Dohan, Sholto Douglas, Tuan Anh Le, Aaron Parisi, Pavel Sountsov, Charles Sutton, Sharad Vikram, and Rif A. Saurous. Training chain-of-thought via latent-variable inference, 2023. URL https://arxiv.org/abs/2312.02179.  

Ben Prystawski, Michael Li, and Noah Goodman. Why think step by step? reasoning emerges from the locality of experience. Advances in Neural Information Processing Systems, 36, 2024.  

Pranav Putta, Edmund Mills, Naman Garg, Sumeet Motwani, Chelsea Finn, Divyansh Garg, and Rafael Rafailov. Agent q: Advanced reasoning and learning for autonomous ai agents. arXiv preprint arXiv:2408.07199, 2024.  

Yuxiao Qu, Tianjun Zhang, Naman Garg, and Aviral Kumar. Recursive introspection: Teaching language model agents how to self-improve, 2024. URL https://arxiv.org/abs/2407.18219.  

Santosh Kumar Radha, Yasamin Nouri Jelyani, Ara Ghukasyan, and Oktay Goktas. Iteration of thought: Leveraging inner dialogue for autonomous large language model reasoning, 2024. URL https://arxiv.org/abs/2409.12618.  

Kate Rakelly, Aurick Zhou, Deirdre Quillen, Chelsea Finn, and Sergey Levine. Efcient of-policy meta-reinforcement learning via probabilistic context variables, 2019. URL https://arxiv.org/ abs/1903.08254.  

Machel Reid, Nikolay Savinov, Denis Teplyashin, Dmitry Lepikhin, Timothy Lillicrap, Jean-baptiste Alayrac, Radu Soricut, Angeliki Lazaridou, Orhan Firat, Julian Schrittwieser, et al. Gemini 1.5: Unlocking multimodal understanding across millions of tokens of context. arXiv preprint arXiv:2403.05530, 2024.  

Christopher D. Rosin. Multi-armed bandits with episode context. Annals of Mathematics and Artifcial Intelligence, 61(3):203–230, 2011. doi: 10.1007/s10472-011-9258-6.  

Adam Santoro, Sergey Bartunov, Matthew Botvinick, Daan Wierstra, and Timothy Lillicrap. One-shot learning with memory-augmented neural networks, 2016. URL https://arxiv.org/abs/1605. 06065.  

John Schulman, Filip Wolski, Prafulla Dhariwal, Alec Radford, and Oleg Klimov. Proximal policy optimization algorithms, 2017.  

John Schultz, Jakub Adamek, Matej Jusup, Marc Lanctot, Michael Kaisers, Sarah Perrin, Daniel Hennes, Jeremy Shar, Cannada Lewis, Anian Ruoss, Tom Zahavy, Petar Veličković, Laurel Prince, Satinder Singh, Eric Malmi, and Nenad Tomašev. Mastering board games by external and internal planning with language models, 2024. URL https://arxiv.org/abs/2412.12119.  

Bilgehan Sel, Ahmad Al-Tawaha, Vanshaj Khattar, Ruoxi Jia, and Ming Jin. Algorithm of thoughts: Enhancing exploration of ideas in large language models, 2024. URL https://arxiv.org/abs/ 2308.10379.  

Amrith Setlur, Saurabh Garg, Xinyang Geng, Naman Garg, Virginia Smith, and Aviral Kumar. Rl on incorrect synthetic data scales the efciency of llm math reasoning by eight-fold, 2024a. URL https://arxiv.org/abs/2406.14532.  

Amrith Setlur, Chirag Nagpal, Adam Fisch, Xinyang Geng, Jacob Eisenstein, Rishabh Agarwal, Alekh Agarwal, Jonathan Berant, and Aviral Kumar. Rewarding progress: Scaling automated process verifers for llm reasoning. arXiv preprint arXiv:2410.08146, 2024b.  

Amrith Setlur, Chirag Nagpal, Adam Fisch, Xinyang Geng, Jacob Eisenstein, Rishabh Agarwal, Alekh Agarwal, Jonathan Berant, and Aviral Kumar. Rewarding progress: Scaling automated process verifers for llm reasoning, 2024c. URL https://arxiv.org/abs/2410.08146.  

Amrith Setlur, Yuxiao Qu, Lunjun Zhang, Matthew Yang, Virginia Smith, and Aviral Kumar. Optimizing llm test-time compute involves solving a meta-rl problem. https://blog.ml.cmu.edu/2025/01/08/ optimizing-llm-test-time-compute-involves-solving-a-meta-rl-problem/, 2025. CMU MLD Blog.  

Zhihong Shao, Peiyi Wang, Qihao Zhu, Runxin Xu, Junxiao Song, Xiao Bi, Haowei Zhang, Mingchuan Zhang, Y. K. Li, Y. Wu, and Daya Guo. Deepseekmath: Pushing the limits of mathematical reasoning in open language models, 2024. URL https://arxiv.org/abs/2402.03300.  

Noah Shinn, Federico Cassano, Edward Berman, Ashwin Gopinath, Karthik Narasimhan, and Shunyu Yao. Refexion: Language agents with verbal reinforcement learning, 2023. URL https://arxiv. org/abs/2303.11366.  

D. Silver, A. Huang, C. Maddison, et al. Mastering the game of go with deep neural networks and tree search. Nature, 529:484–489, 2016. doi: 10.1038/nature16961.  

David Silver, Thomas Hubert, Julian Schrittwieser, Ioannis Antonoglou, Matthew Lai, Arthur Guez, Marc Lanctot, Laurent Sifre, Dharshan Kumaran, Thore Graepel, Timothy P. Lillicrap, Karen Simonyan, and Demis Hassabis. A general reinforcement learning algorithm that masters chess, shogi, and go through self-play. Science, 362(6419):1140–1144, 2018. doi: 10.1126/science. aar6404.  

Avi Singh, John D. Co-Reyes, Rishabh Agarwal, Ankesh Anand, Piyush Patil, Xavier Garcia, Peter J. Liu, James Harrison, Jaehoon Lee, Kelvin Xu, Aaron Parisi, Abhishek Kumar, Alex Alemi, Alex Rizkowsky, Azade Nova, Ben Adlam, Bernd Bohnet, Gamaleldin Elsayed, Hanie Sedghi, Igor Mordatch, Isabelle Simpson, Izzeddin Gur, Jasper Snoek, Jefrey Pennington, Jiri Hron, Kathleen Kenealy, Kevin Swersky, Kshiteej Mahajan, Laura Culp, Lechao Xiao, Maxwell L. Bileschi, Noah Constant, Roman Novak, Rosanne Liu, Tris Warkentin, Yundi Qian, Yamini Bansal, Ethan Dyer, Behnam Neyshabur, Jascha Sohl-Dickstein, and Noah Fiedel. Beyond human data: Scaling self-training for problemsolving with language models, 2024. URL https://arxiv.org/abs/2312.06585.  

Prasann Singhal, Tanya Goyal, Jiacheng Xu, and Greg Durrett. A long way to go: Investigating length correlations in rlhf, 2023.  

Charlie Snell, Jaehoon Lee, Kelvin Xu, and Aviral Kumar. Scaling llm test-time compute optimally can be more efective than scaling model parameters, 2024. URL https://arxiv.org/abs/2408. 03314.  

Ray J Solomonof. A formal theory of inductive inference. part i. Information and control, 7(1):1–22, 1964.  

Bradly C. Stadie, Ge Yang, Rein Houthooft, Xi Chen, Yan Duan, Yuhuai Wu, Pieter Abbeel, and Ilya Sutskever. Some considerations on learning to explore via meta-reinforcement learning, 2019. URL https://arxiv.org/abs/1803.01118.  

Nisan Stiennon, Long Ouyang, Jef Wu, Daniel M. Ziegler, Ryan Lowe, Chelsea Voss, Alec Radford, Dario Amodei, and Paul Christiano. Learning to summarize from human feedback, 2022.   
Fahim Tajwar, Anikait Singh, Archit Sharma, Rafael Rafailov, Jef Schneider, Tengyang Xie, Stefano Ermon, Chelsea Finn, and Aviral Kumar. Preference fne-tuning of llms should leverage suboptimal, on-policy data. arXiv preprint arXiv:2404.14367, 2024.   
Qwen Team. Qwq: Refect deeply on the boundaries of the unknown, November 2024. URL https://qwenlm.github.io/blog/qwq-32b-preview/.   
Ye Tian, Baolin Peng, Linfeng Song, Lifeng Jin, Dian Yu, Haitao Mi, and Dong Yu. Toward selfimprovement of llms via imagination, searching, and criticizing, 2024. URL https://arxiv. org/abs/2404.12253.   
Shubham Toshniwal, Wei Du, Ivan Moshkov, Branislav Kisacanin, Alexan Ayrapetyan, and Igor Gitman. Openmathinstruct-2: Accelerating ai for math with massive open-source instruction data. arXiv preprint arXiv:2410.01560, 2024.   
Peiyi Wang, Lei Li, Zhihong Shao, Runxin Xu, Damai Dai, Yifei Li, Deli Chen, Yu Wu, and Zhifang Sui. Math-shepherd: Verify and reinforce llms step-by-step without human annotations. In Proceedings of the 62nd Annual Meeting of the Association for Computational Linguistics (Volume 1: Long Papers), pp. 9426–9439, 2024.   
Ronald J. Williams. Simple statistical gradient-following algorithms for connectionist reinforcement learning. Mach. Learn., 8(3–4):229–256, may 1992. ISSN 0885-6125. doi: 10.1007/BF00992696. URL https://doi.org/10.1007/BF00992696.   
Yuxi Xie, Anirudh Goyal, Wenyue Zheng, Min-Yen Kan, Timothy P. Lillicrap, Kenji Kawaguchi, and Michael Shieh. Monte carlo tree search boosts reasoning via iterative preference learning, 2024. URL https://arxiv.org/abs/2405.00451.   
Mengjiao Yang, Dale Schuurmans, Pieter Abbeel, and Ofr Nachum. Chain of thought imitation with procedure cloning, 2022. URL https://arxiv.org/abs/2205.10816.   
Shunyu Yao, Dian Yu, Jefrey Zhao, Izhak Shafran, Thomas L. Grifths, Yuan Cao, and Karthik Narasimhan. Tree of thoughts: Deliberate problem solving with large language models, 2023. URL https://arxiv.org/abs/2305.10601.   
Tian Ye, Zicheng Xu, Yuanzhi Li, and Zeyuan Allen-Zhu. Physics of language models: Part 2.1, grade-school math and the hidden reasoning process, 2024a. URL https://arxiv.org/abs/ 2407.20311.   
Tian Ye, Zicheng Xu, Yuanzhi Li, and Zeyuan Allen-Zhu. Physics of language models: Part 2.2, how to learn from mistakes on grade-school math problems, 2024b. URL https://arxiv.org/abs/ 2408.16293.   
Shuo Yin, Weihao You, Zhilong Ji, Guoqiang Zhong, and Jinfeng Bai. Mumath-code: Combining tooluse large language models with multi-perspective data augmentation for mathematical reasoning. arXiv preprint arXiv:2405.07551, 2024.   
Xiao Yu, Baolin Peng, Vineeth Vajipey, Hao Cheng, Michel Galley, Jianfeng Gao, and Zhou Yu. Exact: Teaching ai agents to explore with refective-mcts and exploratory learning, 2024. URL https://arxiv.org/abs/2410.02052.  

Zheng Yuan, Hongyi Yuan, Chengpeng Li, Guanting Dong, Keming Lu, Chuanqi Tan, Chang Zhou, and Jingren Zhou. Scaling relationship on learning mathematical reasoning with large language models, 2023. URL https://arxiv.org/abs/2308.01825.  

Albert S. Yue, Lovish Madaan, Ted Moskovitz, DJ Strouse, and Aaditya K. Singh. HARP: A challenging human-annotated math reasoning benchmark, 2024. URL https://github.com/ aadityasingh/HARP.  

Eric Zelikman, Yuhuai Wu, Jesse Mu, and Noah Goodman. Star: Bootstrapping reasoning with reasoning. Advances in Neural Information Processing Systems, 35:15476–15488, 2022.  

Eric Zelikman, Georges Harik, Yijia Shao, Varuna Jayasiri, Nick Haber, and Noah D Goodman. Quiet-star: Language models can teach themselves to think before speaking. arXiv preprint arXiv:2403.09629, 2024.  

Lunjun Zhang, Arian Hosseini, Hritik Bansal, Mehran Kazemi, Aviral Kumar, and Rishabh Agarwal. Generative verifers: Reward modeling as next-token prediction, 2024a. URL https://arxiv. org/abs/2408.15240.  

Yiming Zhang, Jianfeng Chi, Hailey Nguyen, Kartikeya Upasani, Daniel M. Bikel, Jason Weston, and Eric Michael Smith. Backtracking improves generation safety, 2024b. URL https://arxiv.org/ abs/2409.14586.  

Andy Zhou, Kai Yan, Michal Shlapentokh-Rothman, Haohan Wang, and Yu-Xiong Wang. Language agent tree search unifes reasoning acting and planning in language models, 2024a. URL https: //arxiv.org/abs/2310.04406.  

Yifei Zhou, Andrea Zanette, Jiayi Pan, Sergey Levine, and Aviral Kumar. Archer: Training language model agents via hierarchical multi-turn rl, 2024b. URL https://arxiv.org/abs/2402.19446.  

# A. Prompting  

We investigate whether carefully constructed prompting protocols can induce reasoning capabilities similar to those demonstrated by RL-based models like O1. Through an evaluation of fve increasingly sophisticated instruction sets, we analyze token generation patterns, presence of reasoning-like behaviors, and their correlation with math problem-solving performance.  

Table 4: Input token count across prompting strategies using OpenAI’s GPT-4o tokenizer (excluding problem tokens). Complex instructions like Think (3-shot) use substantially more tokens, primarily due to in-context demonstrations and explicit capabilities descriptions (e.g. thinking, refection, and verifcation requirements).  

<html><body><table><tr><td>Strategy</td><td>Tokens</td></tr><tr><td>Baseline</td><td>23</td></tr><tr><td>Chain-of-Thought (CoT)</td><td>35</td></tr><tr><td>Think</td><td>1,895</td></tr><tr><td>Think withdemonstrations (3-shot)</td><td>4,659</td></tr><tr><td>Think & Verify</td><td>2,111</td></tr></table></body></html>  

# B. Regret Analysis  

Table 5 presents detailed statistics on regret expression across diferent models and prompting strategies. The data shows varying levels of self-correction and error acknowledgment behaviors across model scales and prompting approaches.  

# C. Diferent Instruction Tuning Objectives  

Below we outline a number of potential fnetuning objectives for the instruction tuning stage of our reasoning pipeline described in Section 7.1.  

1. A standard procedural-cloning approach (Yang et al., 2022) which uses traditional supervised fne-tuning  

$$
\mathcal{L}(\theta)=\operatorname*{min}_{\theta}-\mathbb{E}_{({\bf q},{\bf Z},{\bf S})\sim\mathcal{D}_{\mathrm{train}}}\left[\sum_{i=1}^{|\mathbf{Z}|}\log\pi_{\theta}({\bf z}_{i+1}|{\bf Z}_{i},{\bf q})+\sum_{i=1}^{|\mathbf{S}|}\log\pi_{\theta}({\bf s}_{i+1}|{\bf S}_{i},{\bf Z},{\bf q})\right].
$$  

2. Alternatively, we can optimize only over the Meta-CoT tokens with the following optimization objective  

$$
\mathcal{L}(\theta)=\operatorname*{min}_{\theta}-\mathbb{E}_{(\mathbf{q},\mathbf{Z},\mathbf{S})\sim\mathcal{D}_{\mathrm{train}}}\left[\sum_{i=1}^{|\mathbf{Z}|}\log\pi_{\theta}(\mathbf{z}_{i+1}|\mathbf{Z}_{i},\mathbf{q})\right].
$$  

In this formulation, the model will not learn to generate a solution, requiring the use of a separate step to summarize the search process into a fnal solution. One hypothesis is that training on joint sequences with the solution can serve as additional supervision to help the model with maintaining internal state.  

3. One consideration is whether updating model parameters for sub-optimal, or even incorrect, branches can induce the model to generate more errors. To handle this, we can mask incorrect steps/branches in the above loss  

$$
\mathcal{L}(\theta)=\operatorname*{min}_{\theta}-\mathbb{E}_{(\mathbf{q},\mathbf{Z},\mathbf{S})\sim\mathcal{D}_{\mathrm{train}}}\left[\sum_{i=1}^{|\mathbf{Z}|}I\{\mathbf{z}_{i+1}\in\mathbf{S}\}\log\pi_{\theta}(\mathbf{z}_{i+1}|\mathbf{Z}_{i},\mathbf{q})\right].
$$  

That is we only train on the branches that are on the correct path from the root node (problem) to the fnal answer. Prior works (Gandhi et al., 2024; Ye et al., 2024b) did not mask the incorrect steps, and reported no degradation in performance, as long as the data distribution remains reasonable. On the other hand Zhang et al. (2024b) specifcally mask the tokens of the unsafe generation.  

4. Under the above objective, there could be signifcant task mismatch between training and generation, especially in the case of long search chains. One method to mitigate this issue is to generate more synthetic training data by pruning intermediate branches not on the optimal solution path to generate additional sequences. For example this was the SFT objective used by Zhang et al. (2024b), which simultaneously trains on backtracking trajectories (with masking) and the fnal optimal solution (Equation 1 in that paper).  

# D. MCTS Details  

Here we outline the details of our MCTS procedure from Section 4.3.1. Our procedure involves three main steps: selection, backup, and expansion. We describe each step in detail below.  

Selection. Starting at the root node $\mathbf{s}_{0}$ (initialized as the question $\mathbf{q}.$ , a child node s (i.e. the next step in the solution) is selected until a leaf node $\mathbf{s}_{t}$ is reached. A partial solution is then represented as $\mathbf{S}_{t}=(\mathbf{s}_{1},\ldots,\mathbf{s}_{t})$ . The child node is selected according to the policy $\mathbf{s}_{t+1}=\arg\operatorname*{max}_{\mathbf{s}}U(\mathbf{S}_{t},\mathbf{s})$ where $U(\mathbf{S}_{t},\mathbf{s})$ is calculated using UCT (Kocsis & Szepesvári, 2006), defned as  

$$
U(\mathbf{S}_{t},\mathbf{s})=Q(\mathbf{S}_{t},\mathbf{s})+c_{\mathrm{exp}}\sqrt{\frac{\log N(\mathbf{S}_{t},\mathbf{s})}{N(\mathbf{S}_{t})}}.
$$  

In this equation $Q(\mathbf{S}_{t},\mathbf{s})$ is a value function, $N(\mathbf{S}_{t},\mathbf{s})$ is the visit count of selecting step s from the partial solution $\mathbf{S}_{t}$ , $\begin{array}{r}{N(\mathbf{S}_{t})=\sum_{s}N(\mathbf{S}_{t},\mathbf{s})}\end{array}$ is the total visit count of the partial solution $\mathbf{S}_{t}$ , and $c_{\mathrm{exp}}\in\mathbb{R}$ is the exploration constant . We opt for the UCT formulation, instead of the more recent PUCT variant (Rosin, 2011), as in our case actions are represented by logical steps and thus likelihoods could be quite skewed.  

Expansion. After the selection operation, we have chosen a leaf node $\mathbf{s}_{t}$ , and the path from the root node $\mathbf{s}_{0}$ to the leaf node forms a partial solution by concatenating the individual steps into the partial solution, i.e. $\mathbf{S}_{t}=(\mathbf{s}_{1},\ldots,\mathbf{s}_{t})$ . If $\mathbf{s}_{t}$ is terminal, we do not expand it, otherwise the node is expanded by sampling $b$ actions $\{\mathbf{s}_{t+1}^{i}\}_{i=1}^{b}\sim\pi_{\boldsymbol\theta}(\cdot|\mathbf{S}_{t})$ from the policy $\pi_{\theta}$ , and then adding the nodes $\{\mathbf{a}_{t+1}^{i}\}_{i=1}^{b}$ as children of node $\mathbf{S}_{t}$ . The value of each child node is initialized with a value function $v$ as $Q(\dot{\mathbf{S}}_{t},\mathbf{s}_{t}^{i})=v\big((\mathbf{S}_{t},\mathbf{s}_{t+1}^{i}),\mathbf{q}\big)$ . This evaluation essentially replaces the "simulation" step usually carried out in game settings.  

Backup. Finally, the statistics of the expanded node $\mathbf{S}_{t}$ are backed up through the path to the root node $\mathbf{s}_{0}$ . For every ancestor node $\mathbf{S}_{i}$ on the path, the visit count is incremented  

$$
N(\mathbf{S}_{i})=N(\mathbf{S}_{i})+1
$$  

and the value of each ancestor is updated as  

$$
\begin{array}{r l}{Q(\mathbf{S}_{i},\mathbf{s}_{i+1})=}&{{}\;Q(\mathbf{S}_{i},\mathbf{s}_{i+1})+\frac{v(\mathbf{S}_{t},\mathbf{q})-Q(\mathbf{S}_{i},\mathbf{s}_{i+1})}{N(\mathbf{S}_{i},\mathbf{s}_{i+1})}.}\end{array}
$$  

# E. Chains-Of-Thought  

This section contains synthetic CoT traces comparing how diferent models reason through the same planning problems. Specifcally, we show complete reasoning traces from Gemini-1.5 solving problems using $\mathrm{A^{*}}$ and MCTS algorithms, alongside parallel traces from DeepSeek-R1, DeepSeek-O1, and Gemini-2 working through identical tasks. We also include an example trace demonstrating how a smaller SFT-tuned Llama3.1-8B model performs tool-augmented reasoning on a challenging problem using dynamic programming. Each trace details the step-by-step "internal" traces and the fnal solution.  

There were 12 seats prepared for the jury at a round table, each with a name indicated on it. Nikolai Nikolaevich, who arrived frst, absent-mindedly sat not in his designated seat but in the next one clockwise. Each jury member arriving at the table afterward would either sit in their designated seat, or, if it was already occupied, they would walk around the table clockwise and sit in the next available seat. The resulting seating arrangement of the jury members depends on the order in which they approached the table. How many diferent ways can the jury be seated?   
0.0  

Let's break this down step-by-step:  

0.03125  

1. The first person, Nikolai, can sit in any of the 12 seats.   
2. The second person can sit in any of the remaining 11 seats.   
3. The third person can sit in any of the remaining 10 seats.   
4. This pattern continues until all 12 seats are filled.  

0.0  

Alternatively...  

1. The first person, Nikolai Nikolaevich, has only one choice for his seat.   
2. The second person has one choice, as both their designated seat and the seat next to it are taken.   
3. The third person also has one choice, as two seats are taken.   
4. This pattern continues until all seats are filled.  

0.0  

1. The first member can choose any of the 12 seats.   
2. The second member can choose any of the remaining 11 seats, but if the first member chose the next seat clockwise, the second member has only 10 options (not considering the first seat). This pattern continues until the last member, who has only 1 option left.  

0.0  

Alter a V   
1. Nikolai Nikolaevich can only sit in one position (the next seat clockwise from his designated seat).   
2. After Nikolai Nikolaevich sits down, there are 11 remaining seats.   
3. The next person to sit can choose from the 11 remaining seats, but if one of those seats is already occupied, they have to sit in the next available seat. This means that the second person has 11 choices.   
4. The third person then has 10 choices, and so on, until the last person has only 1 choice.  

0.0  

<html><body><table><tr><td>Alternatively...</td><td></td></tr><tr><td></td><td>1. The first person, Nikolai Nikolaevich, has a fixed position. 2. The second person can either sit in their designated seat or move to</td></tr><tr><td></td><td>the next seat clockwise. This gives 2 options for the second person. 3. The third person can either sit in their designated seat or move to</td></tr><tr><td></td><td>the next seat clockwise. This gives 2 options for the third person, but we need to consider the case where the second person moved, so there's only 1 additional option for the third person. 4. Similarly, for each subsequent person, there are 2 options if the</td></tr><tr><td></td><td>previous person didn't move, and 1 additional option if the previous person moved.</td></tr><tr><td>0.046875</td><td></td></tr><tr><td></td><td>Let's analyze the pattern:</td></tr><tr><td>0.0390625</td><td></td></tr><tr><td></td><td></td></tr><tr><td></td><td>- The first person has 1 option. The second person has 2 options.</td></tr><tr><td></td><td>The third person has 2 options.</td></tr><tr><td></td><td>The fourth person has 2 options if the third person didn't move, and 1 option if the third person moved, but since the second person</td></tr><tr><td></td><td>moved, the fourth person still has 2 options. - This pattern continues for the rest of the jury members.</td></tr><tr><td></td><td></td></tr><tr><td>0.03125</td><td>So, the total number of ways the jury can be seated is the product of</td></tr><tr><td></td><td>the options for each person:</td></tr><tr><td>0.03125</td><td></td></tr><tr><td></td><td>\[1\times2\times2\times2\times2\times2\times2\times2\ times 2\times 2\times 2 \times 2 = 2^{11} = 2048 \]</td></tr><tr><td></td><td></td></tr><tr><td>0.0078125</td><td>Wait, however, this process can be simplified by realizing that each</td></tr><tr><td></td><td>person has a unique "decision point" - the first time they have a choice between sitting in their designated seat and moving to the</td></tr><tr><td>choices.</td><td>next seat. After that,their seat is determined by the previous</td></tr><tr><td></td><td></td></tr><tr><td>0.0078125</td><td></td></tr><tr><td></td><td></td></tr><tr><td></td><td></td></tr><tr><td></td><td></td></tr><tr><td></td><td></td></tr><tr><td></td><td></td></tr><tr><td></td><td></td></tr><tr><td></td><td></td></tr><tr><td></td><td></td></tr><tr><td></td><td></td></tr><tr><td></td><td></td></tr><tr><td></td><td>Alternatively, however, this is a circular arrangement, and we need to</td></tr><tr><td></td><td></td></tr><tr><td></td><td></td></tr><tr><td></td><td>account for the fact that the last person can also move. This</td></tr><tr><td></td><td></td></tr><tr><td></td><td></td></tr><tr><td></td><td></td></tr><tr><td></td><td></td></tr><tr><td></td><td></td></tr><tr><td></td><td></td></tr><tr><td></td><td></td></tr><tr><td></td><td></td></tr><tr><td></td><td></td></tr><tr><td></td><td></td></tr><tr><td></td><td>introduces a complication, as the last person will always have only</td></tr><tr><td></td><td></td></tr><tr><td></td><td></td></tr><tr><td></td><td></td></tr><tr><td></td><td></td></tr><tr><td></td><td></td></tr><tr><td></td><td></td></tr><tr><td></td><td></td></tr><tr><td></td><td></td></tr><tr><td></td><td>1 option if everyone else is seated.</td></tr><tr><td></td><td></td></tr><tr><td></td><td></td></tr><tr><td></td><td></td></tr><tr><td></td><td></td></tr><tr><td></td><td></td></tr><tr><td></td><td></td></tr><tr><td></td><td></td></tr><tr><td></td><td></td></tr><tr><td></td><td></td></tr><tr><td></td><td></td></tr><tr><td></td><td></td></tr><tr><td>0.0546875</td><td></td></tr><tr><td></td><td></td></tr><tr><td></td><td></td></tr><tr><td></td><td></td></tr><tr><td></td><td></td></tr><tr><td></td><td></td></tr><tr><td></td><td></td></tr></table></body></html>  

<html><body><table><tr><td>of these 12 people,there are 2 options. However,we need to subtract the cases where the last person doesn't move, as these cases are already counted in the 2 options for each person.</td></tr><tr><td>0.0078125 Wait,to simplify, let's consider the arrangement of people relative to Nikolai Nikolaevich's fixed position. There are 11! ways to arrange the remaining jury members around the table, as each person</td></tr><tr><td>has a fixed position relative to Nikolai. 0.0234375 However,this calculation doesn't take into account the fact that some</td></tr><tr><td>people can move. To correct for this, we need to consider the number of "cycle" arrangements, where each cycle represents a group of people who moved together.</td></tr><tr><td>0.0078125 Wait, let's re-examine the situation:</td></tr><tr><td>0.046875</td></tr><tr><td>1. Nikolai Nikolaevich is Seated. 2. The second person can move or stay, giving 2 options.</td></tr><tr><td>3. For each subsequent person, there's only 1 option left if everyone else is seated.</td></tr><tr><td>0.046875</td></tr><tr><td>option. 0.03125</td></tr><tr><td>So,the total number of arrangements is the product of the options at each step:</td></tr><tr><td>0.0390625</td></tr><tr><td>\[ 2\times 2\times 2\times 2\times 2\times 2\times 2\times 2^ times 2\times2\times 1 = 2~{11} =boxed{2048}\]</td></tr><tr><td>0.0</td></tr><tr><td></td></tr><tr><td></td></tr><tr><td>Wait，let's consider the total number of ways without the circular constraint. For the first person, there's 1 option. For the rest,             option. So, the total number of arrangements without the circular constraint is: \[1\times2~{11}\]</td></tr></table></body></html>  

0.171875  

However, we need to subtract the cases where everyone is seated in their designated seats, which is 1 arrangement. But this is already counted in the 2 options for each person.  

0.03125  

Alternatively, however, this counts each arrangement twice, since the table is round, and we can rotate the arrangement without changing it. To account for this, we divide by the number of seats, which is 12.  

0.0703125  

Alternatively, however, this counts each arrangement twice (once in each direction around the table), so we need to divide by 2: \[ \frac{1 \times $2^{{\displaystyle\hat{\mathbf{\Gamma}}}}\{11\}\}\{2\}{\displaystyle\mathbf{\Gamma}}=\ 2^{{\displaystyle\hat{\mathbf{\Gamma}}}}\{10\}{\displaystyle\mathbf{\Gamma}}\backslash{\mathbf{\Gamma}}]$  

0.328125  

But this is not the correct answer. The correct answer is the number of derangements of 12 objects, which is given by the formula:   
$1\,[\,!\,12\ =\ 12\,!$ \left( 1 - \frac{1}{1!} + \frac{1}{2!} - \frac{1}{3!} + \ cdots $^+$ \frac{(-1)^{12}}{12!} \right) \]  

0.0078125  

Alternatively, but this is still not correct because it doesn't account for the fact that the last person always has only 1 option. Let's reconsider the problem.  

0.0625  

Alternatively, but this is not the correct answer because the problem statement asks for the number of different ways the jury can be seated, considering the circular arrangement. In a circular arrangement, the last person's position is fixed once the first person's position is chosen. Therefore, the correct calculation should consider the number of ways to arrange the remaining 11 people around the table, not the total possible arrangements.  

0.1484375  

Alternatively, but we're not done yet! We need to consider the case where the second person moves, which would cause the last person to have only 1 option. This introduces an overcount. To correct for this, we need to subtract the number of arrangements where the second person moves, which is equivalent to the number of ways the jury can be seated with one person missing (i.e., 11 people). This is:  

0.1796875  

But we still need to account for the fact that the last person has only 1 option. This means that we've overcounted by a factor of 2 for the last person. To correct this, we divide by 2 again:   
$\backslash[\;\;\backslash\mathtt{f r a c}\{2^{\sim}\{10\}\}\{2\}\;=\;2^{\sim}9\;\;\backslash]$   
0.0390625   
Wait, but we also need to consider the case where the first person moves, which would be counted 12 times (once for each possible position). So, we need to subtract these 11 arrangements:   
$\backslash\left[\begin{array}{l}{{\bar{2}}^{\sim}\{10\}~-~11}\end{array}\backslash\right]$   
0.1484375   
Alternatively, this is still not correct, as we haven't accounted for the fact that the last person can only move one seat. We need to consider the cases where the last person moves and subtract those from the total.   
0.1796875   
But we need to correct this approach because we've double-counted due to the nature of the circular arrangement.   
0.015625   
Alternatively, we also need to consider the case where someone arrives after the first person and the table is empty. In this case, the first person still has 1 option, and the remaining 11 people have 2 options each, but again, we need to divide by 2 to account for the circular arrangement:   
\[ \frac{1 \times $\bar{2^{+}}\{11\}\}\{2\}\ =\ 2^{-}\{10\}\ \setminus\}$   
0.2421875   
However, this is not the correct answer. The correct answer is:   
\[ \boxed{2^{11}} \]   
0.0   
Alternatively, but this doesn't account for the fact that the last person has only 1 option. To adjust for this, we need to consider the cases where the last person moves. This introduces a more complex conditional arrangement, but the key insight is that the number of choices for the last person is actually the same as the number of choices for any other person, because the arrangement is circular.   
0.1015625   
Alternatively, but we still need to account for the fact that the last person can only move in one direction. This doesn't affect the number of arrangements, as the circular constraint is already accounted for by the division by 2.  

0.953125  

So, the total number of different ways the jury can be seated is: \[ \boxed{1024} \]  

1.0  

# Solution:  

Let’s break this down step-by-step:  

1. The frst person, Nikolai Nikolaevich, has a fxed position. 2. The second person can either sit in their designated seat or move to the next seat clockwise. This gives 2 options for the second person. 3. The third person can either sit in their designated seat or move to the next seat clockwise. This gives 2 options for the third person, but we need to consider the case where the second person moved, so there’s only 1 additional option for the third person. 4. Similarly, for each subsequent person, there are 2 options if the previous person didn’t move, and 1 additional option if the previous person moved.  

However, this is a circular arrangement, and we need to account for the fact that the last person can also move. This introduces a complication, as the last person will always have only 1 option if everyone else is seated.  

Let’s consider the total number of ways without the circular constraint. For the frst person, there’s 1 option. For the rest, there are 2 options each, except for the last person who has only 1 option. So, the total number of arrangements without the circular constraint is:  

$$
1\times2^{11}
$$  

However, this counts each arrangement twice (once in each direction around the table), so we need to divide by 2:  

$$
{\frac{1\times2^{11}}{2}}=2^{10}
$$  

But we still need to account for the fact that the last person can only move in one direction. This doesn’t afect the number of arrangements, as the circular constraint is already accounted for by the division by 2.  

o, the total number of diferent ways the jury can be seated  

# Trace - Reasoning with External Tools  

# A\* Generated Solution Trace  

Question: There were 12 seats prepared for the jury at a round table, each with a name indicated on it. Nikolai Nikolaevich, who arrived frst, absent-mindedly sat not in his designated seat but in the next one clockwise. Each jury member arriving at the table afterward would either sit in their designated seat, or, if it was already occupied, they would walk around the table clockwise and sit in the next available seat. The resulting seating arrangement of the jury members depends on the order in which they approached the table. How many diferent ways can the jury be seated?  

To solve this problem, we need to determine the total number of different seating arrangements of 12 jury members when each member either sits in their designated seat or walks around the table clockwise to sit in the next available seat.  

0.01171875  

To solve this problem, we need to analyze the seating possibilities given the rules described:  

0.0078125  

1. Nikolai Nikolaevich starts in the next seat clockwise. 2. Each subsequent member either sits in their designated seat or the next available seat.  

0.015625  

To determine the number of ways the jury can be seated, we need to consider the following steps:  

0.01171875  

1. Assign Nikolai Nikolaevich to the next seat clockwise.   
2. Each subsequent jury member has 2 choices: their designated seat or the next available seat.   
3. Since Nikolai Nikolaeich has already changed seats, we need to account for the extra step he took.  

0.046875  

Let's represent this using Python code to calculate the number of different ways the jury can be seated.  

0.05078125  

We can model this problem using dynamic programming, where each state represents the number of available seats and the current seat Nikolai is sitting on. We'll use a recursive approach to count the number of valid seating arrangements.  

0.0390625   
Here's the Python code to solve this problem:   
0.03515625   
Let's implement this in Python.   
0.03515625   
\`\`\`python   
def count\_seating\_arrangements(total\_seats): dp = [0] \* (total\_seats + 1) $\mathrm{{dp}\,[1]\ =\ 1}$ \# Initial case: Nikolai is in the next seat clockwise \# Iterate over all possible remaining seats where Nikolai could be for seat in range(total\_seats): for available\_seats in range(seat + 1, total\_seats): dp[available\_seats] $\mathrel{+{=}}$ dp[available\_seats - seat - 1] return dp[total\_seats]   
\# There are 12 seats prepared for the jury   
total\_seats $=~12$   
seating\_ways $=$ count\_seating\_arrangements(total\_seats)   
print(seating\_ways) \`\`output   
0   
0.03125   
It seems our initial approach overlooked a crucial detail. We need to consider the actual sequence of each member. Each member can either:   
1. Sit in their designated seat.   
2. Walk to the next available seat.   
0.04296875   
It seems there was an issue with the initial state of the dynamic programming approach. Let's refine our approach:  

0.02734375   
1. We should count directly from the number of valid seating   
arrangements starting from Nikolai's initial seat.   
2. For each subsequent member, they have two choices: their designated seat or the next available seat.   
0.046875   
Let's re-implement the logic more carefully:   
0.046875   
1. We need to correctly initialize the base case.   
2. We need to ensure we correctly handle the initial arrangement where Nikolai is in the next seat clockwise.   
0.04296875   
Let's reconsider the logic and the initialization of the DP table: 0.0546875   
Here's an updated approach:   
0.04296875   
1. We start by initializing the base case correctly.   
2. We use an iterative approach instead of dynamic programming to simplify the calculation.   
0.09375   
Let's implement the solution again:   
0.12109375   
Let's implement this revised approach in Python:   
0.09765625   
Let's implement the updated version in Python:   
0.07421875   
\`python   
def   
count\_seating\_arrangements(total\_seats): either sits in their designated seat or the next available seat.   
To determine the number of ways the jury can be seated, we need to consider the following steps:  

![](images/bedee31723d52c1d3721ef66f0c4210abca45ff881484983cb04559b5bf49acb.jpg)  

2. Each subsequent jury member has 2 choices: their designated seat or the next available seat. 3. Since Nikolai Nikolaeich has already changed seats, we need to account for the extra step he took.  

\`python  

def count\_seating\_arrangements(total\_seats):  

$$
\mathrm{{d}p~=~[0]~\ *~\ (\ t o t a l`\_s e a t s~+~1)}
$$  

total\_seats = 12  

print(seating\_ways)  

![](images/1ca9ae6f0fd7cb735e55d5d312b0c2399465b7fb6459edffcd7f2de8eba72cb1.jpg)  

# O1 ofcial math CoT  

# A\* Generated Solution Trace (Gemini 1.5) Problem  

Let $n$ be an even positive integer. Let $p$ be a monic, real polynomial of degree $2n$ ;   
that is to $\mathrm{say},p(x)=x^{2n}+a_{2n-1}x^{2n-1}+\cdot\cdot\cdot+a_{1}x+a_{0}$ for some real coefcients 𝑎0, . . . , 𝑎2𝑛 1.   
Suppose that $\begin{array}{r}{p\left(\frac{1}{k}\right)=k^{2}}\end{array}$ for all integers $k$ such that $1\leq|k|\leq n$ .   
Find all other real numbers $x$ for which $\textstyle p\left({\frac{1}{x}}\right)=x^{2}$ .  

# Solution  

0.0234375  

Let $\Phi q(\mathrm{x})~=~\mathrm{x}^{\sim}\{2\mathrm{n}\}\mathrm{p}\backslash$ left(\frac{1}{x}\right) = a_0 x^{2n} + a_1 x^{2n-1} + \cdots $^+$ a_{2n-1} $\texttt{x}+\texttt{1}\Phi$ .  

0.015625  

The given condition \$p\left(\frac{1}{k}\right) $=\ k^{-}2\Phi$ implies $\Phi{\mathfrak{q}}(\mathbf{k})~=~\mathbf{k}$ ${\hat{\ }}\{2\mathrm{n}\}\;\;\mathrm{k}{\hat{\mathrm{\Sigma}}}2\;=\;\;\mathrm{k}{\hat{\mathrm{\Sigma}}}\{2\mathrm{n}{+}2\}\Phi$ for $\Phi1$ \leq |k| \leq $\mathtt{n}\Phi$ .  

Let $\Phi p({\bf x})~=~{\bf x}^{-}\{2\mathrm{n}\}~+~{\bf a}_{-}\{2\mathrm{n}-1\}{\bf x}^{-}\{2\mathrm{n}-1\}~+~\mathrm{cdots}~+~{\bf a}_{-}\mathrm{1x}~+~{\bf a}_{-}0\Phi$ be a monic polynomial of degree $\Phi2\mathtt{n}\Phi$ , where $\Phi\mathbf{n}\Phi$ is an even positive integer.  

We are given that $\Phi\mathtt{P V}\mathtt{L e f t}(\mathtt{V f r a c}\{1\}\{\mathtt{k}\}\backslash\mathtt{r i g h t})\ =\ \mathtt{k}\not\sim2\Phi$ for all integers $\Phi\mathrm{k}\Phi$ such that $\Phi1$ \leq |k| \leq $\mathtt{n}\Phi$ .  

0.0078125  

We are given that $\Phi\mathtt{P V}\mathtt{L e f t}(\mathtt{V f r a c}\{1\}\{\mathtt{k}\}\backslash\mathtt{r i g h t})\ =\ \mathtt{k}\not\sim2\Phi$ for integers $\Phi\mathbf{\lambda}\Phi$ such that $\Phi1$ \leq |k| \leq $\mathtt{n}\Phi$ .  

0.0  

The condition \$p $\therefore\mathtt{l e f t}(\backslash\mathtt{f r a c}\{1\}\{\mathtt{k}\}\backslash\mathtt{r i g h t})\ =\ \mathtt{k}\not\sim2\Phi$ for $\Phi1$ \leq |k| \leq n\$ is equivalent to \$q(k) = k^{2n} $\mathtt{k}^{\hat{\mathbf{\alpha}}}2\;=\;\mathtt{k}^{\hat{\mathbf{\alpha}}}\{2\mathrm{n}{+}2\}\Phi$ for $\Phi1$ \leq |k| \ leq $\mathtt{n}\Phi$ .  

0.03125  

Consider the polynomial ${\mathfrak{F}}\tau({\mathfrak{x}})~=~{\mathfrak{x}}^{\hat{\mathbf{\Gamma}}}\{2{\mathfrak{n}}+2\}~-~{\mathfrak{q}}({\mathfrak{x}})\oplus$ . We are given that $\Phi\mathbf{r}$ ( $\mathbf{\Psi}\mathbf{k})\ =\ 0\Phi$ for $\Phi1$ \leq |k| \leq $\mathtt{n}\Phi$ .  

0.0625  

Since $\Phi\mathfrak{n}\Phi$ is even, let $\Phi\mathfrak{n}~=~2\mathfrak{m}\Phi$ . Then we have $\Phi2\mathrm{n}~=~4\mathrm{m}\Phi$ . Thus, \$r(x)\$ has the roots $\mathfrak{F}\backslash\mathtt{p m}\ 1$ , $\textsf{setminus p m}\,2$ , \ldots, \pm $2\mathtt{m}\Phi$ .   
0.0703125   
\$r(x) = (x-1) $(\mathbf{x}{+}1)$ )(x-2) $\left(\mathbf{x}+2\right)$ ) \cdots ( $\left[{\tt x}^{-}2{\tt m}\right)$ $\left.x+2\mathfrak{m}\right.$ ) = \prod_{k=1}^{2m} (xk)(x+k) = \prod_{k=1}^{2m} $({\tt x}^{\sim}2^{-}{\tt k}^{\sim}2)\,\Phi$ .   
0.015625   
Since $\Phi{\sf q}\left({\bf x}\right)\Phi$ is a polynomial of degree $\Phi2\mathbf{n}\Phi$ , $\Phi\mathbf{r}\left(\mathbf{x}\right)\Phi$ is a monic polynomial of degree $\Phi2\mathrm{n}{+}2\ =\ 4\mathrm{m}{+}2\Phi$ .   
0.0078125   
Therefore, we can write   
0.0703125   
\$\$ r(x) = (x^2-1)(x^2-2^2)\cdots(x^2 - (2m)^2) = (x^2-1)(x^2-4)\cdots(x $\hat{\mathrm{~\boldmath~\mu~}}_{2}\;-\;4\mathfrak{m}\hat{\mathrm{~\boldmath~\mu~}}_{2}\,$ ) \$\$   
0.0   
Therefore, we can write $\bf\Phi_{F}(\bf x)~=~\bf x\hat{\mu}_{\sf}{\hat{x}}\{2n{+}2\}~-~\mu(\bf x)~\,=~\mu(\bf x\hat{\mu}_{\sf}{\hat{x}}\{2{-}4\})\,(x\hat{\mu}_{\sf}{x}\circ\bf\Phi_{\sf}{x})\sim~\mu(\bf x\hat{\mu}_{\sf}{x})~,~$ ( $\mathtt{x}^{\hat{\mathbf{\Gamma}}}2^{-}\mathtt{n}^{\hat{\mathbf{\Gamma}}}2)\mathtt{R}(\mathtt{x})\mathfrak{F}$ , where $\Phi\mathrm{R}\left(\mathbf{x}\right)\Phi$ is a polynomial of degree $\Phi2\mathrm{n}{+}2\ -\ 2\mathrm{n}\ =$ $^{2\Phi}$ .   
0.1015625   
Since $\Phi{\sf q}\left({\bf x}\right)\Phi$ is monic with degree $\Phi2\mathbf{n}\Phi$ , $\Phi x^{-}\{2\mathrm{n}{+}2\}~-~\mathfrak{q}(\mathfrak{x})\Phi$ has leading coefficient 1. Also, the leading coefficient of $\mathfrak{F}(\mathfrak{x}^{\sim}2{-}1)\,(\mathfrak{x}^{\sim}2{-}4)\backslash$ cdots $(\mathtt{x}^{\sim}2^{-}\mathtt{n}^{\sim}2)\,\mathfrak{F}$ is 1. Thus, $\Phi\mathrm{R}\left(\mathbf{x}\right)\Phi$ must be monic.   
0.03125   
Let $\Re\mathrm{R}(\mathrm{x})~=~\mathrm{x}^{\sim}2~+~\mathrm{bx}~+~\mathrm{c}\Phi$ . Then we have $\oint\!\mathbf{r}\,(\mathbf{x})\ =\ \mathbf{x}\,^{\frown}\{2\mathbf{n}{+}2\}\ -\ \mathbf{q}\,(\mathbf{x})\ =\ (\mathbf{x}$ $\{2{-}1)$ $(\mathtt{x}^{\mathtt{\sim}}2^{-4})$ \cdots $\langle\mathbf{x}^{\sim}2^{-}\mathbf{n}^{\sim}2\rangle$ $(\mathtt{x}^{\sim}2+\mathtt{b x}^{\mathtt{+}}\mathtt{c})\Phi$ .   
0.1953125   
We are looking for values of $\Phi\mathbf{x}\Phi$ such that \$p\left(\frac{1}{x}\right) = $\mathfrak{X}^{-}2\mathfrak{X}$ , which is equivalent to ${\mathfrak{H}}{\mathfrak{q}}({\mathfrak{x}})~=~{\mathfrak{x}}^{\hat{\mathfrak{f}}}\{2{\mathfrak{n}}\!+\!2\}\Phi$ . This means we want to find the roots of $\Phi\mathbf{r}\left(\mathbf{x}\right)\ =\ 0\Phi$ . We know that $\Phi\mathbf{r}\left(\mathbf{x}\right)\ =\ 0\Phi$ for $\Phi\mathbf{x}~=$ $\textsf{V m}\mathtt{k}\Phi$ for $\Phi1$ \leq k \leq $\mathtt{n}\Phi$ . We are looking for the roots of $^{\Phi\mathbb{X}}^{-2\mathrm{~\textcent~}+}$ $b x+c=0\Phi$ .   
0.1796875   
Since \$r(0) = (-1)^n(-1)(-4)\cdots $\left(-\mathbf{n}^{\displaystyle\widehat{\mathbf{\alpha}}}2\right)\mathsf{c}\ =\ \left(-1\right)\widehat{\mathbf{\alpha}}\mathbf{n}\ \mathbf{\alpha}\left(-1\right)\widehat{\mathbf{\alpha}}\mathbf{n}$ (1)(4)\cdots( $\mathrm{n}{\hat{\mathbf{\Gamma}}}2)\,\mathtt{c}\ =\ \mathtt{n}!{\hat{\mathbf{\Gamma}}}2\ \mathsf{c}\Phi$ , we have $\Phi\mathbf{r}\left(0\right)\;\;=\;\;-\mathbf{q}\left(0\right)\;\;=\;\;-1\Phi$ . Thus $\Phi\mathfrak{n}!\hat{\mathbf{\xi}}^{2}\;\textsf{c}=\;-1\Phi$ , so $\Phi\mathtt{C}\ =\ -\backslash\mathtt{f r a c}\{1\}\{\mathtt{n}\{\hat{\mathtt{\Gamma}}^{2}2\}\mathfrak{F}$ .   
0.40625  

\$r(x) = (x^2-1)(x^2-4)\cdots(x^2-n^2)(x^2+bx-\frac{1}{n!^2})\$.  

0.3984375  

Since the coefficients of $\Phi\mathrm{{p}\left(x\right)\Phi}$ are real, the coefficients of $\Phi{\sf q}\left({\bf x}\right)\Phi$ are real, and so the coefficients of $\Phi\mathbf{r}\left(\mathbf{x}\right)\Phi$ are real. Therefore, the coefficients of $\Phi\mathrm{R}\left(\mathbf{x}\right)\Phi$ are real. In particular, $\Phi\mathfrak{b}\Phi$ is real. The roots of $\mathfrak{F x}^{-2+\mathfrak{b x}-\backslash\mathtt{f r a c}\{1\}\cdot\{\mathtt{n}\}^{-}2\}^{=0\Phi}$ are $\Phi\mathbf{x}{=}$ \frac{-b \pm \sqrt{b $\hat{\mathbf{\mu}}_{2+\setminus\ddag\mathbf{rac}\{4\}\cdot\{\mathbf{n}\}}\hat{\mathbf{\mu}}_{2}\hat{\mathbf{\chi}}_{\rightmoon}\{2\}\Updownarrow\Updownarrow$ . Thus the roots are real if \$b^2+\frac{4}{n !^2} \geq ${0\Phi}$ , which is always true since $\Phi\mathfrak{b}\Phi$ is real.  

0.265625  

Consider $\Phi r\left({\bf x}\right)\;=\;{\bf x}^{\hat{\sim}}\{2{\bf n}{+}2\}\;-\;{\sf q}({\bf x})\,\Phi\,.$  

0.359375  

The equation $\Phi\mathbf{p}(\backslash\mathtt{f r a c}\{1\}\{\mathbf{x}\})\ =\ \mathbf{x}\hat{\mathbf{\Omega}}2\Phi$ implies $\Phi\mathbf{x}^{\hat{\mathbf{\alpha}}}\mathbf{\{2n\}}\;\;\mathbf{p}(\backslash\mathbf{frac}\{1\}\mathbf{\{x\}})\ \mathbf{\alpha}=\ \mathbf{x}$ $\hat{\bf\Phi}^{\{2\mathrm{n}+2\}\Phi}$ , so ${\mathfrak{H}}{\mathfrak{q}}({\mathfrak{x}})~=~{\mathfrak{x}}^{\hat{\Gamma}}{\{2\mathfrak{n}+2\}}{\mathfrak{X}}$ , thus $\oint\!\mathbf{x}^{\frown}\{2\mathbf{n}{+}2\}\ -\ \mathbf{q}(\mathbf{x})\ =\ 0\Phi$ . We already know that $\Phi x=\setminus p m\;\;\mathtt{k}\Phi$ , $\Phi1$ \le $\mathtt{k}$ \le $\mathtt{n}\Phi$ , are roots.  

0.390625  

We have \$r(x) = (x^2-1)\cdots $\langle{\bf x}^{\frown}2{-}{\bf n}^{\frown}2\rangle$ )(x^2+bx-\frac{1}{n!^2})\$.  

0.359375  

Since $\Phi\mathbf{r}\left(\mathbf{x}\right)\Phi$ is an even function, $\Phi\mathbf{b}{=}0\Phi$ .  

0.6875  

The other solutions are the roots of $\Phi\mathbf{x}^{-}2$ - \frac{1}{n!^2} = 0\$, so $\Phi\mathbf{x}$ = \pm \frac{1}{n!}\$.  

0.796875  

Final Answer: The final answer is \$\boxed{\pm \frac{1}{n!}}\$  

1.0  

# Solution  

Let $q(x)=x^{2n}p\left({\frac{1}{x}}\right)=a_{0}x^{2n}+a_{1}x^{2n-1}+\cdot\cdot\cdot+a_{2n-1}x+1.$   
The given condition $\textstyle p\left({\frac{1}{k}}\right)=k^{2}$ implies $q(k)=k^{2n}k^{2}=k^{2n+2}$ for $1\leq|k|\leq n$ .   
Consider the polynomial $r(x)=x^{2n+2}-q(x)$ . We are given that $r(k)=0$ for $1\leq|k|\leq n$ . Since $n$ is even, let $n~=~2m$ . Then we have $2n\;=\;4m$ . Thus, $r(x)$ has the roots $\pm1,\pm2,\ldots,\pm2m$ .   
$\begin{array}{r}{r(\underline{{x}})\,=\,(x-\underline{{1}})(x+1)(x-2)(x+2)\cdot\cdot\cdot(x-2m)(x+2m)\,=\,\prod_{k=1}^{2m}(x-k)(x+k)\,=\,\prod_{k=1}^{2m}(1-k)^{2k}\ensuremath{\operatorname{sup}}(1-k)\cdot\ensuremath{\operatorname{sup}}(1-k).}\end{array}$ $\textstyle\prod_{k=1}^{2m}(x^{2}-k^{2})$ .   
Since $q(x)$ is a polynomial of degree $2n$ , $r(x)$ is a monic polynomial of degree $2n+2=$ $4m+2$ . Therefore, we can write $r(x)=x^{2n+2}-q(x)=(x^{2}-1)(x^{2}-4)\cdot\cdot\cdot(x^{2}-n^{2})R(x),$ , where $R(x)$ is a polynomial of degree $2n+2-2n=2$ .   
Since $q(x)$ is monic with degree $2n$ , $x^{2n+2}-q(x)$ has leading coefcient 1. Also, the leading coefcient of $(x^{2}-1)(x^{2}-4)\cdot\cdot\cdot(x^{2}-n^{2})$ is 1. Thus, $R(x)$ must be monic. Let $R(x)=x^{2}+b x+c$ . Then we have $r(x)=x^{2n+2}-q(x)=(x^{2}-1)(x^{2}-4)\cdot\cdot\cdot(x^{2}-$ $n^{2})(x^{2}+b x+c)$ .   
We are looking for values of $x$ such that $\textstyle p\left({\frac{1}{x}}\right)=x^{2}$ , which is equivalent to $q(x)=x^{2n+2}$ . This means we want to fnd the roots of $r(x)=0$ . We know that $r(x)=0$ for $x=\pm k$ for $1\leq k\leq n$ . We are looking for the roots of $x^{2}+b x+c=0$ .   
Since $r(0)=(-1)^{n}(-1)(-4)\cdot\cdot\cdot(-n^{2})c=(-1)^{n}(-1)^{n}(1)(4)\cdot\cdot\cdot(n^{2})c=n!^{2}c_{2}$ we have $r(0)=-q(0)=-1$ . Thus $n!^{2}c=-1$ , so $\begin{array}{r}{c=-\frac{1}{n!^{2}}}\end{array}$   
Since the coefcients of $p(x)$ are real, the coefcients of $q(x)$ are real, and so the coefcients of $r(x)$ are real. Therefore, the coefcients of $R(x)$ are real. In particular, $b$ ,e  rwohoitcsh  oifs $\begin{array}{r}{x^{2}+b x-\frac{1}{n!^{2}}=0}\end{array}$ rise  r𝑥ea=l. −𝑏±√2𝑏2+ 𝑛4!2 . Thus the roots are real if $\begin{array}{r}{b^{2}+\frac{4}{n!^{2}}\geq0}\end{array}$ $b$   
The equation $\textstyle p({\frac{1}{x}})=x^{2}$ implies $\begin{array}{r}{x^{2n}p(\frac{1}{x})=x^{2n+2}}\end{array}$ , s $\circ q(x)=x^{2n+2}$ , thus $x^{2n+2}\!-\!q(x)=0$ We already know that $x=\pm k$ , $1\leq k\leq n_{}$ , are roots.   
We have $\begin{array}{r}{r(x)=(x^{2}-1)\cdot\cdot\cdot(x^{2}-n^{2})(x^{2}+b x-\frac{1}{n!^{2}})}\end{array}$ .   
Since $r(x)$ is an even function, $b=0$ .   
The other solutions are the roots of $\begin{array}{r}{x^{2}-\frac{1}{n!^{2}}=0}\end{array}$ 𝑛1!2 = 0, so 𝑥 = ± 𝑛1! .   
Final Answer: The fnal answer is $\pm{\frac{1}{n!}}$  

# MCTS Generated Solution Trace Problem  

Let $n$ be an even positive integer. Let $p$ be a monic, real polynomial of degree $2n$ ;   
that is to say, $p(x)=x^{2n}+a_{2n-1}x^{2n-1}+\cdot\cdot\cdot+a_{1}x+a_{0}$ for some real coefcients 𝑎0, . . . , 𝑎2𝑛 1.   
Suppose that $\begin{array}{r}{p\left(\frac{1}{k}\right)=k^{2}}\end{array}$ for all integers $k$ such that $1\leq|k|\leq n$ .   
Find all other real numbers $x$ for which $\textstyle p\left({\frac{1}{x}}\right)=x^{2}$ .  

# Solution  

0.03125   
Let \$q(x) = p\left(\frac{1}{x}\right) - x^2\$.   
0.09375   
Let \$q(x) = p\left(\frac{1}{x}\right) - $\mathfrak{x}^{-}2\mathfrak{s}$ . Then $\Phi{\sf q}\left({\bf x}\right)\Phi$ is a rational function of the form   
0.28125   
$\begin{array}{r l}{\mathfrak{F}\mathfrak{G}\mathfrak{q}(\mathfrak{x})}&{=\;\mathrm{bf}\mathtt{r a c}\{1\;+\;\mathtt{a}_{-}\{2\mathfrak{n}^{-1}\}\mathfrak{x}\;+\;\mathrm{\bdots~+~\mathtt{a}_{-}\mathtt{1}~\mathfrak{x}^{\sim}\{2\mathfrak{n}^{-1}\}~+~\mathtt{a}_{-}\mathtt{0}~\mathfrak{x}^{\sim}\{2\mathfrak{n}\}~-~\mathfrak{x}^{\sim}\{2\mathfrak{n}\}~}\\ {\mathfrak{n}^{+2\}\}\{\mathfrak{x}^{\sim}\{2\mathfrak{n}\}\}\;.\;\mathfrak{G}\mathfrak{s}}\end{array}$ 0.3125   
We are given that $\Phi\mathrm{{p}\left(x\right)\Phi}$ is a monic real polynomial of degree $\Phi2\mathtt{n}\Phi$ . 0.0625   
$\begin{array}{r}{\mathfrak{F}\mathfrak{G}\mathfrak{q}(\mathbf{x})\;=\;\mathsf{l f}\,\mathrm{rac}\{1\;+\;\mathbf{a}_{-}\{2\mathbf{n}-1\}\mathbf{x}\;+\;\mathsf{l d o t s}\;+\;\mathbf{a}_{-}\mathbf{lx}\,\widehat{\mathbf{\xi}}\{2\mathbf{n}-1\}\,+\;\mathbf{a}_{-}0\mathbf{x}\,\widehat{\mathbf{\xi}}\{2\mathbf{n}\}\;-\;\mathbf{x}\,\widehat{\mathbf{\xi}}\{2\mathbf{n}-1\}\,+\;\mathbf{a}_{-}\mathbf{l}\,\mathsf{a}_{-}\mathbf{l}\,\mathsf{a}_{-}\widehat{\mathbf{k}}\,\widehat{\mathbf{\xi}}\mathbf{k}\;,}\\ {+2\}\,\mathrm{f}\,\mathrm{x}\,\widehat{\mathbf{\xi}}\{2\mathbf{n}\}\}\;\mathrm{.}\;\mathfrak{F}\mathfrak{F}\;\,\mathsf{G}\;\,\mathsf{G}\;\,\mathsf{m}\;\mathsf{a}_{-}\;\,\mathsf{m}\;\,\mathsf{a}_{-}\;\,\mathsf{m}\;\,\mathsf{c}\;\,\mathsf{m}\;\,\mathsf{c}\;\,\mathsf{c}\;\,\mathsf{m}\;\,\mathsf{c}\;\,\mathsf{c}\;\,\mathsf{m}\;\,\mathsf{c}\;\,\mathsf{c}\;\,\mathsf{m}\;\,\mathsf{c}\;\,\mathsf{c}\;\,\mathsf{m}\;\,\mathsf{c}\;\,\mathsf{c}\;\,\mathsf{m}\;\,\mathsf{c}\;\,\mathsf{c}\;\,\mathsf{m}\;\,\mathsf{c}\;\,\mathsf{c}\;\,\mathsf{c}\;\,\mathsf{m}\;\,\mathsf{c}\;\,\mathsf{c}\;\,\mathsf{c}\;\,\mathsf{m}\;\,\mathsf{c}\;\,\mathsf{c}\;\,\mathsf{c}\;\,\mathsf{m}\;\,\mathsf{c}\;\,\mathsf{c}\;\,\mathsf{c}\;\,\mathsf{c}\;\,\mathsf{m}\;\,\mathsf{c}\;\,\mathsf{c}\;\,\mathsf{c}\;\,\mathsf{c}\;\,\mathsf{c}\;\,\mathsf{c}\;\,\mathsf{c}\;\,\mathsf{c}\;\,\mathsf{c}\;\,\mathsf{c}\;\,\mathsf{c}\;\,\mathsf{c}\;\,\mathsf{c}\;\,\mathsf{c}\;\,\mathsf{c}\;\,\mathsf{c}\;\,\mathsf{c}\;\,\mathsf{c}\;\,\mathsf{c}\;\,\mathsf{c}\;\,\mathsf{c}\;\,\mathsf{c}\;\,\mathsf{c$ 0.25   
$\begin{array}{r l}{\mathfrak{F}\mathfrak{G}\mathfrak{q}(\mathfrak{x})\;=\;\mathrm{~\forall\,frac}\{1\;+\;\mathfrak{a}_{-}\{2\mathfrak{n}^{-1}\}\mathfrak{x}\;+\;\mathrm{~\forall\,cdots~+~\mathfrak{a}_{-}\mathrm{1}\mathfrak{x}^{\sim}}\{2\mathfrak{n}^{-1}\}\;+\;\mathfrak{a}_{-}\mathfrak{d}\mathfrak{x}^{\sim}\{2\mathfrak{n}\}\;-\;\mathrm{~\mathfrak{x}^{\sim}}\{2\mathfrak{n}^{-1}\}\;+\;\mathfrak{a}_{-}\mathfrak{d}\mathfrak{x}^{\sim}\{2\mathfrak{n}\}\;-\;\mathrm{~\mathfrak{x}^{\sim}}\{2\mathfrak{n}^{-1}\}}\\ {+2\}\}\{\mathrm{~\mathfrak{x}^{\sim}}\{2\mathfrak{n}\}\}\;=\;\mathrm{~\forall\,frac}\{\mathfrak{x}\}\{\mathfrak{x}^{\sim}\{2\mathfrak{n}\}\}\mathrm{.}\;\mathfrak{G}\mathfrak{F}}\end{array}$ 0.15625   
We are given that $\Phi\mathrm{{p}\left(x\right)\Phi}$ is a monic real polynomial of degree $\Phi2\mathtt{n}\Phi$ . 0.0625   
We are given that $\Phi\mathtt{p}\mathrm{{V}}\mathrm{{Left}\,(\backslash\{\mathtt{f}\,\mathtt{r a c}\{1\}\{k\}\backslash\mathrm{right}\}\hbar=\ k^{\star}2\Downarrow}$ for $\Phi1$ \leq |k| \ leq $\mathtt{n}\Phi$ . This means $\Phi{\sf q}({\bf k})~=~0\Phi$ for $\Phi1$ \leq |k| \leq $\mathtt{n}\Phi$ . Thus, $\Phi{\sf q}\left({\bf x}\right)\Phi$ has roots $\Phi\mathbf{\lambda}\Phi$ where $\Phi1$ \leq |k| \leq $\mathtt{n}\Phi$ . This accounts for $\Phi2\mathtt{n}\Phi$ roots.   
0.25 We have \$r(x) = \prod_{k=1}^n (x^2-k^2) ( $\mathbf{\nabla}^{-}\mathbf{x}^{-}2^{+}\mathtt{B x}^{+}\mathtt{C})\,\Phi$ . Since $\Phi\mathbf{r}\left(0\right)\ =\ 1\Phi$ , we have \$1 = \prod_{k=1}^n (-k^2) \cdot $\mathtt{C5}$ . Also, $\Phi\mathbf{C}~=~\mathsf{V r a c}\{(-1)\hat{\mathbf{\phi}}\}$ $\mathtt{n}\mathtt{J}\mathtt{\Omega}\{(\mathtt{n}\colon)\,{\hat{\textbf{\Omega}}}2\}\Phi$ .   
0.5 We have $\Phi x\left({\bf x}\right)\;=\;\backslash\mathrm{prod\!_{-}\{k=1\}\hat{\partial}n}\;\left({\bf x}\hat{\partial}\!\!\!\setminus\!\!2\!\!-\!\!\mathrm{k}\hat{\partial}\!\!\!\setminus\!\!2\right)\left(-{\bf x}\hat{\partial}\!\!\!\setminus\!\!2\!\!+\!\!\mathrm{Bx}\!\!\!+\!\!\!\mathrm{C}\right)\Phi\,.$   
0.5 In the expansion of \$\prod_ $\{\mathtt{k}{=}1\}\hat{\mathbf{\Sigma}}{\sim}\mathtt{n}$ $\langle{\bf x}^{\star}2\mathbf{-}\mathbf{k}^{\star}2\rangle$ $\langle-\mathbf{x}^{\frown}2{+}\mathbf{B}\mathbf{x}{+}\mathbf{C})\,\Phi$ , the coefficient of $\Phi\mathbf{x}^{\textstyle\frown}\{2\mathbf{n}{+}1\}\Phi$ is $\tt{\Phi}{B}$ \prod_ $\{\mathtt{k}{=}1\}\,\widehat{\mathrm{~\i~}}_{\mathtt{n}}\;\;(-\mathtt{k}\widehat{\mathrm{~\i~}}{\=}2)\;\;=\;\mathtt{B}\,(-1)\,\widehat{\mathrm{~\i~}}\mathtt{n}\;\;(\mathtt{n}\,!)$ ) $\hat{2}\Phi$ . Since this must equal 0, we have $\Phi B{=}0\Phi$ .   
1.0 In the expansion of \$\prod_ $\{\mathtt{k}{=}1\}\hat{\mathbf{\Sigma}}{\sim}\mathtt{n}$ $(\mathbf{x}^{\sim}2\mathbf{\mu}-\mathbf{\epsilon}\mathbf{k}^{\sim}2)$ $(-{\bf x}\hat{\mathrm{~\boldmath~\mu~}}2{\bf\Phi}+{\bf\Phi}{\bf B}{\bf x}{\bf\Phi}+{\bf\Phi}{\bf C})\,\Phi$ , the coefficient of $\Phi\mathbf{x}^{\textstyle\frown}\{2\mathbf{n}{+}1\}\Phi$ is $\tt{\Phi}{B}$ \prod_ $\{\mathtt{k}{=}1\}\,\widehat{\mathrm{~\i~}}_{\mathtt{n}}\;\;(-\mathtt{k}\widehat{\mathrm{~\i~}}{\=}2)\;\;=\;\mathtt{B}\,(-1)\,\widehat{\mathrm{~\i~}}\mathtt{n}\;\;(\mathtt{n}\,!$ ) ${\hat{\mathbf{\alpha}}}_{2}\ =\ (-1)\,{\mathbf{\hat{\alpha}}}_{\mathtt{n}}\ \ (\mathtt{n}\,!\ )\,{\hat{\mathbf{\alpha}}}_{2}\ \mathtt{B}\vartheta$ . Since this must be zero, we must have $\Phi\mathtt{B}\ =\ 0$ $\Phi$ .   
1.0 So, we have $\Phi\bf{r}(\bf{x})\ =\ \backslash\mathrm{{prod}\_{\bf{-}}\{\bar{\mathrm{x}}=1\}\hat{\mathrm{{~\boldmath~\mu~}}(\bf{x}\hat{\sim}2\bar{\mathrm{-}}\bar{\mathrm{k}}\hat{\Omega})\quad(-x\hat{\bar{\mathrm{~\boldmath~\mu~}}}\bf{\hat{\Omega}})\,\hat{\psi}(\bar{\mathrm{\bf{~\mu}}}\bf{x})\in\bar{\mathrm{~\bf{Virac}}}\{(-1)\hat{\bar{\mathrm{~\boldmath~\mu~}}}\bf{\hat{\Lambda}}(\mathrm{n}!)\}}\,,$ $\hat{\mathbf{\xi}}_{2\})\hat{\mathbf{\xi}}$ .   
1.0 In the expression \$r(x) = \prod_ $\{\mathtt{k}{=}1\}\hat{\mathbf{\Sigma}}{\sim}\mathtt{n}$ $\langle\mathtt{x}^{\sim}2\mathtt{\mu}-\mathtt{k}^{\sim}2\rangle$ ) ( $\mathrm{~\boldmath~\cdot~}\mathbf{x}\hat{\mathrm{~\boldmath~}}2\;+\;\mathtt{B x}\;+\;\mathtt{C})\,\Phi$ , the coefficient of $\Phi\mathbf{x}^{\textstyle\frown}\{2\mathbf{n}{+}1\}\Phi$ is $\Phi\mathtt{B}\Phi$ . Thus, $\Phi B{=}0\Phi$ .   
0.96875 Since $\Phi\mathfrak{n}\Phi$ is even, let $\Phi\mathrm{n}{=}2\mathrm{m}\Phi$ . The coefficient of $\Phi\mathbf{x}^{\textstyle\frown}\{2\mathbf{n}{+}1\}\Phi$ in $\Phi\mathbf{r}\left(\mathbf{x}\right)\Phi$ is $\Phi0\Phi$ . Also, the coefficient of ${\mathfrak{F}}{\mathfrak{X}}^{-}{\{2\mathrm{n}+1\}}{\mathfrak{F}}$ in $\Phi$ \prod_ $\{\mathtt{k}{=}1\}^{-}\{2\mathtt{m}\}$ (xk) $(x+k)$ $(-\mathtt{x}^{-}2+\mathtt{B x}+\mathtt{C})\,\mathfrak{F}$ is $\tt{\Phi}{B}$ \prod_ $\{k{=}1\}^{-}\{2\mathrm{m}\}\;\;\ k^{\mathord{\sim}2}\;=\;\textsf{B}\;\;(\mathrm{n}!)\,{\sim}\,2\;=\;\mathsf{B}\,(\,(2\mathrm{m})$ !) $\hat{2}\Phi$ .   
1.0 Since $\Phi\mathfrak{n}\Phi$ is even, let $\Phi\mathrm{n}{=}2\mathrm{m}\Phi$ . Then the roots are $\mathfrak{F}\backslash\mathtt{p m}\ 1$ , \pm 2, \ldots , $\textsf{l p m}\,2\mathtt{m}\Phi$ .   
0.75 So, $\Phi1\ =\ (-1)\,{\mathsf{\hat{\Pi}}}{\mathsf{n}}\ \left({\mathsf{n}}\,!\right)\,{\mathsf{\hat{\Pi}}}{\mathsf{2}}\ \mathsf{C}\Phi$ . Since $\Phi\mathrm{C}\ =\ \mathsf{V f r a c}\{(-1)\,\hat{\mathbf{\Pi}}_{\mathrm{n}}\}\,\{(\mathbf{n}\,!\,)\,\hat{\mathbf{\Pi}}_{2}\}\Phi$ , the equation holds. The coefficient of $\Phi\mathbf{x}\Phi$ in $\Phi\mathbf{r}\left(\mathbf{x}\right)\Phi$ is $\mathfrak{P}\mathtt{a}\_\{2\mathtt{n}-1\}\mathfrak{P}$ . Since $\mathbb{\Phi}\mathbf{r}\left(\mathbf{x}\right)\ =\ \backslash\mathtt{p r o d\_}\{\mathbf{k}{=}1\}\hat{\mathbf{\Pi}}{\mathbf{n}}$ $(\mathbf{x}^{\hat{\mathbf{\alpha}}}2\mathbf{\alpha}-\mathbf{\alpha}\mathbf{k}^{\hat{\mathbf{\alpha}}}2)$ ) $(-\mathbf{x}^{-}2+\mathsf{B x}+\mathsf{C})\,\Phi$ , the coefficient of $\Phi\mathbf{x}\Phi$ is $\Phi{\mathrm{C}}\Phi$ times the coefficient of $\Phi\mathbf{x}\Phi$ in \$\prod_{k $\mathsf{\Gamma}=\mathtt{1}\,\}\,\mathsf{\Gamma}\,\hat{\mathbf{n}}$ $(\mathtt{x}^{\sim}2\ -\ \mathtt{k}^{\sim}2)\,\mathfrak{F}$ , which is 0. Thus, the coefficient of $\Phi\mathbf{x}\Phi$ in $\Phi\mathbf{r}$ ( ${\mathfrak{x}})\,\Phi$ is 0, s $)\ \mathfrak{P}\mathtt{a\_}{\left\{2\mathrm{n-1}\right\}}\ =\ 0\mathfrak{P}$ .   
0.71875   
So, $\oplus\mathrm{B}\left(\,(2\uppi)\,!\,\right)\widehat{\sf-\,}2\;=\;0\Phi$ , which implies $\Phi B{=}0\Phi$ .   
1.0   
We also know that the coefficient of $\Phi\mathbf{x}^{\textstyle\frown}\{2\mathbf{n}{+}1\}\Phi$ in $\Phi\mathbf{r}\left(\mathbf{x}\right)\Phi$ is $\Phi0\Phi$ . The coefficient of $\Phi\mathbf{x}^{\textstyle\frown}\{2\mathbf{n}{+}1\}\Phi$ in $\mathfrak{F}$ \prod_ $\{\mathtt{k}{=}1\}\hat{\mathbf{\Sigma}}{\sim}\mathtt{n}$ ( $\mathbf{\hat{x}}^{\sim}2^{-}\mathbf{k}^{\sim}2)$ $(-\mathtt{x}^{-}2+\mathtt{B x}+\mathtt{C})\,\mathfrak{F}$ is $\Phi\mathtt{B}\Phi$ . Thus $\mathtt{\Phi}_{\Phi}=\mathtt{O}\Phi$ .   
1.0   
So, $\oplus\mathrm{B}\left(\,(2\uppi)\,!\,\right)\widehat{\sf-\,}2\;=\;0\Phi$ , which implies $\Phi B{=}0\Phi$ .   
1.0   
We have $\Phi\mathtt{r}\left(\mathtt{x}\right)\;=\;\mathsf{\mathsf{\hat{\Pi}}[p r o d\mathrm{\tt_-}\{k=1\}\hat{\Sigma}\hat{\Pi}\{n\}(\hat{\Pi}\hat{\Pi}^{-}2\mathrm{-}\hat{k}\hat{\Sigma}\hat{\Pi})\left(-\hat{x}\hat{\Pi}^{-}2\mathrm{+}\mathrm{Bx}^{+}\mathsf{C}\right)\hat{\Phi}\,.}$   
0.84375   
So we have \$r(x) = \prod_ $\{\mathtt{k}{=}1\}\hat{\mathbf{\Sigma}}{\sim}\mathtt{n}$ (x^2-k^2) (-x^2 + \frac{(-1)^n}{(n!) $\hat{\mathbf{\xi}}_{2\})\hat{\mathbf{\xi}}$ .   
1.0   
Thus, $\oplus\mathrm{B}\left(\,(2\uppi)\,!\,\right)\widehat{\sf-\,}2\;=\;0\Phi$ , which means $\Phi B~=~0\Phi$ . Therefore,   
1.0   
Since $\Phi\mathfrak{n}\Phi$ is even, $\Phi\mathbf{C}\ =\ \backslash\mathtt{f r a c}\{1\}\cdot\{(\mathtt{n}\{\mathtt{)}\cdot2\}\mathfrak{F}$ . Also, $\Phi\mathtt{A}~=~-1\Phi$ . So,   
0.71875   
The constant term of $\Phi\mathbf{r}\left(\mathbf{x}\right)\Phi$ is $\Phi1\Phi$ . Also, the constant term is given by \$\prod_ $\{\mathtt{k}{=}1\}\hat{\mathbf{\Sigma}}^{\sim}\mathtt{n}$ (-k)(k) \cdot $\textsf{C}=(-1)\,\hat{\textsf{r n}}\left(\mathtt{n}\,!\right)\hat{\textsf{\i}}2\;\;\mathbb{C}\varPhi$ . Thus, $\Phi(-1)\hat{\mathbf{\Sigma}}\hat{\mathbf{n}}$ ( $\mathtt{n!})\!\hat{\mathbf{\Sigma}}_{2}\texttt{C}=\,1\Phi$ , which implies ${\mathfrak{P C}}\ =\ {\mathsf{V f r a c}}{\{1\}}{\{\mathsf{(-1)}\,{\mathsf{\hat{\mathbf{\Pi}}}}{\mathbf{\hat{n}}}}\,$ (n!)^2} = \frac ${\{(-1)\hat{\mathbf{\alpha}}\mathbf{n}\}}\{(\mathbf{n}\!\mid\!)\hat{\mathbf{\alpha}}^{2}\}\Downarrow$ . Since $\Phi\mathfrak{n}\Phi$ is even, $\Phi\mathrm{C}\ =\ \mathsf{V f r a c}\{1\}\{\left(\mathrm{n}\,!\right)\!\sim\!2\}\Phi\,.$   
0.71875   
So, we have $\Phi\bf{r}(\bf{x})\ =\ \backslash\mathrm{{prod}\_{\bf{-}}\{\bar{\mathrm{x}}=1\}\hat{\mathrm{{~\boldmath~\mu~}}(\bf{x}\hat{\sim}2\bar{\mathrm{-}}\bar{\mathrm{k}}\hat{\Omega})\quad(-x\hat{\bar{\mathrm{~\boldmath~\mu~}}}\bf{\hat{\Omega}})\,\hat{\psi}(\bar{\mathrm{\bf{~\mu}}}\bf{x})\in\bar{\mathrm{~\bf{Virac}}}\{(-1)\hat{\bar{\mathrm{~\boldmath~\mu~}}}\bf{\hat{\Lambda}}(\mathrm{n}!)\}}\,,$ $\hat{\mathbf{\xi}}_{2\})\hat{\mathbf{\xi}}$ .   
1.0   
So we have \$r(x) = \prod_{k=1}^{n} (x^2 - k^2)(-x^2 + \frac{(-1)^n}{(n $!\,\rangle\!\-\!\,2\!\}\rangle\!\,\mathfrak{G}$ .   
1.0   
So we have \$r(x) = \prod_{k=1}^{n}(x^2-k^2)(-x^2 + \frac{(-1)^n}{(n!) $\hat{\mathbf{\xi}}_{2\})\hat{\mathbf{\xi}}$ .   
0.96875  

<html><body><table><tr><td>Since $p(1/k) = k~2$ for $1 \le Ikl \le n$， we have $q(k) = O$ for $1\ le lkl \le n$. 0.21875</td></tr><tr><td>We are given that $p\left(\frac{1}{k}\right) = k^2$ for $1 \leq |kl\ leq n$,which means $q(k) = p\left(\frac{1}{k}\right） - k^2 = O$ for $1 \leq Ikl \leq n$. Thus, $r(x)$ has roots at $x = k$ for $1 \ ‘ 91 1  5  s()   u  1 can write</td></tr><tr><td>0.25 We are given that $p\left(\frac{1}{k}\right） = k~2$ for $1 \leq lkl \ =  - (11) = ()    \ ‘ \ =    ‘ ‘  1   1g</td></tr><tr><td>ldots,\pm n$ are roots of the polynomial $r(x) = 1 + a_{2n-1}x + \ cdots + a_1x~{2n-1} + a_0x~{2n} - x~{2n+2}$. 0.3125</td></tr><tr><td>Since $r(x)$ is a polynomial of degree $2n+2$, and we know $2n$ distinct roots， we can write</td></tr><tr><td>0.1875</td></tr><tr><td>Since $r(x)$ has degree $2n+2$ and has roots $\pm 1,\pm 2,\ldots,\pm n$, there are $2n$ such roots. Thus, we can write</td></tr><tr><td>0.375 Since $p\left(\frac{1}{k}\right） = k~2$ for $1 \leq Ikl \leq n$， we</td></tr><tr><td> 1  ‘ i  l     =  x)$， which we denote by $r(x)$, has roots $k$ where $1 \leq lkl\ leq n$. Since $n$ is even, we have $2n$ such values of $k$. Also, since $p$ has real coefficients, if $k$ is a root, so is $-k$.</td></tr><tr><td>Since $r(x)$ has degree $2n+2$, we can write 0.15625</td></tr><tr><td>+  =(= x + a_{2n-2}x~2 + \dots + a_1x^{2n-1} + a_0x~{2n}$.</td></tr><tr><td>0.03125 Thus, $r(x)$ is a polynomial of degree $2n+2$ with roots $k$ for $1 \</td></tr><tr><td>leq lkl \leq n$. Since there are $2n$ such values of $k$, we can write</td></tr><tr><td>0.3125</td></tr><tr><td></td></tr><tr><td></td></tr><tr><td>We are given that $p(x)$ is a monic real polynomial of degree $2n$. 0.0625</td></tr></table></body></html>  

<html><body><table><tr><td>Since $p\left(\frac{1}{k}\right) = k2$ for all integers $k$ such that $1 \leq Ikl \leq n$,we have $q(k) = p\left(\frac{1}{k}\right） - k 2 = o$ for $1 \leq lkl \leq n$.</td></tr><tr><td>0.34375 Thus， $q(x)$ has $2n$ distinct roots $k$ where $1 \le Ikl \le n$.</td></tr><tr><td>0.15625</td></tr><tr><td>Thus, $q(x)$ has roots $k$ for $1 \leq Ikl \leq n$. These are $2n$ distinct integer roots. Since the numerator of $q(x)$ has degree $2n+2$, we can write</td></tr><tr><td>0.34375</td></tr><tr><td>Thus, $q(x)$ has roots at $k = \pm 1,^pm 2,\ldots,^pm n$. These are $2n$ distinct roots.</td></tr><tr><td>0.375</td></tr><tr><td> =      1 = \leq Ikl \leq n$.</td></tr><tr><td>0.28125</td></tr><tr><td>Thus, $q(x)$ has roots $k$ for $1 \leq Ikl \leq n$. Since $n$ is an even integer, there are $2n$ such values of $k$. We can write</td></tr><tr><td>0.125</td></tr><tr><td>Continuing the solution:</td></tr><tr><td>0.40625 Since $p$ has degree $2n$, $p(1/x)$ has degree $2n$ in $\frac{1}{x}$,</td></tr><tr><td>So when we consider $p(1/x) = x^2$, the numerator of $q(x)$ has degree $2n+2$. Thus， $q(x)$ has $2n$ zeros at $\pm 1，\pm 2,\dots, Ipm n$.</td></tr><tr><td>0.40625 }x - u}x  + {-u}x  + \ + x{-u} +  = (x) </td></tr><tr><td>+2}$. 0.34375</td></tr><tr><td>}x - u}x  + {-u}x  + \ + x{-u} +  = (x) </td></tr><tr><td>+2}$.</td></tr><tr><td>0.375</td></tr></table></body></html>  

<html><body><table><tr><td>$$q(x） =\frac{(x-1)(x+1）\dots (x-n)(x+n)(x-\alpha)(x-\beta)}{x~{2n}} $$</td></tr><tr><td>0.1875 }<x - {u}x 0e + {-u}x " + so\ + x{-u} + e\ = (x)b$$</td></tr><tr><td>n+2}}{x~{2n}} = \frac{-x~{2n+2} + a_0 x~{2n} + \dots + a_{2n-1}x + 1}{x~{2n}}.$$ 0.375</td></tr><tr><td>x - {u}x  + {-u}x  + \ + x{-u} +  = (x) )  1  1d  ()1  1   ' Pog  m  ‘u 1p o od e 1t (x)1 $2n$ distinct roots, we can write</td></tr><tr><td>0.40625</td></tr><tr><td>of $r(x)$. Thus， $r(x)$ has roots $k$ where $1 \le Ikl \le n$. 0.28125</td></tr><tr><td>Let $r(x) = 1 + a_{2n-1}x + \dots + a_1 x~{2n-1} + a_0 x^{2n} - x~{2n +2}$. Then the roots of $q(x)$ are the roots of $r(x)$. Since $r(x) $ is a polynomial of degree $2n+2$, and we know $2n$ roots, we can write</td></tr><tr><td>0.5</td></tr><tr><td> 1   g   ( ~{2n}}$, the numerator of $q(x)$ must have degree $2n+2$.</td></tr><tr><td>0.375</td></tr><tr><td>$$r(x） = (x-1)(x+1)(x-2)(x+2)\cdots(x-n)(x+n)(Ax~2 + Bx + C)$$</td></tr><tr><td>0.6875</td></tr><tr><td>$$r(x）=(x-1)(x+1)(x-2)(x+2)\cdots(x-n)(x+n)(cx~2+bx+a）=(x~2-1)(x 2-2~2)\cdots(x~2-n~2)(cx~2+bx+a）.$$</td></tr><tr><td>0.5</td></tr><tr><td>$$r(x） =(x-1)(x+1)(x-2)(x+2)\cdots(x-n)(x+n)(cx^2 + bx + a） =\prod_{k =1}~n (x~2-k~2) （cx~2+bx+a)$$</td></tr><tr><td>0.5625</td></tr><tr><td>Since $r(O) = 1$，we must have $(-1)~n (n!)~2 C = 1$，So $C = \frac</td></tr><tr><td>{1}{(n!)~2}$.</td></tr><tr><td>0.9375</td></tr></table></body></html>  

<html><body><table><tr><td>Since $r(x) = 1 + a_{2n-1}x + \dots + a_0 x~{2n} - x~{2n+2}$，the coefficient of $x~{2n+2}$ is $-1$. Also， since $r(x) = \prod_{k=1} Thus, $A = -1$.</td></tr><tr><td>0.6875 The coefficient of $x~{2n+2}$ in $r(x)$ is $-1$.Also，the coefficient of $x^{2n+2}$ in the expansion of $(x-1)(x+1)\cdots(x-n)(x+n)(Ax~2+</td></tr><tr><td>Bx+C)$ is $A$. ThuS， $A = -1$. 0.875</td></tr><tr><td>Since $r(x) = 1 + a_{2n-1}x + \dots + a_0 x~{2n} - x~{2n+2}$， the coefficient of $x~{2n+2}$ is $-1$.</td></tr><tr><td>0.53125</td></tr><tr><td>Continuing the Solution:</td></tr><tr><td>0.6875</td></tr><tr><td>The constant term of $r(x)$ is $1$. The constant term of $\prod_{k=1}~n (x-k)(x+k)(-x~2+Bx+C)$ is $(-1)~n (n!)~2 C$.Therefore，$(-1)~n (n !)~2 C = 1$，so $C = \frac{1}{(-1)~n (n!)~2} = \frac{(-1)~n}{(n!)~2} $.</td></tr><tr><td>0.65625</td></tr><tr><td>The constant term of $r(x)$ is $1$. The constant term in $\prod_{k=1}^n (x-k)(x+k)(-x~2+Bx+C)$ is $(-1)~n(n!)~2 C$.So，$1 =（-1)~n (n!) 2 C$， which implies $C =\frac{(-1)~n}{(n!)~2}$.</td></tr><tr><td>0.5625</td></tr><tr><td>The constant term of $r(x)$ is $1$. 0.5</td></tr><tr><td>The coefficient of $x~{2n}$ in $r(x)$ is $a_O$. The coefficient of $x ^{2n}$ in the factored form is $A - \sum_{k=1}~n k^2$. The coefficient of $x^{2n+1}$ in $r(x)$ is $a_1$，while in the factored</td></tr><tr><td>form,the coefficient is $B$. The coefficient of $x~{2n+2}$ in $r( x)$ is $-1$， and in the factored form， it is $A$. Therefore,$A = -1$. 0.71875</td></tr><tr><td></td></tr><tr><td>1.0</td></tr></table></body></html>  

So, we have \$r(x) = \prod_{k=1}^n (x^2 - k^2)(-x^2 + \frac{(-1)^n}{(n!) $\hat{\mathbf{\xi}}_{2\})\hat{\mathbf{\xi}}$ . Thus  

1.0  

We have $\Phi x\left({\bf x}\right)\;=\;\backslash\mathrm{prod\!_{-}\{k=1\}\hat{\partial}n}\;\left({\bf x}\hat{\partial}\!\!\!\setminus\!\!2\!\!-\!\!\mathrm{k}\hat{\partial}\!\!\!\setminus\!\!2\right)\left(-{\bf x}\hat{\partial}\!\!\!\setminus\!\!2\!\!+\!\!\mathrm{Bx}\!\!\!+\!\!\!\mathrm{C}\right)\Phi\,.$  

0.8125  

So we have $\Phi\bf{r}(\bf{x})\ =\ \backslash\mathrm{{prod}\_{\bf{-}}\{\bar{\mathrm{x}}=1\}\hat{\mathrm{{~\boldmath~\mu~}}(\bf{x}\hat{\sim}2\bar{\mathrm{-}}\bar{\mathrm{k}}\hat{\Omega})\quad(-x\hat{\bar{\mathrm{~\boldmath~\mu~}}}\bf{\hat{\Omega}})\,\hat{\psi}(\bar{\mathrm{\bf{~\mu}}}\bf{x})\in\bar{\mathrm{~\bf{Virac}}}\{(-1)\hat{\bar{\mathrm{~\boldmath~\mu~}}}\bf{\hat{\Lambda}}(\mathrm{n}!)\}}\,,$ $\hat{\mathbf{\xi}}_{2\})\hat{\mathbf{\xi}}$ .  

1.0  

Thus, we have $\Phi\mathbf{r}\left(\mathbf{x}\right)\ =\ \backslash\mathtt{p r o d\_}\{\mathbf{k}{=}1\}\hat{\mathbf{\Pi}}{\mathbf{n}}$ ( $\mathbf{\hat{x}}^{\sim}2^{-}\mathbf{k}^{\sim}2)$ (-x^2 + \frac{(-1)^n}{(n!) $\hat{\mathbf{\xi}}_{2\})\hat{\mathbf{\xi}}$ . Since $\Phi\mathfrak{n}\Phi$ is even, let $\Phi\mathrm{n}{=}2\mathrm{m}\Phi$ .  

1.0  

We have ${\mathfrak{G r}}(\mathfrak{x})\ =\ \backslash{\mathfrak{p r o d}}_{-}$ $\{\mathtt{k}{=}1\}\hat{\mathbf{\Sigma}}{\sim}\mathtt{n}$ $\langle{\bf x}^{\frown}2{-}{\bf k}^{\frown}2\rangle$ ) $(-x^{-}2+\sqrt{\pm x^{2}+(-1)^{-}n})\{(n!)^{-}2\})\nS$ . The roots of $\Phi\mathbf{r}\left(\mathbf{x}\right)\Phi$ other than $\mathfrak{F}\backslash\mathsf{p m}$ 1, \pm 2, .. , \pm n\$ satisfy $\oint\!\!-\mathbf{x}\,\hat{\mathrm{~\boldmath~\mu~}}2\ +\ \backslash\mathtt{f r a c}\{(-1)\hat{\mathrm{~\boldmath~\mu~}}\}\{(\mathtt{n}!)\hat{\mathrm{~\boldmath~\mu~}}\}=\ 0\Phi$ , so \$x^2 = \frac{(-1)^n}{(n!)^2} $\Phi$ .  

1.0  

$$
\begin{array}{r l}&{\mathfrak{F}\mathtt{r a c}(\mathtt{x})\;=\;\operatorname{brod\mathrm{\tt-}\{k=1\}\,\hat{\mathrm{\boldmath~\mu~}}(\mathtt{x}^{\star}2\;-\;\mathtt{k}^{\sim}2)\;\;(-x^{\star}2\;+\;\operatorname{lf}x\mathtt{a c}\{1\}\{(\mathtt{n}!)\wedge2\})\;\;=\;-x^{\sim}2\;\;\backslash}\\ &{\mathtt{p r o d\mathrm{\tt-}\{k=1\}\,\hat{\mathrm{\boldmath~\mu~}}(\mathtt{x}^{\star}2\;-\;\mathtt{k}^{\sim}2)\;+\;\backslash\{\mathtt{r a c}\{1\}\,\{(\mathtt{n}!)\}\,\hat{\mathrm{\boldmath~}}2\}\;\;\operatorname{sprod\mathrm{\tt-}\{k=1\}\,\hat{\mathrm{\boldmath~\mu~}}(\mathtt{x}\,\hat{\mathrm{\boldmath~}}2\;-\;\mathtt{k}^{\sim}2)\;\;\hat{\mathrm{\boldmath~\mu~}}(\mathtt{n})\}}\\ &{\;\;\;\;\;\;\;\;\;\;\;\;\;\;\;\;\;\;\;\;\;\;\;\;\;\;\;\;\;\;\;\;\;\;\;\;\;\;\;\;\;\;\;\;\;\;\;\;\;\;\;\;\;\;\;\;\;}\end{array}
$$  

1.0  

So, \$r(x) = \prod_ $\{\mathtt{k}{=}1\}\hat{\mathbf{\Sigma}}{\sim}\mathtt{n}$ $({\tt x}^{\hat{\l}}{2}{-}{\tt k}^{\hat{\l}}{2})$ ) $(-x^{-}2+\sqrt{\pm x^{2}+(-1)^{-}n})\{(n!)^{-}2\})$ . We are looking for the roots of $\Phi\mathbf{r}\left(\mathbf{x}\right)\ =\ 0\Phi$ , other than $\mathfrak{F}\backslash\mathtt{p m}\ 1$ , \pm 2, \ldots, \pm $\mathtt{n}\Phi$ . These must be the roots of $\Phi–\mathbf{x}^{-}2+\mathsf{V r a c}\{(-1)^{}\hat{\mathbf{\xi}}\mathbf{n}\}\{($ ( $\mathbf{n}\colon\right)\{2\}\ =\ 0\Phi$ , which gives us $\Phi x^{-}2=\sqrt{\bf f r a c}\{(-1)\hat{\bf\Delta}\hat{\bf\mu}\}\{({\bf n}!)\hat{\bf\Delta}^{~}2\}\Phi$ . Since $\Phi\mathbf{n}\Phi$ is an even integer, we have ${\mathfrak{F}}{\mathfrak{X}}^{\hat{-}2}\ =\ {\mathsf{V f r a c}}{\{1\}}{\{\mathfrak{f}}\,({\mathfrak{n}}!)}^{\hat{-}2\}{\mathfrak{F}}\,.$ , so \boxed $\{x=\setminus\mathtt{p m}\setminus\mathtt{f r a c}\{1\}\{\mathtt{n}:\}\}$ .  

1.0  

Let $\begin{array}{r}{q(x)=p\left(\frac{1}{x}\right)-x^{2}}\end{array}$ . Then $q(x)$ is a rational function of the form  

$$
q(x)={\frac{1+a_{2n-1}x+\cdot\cdot\cdot+a_{1}x^{2n-1}+a_{0}x^{2n}-x^{2n+2}}{x^{2n}}}.
$$  

Since $\textstyle p\left({\frac{1}{k}}\right)=k^{2}$ for all integers $k$ such that $1\leq|k|\leq n$ , we have $\begin{array}{r}{q(k)=p\left(\frac{1}{k}\right)-k^{2}=0}\end{array}$ for $1\leq|k|\leq n$ .   
Thus, $q(x)$ has roots at $k=\pm1,\pm2,\ldots,\pm n.$ . These are $2n$ distinct roots.   
Let $r(x)\,=\,1+a_{2n-1}x+\cdots+a_{1}x^{2n-1}+a_{0}x^{2n}-x^{2n+2}$ . Then the roots of $r(x)$ are precisely the roots of $q(x)$ . Since $r(x)$ is a polynomial of degree $2n+2$ , and we have found $2n$ distinct roots, we can write  

$r(x)=(x-1)(x+1)(x-2)(x+2)\cdot\cdot\cdot(x-n)(x+n)(A x^{2}+B x+C)$ Since $r(x)=1+a_{2n-1}x+\cdot\cdot\cdot+a_{0}x^{2n}-x^{2n+2}$ , the coefcient of $x^{2n+2}$ is $-1$ . Also, since $\begin{array}{r}{r(x)=\prod_{k=1}^{n}(x-k)(x+k)(A x^{2}+B x+C).}\end{array}$ , the coefcient of $x^{2n+2}$ is $A$ . Thus, $A=-1$ . The co e∏fcient of $x^{2n+2}$ in $r(x)$ is $-1$ . Also, the coefcient of $x^{2n+2}$ in the expansion of $(x-1)(x+1)\cdots(x-n)(x+n)(A x^{2}+B x+C)$ is $A$ . Thus, $A=-1$ . The constant term of $r(x)$ is 1. The constant term of $\textstyle\prod_{k=1}^{n}(x-k)(x+k)(-x^{2}+B x+C)$ is (−1)𝑛(𝑛!)2𝐶. Therefore, (−1)𝑛(𝑛!)2𝐶 = 1, so 𝐶 = ( 1)𝑛1(𝑛!)2 = ((−𝑛!1))2𝑛 . We also know that the coefcient of $x^{2n+1}$ in $r(x)$ is 0. The coefcient of $x^{2n+1}$ in $\textstyle\prod_{k=1}^{n}(x^{2}-k^{2})(-x^{2}+B x+C)$ is $B$ . Thus $B=0$ . S∏o, $\begin{array}{r}{r(x)\,=\,\prod_{k=1}^{n}(x^{2}-k^{2})(-x^{2}+\frac{(-1)^{n}}{(n!)^{2}})}\end{array}$ . We are looking for the roots of $r(x)\,=\,0_{\mathrm{.}}$ , other than $\pm1,\pm2,\ldots,\pm n$ . These must be the roots of $\begin{array}{r}{-x^{2}+\frac{(-1)^{n}}{(n!)^{2}}=0,}\end{array}$ ((−𝑛!1))2 = 0, which gives us $\begin{array}{r}{x^{2}=\frac{(-1)^{n}}{(n!)^{2}}}\end{array}$ ((−𝑛!1))2𝑛 . Since 𝑛 is an even integer, we have 𝑥2 = $\begin{array}{r}{x^{2}=\frac{1}{(n!)^{2}}}\end{array}$ (𝑛!)2 , so $x=\pm{\frac{1}{n!}}$  

# DeepSeek R1 Solution Trace Problem  

Let $n$ be an even positive integer. Let $p$ be a monic, real polynomial of degree $2n$ ;   
that is to $\mathrm{say},p(x)=x^{2n}+a_{2n-1}x^{2n-1}+\cdot\cdot\cdot+a_{1}x+a_{0}$ for some real coefcients 𝑎0, . . . , 𝑎2𝑛 1.   
Suppose that $\begin{array}{r}{p\left(\frac{1}{k}\right)=k^{2}}\end{array}$ for all integers $k$ such that $1\leq|k|\leq n$ .   
Find all other real numbers $x$ for which $\textstyle p\left({\frac{1}{x}}\right)=x^{2}$ .  

# Solution  

So i have this problem here. let’s see what it’s about. we have an even positive integer $n$ , and a monic real polynomial $p(x)$ of degree $2\mathrm{n}$ , which means it looks like this:  

$$
p(x)=x^{2n}+a_{2n-1}x^{2n-1}+\cdot\cdot\cdot+a_{1}x+a_{0}
$$  

for some real coefcients 𝑎0, 𝑎1, ..., 𝑎2𝑛 1  

the condition given is that $\textstyle p\big(\frac{1}{k}\big)\,=\,k^{2}$ for all integers $\boldsymbol{\mathrm{k}}$ such that $1\,\leq\,|k|\,\leq\,n$ . so, k takes integer values from -n to $^{-1}$ and from 1 to $\mathbf{n}$ , excluding zero because $\textstyle{\frac{1}{k}}$ would be undefned.  

okay, so frst, i need to understand what is being asked. we have a polynomial of degree $2\mathtt{n}$ , and it satisfes the equation $\textstyle p\big(\frac{1}{k}\big)=k^{2}$ for these specifc values of k. i need to fnd all real x (other than the ones already specifed, i assume) that satisfy $\textstyle p({\frac{1}{x}})=x^{2}$ . one thing that comes to mind is that since $p(x)$ is a polynomial of degree $2\Omega$ , and we have 2n conditions (because $\boldsymbol{\mathrm{k}}$ ranges from -n to $n$ , excluding zero, which gives $2\Omega$ points), this should determine the polynomial uniquely. that is, there should be only one such polynomial $p(x)$ that satisfes these conditions.  

$$
p\left({\frac{1}{k}}\right)-\left({\frac{1}{k}}\right)^{-2}=0,
$$  

let me try to think diferently. since $p(x)$ is a polynomial of degree $2n$ , and i have $2n$ points where $\begin{array}{r}{p\left({\frac{1}{k}}\right)=k^{2}}\end{array}$ , perhaps i can construct another polynomial that captures the diference between $\textstyle p\left({\frac{1}{x}}\right)$ and $x^{2}$ , and see its roots.  

$$
q(x)=p\left({\frac{1}{x}}\right)-x^{2}
$$  

i need to fnd the real roots of $q(x)$ , i.e., the real $x$ such that $q(x)=0$ . given that $\begin{array}{r}{p\left(\frac{1}{k}\right)=k^{2}}\end{array}$ for $1\leq|k|\leq n$ , that means that for each such $\begin{array}{r}{k,q(k)=p\left(\frac{1}{k}\right)\!-\!k^{2}=}\end{array}$ 0. so, $k$ are roots of $q(x)$ .  

but $q(x)$ is $\textstyle p\left({\frac{1}{x}}\right)-x^{2}$ , and $p(x)$ is a degree $2n$ polynomial, so $\textstyle p\left({\frac{1}{x}}\right)$ is a rational function, not a polynomial, unless we multiply by $x^{2n}$ to clear denominators. wait, maybe i should consider multiplying $q(x)$ by $x^{2n}$ to make it a polynomial. so, perhaps i should consider the equation $\begin{array}{r}{p\left({\frac{1}{x}}\right)-x^{2}=0}\end{array}$ and see for which real $x$ this holds, beyond the already given $x=k$ for $1\leq|k|\leq n$ . but wait, actually, in the given condition, it’s $\textstyle p\left({\frac{1}{k}}\right)\,=\,k^{2}$ , which can be rewritten as $\begin{array}{r}{p\left(\frac{1}{k}\right)-\left(\frac{1}{k}\right)^{-2}=0}\end{array}$ , since $\textstyle x^{2}=\left({\frac{1}{k}}\right)^{-2}$ when $\textstyle x={\frac{1}{k}}$ . hmm, not sure if that helps directly.  

$$
p\left({\frac{1}{k}}\right)-\left({\frac{1}{k}}\right)^{-2}=0,
$$  

let me try to think diferently. since $p(x)$ is a polynomial of degree $2n$ , and i have $2n$ points where $\begin{array}{r}{p\left({\frac{1}{k}}\right)=k^{2}}\end{array}$ , perhaps i can construct another polynomial that captures the diference between $\textstyle p\left({\frac{1}{x}}\right)$ and $x^{2}$ , and see its roots.  

$$
q(x)=p\left({\frac{1}{x}}\right)-x^{2}
$$  

$$
p\left({\frac{1}{k}}\right)-k^{2}=0
$$  

for $k=\pm1,\pm2,\ldots,\pm n$ . so, there are $2n$ such equations.   
since $p(x)$ is a monic polynomial of degree $2n$ , these $2n$ equations should determine $p(x)$ uniquely.  

<html><body><table><tr><td>but i need to find  such that p () = ², which is similar to the condition given, but for arbitrary real c, not just integer k. so, perhaps i can consider the equation p () - x2 = O, and find its real roots, given that it already has roots at  = k for 1 ≤ |kl ≤ n. but to find the other roots, i need to know more about this equation. alternatively, maybe i can consider the polynomial q(x) = p(x) -  , but again, that seems messy because of the  term. wait, perhaps i should consider multiplying both sides of the equation by x2 to eliminate the denominator. so, starting from p () = c², multiply both sides by x2 : now, x?p () can be expressed in terms of p(x), but i'm not sure. alternatively, since p(x) is a polynomial, p () is a rational function, and multiplying by α2n gives a polynomial. let me consider that again. define q(x) = x2np (±) - x2n+2, which is a polynomial of degree 2n + 2, as previously established. we know that q(k) = 0 for k = ±1, ±2,...,±n, because p(t) = k², so q(k) = k2np (t)- k2n+2 = k2n .k2 - k2n+2 = k2n+2 - k2n+2 = 0. therefore, q(x) has 2n roots at  = ±1,±2, ...,±n. since q(α) is a degree 2n + 2 polynomial, there are two more roots beyond these 2n roots. so, q(x) can be factored as: q(x) =(x²-1)(x²- 4)···(x²-n²)·(x²-α) for some constant a, because the roots come in pairs  = k and  = -k, and possibly another pair x = Va and x = -√a. wait, but q(x) is degree 2n + 2, and (x2 - k²) for k = 1 to n gives n quadratic factors, which is degree 2n. so, there are two more roots to account for. alternatively, perhaps q(x) = (x² - 1)(x² - 4) .-· (x² - n²) · (x² - a)(x² - b), where a and b are the remaining roots. but, to determine a and b, i need more information about q(). alternatively, perhaps i can look at the leading coefficient. since g(x) = x2np(±) - x2n+2, and p(±) = x-2n + a2n-1α-2n+1 + ·· + ao, then x2np (±) = 1+ a2n-12 +... + aox2n, so q(x) = 1 + a2n-1c + ... + aox2n - x2n+2. the leading term is -c2n+2 + aox2n but since p(c) is monic, ao is just a constant. wait, perhaps there's a better approach. maybe i can consider the reciprocal polynomial or something like that. alternatively, since p(xc) is a polynomial of degree 2n, and i have p (t) = k2 for k = ±l, ±2, ..., ±n, perhaps i can consider defining a new polynomial that has roots at = , but that seems off-track.        suppose i define a new variable y = , then, the equation p () = c2 becomes p(y) = c） y y, or p(y)y² = 1. so, p(y) = </td></tr></table></body></html>  

<html><body><table><tr><td>so, p(y)y2 - 1 = 0. now, p(y)y? - 1 is a polynomial of degree 2n + 2, since p(y) is degree 2n and multiplied by y2. given that p(t) = k², for k = ±l,±2,..,±n, that means p(y) =  at y = , so</td></tr><tr><td>p(y)y² - 1 = 0 at y = k· 1</td></tr><tr><td>but p(y)y2 - 1 is a degree 2n + 2 polynomial, so there are 2n + 2 roots in total (counting multiplicities).</td></tr><tr><td></td></tr><tr><td>that i need to find. let me denote these additional roots as y = Q and y = β, so:</td></tr><tr><td></td></tr><tr><td>p(y)y² - 1 = (y -α)(y-β)</td></tr><tr><td>n</td></tr><tr><td>but since p(y)y2 - 1 is a polynomial of degree 2n + 2, and we have 2n roots from y = ±, there must be two more roots, say y = α and y = β.</td></tr><tr><td>now, to find α and β, perhaps i can look at the behavior of p(y)y2 - 1 as y approaches</td></tr><tr><td>infinity. the leading term of p(y)y2 is y2n · y2 = y2n+2, and the constant term is -1, so as y -> 00, p(y)y² - 1 ≈ y2n+2 , which goes to infinity.</td></tr><tr><td>similarly, as y → 0, p(y)y² - 1 ≈ aoy² - 1, which approaches -1. but i'm not sure if that helps directly.</td></tr><tr><td>alternatively, perhaps i can consider that p(y)y2 - 1 has a double root at y = 0, but that seems unlikely.</td></tr><tr><td>wait, let's consider the multiplicity of roots.</td></tr><tr><td>y = Q and y = β, i need to determine what α and β are.</td></tr><tr><td>alternatively, perhaps i can consider expressing p(y)y2 - 1 in terms of its roots. but that seems complicated.</td></tr><tr><td>maybe there's a better way to approach this problem. let me try to consider the function p () - x2 and see if i can find a relationship between</td></tr><tr><td>r and ↓.</td></tr><tr><td>but that seems to just give back the original condition.</td></tr><tr><td>wait, perhaps i can consider the fact that p(x) is a polynomial, and consider its reciprocal</td></tr><tr><td>polynomial. the reciprocal polynomial of p(c) is x?np () , which is indeed a polynomial, as we've</td></tr><tr><td>seen earlier. let me denote this reciprocal polynomial as r(α) = x2np (), which is a degree 2n polynomial. but earlier, ihad q(x) = p(±) -x², and x2nq(α) = r(x)- x2n+2, which is a degree 2n + 2 polynomial. but perhaps focusing on r(x) directly might help. given that p (k) = k², then r(k) = k2np (t) = k2n . k² = k2n+2. 7+z = (y) os 0 = z+uz - (y)4 on p ym 0 = (y) pu  1 nq m therefore, r(k) - k2n+2 = O, meaning that r(x) - x2n+2 has roots at x = k for k = ±1,±2,...,±n.</td></tr></table></body></html>  

but $r(x)$ is a degree $2n$ polynomial, and $x^{2n+2}$ is a monomial of degree $2n+2$ , so $r(x)-x^{2n+2}$ is a degree $2n+2$ polynomial with $2n$ roots at $x=\pm1,\pm2,\ldots,\pm n$ , and two additional roots.  

wait, but $r(x)$ is degree $2n$ , and $x^{2n+2}$ is degree $2n+2$ , so their diference is degree $2n+2$ , as previously established.   
but if $r(x)\!-\!x^{2n+2}$ has $2n$ roots at $x=\pm1,\pm2,\ldots,\pm n$ , and it’s a degree $2n\!+\!2$ polynomial, then there are two more roots.   
now, perhaps i can factor $r(x)-x^{2n+2}$ as:   
$r(x)-x^{2n+2}=(x^{2}-1)(x^{2}-4)\cdot\cdot\cdot(x^{2}-n^{2})\cdot(x^{2}-a)$   
for some constant $a$ , since the roots come in pairs $x=\pm k$ for $k=1$ to $n$ , and possibly another pair $x=\pm{\sqrt{a}}$ .   
but i need to fnd $a$ .   
alternatively, perhaps i can look at the behavior of $r(x)\ –\ x^{2n+2}$ at specifc points to determine $a$ .   
but this seems complicated. maybe there’s a diferent approach.   
let me consider that $p(x)$ is a monic polynomial of degree $2n$ , and $\textstyle p\left({\frac{1}{k}}\right)\;=\;k^{2}$ for $k=\pm1,\pm2,\ldots,\pm n$ .   
perhaps i can consider the polynomial $\textstyle p(x)-{\frac{1}{x^{2}}}$ , but as before, that introduces a rational function.   
alternatively, maybe i can consider building $p(x)$ explicitly.   
since $p(x)$ is monic of degree $2n$ , and it satisfes $\textstyle p\left({\frac{1}{k}}\right)\,=\,k^{2}$ for $k=\pm1,\pm2,\ldots,\pm n$ , perhaps i can write $p(x)$ in terms of these roots.   
wait, but $\begin{array}{r}{p\left(\frac{1}{k}\right)=k^{2}}\end{array}$ , which is like saying $\begin{array}{r}{p\left(\frac{1}{k}\right)-k^{2}=0}\end{array}$ , meaning that $\textstyle{\frac{1}{k}}$ are roots of the equation $\begin{array}{r}{p(x)-\frac{1}{x^{2}}=0}\end{array}$ , but again, that’s a rational function.   
alternatively, perhaps i can consider that $\textstyle p(x)\,-\,{\frac{1}{x^{2}}}$ has roots at $\scriptstyle x\ =\ {\frac{1}{k}}$ for $k\ =$ $\pm1,\pm2,\ldots,\pm n$ , and try to factor it accordingly.   
but this seems tricky because of the $\textstyle{\frac{1}{x^{2}}}$ term.   
maybe a better approach is to consider the polynomial $x^{2}p(x)-1$ , which is a degree $2n+2$ polynomial.   
given that $\textstyle p\left({\frac{1}{k}}\right)\;=\;k^{2}$ , then $x^{2}p(x)\mathrm{~-~}1$ evaluated at $\scriptstyle x\ =\ {\frac{1}{k}}$ gives $\begin{array}{r l r}{\left(\frac{1}{k}\right)^{2}p\left(\frac{1}{k}\right)\,-\,1\,=}&{{}}\end{array}$ $\begin{array}{r}{\frac{1}{k^{2}}\cdot k^{2}-1=\overset{\cdot}{1}-1=0}\end{array}$ , so $\textstyle x={\frac{1}{k}}$ are roots of $x^{2}p(x)-1=0$ .   
therefore, $x^{2}p(x)-1$ is a degree $2n+2$ polynomial with roots at $x=\pm\textstyle{\frac{1}{1}},\pm\textstyle{\frac{1}{2}},\ldots,\pm\textstyle{\frac{1}{n}}$ , and possibly two more roots.   
wait, but $x^{2}p(x)-1$ is a polynomial, and its roots are the solutions to $x^{2}p(x)=1$ , which is similar to the original equation $\textstyle p\left({\frac{1}{x}}\right)=x^{2}$ .   
so, perhaps i can consider the roots of $x^{2}p(x)\!-\!1=0$ , which are the same as the solutions to $\textstyle p\left({\frac{1}{x}}\right)=x^{2}$ due to the substitution $\textstyle y={\frac{1}{x}}$   
therefore, fnding the roots of ${x^{2}}p(x)\mathrm{~-~}1\;=\;0$ is equivalent to fnding $x$ such that $\textstyle p\left({\frac{1}{x}}\right)=x^{2}$ .   
given that, and knowing that $p(x)$ is a monic polynomial of degree $2n$ , and that $\textstyle p\left({\frac{1}{k}}\right)=k^{2}$ for $k=\pm1,\pm2,\ldots,\pm n$ , perhaps i can fnd a relationship between $p(x)$ and these roots. alternatively, perhaps i can consider the lagrange interpolation formula for $p(x)$ , given its values at specifc points.   
since $p(x)$ is a monic polynomial of degree $2n$ , and i have its values at $2n$ points, i can indeed fnd $p(x)$ explicitly using interpolation, but that seems complicated for this problem. moreover, the problem is not asking for $p(x)$ , but for the real numbers $x$ that satisfy $\textstyle p\left({\frac{1}{x}}\right)=x^{2}$ , beyond the given $x=k$ for $k=\pm1,\pm2,\ldots,\pm n.$ .   
so, perhaps there is a symmetry or a particular property of $p(x)$ that i can exploit. wait, perhaps i can consider that $p(x)$ and $\textstyle p\left({\frac{1}{x}}\right)$ are related in a specifc way.   
given that $p(x)$ is a polynomial, $\textstyle p\left({\frac{1}{x}}\right)$ is a rational function, and multiplying by $x^{2n}$ gives a polynomial.   
alternatively, perhaps there is a functional equation that $p(x)$ satisfes.   
let me consider writing $p(x)$ in terms of its roots.   
but $p(x)$ is a monic polynomial of degree $2n$ , so it can be written as:   
$p(x)=(x-r_{1})(x-r_{2})\cdot\cdot\cdot(x-r_{2n})$   
for some roots 𝑟1, 𝑟2, . . . , 𝑟2𝑛.   
but i don’t know these roots, so that might not help directly.   
alternatively, perhaps i can consider the fact that $p(x)$ has certain symmetries given the condition $\begin{array}{r}{p\left(\frac{1}{k}\right)=k^{2}}\end{array}$ .   
wait, maybe i can consider the function $\textstyle p(x)-{\frac{1}{x^{2}}}$ , but again, that introduces a rational function.   
alternatively, perhaps i can consider the derivative of $p(x)$ , but i don’t see an immediate connection.   
let me try to think diferently.   
suppose i defne a new function $\begin{array}{r}{f(x)=p\left(\frac{1}{x}\right)-x^{2}}\end{array}$ , as i did earlier.   
then, $f(x)=0$ for $x=\pm1,\pm2,\ldots,\pm n$ , as given.   
i need to fnd all real $x$ such that $f(x)=0$ , beyond these known roots.   
now, $f(x)$ is a rational function, but $x^{2n}f(x)$ is a polynomial of degree $2n\,+\,2$ , as established earlier.   
therefore, $x^{2n}f(x)=0$ has roots at $x=\pm1,\pm2,\ldots,\pm n$ , and possibly two more roots. so, if i can fnd this polynomial $x^{2n}f(x)$ , i can factor it and fnd the additional roots. but to fnd $x^{2n}f(x)$ , i need to know more about $p(x)$ , which i don’t.   
alternatively, perhaps i can consider the behavior of $f(x)$ as $x$ approaches infnity or zero.   
as $x\to\infty$ , $\textstyle p\left({\frac{1}{x}}\right)$ behaves like $\left({\frac{1}{x}}\right)^{2n}$ , which approaches 0, while $x^{2}$ approaches infnity, so $f(x)\rightarrow-\infty$ .   
as $\begin{array}{r}{x\rightarrow0,\,p\left(\frac{1}{x}\right)}\end{array}$ behaves like $\left({\frac{1}{x}}\right)^{2n}$ , which goes to infnity, while $x^{2}$ approaches 0, so $f(x)\rightarrow\infty$ .   
this suggests that there are additional roots beyond the given $x\,=\,\pm1,\pm2,\ldots,\pm n$ , possibly at infnity or near zero, but i need to fnd them explicitly.   
wait, perhaps there’s a smarter way to approach this.   
given that $p(x)$ is a monic polynomial of degree $2n$ , and it satisfes $\textstyle p\left({\frac{1}{k}}\right)\;=\;k^{2}$ for $k=\pm1,\pm2,\ldots,\pm n$ , perhaps i can consider the minimal polynomial that satisfes these conditions.   
alternatively, maybe i can consider that $p(x)$ can be expressed in terms of a product involving these roots.   
wait, perhaps i can consider that $\textstyle p(x)-{\frac{1}{x^{2}}}$ has roots at $x=\pm\textstyle{\frac{1}{1}},\pm\textstyle{\frac{1}{2}},\ldots,\pm\textstyle{\frac{1}{n}}$ , but again, that introduces a rational function.   
alternatively, perhaps i can consider that $x^{2}p(x)-1$ has roots at $x=\pm\textstyle{\frac{1}{1}},\pm\textstyle{\frac{1}{2}},\ldots,\pm\textstyle{\frac{1}{n}}$ , as previously established, and try to fnd a general form for this polynomial.   
let me consider that $x^{2}p(x)\mathrm{~-~}1$ is a degree $2n\,+\,2$ polynomial with roots at $x=$ $\pm{\frac{1}{1}},\pm{\frac{1}{2}},\ldots,\pm{\frac{1}{n}}$ , and two additional roots.  

given that, perhaps i can express $x^{2}p(x)-1$ as:  

$$
x^{2}p(x)-1=\prod_{k=1}^{n}\left(x^{2}-{\frac{1}{k^{2}}}\right)\cdot(x^{2}-a)
$$  

for some constant $a$ , since the roots come in pairs $x=\pm{\frac{1}{k}}$ and possibly $x=\pm{\sqrt{a}}$ . but i need to determine $a$ .   
alternatively, perhaps i can consider that the product $\textstyle\prod_{k=1}^{n}\left(x^{2}-{\frac{1}{k^{2}}}\right)$ is a degree $2n$ polynomial, and $x^{2}p(x)-1$ is degree $2n+2$ , so there a re two more roots.   
but without knowing more about $p(x)$ , it’s hard to pin down $a$ .   
wait, perhaps i can consider the behavior at $x=0$ .   
at $x=0$ , $x^{2}p(x)-1=-1$ , so $x=0$ is not a root.   
similarly, as $x\to\infty$ , $x^{2}p(x)-1$ behaves like $x^{2n+2}$ , which goes to infnity.   
but i still don’t see how to fnd $a$ .   
alternatively, perhaps i can consider the value of $x^{2}p(x)-1$ at a specifc point, say $x=1$ , but that’s already a root.   
wait, perhaps i can consider $x=\infty$ , but that’s not helpful.   
alternatively, perhaps i can consider that $p(x)$ is determined by its values at $2n$ points, so i can use interpolation to fnd $p(x)$ , but that seems too involved for this problem. alternatively, perhaps there’s a symmetry in the equation that i can exploit.   
given that $p(x)$ is a polynomial of even degree $2n$ , and the conditions are given for $k=\pm1,\pm2,\ldots,\pm n$ , perhaps $p(x)$ has some symmetry properties.   
for example, maybe $p(x)$ is an even function, but i don’t know that for sure.   
wait, let’s check: if $p(x)$ were even, then $p(-x)=p(x)$ , but the conditions are given for both positive and negative $k$ , so perhaps that’s a clue.   
given that $\begin{array}{r}{p\left(\frac{1}{k}\right)=k^{2}}\end{array}$ for both $k$ and $-k$ , and $k^{2}$ is the same for both, it suggests that $p(x)$ might be even, but i need to confrm.   
however, $p(x)$ is a general monic polynomial of degree $2n$ , and its coefcients could make it even or not.   
but in this case, since $\begin{array}{r}{p\left(\frac{1}{k}\right)=p\left(\frac{1}{-k}\right)=k^{2}}\end{array}$ , which is the same, it suggests that $\textstyle p\left({\frac{1}{x}}\right)$ is even in $x$ , but $p(x)$ itself doesn’t have to be even.   
alternatively, perhaps i can consider that $p(x)$ has a reciprocal property, given the conditions involving $\textstyle{\frac{1}{k}}$ .   
wait, perhaps i should consider the reciprocal polynomial of $p(x)$ , which is $x^{2n}p\left({\frac{1}{x}}\right)$ as before.   
let me denote this as $\begin{array}{r}{r(x)=x^{2n}p\left(\frac{1}{x}\right)}\end{array}$ , which is a degree $2n$ polynomial.   
given that, and knowing that $\textstyle p\left({\frac{1}{k}}\right)=k^{2}$ , then $\begin{array}{r}{r(k)=k^{2n}p\left(\frac{1}{k}\right)=k^{2n}\cdot k^{2}=k^{2n+2},}\end{array}$ . therefore, $r(k)=k^{2n+2}$ for $k=\pm1,\pm2,\ldots,\pm n.$ .   
now, since $r(x)$ is a degree $2n$ polynomial, and it equals $x^{2n+2}$ at $2n$ points, which is more points than the degree, unless there’s some dependency.   
wait, but $r(x)$ is degree $2n$ , and $x^{2n+2}$ is degree $2n+2$ , so they can’t be equal at more points than their degree diference unless they are the same polynomial, which they aren’t.   
this suggests that my earlier assumption is incorrect, or perhaps i need to consider that $r(x)-x^{2n+2}$ is a polynomial of degree $2n+2$ , but that seems inconsistent because $r(x)$ is degree $2\mathrm{n}$ and $x^{2n+2}$ is degree $2n+2$ , so their diference should be degree $2n+2$ . wait, perhaps i should correct that.   
given $\begin{array}{r}{r(x)=x^{2n}p\left(\frac{1}{x}\right)}\end{array}$ , which is degree $2n$ , and $x^{2n+2}$ is degree $2n+2$ , their diference $r(x)-x^{2n+2}$ is deg(re)e $2n+2$ , as the highest degree term comes from $-x^{2n+2}$ .  

given that $r(k)=k^{2n+2}$ for $k=\pm1,\pm2,\ldots,\pm n$ , then $r(x)-x^{2n+2}=0$ at these points. therefore, $r(x)-x^{2n+2}$ has $2n$ roots at $x=\pm1,\pm2,\ldots,\pm n$ , and since it’s degree $2n+2$ , there are two more roots.  

now, perhaps i can factor $r(x)-x^{2n+2}$ as:  

$$
r(x)-x^{2n+2}=(x^{2}-1)(x^{2}-4)\cdot\cdot\cdot(x^{2}-n^{2})\cdot(x^{2}-a)
$$  

for some constant $a$ , since the roots come in pairs $x=\pm k$ , and possibly another pair $x=\pm{\sqrt{a}}$ .   
now, to fnd $a$ , perhaps i can consider the behavior at infnity or use the fact that the leading coefcients must match.   
given that $\begin{array}{r}{r(x)=x^{2n}p\left(\frac{1}{x}\right)}\end{array}$ and $p(x)$ is monic of degree $2n$ , the leading term of $p(x)$ is $x^{2n}$ , so $\begin{array}{r}{p\left(\frac{1}{x}\right)=x^{-2n}+\cdots}\end{array}$ , and thus $r(x)=x^{2n}\cdot x^{-2n}+\cdot\cdot\cdot=1+\cdot\cdot\cdot$ , so the leading term of $r(x)$ is 1.   
on the other hand, $x^{2n+2}$ has leading term $x^{2n+2}$ , so $r(x)\ –\ x^{2n+2}$ has leading term 𝑥2𝑛+2.   
now, the right-hand side, $(x^{2}-1)(x^{2}-4)\cdot\cdot\cdot(x^{2}-n^{2})(x^{2}-a)$ , expands to $x^{2n+2}-(1+$ $4+\cdots+n^{2}+a)x^{2n}+\cdots$   
wait, actually, the product $(x^{2}-1)(x^{2}-4)\cdot\cdot\cdot(x^{2}-n^{2})(x^{2}-a)$ would have leading term $x^{2n+2}$ , and the coefcient of $x^{2n}$ would be $-(1+4+\cdot\cdot\cdot+n^{2}+a)$ .   
but $r(x)-x^{2n+2}$ has leading term $-x^{2n+2}$ , and the coefcient of $x^{2n}$ would be something else, depending on   
on the other hand, $x^{2n+2}$ has leading term $x^{2n+2}$ , so $r(x)\ –\ x^{2n+2}$ has leading term $-x^{2n+2}$ .   
now, the right-hand side, $(x^{2}-1)(x^{2}-4)\cdot\cdot\cdot(x^{2}-n^{2})(x^{2}-a)$ , expands to $x^{2n+2}-(1+$ $4+\cdot\cdot\cdot+n^{2}+a)x^{2n}+\cdot\cdot\cdot$   
wait, actually, the product $(x^{2}-1)(x^{2}-4)\cdot\cdot\cdot(x^{2}-n^{2})(x^{2}-a)$ would have leading term $x^{2n+2}$ , and the coefcient of $x^{2n}$ would be $-(1+4+\cdot\cdot\cdot+n^{2}+a)$ .   
but $r(x)-x^{2n+2}$ has leading term $-x^{2n+2}$ , and the coefcient of $x^{2n}$ would be something else, depending on the expansion of $r(x)$ .   
therefore, to match the leading coefcients, i must have:  

$$
-x^{2n+2}=x^{2n+2}-(1+4+\cdot\cdot\cdot+n^{2}+a)x^{2n}+\cdot\cdot\cdot
$$  

this implies that:  

$$
-x^{2n+2}=x^{2n+2}-(1+4+\cdot\cdot\cdot+n^{2}+a)x^{2n}+\cdot\cdot\cdot
$$  

comparing coefcients, the coefcient of $x^{2n+2}$ on the left is $-1$ , and on the right is 1.   
this suggests that there’s an inconsistency unless i made a mistake in my earlier steps.   
wait, perhaps i need to reconsider the factorization.  

given that ${\bar{r}}(x)-x^{2n+2}=(x^{2}-1)(x^{2}-4)\cdot\cdot\cdot(x^{2}-n^{2})(x^{2}-a)$ , and the leading term on the left is $-x^{2n+2}$ , while on the right, it’s $x^{2n+2}$ , this suggests that there is a sign discrepancy.  

therefore, perhaps the correct factorization should include a negative sign:  

$$
r(x)-x^{2n+2}=-(x^{2}-1)(x^{2}-4)\cdot\cdot\cdot(x^{2}-n^{2})(x^{2}-a)
$$  

this way, the leading coefcient matches: $-x^{2n+2}=-x^{2n+2}$ .  

now, to fnd $a$ , perhaps i can consider the constant term.  

the constant term of $r(x)-x^{2n+2}$ is $r(0)-0=r(0)$ .  

but $\begin{array}{r}{r(x)=x^{2n}p\left(\frac{1}{x}\right)}\end{array}$ , so $\begin{array}{r}{r(0)=\operatorname*{lim}_{x\to0}x^{2n}p\left(\frac{1}{x}\right)}\end{array}$ .   
but $\textstyle p\left({\frac{1}{x}}\right)$ behaves like $\left({\frac{1}{x}}\right)^{2n}$ as $x\to0$ , so 1 2𝑛 = 1, so 𝑟(0) = 1.   
on the other hand, the constant term of $-(x^{\tilde{2}^{*}}-1)(x^{2}\,-\,4)\cdot\cdot\cdot(x^{2}\,-\,n^{2})(x^{2}\,-\,a)$ is $-(-1)^{n}\cdot1\cdot4\cdot9\cdot\cdot\cdot n^{2}\cdot a.$ .   
wait, actually, expanding the product:   
$(x^{2}-1)(x^{2}-4)\cdot\cdot\cdot(x^{2}-\bar{n}^{2})(x^{\bar{2}}-a)=x^{2n+2}-(1+4+\cdot\cdot\cdot+n^{2}+a)x^{2n}+\cdot\cdot\cdot+(-1)^{n+1}(1+\cdot\cdot\cdot+(-1)^{2n})$ $4\cdot9\cdot\cdot\cdot n^{2}\cdot a)$   
therefore, the constant term is $(-1)^{n+1}(1\cdot4\cdot9\cdot\cdot\cdot n^{2}\cdot a)$ .   
but since $r(x)-x^{2n+2}$ has constant term $r(0)=1$ , and the right-hand side has constant term $-(-1)^{n+1}(1\cdot4\cdot9\cdot\cdot\cdot n^{2}\cdot a)=(-1)^{n}(1\cdot4\cdot9\cdot\cdot\cdot n^{2}\cdot a)$ , we have:  

$$
(-1)^{n}(1\cdot4\cdot9\cdot\cdot\cdot n^{2}\cdot a)=1
$$  

therefore:  

$$
a={\frac{1}{(-1)^{n}(1\cdot4\cdot9\cdot\cdot\cdot n^{2})}}
$$  

but $1\cdot4\cdot9\cdot\cdot\cdot n^{2}=(n!)^{2}$ , since $1=1^{2},4=2^{2},9=3^{2}$ , etc., up to $n^{2}$ . therefore:  

$$
a={\frac{1}{(-1)^{n}(n!)^{2}}}
$$  

but since $a$ is a real number, and $n$ is even (as given in the problem), because $n$ is an even positive integer, $(-1)^{n}=1$ , so:  

$$
a=\frac{1}{(n!)^{2}}
$$  

therefore, the factorization is:  

$$
r(x)-x^{2n+2}=-\left(x^{2}-1\right)\left(x^{2}-4\right)\cdot\cdot\cdot\left(x^{2}-n^{2}\right)\left(x^{2}-{\frac{1}{(n!)^{2}}}\right)
$$  

now, since $\begin{array}{r}{r(x)=x^{2n}p\left(\frac{1}{x}\right)}\end{array}$ , and $r(x)-x^{2n+2}=0$ when $\textstyle p\left({\frac{1}{x}}\right)=x^{2}$ , the roots of this equation are $x=\pm1,\pm2,\ldots,\pm n,\pm\frac{1}{n!}$ .   
wait, no. actually, from the factorization, the roots are $x=\pm1,\pm2,\ldots,\pm n,\pm\frac{1}{n!}$ because $\begin{array}{r}{x^{2}-\frac{1}{(n!)^{2}}=0}\end{array}$ (𝑛1!)2 = 0 gives  

$$
x=\pm{\frac{1}{n!}}.
$$  

therefore, the solutions to $\textstyle p\left({\frac{1}{x}}\right)=x^{2}$ are $x=\pm1,\pm2,\ldots,\pm n,\pm\frac{1}{n!}$ . but the problem states that $\begin{array}{r}{p\left(\frac{1}{k}\right)=k^{2}}\end{array}$ for $1\leq|k|\leq n$ , so these $x=\pm1,\pm2,\ldots,\pm n$ are already known solutions.  

therefore, the "other real numbers" $x$ that satisfy $\textstyle p\left({\frac{1}{x}}\right)=x^{2}$ are $\textstyle x=\pm{\frac{1}{n!}}$ wait, but in the factorization, i have  

$$
x^{2}-{\frac{1}{(n!)^{2}}},
$$  

which gives $x=\pm\frac{1}{n!}$ , assuming that’s correct.   
but i need to verify this conclusion.  

given that $\begin{array}{r}{a=\frac{1}{(n!)^{2}}}\end{array}$ and the factorization is  

$$
r(x)-x^{2n+2}=-\left(x^{2}-1\right)\left(x^{2}-4\right)\cdot\cdot\cdot\left(x^{2}-n^{2}\right)\left(x^{2}-{\frac{1}{(n!)^{2}}}\right),
$$  

then indeed, the roots are $x=\pm1,\pm2,\ldots,\pm n,\pm\frac{1}{n!}$ . therefore, the additional solutions beyond $x=\pm1,\pm2,\ldots,\pm n$ are  

$$
x=\pm{\frac{1}{n!}}.
$$  

but wait, is $n$ ! correct here? let me double-check the calculation of $a$ . earlier, i had:  

$$
a={\frac{1}{(-1)^{n}(1\cdot4\cdot9\cdot\cdot\cdot n^{2})}}={\frac{1}{(n!)^{2}}}
$$  

since 𝑛 is even, (−1)𝑛 = 1, so indeed, 𝑎 = (𝑛1!)2 .   
therefore, the additional roots are $x=\pm{\frac{1}{n!}}$   
hence, the other real numbers $x$ that satisfy $\textstyle p\left({\frac{1}{x}}\right)=x^{2}$ are $x=\pm{\frac{1}{n!}}$ \*\*fnal answer\*\*  

$$
x=\pm{\frac{1}{n!}}
$$  

Table 5: Regret Statistics by Model and Prompting Strategy   


<html><body><table><tr><td>Model</td><td>Prompting Strategy</td><td>Regret Count</td><td>Regret %</td></tr><tr><td rowspan="5">Llama 3.1 405B</td><td>Baseline</td><td>6</td><td>0.24</td></tr><tr><td>CoT</td><td>8</td><td>0.32</td></tr><tr><td>Think</td><td>87</td><td>3.49</td></tr><tr><td>Think (3-shot)</td><td>216</td><td>8.69</td></tr><tr><td>Think & Verify</td><td>391</td><td>15.67</td></tr><tr><td rowspan="5">Llama 3.1 8B</td><td>Baseline</td><td>7</td><td>0.30</td></tr><tr><td>CoT</td><td>12</td><td>0.50</td></tr><tr><td>Think</td><td>103</td><td>4.51</td></tr><tr><td>Think (3-shot)</td><td>103</td><td>4.28</td></tr><tr><td>Think & Verify</td><td>454</td><td>20.87</td></tr><tr><td rowspan="5">Llama 3.1 70B</td><td>Baseline</td><td>5</td><td>0.20</td></tr><tr><td>CoT</td><td>6</td><td>0.25</td></tr><tr><td>Think</td><td>313</td><td>12.65</td></tr><tr><td>Think (3-shot)</td><td>231</td><td>9.33</td></tr><tr><td>Think & Verify</td><td>638</td><td>25.67</td></tr><tr><td rowspan="5">GPT-4o-mini</td><td>Baseline</td><td>1</td><td>0.04</td></tr><tr><td>CoT</td><td>2</td><td>0.08</td></tr><tr><td>Think</td><td>66</td><td>2.64</td></tr><tr><td>Think (3-shot)</td><td>53</td><td>2.12</td></tr><tr><td>Think & Verify</td><td>62</td><td>2.48</td></tr><tr><td rowspan="5">GPT-40</td><td>Baseline</td><td>0</td><td>0.00</td></tr><tr><td>CoT</td><td>0</td><td>0.00</td></tr><tr><td>Think</td><td>34</td><td>1.36</td></tr><tr><td>Think (3-shot)</td><td>28</td><td>1.12</td></tr><tr><td>Think & Verify</td><td>38</td><td>1.52</td></tr><tr><td rowspan="5">GPT-3.5-turbo</td><td>Baseline</td><td>1</td><td>0.04</td></tr><tr><td>CoT</td><td>2</td><td>0.08</td></tr><tr><td>Think</td><td>24</td><td>0.96</td></tr><tr><td>Think (3-shot)</td><td>25</td><td>1.00</td></tr><tr><td>Think & Verify</td><td>12</td><td>0.48</td></tr><tr><td rowspan="5">Claude 3.5 Sonnet</td><td>Baseline</td><td>0</td><td>0.00</td></tr><tr><td>CoT</td><td>3</td><td>0.12</td></tr><tr><td>Think</td><td>81</td><td>3.24</td></tr><tr><td>Think (3-shot)</td><td>99</td><td>3.96</td></tr><tr><td>Think & Verify</td><td>111</td><td>4.44</td></tr><tr><td rowspan="5">Claude 3.5 Haiku</td><td>Baseline</td><td>0</td><td>0.00</td></tr><tr><td>CoT</td><td>1</td><td>0.04</td></tr><tr><td>Think</td><td>16</td><td>0.64</td></tr><tr><td>Think (3-shot)</td><td>16</td><td>0.64</td></tr><tr><td>Think & Verify</td><td>4</td><td>0.16</td></tr></table></body></html>  