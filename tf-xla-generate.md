---
title: 'Faster TensorFlow Text Generation with XLA'
thumbnail: /blog/assets/87_tf_xla_generate/thumbnail.png
---

<h1>
    Faster TensorFlow Text Generation with XLA
</h1>

<div class="blog-metadata">
    <small>Published July 25, 2022.</small>
    <a target="_blank" class="btn no-underline text-sm mb-5 font-sans" href="https://github.com/huggingface/blog/blob/main/tf_xla_generate.md">
        Update on GitHub
    </a>
</div>

<div class="author-card">
    <a href="/joaogante">
        <img class="avatar avatar-user" src="https://aeiljuispo.cloudimg.io/v7/https://s3.amazonaws.com/moonup/production/uploads/1641203017724-noauth.png?w=200&h=200&f=face">
        <div class="bfc">
            <code>joaogante</code>
            <span class="fullname">João Gante</span>
        </div>
    </a>
</div>

<em>TL;DR</em>: Text Generation on TensorFlow can now be massively accelerated with XLA, often being faster than
Pytorch, using a single line of additional code -- check the colab below!

<a target="_blank" href="https://colab.research.google.com/github/huggingface/blog/blob/main/notebooks/87_tf_xla_generate.ipynb">
    <img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/>
</a>


## Text Generation

As the quality of large language models increased, so did our expectations of what those models could do. Especially
since the release of OpenAI's [GPT2 model](https://openai.com/blog/better-language-models/), models with text
generation capabilities have been in the spotlight. And for legitimate reasons -- these models can be used to
summarize, to translate, and even demonstrate zero-shot learning capabilities on language tasks.

The Hugging Face `transformers` library started off with NLP models, and text generation is of utmost importance to us.
It is part of our democratization efforts to ensure text generation is accessible, easily controllable, and efficient.
We have written a [blog post](https://huggingface.co/blog/how-to-generate) about the different types of text
generation, and below there's a quick recap of the basic functionality -- feel free to skip it if you're familiar with
our `generate` function.

```python
# Requires transformers >= 4.21.0; Sample outputs may differ if run on a CPU.
from transformers import AutoTokenizer, TFAutoModelForCausalLM

tokenizer = AutoTokenizer.from_pretrained("gpt2")
model = TFAutoModelForCausalLM.from_pretrained("gpt2")
model.config.pad_token_id = model.config.eos_token_id
input_tokens = tokenizer(["TensorFlow is"], return_tensors="tf")

# 1. Sample
# `do_sample=True` causes the selected token at each iteration to be the outcome
# of sampling, also known as "sample". `seed`, a tuple of two integers for
# stateless TF random number generation, can be used to enforce determinism.
generated = model.generate(**input_tokens, do_sample=True, seed=(42, 0))
print("Sample output: ", tokenizer.decode(generated[0]))
# > Sample output: TensorFlow is a great learning platform for learning about
# data structure and structure in data science..

# 1.1. Controlling generation length
# `max_new_tokens` controls the maximum number of additional tokens to sample,
# in addition to the original prompt.
generated = model.generate(
    **input_tokens, do_sample=True, seed=(42, 0), max_new_tokens=5
)
print("Limiting to 5 new tokens:", tokenizer.decode(generated[0]))
# > Limiting to 5 new tokens: TensorFlow is a great learning platform for
generated = model.generate(
    **input_tokens, do_sample=True, seed=(42, 0), max_new_tokens=30
)
print("Limiting to 30 new tokens:", tokenizer.decode(generated[0]))
# > Limiting to 30 new tokens: TensorFlow is a great learning platform for
# learning about data structure and structure in data science................

# 1.2. Controlling sampling temperature
# `temperature` controls the randomness of the sampling -- values below 1.0
# prioritize sampling tokens with a higher likelihood, where values above 1.0 do
# the opposite. Setting it to 0.0 means it will be doing greedy decoding (see
# below), where very large values approximate uniform sampling.
generated = model.generate(
    **input_tokens, do_sample=True, seed=(42, 0), temperature=0.7
)
print("Temperature 0.7: ", tokenizer.decode(generated[0]))
# > Temperature 0.7: TensorFlow is a great way to do things like this........
generated = model.generate(
    **input_tokens, do_sample=True, seed=(42, 0), temperature=1.5
)
print("Temperature 1.5: ", tokenizer.decode(generated[0]))
# > Temperature 1.5: TensorFlow is being developed for both Cython and Bamboo.
# On Bamboo...

# 1.3. Controlling top-p sampling
# `top_p` limits the sampling to the most likely tokens that cumulatively account
# for at least `top_p` of the total probability. In other words, it will ensure
# that very unlikely tokens are never sampled, and will likely improve the quality
# of the generation
generated = model.generate(**input_tokens, do_sample=True, seed=(42, 0), top_p=0.9)
print("Top p 0.9: ", tokenizer.decode(generated[0]))
# > Top p 0.9: TensorFlow is a great learning platform for learning about data
# structure and structure in data science. It

# 2. Beam Search
# When `do_sample=False` (default), the most likely token is picked at each
# iteration ("greedy decoding"). More interestingly, when `num_beams` is larger
# than 1, it triggers "beam search", which continuously explore high-probability
# sequences
generated = model.generate(**input_tokens, num_beams=2)
print("Beam Search output:", tokenizer.decode(generated[0]))
# > Beam Search output: TensorFlow is an open-source, open-source,
# distributed-source application framework for the

# 2.1. `num_return_sequences` controls the number of outputs to return per input.
# You can use it to see the multiple hypotheses generated by beam search, and it
# can also be used with sampling.
generated = model.generate(**input_tokens, num_beams=2, num_return_sequences=2)
print(
    "All generated hypotheses:",
    "\n".join(tokenizer.decode(out) for out in generated)
)
# > All generated hypotheses: TensorFlow is an open-source, open-source,
# distributed-source application framework for the
# > TensorFlow is an open-source, open-source, distributed-source application
# framework that allows
```

The basics of Sample and Beam Search are straightforward to control. However there are many options not covered in the
example above, and we encourage you to read the
[documentation](https://huggingface.co/docs/transformers/main/en/main_classes/text_generation#transformers.generation_tf_utils.TFGenerationMixin.generate)
for advanced use cases.
Sadly, when you run `generate` with TensorFlow, you might notice that it takes a while to execute.
If your target application expects low latency or a large amount of input prompts, running text generation with
TensorFlow may seem an expensive endeavour. 😬

Fear not, for the remainder of this blog post aims to demonstrate that one line of code can make a drastic improvement.
If you'd rather jump straight into action,
[our colab](https://colab.research.google.com/github/huggingface/blog/blob/main/notebooks/87_tf_xla_generate.ipynb)
has an interactive example you can fiddle with!

## TensorFlow and XLA

[XLA](https://www.tensorflow.org/xla), or Accelerated Linear Algebra, is a compiler originally developed to accelerate
TensorFlow models. Nowadays, it is also the compiler behind [JAX](https://github.com/google/jax), and it can even
be [used with PyTorch](https://huggingface.co/blog/pytorch-xla). Although the word "compiler" might sound dauting for
some, XLA is simple to use with TensorFlow -- it comes packaged inside the `tensorflow` library, and it can be
triggered with the `jit_compile` argument in any graph-creating function.

For those of you familiar with TensorFlow 1, the concept of a TensorFlow graph comes naturally, as it was the only
mode of operation. First you defined your operations in a declarative fashion to create a graph, then you could pipe
inputs through the graph and observe the outputs. Fast, efficient, but painful to debug. With TensorFlow 2 came
Eager Execution and the ability to code our models imperatively -- the TensorFlow team explains the difference in more
detail in [their blog post](https://blog.tensorflow.org/2019/01/what-are-symbolic-and-imperative-apis.html).

Hugging Face writes their TensorFlow models with Eager Execution in mind. Transparency is a core value, and being able
to inspect the model internals at any point is very benefitial to that end. However, that does mean that some uses of
our models do not benefit from the graph mode performance advantages out of the box (e.g. when calling `model(args)`).

Fortunately, the TensorFlow team has users like us covered! Wrapping a function containing TensorFlow code with
[`tf.function`](https://www.tensorflow.org/api_docs/python/tf/function) will attempt to convert it into a graph when
you call the wrapped function. If you're training a model, calling `model.compile()` (without `run_eagerly=True`) does
precisely that wrapping, so that you benefit from graph mode when you call `model.fit()`. Since `tf.function` can be
used in any function containing TensorFlow code, it means you can use it on functions that go beyond model inference,
creating a single optimized graph.

Now that we know how to create TensorFlow graphs, compiling them with XLA is straightforward -- simply add `jit_compile=True`
as an argument to the functions mentioned above (`tf.function` and `tf.keras.Model.compile`). Assuming everything went well
(more on that below) and that you are using a GPU or a TPU, you will notice that the first call will take a while, but
that the remaining ones are much, much faster. Here's a simple example:

```python
import tensorflow as tf
from transformers import AutoTokenizer, TFAutoModelForCausalLM

tokenizer = AutoTokenizer.from_pretrained("gpt2")
model = TFAutoModelForCausalLM.from_pretrained("gpt2")
model.config.pad_token_id = model.config.eos_token_id
input_tokens = tokenizer(["TensorFlow is"], return_tensors="tf")

def most_likely_next_token(input_ids):
    model_output = model(input_ids)
    return tf.argmax(model_output.logits[:, -1, :], axis=-1)

# Now let's create an XLA function from the function above
xla_most_likely_next_token = tf.function(most_likely_next_token, jit_compile=True)

print("Calling regular function with TensorFlow code...")
most_likely_next_token(input_tokens)
print("Calling XLA function... (for the first time -- will be slow)")
xla_most_likely_next_token(input_tokens)
print("Calling XLA function... (for the second time -- will be fast)")
xla_most_likely_next_token(input_tokens)
```

## Text Generation using TensorFlow with XLA

As with any optimization procedure, there is no free lunch -- XLA is no exception. From the perspective of a text
generation user, there is only one technical detail that you need to keep in mind. Without digging too much into
[details](https://www.tensorflow.org/guide/function#rules_of_tracing), XLA used in this fashion does just-in-time (JIT)
compilation of a `tf.function`, which relies on polymorphism. In other words, every time you call a `tf.function` with
different inputs, with the exception of native TensorFlow types (such as `tf.Tensor` or `tf.Variable`) with the same
data type and shape, it will go through a slow compilation step also known as tracing. In practice, for text generation,
it simply means the input should be padded to a multiple of a certain length (so it has a limited number of possible
shapes), and that using different options will be slow for the first time you use them. Illustrated as an example:

```python
import tensorflow as tf
from transformers import AutoTokenizer, TFAutoModelForCausalLM

# Notice the new argument, `padding_side="left"` -- decoder-only models, which can
# be instantiated with TFAutoModelForCausalLM, should be left-padded, as they
# continue generating from the input prompt.
tokenizer = AutoTokenizer.from_pretrained("gpt2", padding_side="left", pad_token="</s>")
model = TFAutoModelForCausalLM.from_pretrained("gpt2")
model.config.pad_token_id = model.config.eos_token_id
input_1 = ["TensorFlow is"]
input_2 = ["TensorFlow is a"]

# One line to create a XLA generation function
xla_generate = tf.function(model.generate, jit_compile=True)

# Let's see what happens when no padding is done
tokenized_input_1 = tokenizer(input_1, return_tensors="tf")  # length = 4
tokenized_input_2 = tokenizer(input_2, return_tensors="tf")  # length = 5
print(f"`tokenized_input_1` shape = {tokenized_input_1.input_ids.shape}")
print(f"`tokenized_input_2` shape = {tokenized_input_2.input_ids.shape}")
print("Calling XLA generation with tokenized_input_1...")
print("(will be slow as it is the first call)")
xla_generate(**tokenized_input_1)
print("Calling XLA generation with tokenized_input_2...")
print("(has a different length = will trigger tracing again)")
xla_generate(**tokenized_input_2)

# Oh no, that's terrible! Let's try the same, now with padding
# `pad_to_multiple_of` should be used to achieve a balance between accepting any
# input length and limiting tracing.
padding_kwargs = {"pad_to_multiple_of": 8, "padding": True}
tokenized_input_1_with_padding = tokenizer(
    input_1, return_tensors="tf", **padding_kwargs
)  # length = 8
tokenized_input_2_with_padding = tokenizer(
    input_2, return_tensors="tf", **padding_kwargs
)  # length = 8
print(
    "`tokenized_input_1_with_padding` shape = ",
    f"{tokenized_input_1_with_padding.input_ids.shape}"
)
print(
    "`tokenized_input_2_with_padding` shape = ",
    f"{tokenized_input_2_with_padding.input_ids.shape}"
)
print("Calling XLA generation with tokenized_input_1_with_padding...")
print("(slow, first time running with this length)")
xla_generate(**tokenized_input_1_with_padding)
print("Calling XLA generation with tokenized_input_2_with_padding...")
print("(will be fast!)")
xla_generate(**tokenized_input_2_with_padding)

# Be careful -- if you suddendly change the input options, it will trigger tracing
print("Calling XLA generation with the same input, but with new options...")
print("(slow again)")
xla_generate(**tokenized_input_1_with_padding, num_beams=2)
```

From a developer perspective, relying on XLA implies being aware of a few additional nuances. XLA shines when the size
of the data structures are known in advance, such as in model training. On the other hand, when their dimensions are
impossible to determine or certain dynamic slices are used, XLA fails to compile. Modern implementations of text
generation are auto-regressive, whose natural behavior is to expand tensors and to abruptly interrupt some operations
as it goes -- in other words, not XLA-friendly by default.
We have rewritten our entire TensorFlow text generation codebase to vectorize operations and use fixed-sized
structures with padding. Our NLP models were also modified to correctly use their positional embeddings in the
presence of padded structures. The result should be invisible to TensorFlow text generation users, except for the
availability of XLA compilation.

## Benchmarks and Conclusions

We have seen above that we can convert TensorFlow functions into a graph and accelerate them with XLA compilation.
Current forms of text generation are simply an auto-regressive function that alternate between a model forward pass
and some post-processing, producing one token per iteration. Through XLA compilation, the entire process gets
optimized, resulting in faster execution. But how much faster? The Gradio demo below contains some benchmarks
comparing Hugging Face's text generation on multiple GPU models for the two main ML frameworks, TensorFlow and PyTorch.

<div class="hidden xl:block">
<div style="display: flex; flex-direction: column; align-items: center;">
<iframe src="https://hf.space/embed/joaogante/tf_xla_generate_benchmarks/+" frameBorder="0" width="1200px" height="760px" title="Gradio app" allow="accelerometer; ambient-light-sensor; autoplay; battery; camera; document-domain; encrypted-media; fullscreen; geolocation; gyroscope; layout-animations; legacy-image-formats; magnetometer; microphone; midi; oversized-images; payment; picture-in-picture; publickey-credentials-get; sync-xhr; usb; vr ; wake-lock; xr-spatial-tracking" sandbox="allow-forms allow-modals allow-popups allow-popups-to-escape-sandbox allow-same-origin allow-scripts allow-downloads"></iframe>
</div>
</div>

If you explore the results, two conclusions become quickly visible:

1. As this blog post has been building up to here, TensorFlow text generation is much faster with XLA. We are talking
about speedups larger than 100x in some cases, which trully demonstrate the power of a compiled graph 🚀
2. TensorFlow text generation with XLA is the fastest option in the vast majority of cases, in some of them by as
much as 9x faster, debunking the myth that Pytorch is the go-to framework for serious NLP tasks 💪

Give [our colab](https://colab.research.google.com/github/huggingface/blog/blob/main/notebooks/87_tf_xla_generate.ipynb)
a go, and enjoy the power of text generation supercharged with XLA!