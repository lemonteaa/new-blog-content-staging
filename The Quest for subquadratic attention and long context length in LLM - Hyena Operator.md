# The Quest for subquadratic attention and long context length in LLM - Hyena Operator

LLM (Large Language Model) have recently become super-hyped due to ChatGPT with an explosion of activities seem in the open source community, as well as a furry of developments across the platform, science, tooling, and application layers. One topic that has been highly anticipated and wished for is to find ways to overcome the **context length limit** of current LLM through improved or novel architecture/design.

In this direction the recent Hyena paper has received some attention. However, a quick web search didn't reveal anything that explain this in a way that is more accessible. So I'd explain how I myself understand it after reading the paper.

**Caveat:** *It's done quickly and without further reading the background literature. I also think about some of its part and try to come up with my own thought, but those might be inaccurate. I will explain it in my own word with my own framing/interpretation/organization so that's quite definitely not how the original author come up with it (which is likely way more messy).*

## Prior act

This problem is well known for a relatively long time already, and there have been many competing proposals, such as longformer (not a global attention), lineformer, etc. (TODO: I read about another post that made a more thorough comparison with neatly formatted table showing key theoretical properties side-by-side) The paper itself talked about Attention-Free Transformer (AFT), Gated State Space (GSS), and Hungry Hungry Hippo (H3).

It is important to note that while many of these proposed solution are promising, after passing a theoretical evaluation, they ultimately still need to be empirically verified because the properties that we care about do not (and likely cannot) have a water-tight math proof. Due to the extreme computational demand of training a GPT style LLM from scratch, researches often are only able to train a down-scaled model and then predict performance by extrapolation. Depending on your level of optimism/skepticism, this can either be a reasonably justified inference, or pure speculation.

At any rate, when an organization finally get to actually train a LLM seeking specifically to solve this issue, they have to pick a method in what could amount to a high stake gamble. (i.e. What if it is trained but the result is disappointing?) At the end of the day, after-the-fact explanation of success is easy - making a choice when there is no gaurantee of success but where any choice requires a huge commitment/investment is a much more difficult situation.

## Review on self-attention mechanism

**Prerequisite:** I assume you are already familiar with transformer architecture and attention, here we are just stating the key formulas and key facts for quick lookup/cross-referencing.

Self-attention is verbally described as:

1. Doing three separate linear projections into latent/semantic spaces: these represent the query $Q = u M_q$, the key $K = u M_k$, and the value $V = u M_v$.
2. Then compute dot-product based similarity score: $QK^T$, with entries given by $ < q_i, k_j > $ across all $i, j$. This is where the **quadratic bottleneck** comes from.
3. Normalize/convert from logit into probabilities/weights by applying softmax: $\text{softmax}(\frac{1}{\sqrt{D}} QK^T)$.
4. Compute answer as linear weighed combinations of matched values corresponding to the keys: $ y = \text{softmax}(\frac{1}{\sqrt{D}} QK^T) V$.
5. This correspond to one *head* of attention. In practise, multi-headed attention is used - just have multiple copies of the same as above but with different weights. They correspond to performing different functions/search/concept on the same sequence. From an implementation point of view this is conceptualized as just parallelizing/adding an additional tensor dimension. This common practise can also be compared to essentially the same thing done to the convolution layers in CNN.

(Note that in a decoder only transformer architecture self attention is the only part where the tokens interact - the feedforward network are identical for each token and can be parallelized)

In words again, self-attention involves doing a form of information retrieval by performing (exhaustive) searches across the sequence of tokens using dot-product based similarity score, thus allowing it to pay attention to token globally/across long distance. This in retrospect proves to be a huge advantage in NLP tasks compared to methods before its invention, such as RNN, which have difficulties with long distance recall. ("Attention is all you need" landmark paper and all that)

Unfortunately, this comes at the cost of quadratic computational cost, which makes evaluating (i.e. inference) or training neural network based on this with long sequences expensive. This has two consequences:

1. Deployed LLM models have to set a short context length limit (both because otherwise the time to output reply will be unacceptably slow for real time applications such as chat, and due to output quality issue explained next), which makes many highly desirable applications impractical or at least difficult at the moment (you may work around this to some extent using various segmenting/external indexing/prompting strategy, but it can't beat giving LLM direct access to the whole long document)
2. Foundation models cannot be trained at long context length, so output quality/intelligence significantly degrade once you start giving it texts longer than what it have seen during training anyway. (Though there are recent proposal such as ALiBi (Attention with Linear Bias) that can mitigate this somewhat - the ability for a LLM to give reasonable output beyond the typical context length it is trained on is known as **extrapolation**).

(Skip: autoregressive point of view?)

For the purpose of explaining Hyena Operator, we will need to provide a framework to interpret attention from a purely computational point of view/specify the IO format:

Self Attention belong to a class of functions known as **Data-controlled linear operator**. For this class of function, the output is just a linear transform of the input, but with a catch - the linear transform/matrix is itself a (possibly nonlinear) function of the input. So they can be written as $y = A(u) u$ where $A$ is a data-controlled matrix.

We can recast self-attention in this framework by writing out the data-dependencies more explicitly in the formula above:

$$
\begin{align}
y &=& \text{softmax}(\frac{1}{\sqrt{D}} QK^T) V \\
  &=& A(Q, K) V \\
  &=& A(Q(u), K(u)) u M_v \\
\end{align}
$$

For some function $A$ whose output is a matrix. Using function composition we can then define another function $A'$ such that $A'(u) := A(Q(u), K(u))$. Or more explicitly: $A'(u) = \text{softmax}(\frac{1}{\sqrt{D}} u M_q M_k^T u^T)$.

(Observation from the paper and other posts on the web: one nice thing about transformer based architecture is that they are *almost linear* and so in many case you can apply linearity based thinking. The almost is important - you need at least a little bit of nonlinearity otherwise a plain linear transform can't do much. The paper observed that in previous experiments done you can remove a majority of the nonlinear/quadratic operations from attention without causing issues, but the last-mile seems critical.)

## Hyena Operator

### Introduction and first definition

Let's give a crisp one sentence description that clarify what Hyena operator actually is:

> Hyena operator is a new family of data-controlled linear operator that is based on neural network/learnable, and designed to be able to emulate self-attention but with a sub-quadratic time algorithm to compute it.

That still seems a mouthful, so let's go slowly and explain its element step by step. We begin by looking at this figure from the paper:


Our first step is to choose a top level architecture/design. In prior acts there have been attempt to do the same or similar computation as attention using ideas from either classical math or software engineering (clustering of similarity vectors, inspiration from cache in KV-store), but they seem to fall short somewhat on expressive power/ability to fully capture attention. We know that neural network is a very powerful and expressive class of functions, so why not try that? (Besides, there's the "bitter lesson" that human attempt to capture complicated structure using human understanding or human coded rules tend to eventually be out-performed by big-data based, brute force approach)

Here it seems to me personally that the author took a stroke of genius where different elements come together in just the right pieces - convolutional filter based prior acts + inspiration from CNN (Convolutional Neural Network) + nonlinear gating.

So here's the plan. We know that convolutional filter is able to, to some extent, replace what attention does. So let's try to slap in a whole (one dimensional) CNN? Of course we can't *just* do that, we'd need to adapt all of its component for our situation.

First thing is that since attention is a data-controlled linear operator, we need to modify the CNN to conform to this IO format. A CNN is basically an interleaved, multi-layer network where on each layer, we "do a convolution, then apply some non-linearity". So what we do is to make both of them a (learned) linear matrix. A convolutional filter *is* already equivalently a (Toeplitz/Circulant) matrix [^1]. For the nonlinearity, we can choose something like the nonlinear elementwise activation function - the author settled on a data-controlled diagonal matrix (data-control is important here because the convolutional filter aren't data controlled, so this is the only place we have left to do it).

![Figure representing the proposed Hyena Operator in their paper](https://github.com/lemonteaa/new-blog-content-staging/blob/main/hyena_operator.png)

(Figure from authors, source: https://hazyresearch.stanford.edu/static/posts/2023-03-07-hyena/diagram.png )

Notes:

- "Dense" here refer to a dense layer in neural network model architecture. Which can be misleading as it actually represent a linear projection matrix here. (Still technically correct as for such a matrix all coefficients are a trainable parameter)
- The filter's matrix is triangular due to the requirement in the case of attention for it to be causal.

**Definition of Hyena Operator as data-controlled linear operator**

Dimensional contants:

- $N$ is order of the Hyena Operator/number of layers. Fixed to a small constant.
- $D$ is width/channel/dimension of latent space in attention. To simplify exposition we set $D=1$. (This is the tensor dimension that trivially parallelize)
- $L$ is the length of sequence.

Definition:

- Let $\theta$ be the learnable parameters for our neural network.
- Let $h^n_\theta$ be a learnable filter, for $n = 1 \cdots N$; and let $P^n_\theta$ be a learnable linear projection matrix, for $n = 0 \cdots N$. Note that the parametrization is explicit/direct for $P^n_\theta$ *after factorization* [^2], that is, each coefficient in the factorized form of matrix $P^n_\theta$ is a learnable paramter corresponding to some entry in $\theta$. For the situation of the filters, refer to next section.
- Let $x^n := u P^n_\theta$ be the projection of input $u$ into a sequence of derived vectors/tensor. Set $x^0 = u P^0_\theta = v$ corresponding to the situation in attention.
- Let

$$
\begin{align}
D^n_x &=& \text{diag}(x^n) \in \mathbb{R}^{L \times L} \\
S^n_h &=& \text{Toeplitz matrix corresponding to } h^n
\end{align}
$$

- Then for the Hyena Operator $H$, we have $y = H(u) v = (D^N_x S^N_h \cdots D^2_x S^2_h D^1_x S^1_h) v$.

----

At this point, we already see that a Hyena Operator [^3]:

- Is highly expressive as it is a neural network and based on CNN model architecture but adapted
- Is a data controlled (factorized) matrix, as multiplication of matrices is still a matrix

### Intended Usage

(*Big caveat: this whole subsection is my personal guesswork. Take with a grain of salt.*)

Maybe we should also briefly mention how Hyena Operator is intended to be used. The operator by itself isn't magic - it simply is, like all neural network, a *class* of possible functions which you still need to train to fit to whatever you want. In the context of the problem we are discussing here, we want to train it to emulate attention.

However, we probably can't just directly train a Hyena Operator in an isolated way - we ideally want to train it on inputs that are typical of those encountered during an actual inference run, such as the intermediate activations/states in the attention layer when feed with user inputs. If we only train on such inputs of typical sequence length, then we may face the extrapolation problem. On the other hand, if we want to feed it inputs of longer length, then the slowness of the original attention (which we need to run to get the "desired output" to feed to Hynea Operator as training data) will again become a bottleneck.

All in all, perhaps their intended use is to take an empty network, replace with Hyena, and then train from scratch. The fact that it is subquadratic will allow us to train on longer length, and hopefully, the designs and properties the author tried to install into Hyena actually work - in the sense that there do in fact exists some configuration of the learning parameter where the Hyena will morph to the desired attention, or at least an approximation that produce good quality output.

### Designing the Hyena Filters

Our next step is to specify the details of the filters $h^n_\theta$. This is critical and can be considered a "secret sauce" of the author.

What they did:

- They first observed that CNN uses local/short convolutional filters/finite impulse response (FIR). They noted that one reason is because computational cost is size of filter times sequence length, so without this local restriction we'd again get the dreaded quadratic bottleneck.
- On the other hand, for our situation a long convolutional filter - one where the impulse response/receptive field is long/global/unrestricted is non-negotiable because this is directly related to whether we can get the long range recall/unrestricted context like attention do.

So, they settled on long convolution. [^4] The quadratic time problem is solved using the standard FFT idea that is drilled into the head of every undergrad students in signal processing (we explain in next section in case someone slept through their class). [^5]

They then move on to pull another really clever trick:

- Observe that attention has constant parameter count - the only parameters are the projection to latent space, after that the search and match/attention itself is fully generic and generalizes to any sequence length without new parameter.
- At the same time, also observe that prior act using convolutional filter may fail if the construct they use to define the filter is not flexible/expressive enough... which hints at...

Using a neural network to compute the filter (!). This changes thing, one of which is that the filter is now implicitly parametrized, allowing us to decouple filter size from parameter count, we then simply allocate parameter count in a way that ensure sublinear parameter scaling.

----

There are some further fine prints on the filter design which you can read on the paper - those are, I think, less tricky then the above. In brief:

- Windowing and positional embedding is added around the filter's neural network to bias it towards certain shapes that are known to have good quality empirically.
- For the use case of language modelling, we need the filter to be causal, which they did by doing some masking.

The formula in the paper: $h_t = \text{Window}(t) \cdot (\text{FFN} \circ \text{PositionalEmbedding})(t)$ (FFN = Feedforward Neural Network, empty circle is function composition while the dot is multiplication)

### Evaluating Hyena Operator quickly

Having specified what function does Hyena Operator represents, we move on to the problem of efficiently evaluating them (Which is the whole point of subquadratic attention!).

- Of course we do not multiply the matrices together in practise, we don't even materialize the matrices actually, instead we apply the effect of the matrices to the input vector iteratively.
- As hinted at above, convolution can be done in subquadratic time using Fast Fourier Transform (FFT). The basic idea: convolutional in time domain is same as pointwise/elementwise multiplication in frequency domain. We can convert between the two domain using Fourier Transform, which when naively done is quadratic time, but FFT bring it down to $O(n \log n)$. Ergo, apply FFT to both the filter and the input, pointwise multiply (linear time), then inverse FFT, and we're done.
- The actual algorithm in production then modify the above to parallelize/reorder some computation to exploit GPU acceleration. Maybe hardware cache-awareness too.

Skipping the hardware optimizations, a basic algorithm is then:

1. Compute $x^n = u P^n_\theta$. (Relabel $x^0 = v$) (Using the factorization of $P^n_\theta$ to avoid the quadratic memory overhead and runtime)
2. Set $z^0 := v$, then loop for $n = 0 \cdots N-1$: $z^{n+1} := x^n \circ \text{FFT-conv}(h^n, z^n)$.
3. Output $y = z^N$.

Here $\text{FFT-conv}(a, b) = a * b$ is a FFT based acceleration of convolution operation, and $\circ$ is elementwise multiplication. You may also refer to the diagram above and cross reference.

### Recap of properties of Hyena Operator

If you're read to this point, congratulation! You now kind of understand Hyena Operator. 

Since I find this to be quite tricky to read, having so many different properties in an interlocking way, maybe summarizing things in a neat table would help:

Desired Property | Hyena feature to satisfy it | Component
----|----|----
Expressive class of function | Neural network of neural network | Top level architecture, Filter
Sublinear parameter scaling | Implicit parametrization of the filter's neural network | Filter
Unrestricted context/long distance associative recall | Long convolution | Filter
Sub-quadratic runtime | Fast Fourier Transform (FFT), not materializing matrices | ?
Data-controlled linear operator | Data-controlled gating, Factorized Matrices | Top level architecture


## Reference

*The paper itself:*

- https://paperswithcode.com/paper/hyena-hierarchy-towards-larger-convolutional
- https://arxiv.org/pdf/2302.10866v3.pdf

*Promotion and review:*

- https://ermongroup.github.io/blog/hyena/
- https://hazyresearch.stanford.edu/blog/2023-03-07-hyena
- https://andlukyane.com/blog/paper-review-hyena
- https://phdstudio.org/2023/05/08/stanford-and-mila-researchers-propose-hyena-an-attention-free-drop-in-replacement-to-the-core-building-block-of-many-large-scale-language-models-anant-shahi-artificial-intelligence-category-marktec/



[^1]: Recall that it simply means that entries with the same relative difference in row/column (on same diagonal) are the same: $a_{i,j} = a_{| i - j |}$. Hopefully the way is obvious. Alternatively, examine the output for each positional basis, as we shift the index by the property of convolution we expect the output to just shift too, which is exactly what it is.

[^2]: As we will see later, we need sublinear parameter count, so a naive approach here would result in $O(L^2)$ memory which seems to defeat the whole point of sub-quadratic attention if we need a space-time tradeoff. Luckily we don't. The reason is because projections have a natural factorization: first a matrix to compute the coefficients/coordinates with respect to the subspace basis, then a second matrix to convert into vector in the ambient space. If the subspace has dimension $b$, then matrices are $O(bL)$ and would be okay as long as $b$ is small. Note that this is the same/similar math behind LoRa (Low rank approximation).

[^3]: The author then also make a clever observation about how the two primitives here are dual of each other: convolution in time domain *is* same as elementwise multiplication in frequency domain and vice versa, and that one spread information around while the other allows fine-grained control of data at specific point.

[^4]: Actually, the author did state that both short and long convolutional filter is necessary - there should be at least one of each type. The reason is that although long range recall is necessary, we also need the ability to do local processing.

[^5]: One may ask if FFT is so great, why doesn't CNN apply it also? I don't have a definitive answer, but I guess it may have to do with the fact that we're doing implicit parametrization here (CNN doesn't becoz the filter itself *is* the point of CNN)... would welcome any insights on this question though.
