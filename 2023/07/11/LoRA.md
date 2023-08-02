# LoRA - Note

LoRA: Low-Rank Adaption of Large Language Models

**Full fine-tuning**: updates all the parameters of the pre-trained model, increasingly expensive for LLMs like GPT-3 with 175B parameters.
For a pretrained model $P_{\Phi}(y|x)$ and a dataset =$\mathcal{Z}{(x_i,y_i)}_{i=1,\dots,N}$ (where $x_i$ and $y_i$ are sequences of tokens), full fine-tuing aims to update initialized weights $\Phi_0$ into $\Phi_0+\triangle\Phi$, written as:
$$
max_{\Phi} \sum_{(x,y)\subset\mathcal{Z}} \sum_{t=1}^{|y|}log(P_\Phi(y_t|x,y_{<t}))
$$


——> Solutions
- update only part of the parameters e.g. Bias-only or BitFit

- add adapter layers e.g. Adapter tuning

  ==compute $|\Theta|$==
  
----------------
$$
  |\Theta|=\hat L_{Adapt}\times(2\times d_{model}\times r+r+d_{model})+2\times \hat L_{LN}\times d_{model},\\
  \hat L_{Adapt}= \#\quad of\quad adapter\quad layers,\\
  \hat L_{LN}= \#\quad of\quad traniable\quad LayerNorms
$$

-------------
- optimize some forms of the input layer/embedding layer activations

- [ ] ==prefix-tuning==: e.g. PreEmbed, PreLayer

- directly optimize prompts - hard

**Challenges to fine-tuning**
1) introduce inference latency
2) extend model depth
3) reduce available sequence length
4) pose a trade-off between efficiency and model quality
#### Inspiration
Previous papers show that the <u>learned over-parametrized models</u> in fact reside on a low intrinsic dimension and can still learn efficiently despite a random projection to a smaller subspace.
(which means that pruning and distilling are intrinsicly feasible and promising for LLMs?)

——> Hypothesize that <u>weights update during model adaption</u> also has a "low intrinsic rank".

[^low intrinsic dimension]: Measuring the Intrinsic Dimension of Objective Langscapes; Intrinsic Dimensionality Explains the Effectiveness of Language Model Fine-Tuning;

#### Main Idea
Keep the pre-trained weights frozen.
The pre-trained model could be shared and used to build&optimize many small LoRA modules across different tasks.
These LoRA moduls are in fact low-rank matrices used to "encode" these parameters, thus indirectly optimizing them.

i.e. only a much smaller-sized set of parameters $\Theta$ needs to be updated and $|\Theta|\ll|\Phi_0|$ (0.01%). This task is written as:
$$
max_{\Theta} \sum_{(x,y)\subset\mathcal{Z}}\sum^{|y|}_{t=1}log(P_{\Phi_0+\triangle\Phi(\Theta)}(y_t|x,y_{<t}))
$$

#### Method
1. Frozen weight matrix  $W_0\in\mathbb{R}^{d\times k}$

Represent weight updates using low-rank decomposition $W_0+\triangle W=W_0+BA$, where $B\in\mathbb R^{d\times r}, A\in\mathbb R^{r\times k},r\ll min(d,k)$ 

Output are summed coordinate-wise, so modified forward pass yields:
$$
h=W_0x+\triangle Wx=W_0x+BAx
$$
2. Initialization: $A=\mathcal N(0,\sigma^2), B=0$,  ==scale $\triangle Wx$ by $\frac{\alpha}{r}$==
(Scaling reduces the need to retune hyperparameters when varying r)
Optimize: with Adam
3. Applying to Transformers: 
4 weight matrices in the self-attention module ($W_k, W_q, W_v, W_o$) and 2 in the MLP module. This paper **freezes the MLP module and only adapts the attention weigths**.
#### Analysis
**Key functional difference** The learned weights A, B can be merged with the frozen pre-trained weights during inference, thus introducing no latency.
**A Generalization of Full Fine-tuning**:==When increasing # of trainable parameters for hard tasks, training LoRA roughly converges to training the original model.==
**No Additional Inference Latency**: When deployed for inference, additional bottleneck layer could only be forwarded sequentially, which is relatively a large overhead when sequence length and batch_size are small (like online inference, in which case, the hardware parallelism cannot be fully utilized). But since $(W_0+\triangle W)\cdot x$ is performed in a coordinate-wise way, updated weights are absorbed. This external LoRA module could be seen as added in a parallel manner.
**Switch between downstream tasks with lower costs**

**Limitation:**
- When absorbing A and B into W (to eliminate inference latency), we cannot batch inputs to different tasks with different A and B in a single forward pass.

==compute $|\Theta|$==

1. Adapter tuning

$$
|\Theta|=\hat L_{Adapt}\times(2\times d_{model}\times r+r+d_{model})+2\times \hat L_{LN}\times d_{model}
$$

$\hat L_{Adapt}$= \number of adapter layers
$  \hat L_{LN}$= \number of traniable LayerNorms
2) LoRA (Only adapte $W_q$ and $W_v$)

$$
|\Theta|=2\times \hat L_{LoRA}\times d_{model}\times r
$$
$\hat L_{LoRA}=$ # of weight matrices where LoRA is applied
#### Experiment
Not all fine-tuning methods benefit from increasing trainable parameters ——> model instability? parameter redundancy? inappropriate optimization approach?
#### Understanding the Low-Rank Updates
**Three questions**
1. To maximize performance on downstream tasks, which subset of weight matrices in a pre-trained Transformer should be adapted? (given constrained parameter budget)
    -> Adapt more weight matrices seems better than adapting a single type of weight matrix with a larger rank (like $W_k$ or $W_q$)

2. What is a good rand r to use in practice to avoid rand deficiency?
	-> Increasing r does not yield significantly better result, so the updated matrix $\triangle W$ may have a very small "intrinsic rank". A low-rank adaption may be sufficient.
	
	-> Subspace similarity between different r: ==Use singular value decomposition and obtain the right-singular unitary matrices $U$;== Measure similarity based on ==Grassmann distance==;
	
	-> The similarity results indicate that the top singular-vector directions of adaption matrices $A_{r=8}$ and $A_{r=64}$ are the most useful, while other directions potentially contain mostly random noises during training. That's the deep reason for low-rank sufficiency.
	
	-> $\triangle W_q$ <u>seems to hava a higher "intrinsic rank" than</u>  $\triangle W_v$
	
	->Similarity equation:
$$
\phi(A_{r=8},A_{r=64},i,j)=\frac{\Vert U^{i\top}_{A_{r=8}}U^j_{A_{r=64}}\Vert ^2_F}{min(i,j)}\in [0,1]
$$

3) What is the connection and correlation between $\triangle W$ and $W$?
    -> project $W$ onto the r-dimensional subspace of $\triangle W$ and compute the ==Frobenius norm== between $\Vert U^\top WV^\top\Vert_F$ and $\Vert W\Vert_F$ ($U$ and $V$ are the ==left- and right-singular matrices of the SVD decomposition of $\triangle W$==)  
    
    -> Intuitively consider a feature amplification factor as the ratio $\frac{\Vert \triangle W \Vert_F}{\Vert U^\top WV\top \Vert_F}$
    
    -> ==Results suggest that adaption matrix amplifies directions that are not emphasized in $W$.==
#### Future work
1. Combine LoRA with other tensor product-based methods, potentially providing orthogonal improvement

2. Mechanism behind fine-tuning 
3. Principles to select the weight matrices to apply LoRA instead of relying on heuristics (in which layer, which kind of weight)
4. Relationship between model size and the optimal rank for adaption
4. The rank-deficiency of $\triangle W$ means potential rank-deficiency of $W$



   
