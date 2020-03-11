# ELECTRA

## Introduction

**ELECTRA** is a new method for self-supervised language representation learning. It can be used to pre-train transformer networks using relatively little compute. ELECTRA models are trained to distinguish "real" input tokens vs "fake" input tokens generated by another neural network, similar to the discriminator of a [GAN](https://arxiv.org/pdf/1406.2661.pdf). At small scale, ELECTRA achieves strong results even when trained on a single GPU. At large scale, ELECTRA achieves state-of-the-art results on the [SQuAD 2.0](https://rajpurkar.github.io/SQuAD-explorer/) dataset.

For a detailed description and experimental results, please refer to our paper [ELECTRA: Pre-training Text Encoders as Discriminators Rather Than Generators](https://openreview.net/pdf?id=r1xMH1BtvB).

This repository contains code to pre-train ELECTRA, including small ELECTRA models on a single GPU. It also supports fine-tuning ELECTRA on downstream tasks including classification tasks (e.g,. [GLUE](https://gluebenchmark.com/)), QA tasks (e.g., [SQuAD](https://rajpurkar.github.io/SQuAD-explorer/)), and sequence tagging tasks (e.g., [text chunking](https://www.clips.uantwerpen.be/conll2000/chunking/)).



## Released Models

We are initially releasing three pre-trained models:

| Model | Layers | Hidden Size | Params | GLUE score | Download |
| --- | --- | --- | --- | ---  | --- |
| ELECTRA-Small | 12 | 256 | 14M | 77.4  | [link](https://storage.googleapis.com/electra-data/electra_small.zip) |
| ELECTRA-Base | 12 | 768 | 110M | 82.7 | [link](https://storage.googleapis.com/electra-data/electra_base.zip) |
| ELECTRA-Large | 24 | 1024 | 335M |  85.2 | [link](https://storage.googleapis.com/electra-data/electra_large.zip) |

The models were trained on uncased English text. They correspond to ELECTRA-Small++, ELECTRA-Base++, ELECTRA-1.45M  in our paper. We hope to release other models, such as multilingual models, in the future.

On [GLUE](https://gluebenchmark.com/), ELECTRA-Large scores slightly better than ALBERT/XLNET, ELECTRA-Base scores better than BERT-Large, and ELECTRA-Small scores slightly worst than [TinyBERT](https://arxiv.org/abs/1909.10351) (but uses no distillation). See the expected results section below for detailed performance numbers.



## Requirements
* Python 3
* [TensorFlow](https://www.tensorflow.org/) 1.15 (although we hope to support TensorFlow 2.0 at a future date)
* [NumPy](https://numpy.org/)
* [scikit-learn](https://scikit-learn.org/stable/) and [SciPy](https://www.scipy.org/) (for computing some evaluation metrics).

## Pre-training
Use `build_pretraining_dataset.py` to create a pre-training dataset from a dump of raw text. It has the following arguments:

* `--corpus-dir`: A directory containing raw text files to turn into ELECTRA examples. A text file can contain multiple documents with empty lines separating them.
* `--vocab-file`: File defining the wordpiece vocabulary.
* `--output-dir`: Where to write out ELECTRA examples.
* `--max-seq-length`: The number of tokens per example (128 by default).
* `--num-processes`: If >1 parallelize across multiple processes (1 by default).
* `--blanks-separate-docs`: Whether blank lines indicate document boundaries (True by default).
* `--do-lower-case/--no-lower-case`: Whether to lower case the input text (True by default).

Use `run_pretraining.py` to pre-train an ELECTRA model. It has the following arguments:

* `--data-dir`: a directory where pre-training data, model weights, etc. are stored. By default, the training loads examples from `<data-dir>/pretrain_tfrecords` and a vocabulary from `<data-dir>/vocab.txt`.
*  `--model-name`: a name for the model being trained. Model weights will be saved in `<data-dir>/models/<model-name>` by default.
* `--hparams` (optional): a JSON dict or path to a JSON file containing model hyperparameters, data paths, etc. See `configure_pretraining.py` for the supported hyperparameters.

If training is halted, re-running the `run_pretraining.py` with the same arguments will continue the training where it left off.

##  Quickstart: Pre-train a small ELECTRA model.
These instructions pre-train a small ELECTRA model (12 layers, 256 hidden size). Unfortunately, the data we used in the paper is not publicly available, so we will use the [OpenWebTextCorpus](https://skylion007.github.io/OpenWebTextCorpus/) released by Aaron Gokaslan and Vanya Cohen instead. The fully-trained model (~4 days on a v100 GPU) should perform roughly in between [GPT](https://s3-us-west-2.amazonaws.com/openai-assets/research-covers/language-unsupervised/language_understanding_paper.pdf) and BERT-Base in terms of GLUE performance. By default the model is trained on length-128 sequences, so it is not suitable for running on question answering. See the "expected results" section below for more details on model performance.

#### Setup
1. Place a vocabulary file in `$DATA_DIR/vocab.txt`. Our ELECTRA models all used the exact same vocabulary as English uncased BERT, which you can download [here](https://storage.googleapis.com/electra-data/vocab.txt).
2. Download the [OpenWebText](https://skylion007.github.io/OpenWebTextCorpus/) corpus (12G) and extract it  (i.e., run `tar xf openwebtext.tar.xz`). Place it in `$DATA_DIR/openwebtext`.
3. Run `python3 build_openwebtext_pretraining_dataset.py --data-dir $DATA_DIR --num-processes 5`. It pre-processes/tokenizes the data and outputs examples as [tfrecord](https://www.tensorflow.org/tutorials/load_data/tfrecord) files under `$DATA_DIR/pretrain_tfrecords`. The tfrecords require roughly 30G of disk space.

#### Pre-training the model.
Run `python3 run_pretraining.py --data-dir $DATA_DIR --model-name electra_small_owt`
to train a small ELECTRA model for 1 million steps on the data. This takes slightly over 4 days on a Tesla V100 GPU. However, the model should achieve decent results after 200k steps (10 hours of training on the v100 GPU).

To customize the training, add `--hparams '{"hparam1": value1, "hparam2": value2, ...}'` to the run command. `--hparams` can also be a path to a `.json` file containing the hyperparameters. Some particularly useful options:

* `"debug": true` trains a tiny ELECTRA model for a few steps.
* `"model_size": one of "small", "base", or "large"`: determines the size of the model
* `"electra_objective": false` trains a model with masked language modeling instead of replaced token detection (essentially BERT with dynamic masking and no next-sentence prediction).
* `"num_train_steps": n` controls how long the model is pre-trained for.
* `"pretrain_tfrecords": <paths>` determines where the pre-training data is located. Note you need to specify the specific files not just the directory (e.g., `<data-dir>/pretrain_tf_records/pretrain_data.tfrecord*`)
* `"vocab_file": <path>` and `"vocab_size": n` can be used to set a custom wordpiece vocabulary.
* `"learning_rate": lr, "train_batch_size": n`, etc. can be used to change training hyperparameters
* `"model_hparam_overrides": {"hidden_size": n, "num_hidden_layers": m}`, etc. can be used to changed the hyperparameters for the underlying transformer (the `"model_size"` flag sets the default values).

See `configure_pretraining.py` for the full set of supported hyperparameters.

#### Evaluating the pre-trained model.

To evaluate the model on a downstream task, see the below finetuning instructions. To evaluate the generator/discriminator on the openwebtext data run `python3 run_pretraining.py --data-dir $DATA_DIR --model-name electra_small_owt --hparams '{"do_train": false, "do_eval": true}'`. This will print out eval metrics such as the accuracy of the generator and discriminator, and also writing the metrics out to `data-dir/model-name/results`.

## Fine-tuning

Use `run_finetuning.py` to fine-tune and evaluate an ELECTRA model on a downstream NLP task. It expects three arguments:

* `--data-dir`: a directory where data, model weights, etc. are stored. By default, the script loads finetuning data from `<data-dir>/finetuning_data/<task-name>` and a vocabulary from `<data-dir>/vocab.txt`.
*  `--model-name`: a name of the pre-trained model: the pre-trained weights should exist in `data-dir/models/model-name`.
* `--hparams`: a JSON dict containing model hyperparameters, data paths, etc. (e.g., `--hparams '{"task_names": ["rte"], "model_size": "base", "learning_rate": 1e-4, ...}'`). See `configure_pretraining.py` for the supported hyperparameters.  Instead of a dict, this can also be a path to a `.json` file containing the hyperparameters. You must specify the `"task_names"` and `"model_size"` (see examples below).

By default eval metrics will be saved in `data-dir/model-name/results` and model weights will be saved in `data-dir/model-name/finetuning_models`. To customize the training, add `--hparams '{"hparam1": value1, "hparam2": value2, ...}'` to the run command. Some particularly useful options:

* `"debug": true` fine-tunes a tiny ELECTRA model for a few steps.
* `"task_names": ["task_name"]`: specifies the tasks to train on. A list because the codebase nominally supports multi-task learning, (although be warned this has not been thoroughly tested).
* `"model_size": one of "small", "base", or "large"`: determines the size of the model; you must set this to the same size as the pre-trained model.
* `"do_train" and "do_eval"`: train and/or evaluate a model (both are set to true by default). For using `"do_eval": true` with `"do_train": false`, you need to specify the `init_checkpoint`, e.g., `python3 run_finetuning.py --data-dir $DATA_DIR --model-name electra_base --hparams '{"model_size": "base", "task_names": ["mnli"], "do_train": false, "do_eval": true, "init_checkpoint": "<data-dir>/models/electra_base/finetuning_models/mnli_model_1"}'`
* `"num_trials": n`: If >1, does multiple fine-tuning/evaluation runs with different random seeds.
* `"learning_rate": lr, "train_batch_size": n`, etc. can be used to change training hyperparameters.
* `"model_hparam_overrides": {"hidden_size": n, "num_hidden_layers": m}`, etc. can be used to changed the hyperparameters for the underlying transformer (the `"model_size"` flag sets the default values).

### Setup
Get a pre-trained ELECTRA model either by training your own (see pre-training instructions above), or downloading the release ELECTRA weights and unziping them under `$DATA_DIR/models` (e.g., you should have a directory`$DATA_DIR/models/electra_large` if you are using the large model).


### Finetune ELECTRA on a GLUE  task

Download the GLUE data by running [this script](https://gist.github.com/W4ngatang/60c2bdb54d156a41194446737ce03e2e). Set up the data by running `mv CoLA cola && mv MNLI mnli && mv MRPC mrpc && mv QNLI qnli && mv QQP qqp && mv RTE rte && mv SST-2 sst && mv STS-B sts && mv diagnostic/diagnostic.tsv mnli && mkdir -p $DATA_DIR/finetuning_data && mv * $DATA_DIR/finetuning_data`.

Then run `run_finetuning.py`. For example, to fine-tune ELECTRA-Base  on MNLI
```
python3 run_finetuning.py --data-dir $DATA_DIR --model-name electra_base --hparams '{"model_size": "base", "task_names": ["mnli"]}'
```
Or fine-tune a small model pre-trained using the above instructions on CoLA.
```
python3 run_finetuning.py --data-dir $DATA_DIR --model-name electra_small_owt --hparams '{"model_size": "small", "task_names": ["cola"]}'
```

### Finetune ELECTRA on question answering

The code supports [SQuAD](https://rajpurkar.github.io/SQuAD-explorer/) 1.1 and 2.0, as well as datasets in [the 2019 MRQA shared task](https://github.com/mrqa/MRQA-Shared-Task-2019)

* **Squad 1.1**: Download the [train](https://rajpurkar.github.io/SQuAD-explorer/dataset/train-v1.1.json) and [dev](https://rajpurkar.github.io/SQuAD-explorer/dataset/dev-v1.1.json) datasets and move them under `$DATA_DIR/finetuning_data/squadv1/(train|dev).json`
* **Squad 2.0**: Download the datasets from the [SQuAD Website](https://rajpurkar.github.io/SQuAD-explorer/) and move them under `$DATA_DIR/finetuning_data/squad/(train|dev).json`
* **MRQA tasks**: Download the data from [here](https://github.com/mrqa/MRQA-Shared-Task-2019#datasets). Move the data to `$DATA_DIR/(newsqa|naturalqs|triviaqa|searchqa)/(train|dev).jsonl`.

Then run (for example)
```
python3 run_finetuning.py --data-dir $DATA_DIR --model-name electra_base --hparams '{"model_size": "base", "task_names": ["squad"]}'
```

This repository uses the official evaluation code released by the [SQuAD](https://rajpurkar.github.io/SQuAD-explorer/) authors and [the MRQA shared task](https://github.com/mrqa/MRQA-Shared-Task-2019) to compute metrics

### Finetune ELECTRA on sequence tagging

Download the CoNLL-2000 text chunking dataset from [here](https://www.clips.uantwerpen.be/conll2000/chunking/) and put it under `$DATA_DIR/chunk/(train|dev).txt`. Then run
```
python3 run_finetuning.py --data-dir $DATA_DIR --model-name electra_base --hparams '{"model_size": "base", "task_names": ["chunk"]}'
```

### Adding a new task
The easiest way to run on a new task is to implement a new `finetune.task.Task`, add it to `finetune.task_builder.py`, and then use `run_finetuning.py` as normal. For classification/qa/sequence tagging, you can inherit from a `finetune.classification.classification_tasks.ClassificationTask`, `finetune.qa.qa_tasks.QATask`, or `finetune.tagging.tagging_tasks.TaggingTask`.
For preprocessing data, we use the same tokenizer as [BERT](https://github.com/google-research/bert).


## Expected Results
Here are expected results for ELECTRA on various tasks. Note that variance in fine-tuning can be [quite large](https://arxiv.org/abs/2002.06305), so for some tasks you may see big fluctuations in scores when fine-tuning from the same checkpoint multiple times. The below scores show median performance over a large number of random seeds.  ELECTRA-Small/Base/Large are our released models. ELECTRA-Small-OWT is the OpenWebText-trained model from above (it performs a bit worse than ELECTRA-Small due to being trained for less time and on a smaller dataset).

|  | CoLA | SST | MRPC | STS  | QQP  | MNLI | QNLI | RTE | SQuAD 1.1 | SQuAD 2.0 | Chunking |
| --- | --- | --- | --- | ---  | ---  | --- | --- | --- | ---| ---| --- |
| Metrics | MCC | Acc | Acc | Spearman  | Acc  | Acc | Acc | Acc | EM | EM | F1 |
| ELECTRA-Large| 69.1 | 96.9 | 90.8 | 92.6 | 92.4 | 90.9 | 95.0 | 88.0 | 89.7 | 88.1 | 97.2 |
| ELECTRA-Base | 67.7 | 95.1 | 89.5 | 91.2 | 91.5  | 88.8  | 93.2 | 82.7 | 86.8 | 83.7| 97.1 |
| ELECTRA-Small | 57.0 | 91.2 | 88.0 |  87.5 | 89.0  | 81.3 | 88.4 | 66.7 | 75.8 | 70.1 |  96.5 |
| ELECTRA-Small-OWT | 56.8 | 88.3 | 87.4 |  86.8 | 88.3  | 78.9 | 87.9 | 68.5 | -- | -- |  -- |


## Citation
If you use this code for your publication, please cite the original paper:
```
@inproceedings{clark2019electra,
  title = {{ELECTRA}: Pre-training Text Encoders as Discriminators Rather Than Generators},
  author = {Kevin Clark and Minh-Thang Luong and Quoc V. Le and Christopher D. Manning},
  booktitle = {ICLR},
  year = {2020}
}
```

## Contact Info
For help or issues using ELECTRA, please submit a GitHub issue.

For personal communication related to ELECTRA, please contact [Kevin Clark](https://cs.stanford.edu/~kevclark/) (`kevclark@cs.stanford.edu`).
