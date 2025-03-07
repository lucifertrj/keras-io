# English-to-Spanish translation with KerasNLP

**Author:** [Abheesht Sharma](https://github.com/abheesht17/)<br>
**Date created:** 2022/05/26<br>
**Last modified:** 2022/12/21<br>
**Description:** Use KerasNLP to train a sequence-to-sequence Transformer model on the machine translation task.


<img class="k-inline-icon" src="https://colab.research.google.com/img/colab_favicon.ico"/> [**View in Colab**](https://colab.research.google.com/github/keras-team/keras-io/blob/master/examples/nlp/ipynb/neural_machine_translation_with_keras_nlp.ipynb)  <span class="k-dot">•</span><img class="k-inline-icon" src="https://github.com/favicon.ico"/> [**GitHub source**](https://github.com/keras-team/keras-io/blob/master/examples/nlp/neural_machine_translation_with_keras_nlp.py)



---
## Introduction

KerasNLP provides building blocks for NLP (model layers, tokenizers, metrics, etc.) and
makes it convenient to construct NLP pipelines.

In this example, we'll use KerasNLP layers to build an encoder-decoder Transformer
model, and train it on the English-to-Spanish machine translation task.

This example is based on the
[English-to-Spanish NMT
example](https://keras.io/examples/nlp/neural_machine_translation_with_transformer/)
by [fchollet](https://twitter.com/fchollet). The original example is more low-level
and implements layers from scratch, whereas this example uses KerasNLP to show
some more advanced approaches, such as subword tokenization and using metrics
to compute the quality of generated translations.

You'll learn how to:

- Tokenize text using `keras_nlp.tokenizers.WordPieceTokenizer`.
- Implement a sequence-to-sequence Transformer model using KerasNLP's
`keras_nlp.layers.TransformerEncoder`, `keras_nlp.layers.TransformerDecoder` and
`keras_nlp.layers.TokenAndPositionEmbedding` layers, and train it.
- Use `keras_nlp.utils.top_p_search` function to generate translations
of unseen input sentences using the top-p decoding strategy!

Don't worry if you aren't familiar with KerasNLP. This tutorial will start with
the basics. Let's dive right in!

---
## Setup

Before we start implementing the pipeline, let's import all the libraries we need.


```python
!!pip install -q rouge-score
!!pip install -q git+https://github.com/keras-team/keras-nlp.git --upgrade
```




```python
import keras_nlp
import pathlib
import random
import tensorflow as tf

from tensorflow import keras
from tensorflow_text.tools.wordpiece_vocab import (
    bert_vocab_from_dataset as bert_vocab,
)
```
<div class="k-default-codeblock">
```
[]

```
</div>
Let's also define our parameters/hyperparameters.


```python
BATCH_SIZE = 64
EPOCHS = 1  # This should be at least 10 for convergence
MAX_SEQUENCE_LENGTH = 40
ENG_VOCAB_SIZE = 15000
SPA_VOCAB_SIZE = 15000

EMBED_DIM = 256
INTERMEDIATE_DIM = 2048
NUM_HEADS = 8
```

---
## Downloading the data

We'll be working with an English-to-Spanish translation dataset
provided by [Anki](https://www.manythings.org/anki/). Let's download it:


```python
text_file = keras.utils.get_file(
    fname="spa-eng.zip",
    origin="http://storage.googleapis.com/download.tensorflow.org/data/spa-eng.zip",
    extract=True,
)
text_file = pathlib.Path(text_file).parent / "spa-eng" / "spa.txt"
```

---
## Parsing the data

Each line contains an English sentence and its corresponding Spanish sentence.
The English sentence is the *source sequence* and Spanish one is the *target sequence*.
Before adding the text to a list, we convert it to lowercase.


```python
with open(text_file) as f:
    lines = f.read().split("\n")[:-1]
text_pairs = []
for line in lines:
    eng, spa = line.split("\t")
    eng = eng.lower()
    spa = spa.lower()
    text_pairs.append((eng, spa))
```

Here's what our sentence pairs look like:


```python
for _ in range(5):
    print(random.choice(text_pairs))
```

<div class="k-default-codeblock">
```
('tom and mary are dancing.', 'tom y mary están bailando.')
("it can't be removed. it's fixed.", 'no se puede quitar. está fijo.')
('you can get a loan from a bank.', 'puedes conseguir un préstamo de un banco.')
("you're so stuck up.", 'sois tan engreídas.')
('can man live without having a social life?', '¿puede un hombre vivir sin tener una vida social?')

```
</div>
Now, let's split the sentence pairs into a training set, a validation set,
and a test set.


```python
random.shuffle(text_pairs)
num_val_samples = int(0.15 * len(text_pairs))
num_train_samples = len(text_pairs) - 2 * num_val_samples
train_pairs = text_pairs[:num_train_samples]
val_pairs = text_pairs[num_train_samples : num_train_samples + num_val_samples]
test_pairs = text_pairs[num_train_samples + num_val_samples :]

print(f"{len(text_pairs)} total pairs")
print(f"{len(train_pairs)} training pairs")
print(f"{len(val_pairs)} validation pairs")
print(f"{len(test_pairs)} test pairs")

```

<div class="k-default-codeblock">
```
118964 total pairs
83276 training pairs
17844 validation pairs
17844 test pairs

```
</div>
---
## Tokenizing the data

We'll define two tokenizers - one for the source language (English), and the other
for the target language (Spanish). We'll be using
`keras_nlp.tokenizers.WordPieceTokenizer` to tokenize the text.
`keras_nlp.tokenizers.WordPieceTokenizer` takes a WordPiece vocabulary
and has functions for tokenizing the text, and detokenizing sequences of tokens.

Before we define the two tokenizers, we first need to train them on the dataset
we have. The WordPiece tokenization algorithm is a subword tokenization algorithm;
training it on a corpus gives us a vocabulary of subwords. A subword tokenizer
is a compromise between word tokenizers (word tokenizers need very large
vocabularies for good coverage of input words), and character tokenizers
(characters don't really encode meaning like words do). Luckily, KerasNLP
makes it very simple to train WordPiece on a corpus with the 
`keras_nlp.tokenizers.compute_word_piece_vocabulary` utility.


```python

def train_word_piece(text_samples, vocab_size, reserved_tokens):
    word_piece_ds = tf.data.Dataset.from_tensor_slices(text_samples)
    vocab = keras_nlp.tokenizers.compute_word_piece_vocabulary(
        word_piece_ds.batch(1000).prefetch(2),
        vocabulary_size=vocab_size,
        reserved_tokens=reserved_tokens,
    )
    return vocab

```

Every vocabulary has a few special, reserved tokens. We have four such tokens:

- `"[PAD]"` - Padding token. Padding tokens are appended to the input sequence
length when the input sequence length is shorter than the maximum sequence length.
- `"[UNK]"` - Unknown token.
- `"[START]"` - Token that marks the start of the input sequence.
- `"[END]"` - Token that marks the end of the input sequence.


```python
reserved_tokens = ["[PAD]", "[UNK]", "[START]", "[END]"]

eng_samples = [text_pair[0] for text_pair in train_pairs]
eng_vocab = train_word_piece(eng_samples, ENG_VOCAB_SIZE, reserved_tokens)

spa_samples = [text_pair[1] for text_pair in train_pairs]
spa_vocab = train_word_piece(spa_samples, SPA_VOCAB_SIZE, reserved_tokens)
```

<div class="k-default-codeblock">
```
WARNING:tensorflow:From /usr/local/google/home/chenmoney/anaconda3/envs/keras-io-dev/lib/python3.9/site-packages/tensorflow/python/autograph/pyct/static_analysis/liveness.py:83: Analyzer.lamba_check (from tensorflow.python.autograph.pyct.static_analysis.liveness) is deprecated and will be removed after 2023-09-23.
Instructions for updating:
Lambda fuctions will be no more assumed to be used in the statement where they are used, or at least in the same block. https://github.com/tensorflow/tensorflow/issues/56089

```
</div>
Let's see some tokens!


```python
print("English Tokens: ", eng_vocab[100:110])
print("Spanish Tokens: ", spa_vocab[100:110])
```

<div class="k-default-codeblock">
```
English Tokens:  ['him', 'there', 'they', 'go', 'her', 'has', 'will', 'time', 'how', 're']
Spanish Tokens:  ['mary', 'las', 'más', 'al', 'yo', 'tu', 'estoy', 'eso', 'este', 'muy']

```
</div>
Now, let's define the tokenizers. We will configure the tokenizers with the
the vocabularies trained above.


```python
eng_tokenizer = keras_nlp.tokenizers.WordPieceTokenizer(
    vocabulary=eng_vocab, lowercase=False
)
spa_tokenizer = keras_nlp.tokenizers.WordPieceTokenizer(
    vocabulary=spa_vocab, lowercase=False
)
```

Let's try and tokenize a sample from our dataset! To verify whether the text has
been tokenized correctly, we can also detokenize the list of tokens back to the
original text.


```python
eng_input_ex = text_pairs[0][0]
eng_tokens_ex = eng_tokenizer.tokenize(eng_input_ex)
print("English sentence: ", eng_input_ex)
print("Tokens: ", eng_tokens_ex)
print(
    "Recovered text after detokenizing: ",
    eng_tokenizer.detokenize(eng_tokens_ex),
)

print()

spa_input_ex = text_pairs[0][1]
spa_tokens_ex = spa_tokenizer.tokenize(spa_input_ex)
print("Spanish sentence: ", spa_input_ex)
print("Tokens: ", spa_tokens_ex)
print(
    "Recovered text after detokenizing: ",
    spa_tokenizer.detokenize(spa_tokens_ex),
)
```

<div class="k-default-codeblock">
```
English sentence:  think about it.
Tokens:  tf.Tensor([120 117  70  12], shape=(4,), dtype=int32)
Recovered text after detokenizing:  tf.Tensor(b'think about it .', shape=(), dtype=string)
```
</div>
    
<div class="k-default-codeblock">
```
Spanish sentence:  piénsalo.
Tokens:  tf.Tensor([  44 1677 2782  207   15], shape=(5,), dtype=int32)
Recovered text after detokenizing:  tf.Tensor(b'pi\xc3\xa9nsalo .', shape=(), dtype=string)

```
</div>
---
## Format datasets

Next, we'll format our datasets.

At each training step, the model will seek to predict target words N+1 (and beyond)
using the source sentence and the target words 0 to N.

As such, the training dataset will yield a tuple `(inputs, targets)`, where:

- `inputs` is a dictionary with the keys `encoder_inputs` and `decoder_inputs`.
`encoder_inputs` is the tokenized source sentence and `decoder_inputs` is the target
sentence "so far",
that is to say, the words 0 to N used to predict word N+1 (and beyond) in the target
sentence.
- `target` is the target sentence offset by one step:
it provides the next words in the target sentence -- what the model will try to predict.

We will add special tokens, `"[START]"` and `"[END]"`, to the input Spanish
sentence after tokenizing the text. We will also pad the input to a fixed length.
This can be easily done using `keras_nlp.layers.StartEndPacker`.


```python

def preprocess_batch(eng, spa):
    batch_size = tf.shape(spa)[0]

    eng = eng_tokenizer(eng)
    spa = spa_tokenizer(spa)

    # Pad `eng` to `MAX_SEQUENCE_LENGTH`.
    eng_start_end_packer = keras_nlp.layers.StartEndPacker(
        sequence_length=MAX_SEQUENCE_LENGTH,
        pad_value=eng_tokenizer.token_to_id("[PAD]"),
    )
    eng = eng_start_end_packer(eng)

    # Add special tokens (`"[START]"` and `"[END]"`) to `spa` and pad it as well.
    spa_start_end_packer = keras_nlp.layers.StartEndPacker(
        sequence_length=MAX_SEQUENCE_LENGTH + 1,
        start_value=spa_tokenizer.token_to_id("[START]"),
        end_value=spa_tokenizer.token_to_id("[END]"),
        pad_value=spa_tokenizer.token_to_id("[PAD]"),
    )
    spa = spa_start_end_packer(spa)

    return (
        {
            "encoder_inputs": eng,
            "decoder_inputs": spa[:, :-1],
        },
        spa[:, 1:],
    )


def make_dataset(pairs):
    eng_texts, spa_texts = zip(*pairs)
    eng_texts = list(eng_texts)
    spa_texts = list(spa_texts)
    dataset = tf.data.Dataset.from_tensor_slices((eng_texts, spa_texts))
    dataset = dataset.batch(BATCH_SIZE)
    dataset = dataset.map(preprocess_batch, num_parallel_calls=tf.data.AUTOTUNE)
    return dataset.shuffle(2048).prefetch(16).cache()


train_ds = make_dataset(train_pairs)
val_ds = make_dataset(val_pairs)
```

Let's take a quick look at the sequence shapes
(we have batches of 64 pairs, and all sequences are 40 steps long):


```python
for inputs, targets in train_ds.take(1):
    print(f'inputs["encoder_inputs"].shape: {inputs["encoder_inputs"].shape}')
    print(f'inputs["decoder_inputs"].shape: {inputs["decoder_inputs"].shape}')
    print(f"targets.shape: {targets.shape}")

```

<div class="k-default-codeblock">
```
inputs["encoder_inputs"].shape: (64, 40)
inputs["decoder_inputs"].shape: (64, 40)
targets.shape: (64, 40)

```
</div>
---
## Building the model

Now, let's move on to the exciting part - defining our model!
We first need an embedding layer, i.e., a vector for every token in our input sequence.
This embedding layer can be initialised randomly. We also need a positional
embedding layer which encodes the word order in the sequence. The convention is
to add these two embeddings. KerasNLP has a `keras_nlp.layers.TokenAndPositionEmbedding `
layer which does all of the above steps for us.

Our sequence-to-sequence Transformer consists of a `keras_nlp.layers.TransformerEncoder`
layer and a `keras_nlp.layers.TransformerDecoder` layer chained together.

The source sequence will be passed to `keras_nlp.layers.TransformerEncoder`, which
will produce a new representation of it. This new representation will then be passed
to the `keras_nlp.layers.TransformerDecoder`, together with the target sequence
so far (target words 0 to N). The `keras_nlp.layers.TransformerDecoder` will
then seek to predict the next words in the target sequence (N+1 and beyond).

A key detail that makes this possible is causal masking.
The `keras_nlp.layers.TransformerDecoder` sees the entire sequence at once, and
thus we must make sure that it only uses information from target tokens 0 to N
when predicting token N+1 (otherwise, it could use information from the future,
which would result in a model that cannot be used at inference time). Causal masking
is enabled by default in `keras_nlp.layers.TransformerDecoder`.

We also need to mask the padding tokens (`"[PAD]"`). For this, we can set the
`mask_zero` argument of the `keras_nlp.layers.TokenAndPositionEmbedding` layer
to True. This will then be propagated to all subsequent layers.


```python
# Encoder
encoder_inputs = keras.Input(shape=(None,), dtype="int64", name="encoder_inputs")

x = keras_nlp.layers.TokenAndPositionEmbedding(
    vocabulary_size=ENG_VOCAB_SIZE,
    sequence_length=MAX_SEQUENCE_LENGTH,
    embedding_dim=EMBED_DIM,
    mask_zero=True,
)(encoder_inputs)

encoder_outputs = keras_nlp.layers.TransformerEncoder(
    intermediate_dim=INTERMEDIATE_DIM, num_heads=NUM_HEADS
)(inputs=x)
encoder = keras.Model(encoder_inputs, encoder_outputs)


# Decoder
decoder_inputs = keras.Input(shape=(None,), dtype="int64", name="decoder_inputs")
encoded_seq_inputs = keras.Input(shape=(None, EMBED_DIM), name="decoder_state_inputs")

x = keras_nlp.layers.TokenAndPositionEmbedding(
    vocabulary_size=SPA_VOCAB_SIZE,
    sequence_length=MAX_SEQUENCE_LENGTH,
    embedding_dim=EMBED_DIM,
    mask_zero=True,
)(decoder_inputs)

x = keras_nlp.layers.TransformerDecoder(
    intermediate_dim=INTERMEDIATE_DIM, num_heads=NUM_HEADS
)(decoder_sequence=x, encoder_sequence=encoded_seq_inputs)
x = keras.layers.Dropout(0.5)(x)
decoder_outputs = keras.layers.Dense(SPA_VOCAB_SIZE, activation="softmax")(x)
decoder = keras.Model(
    [
        decoder_inputs,
        encoded_seq_inputs,
    ],
    decoder_outputs,
)
decoder_outputs = decoder([decoder_inputs, encoder_outputs])

transformer = keras.Model(
    [encoder_inputs, decoder_inputs],
    decoder_outputs,
    name="transformer",
)
```

---
## Training our model

We'll use accuracy as a quick way to monitor training progress on the validation data.
Note that machine translation typically uses BLEU scores as well as other metrics,
rather than accuracy. However, in order to use metrics like ROUGE, BLEU, etc. we
will have decode the probabilities and generate the text. Text generation is
computationally expensive, and performing this during training is not recommended.

Here we only train for 1 epoch, but to get the model to actually converge
you should train for at least 10 epochs.


```python
transformer.summary()
transformer.compile(
    "rmsprop", loss="sparse_categorical_crossentropy", metrics=["accuracy"]
)
transformer.fit(train_ds, epochs=EPOCHS, validation_data=val_ds)
```

<div class="k-default-codeblock">
```
Model: "transformer"
__________________________________________________________________________________________________
 Layer (type)                   Output Shape         Param #     Connected to                     
==================================================================================================
 encoder_inputs (InputLayer)    [(None, None)]       0           []                               
                                                                                                  
 token_and_position_embedding (  (None, None, 256)   3850240     ['encoder_inputs[0][0]']         
 TokenAndPositionEmbedding)                                                                       
                                                                                                  
 decoder_inputs (InputLayer)    [(None, None)]       0           []                               
                                                                                                  
 transformer_encoder (Transform  (None, None, 256)   1315072     ['token_and_position_embedding[0]
 erEncoder)                                                      [0]']                            
                                                                                                  
 model_1 (Functional)           (None, None, 15000)  10203288    ['decoder_inputs[0][0]',         
                                                                  'transformer_encoder[0][0]']    
                                                                                                  
==================================================================================================
Total params: 15,368,600
Trainable params: 15,368,600
Non-trainable params: 0
__________________________________________________________________________________________________
1302/1302 [==============================] - 548s 418ms/step - loss: 3.8116 - accuracy: 0.4294 - val_loss: 2.8681 - val_accuracy: 0.5342

<keras.callbacks.History at 0x7f2af05291c0>

```
</div>
---
## Decoding test sentences (qualitative analysis)

Finally, let's demonstrate how to translate brand new English sentences.
We simply feed into the model the tokenized English sentence
as well as the target token `"[START]"`. The model outputs probabilities of the
next token. We then we repeatedly generated the next token conditioned on the
tokens generated so far, until we hit the token `"[END]"`.

For decoding, we will use the `keras_nlp.utils.greedy_search` function from
KerasNLP. Greedy Decoding is a text decoding method which outputs the most
likely next token at each time step, i.e., the token with the highest probability.


```python

def decode_sequences(input_sentences):
    batch_size = tf.shape(input_sentences)[0]

    # Tokenize the encoder input.
    encoder_input_tokens = eng_tokenizer(input_sentences).to_tensor(
        shape=(None, MAX_SEQUENCE_LENGTH)
    )

    # Define a function that outputs the next token's probability given the
    # input sequence.
    def token_probability_fn(decoder_input_tokens):
        return transformer([encoder_input_tokens, decoder_input_tokens])[:, -1, :]

    # Set the prompt to the "[START]" token.
    prompt = tf.fill((batch_size, 1), spa_tokenizer.token_to_id("[START]"))

    generated_tokens = keras_nlp.utils.top_p_search(
        token_probability_fn,
        prompt,
        p=0.1,
        max_length=40,
        end_token_id=spa_tokenizer.token_to_id("[END]"),
    )
    generated_sentences = spa_tokenizer.detokenize(generated_tokens)
    return generated_sentences


test_eng_texts = [pair[0] for pair in test_pairs]
for i in range(2):
    input_sentence = random.choice(test_eng_texts)
    translated = decode_sequences(tf.constant([input_sentence]))
    translated = translated.numpy()[0].decode("utf-8")
    translated = (
        translated.replace("[PAD]", "")
        .replace("[START]", "")
        .replace("[END]", "")
        .strip()
    )
    print(f"** Example {i} **")
    print(input_sentence)
    print(translated)
    print()
```

<div class="k-default-codeblock">
```
** Example 0 **
when did this occur?
¿ cuándo se apulul ?
```
</div>
    
<div class="k-default-codeblock">
```
** Example 1 **
have you ever interrupted your manager?
¿ has dis tu opul ?
```
</div>
    


---
## Evaluating our model (quantitative analysis)

There are many metrics which are used for text generation tasks. Here, to
evaluate translations generated by our model, let's compute the ROUGE-1 and
ROUGE-2 scores. Essentially, ROUGE-N is a score based on the number of common
n-grams between the reference text and the generated text. ROUGE-1 and ROUGE-2
use the number of common unigrams and bigrams, respectively.

We will calculate the score over 30 test samples (since decoding is an
expensive process).


```python
rouge_1 = keras_nlp.metrics.RougeN(order=1)
rouge_2 = keras_nlp.metrics.RougeN(order=2)

for test_pair in test_pairs[:30]:
    input_sentence = test_pair[0]
    reference_sentence = test_pair[1]

    translated_sentence = decode_sequences(tf.constant([input_sentence]))
    translated_sentence = translated_sentence.numpy()[0].decode("utf-8")
    translated_sentence = (
        translated_sentence.replace("[PAD]", "")
        .replace("[START]", "")
        .replace("[END]", "")
        .strip()
    )

    rouge_1(reference_sentence, translated_sentence)
    rouge_2(reference_sentence, translated_sentence)

print("ROUGE-1 Score: ", rouge_1.result())
print("ROUGE-2 Score: ", rouge_2.result())
```

<div class="k-default-codeblock">
```
ROUGE-1 Score:  {'precision': <tf.Tensor: shape=(), dtype=float32, numpy=0.4015512>, 'recall': <tf.Tensor: shape=(), dtype=float32, numpy=0.34473664>, 'f1_score': <tf.Tensor: shape=(), dtype=float32, numpy=0.3574271>}
ROUGE-2 Score:  {'precision': <tf.Tensor: shape=(), dtype=float32, numpy=0.1584127>, 'recall': <tf.Tensor: shape=(), dtype=float32, numpy=0.12230159>, 'f1_score': <tf.Tensor: shape=(), dtype=float32, numpy=0.13493896>}

```
</div>
After 10 epochs, the scores are as follows:

|               | **ROUGE-1** | **ROUGE-2** |
|:-------------:|:-----------:|:-----------:|
| **Precision** |    0.568    |    0.374    |
|   **Recall**  |    0.615    |    0.394    |
|  **F1 Score** |    0.579    |    0.381    |
