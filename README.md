# Re3: Generating Longer Stories With Recursive Reprompting and Revision

This repo contains code for Re3: Generating Longer Stories With Recursive Reprompting and Revision (https://arxiv.org/abs/2210.06774) by Kevin Yang, Yuandong Tian, Nanyun Peng, and Dan Klein, to appear at EMNLP 2022. In this codebase we provide instructions for automatically generating stories of 2000+ words (or even much longer), as well as reproducing all of our baselines/ablations/analyses from the paper. We hope that this work can be useful for future research on automatic long story generation. 

## Installation / Data

Install Python 3.8.13 and PyTorch 1.12.1 (other versions are probably also fine for both), then install the remaining requirements via `pip install -r requirements.txt`. Additionally install this repo with `pip install -e .`.

Also run `export OPENAI_API_KEY=$YOUR_API_KEY` in your terminal so that the code can call the GPT3 API with your key. 

Meanwhile, run `wget https://emnlp22-re3-data.s3.amazonaws.com/emnlp22_re3_data.zip` and unzip the folder. This folder contains some pretrained reranker ckpts as well as all of the data we used in all components/analyses. It also contains the final generated stories and MTurk annotation results from our main experiments (note: some generated stories may contain sensitive/NSFW content, since we didn't attempt to filter these).

## Main Story Generation

Main story generation command matching the settings used for our main paper experiments:
```
CUDA_VISIBLE_DEVICES=0 python -u scripts/main.py --summarizer gpt3_summarizer --controller longformer_classifier longformer_classifier --loader alignment coherence --controller-load-dir emnlp22_re3_data/ckpt/relevance_reranker emnlp22_re3_data/ckpt/coherence_reranker --controller-model-string allenai/longformer-base-4096 allenai/longformer-base-4096 --save-outline-file output/outline0.pkl --save-complete-file output/complete_story0.pkl --log-file output/story0.log
```

This command uses our existing relevance and coherence reranker ckpts included in the download (note: these were retrained after submission to be compatible with an updated version of the HuggingFace transformers package, but otherwise are effectively the same as the ones we used in our paper experiments). If you want to use your own ckpts, see the instructions further down for training, and change the paths in this command to point to the correct ckpts. 

### Other Arguments

Main story generation arguments are compiled in `scripts/main.py`; follow the links there to see a complete list. Some particular arguments of interest:

* Add the `--setup-only` flag to stop after generating the initial plan; the plan can then be reloaded by specifying `--load-outline-file` instead of `--save-outline-file` later.
* Add `--no-editor` to turn off the Edit module. This will make generation a decent amount faster without sacrificing too much. This is the Plan-Draft-Rewrite ablation in our paper.
* Add `--no-planner` to turn off the Plan module, which will make performance a good bit worse. This is the Draft-Rewrite-Edit ablation in our paper.
* Set `--max-candidates 1` to turn off the Rewrite module, which will make performance a good bit worse. This is the Plan-Draft-Edit ablation in our paper. 
* Increase `--max-beam-size` (defaults to 1) to turn on a passage-level variable-size beam search procedure based on the rerankers. This is off for the paper experiments (makes the system several times slower) but should improve performance a bit. 
* Change `--fixed-outline-length` (defaults to 3) to set a desired length for your outline (i.e., how many numbered items it will have) or set it to -1 to accept variable length.
* Change `--max-continuation-substeps` (defaults to 4) and `--generation-max-length` (defaults to 256) to change how much story text to write for each numbered item of the outline. With the default settings, it will write four 256-token passages for each.
* For the longer story in Appendix L of the paper, we added `--outline-levels 2 --fixed-outline-length -1 --continuation-threshold 0.5 --max-continuation-substeps 5` which (1) generates a 2-level outline (note: the current version of this generates somewhat repetitive plans sometimes) and (2) dynamically decides when to move on to the next part of the outline during story generation based on reranker scores, instead of just using a fixed length for each part of the outline. 
* Set `--log-level` to be something between 21 and 25 to vary the verbosity of logging (higher = less verbose). 

### Baselines

The simplest Rolling baseline in the paper can be run by simply adding all of the flags `--no-editor`, `--no-planner`, and `--max-candidates 1` to the main story generation command above. 

To run the Rolling-Finetune baseline requires one to additionally finetune OpenAI's davinci model, for which we used passages from the WritingPrompts dataset (Fan et al https://arxiv.org/pdf/1805.04833.pdf). The data we used for finetuning can be found in `emnlp22_re3_data/data/rollingwindow_finetune_data.json`. Alternatively you can generate your own finetuning data via the command

```
python scripts/data/create_gpt3_finetuning_data_rollingwindow.py --expander --data-dir emnlp22_re3_data/data/writing_prompts --limit 10000 --length-limit 10000000 --lower-length-limit 3000 --save-json $PATH_TO_SAVE_FINETUNING_DATA --track-num-tokens
```

Then, using the OpenAI command line API (https://beta.openai.com/docs/guides/fine-tuning), we finetuned as follows. Replace the json with your own path if you re-created the data. 

```
openai api fine_tunes.create -t emnlp22_re3_data/data/rollingwindow_finetune_data.json -m davinci --learning_rate_multiplier 0.02 --n_epochs 1
```

After finetuning, you can run the same command as for the Rolling baseline to generate stories, but additionally specifying `--draft-model-string $OPENAI_CKPT` where `$OPENAI_CKPT` is the name of your finetuned davinci checkpoint (should be a string starting with `davinci:ft-personal-`)

## Reranker Training

If you wanted to retrain the relevance and coherence rerankers yourself, follow the instructions below. Feel free to adjust the batch sizes depending on your GPU memory. 

### Training Relevance Reranker:

The data can be found in `emnlp22_re3_data/data/alignment_data.csv`, and was generated by chunking WritingPrompts stories and prompting GPT3-Instruct-13B (text-curie-001) for summaries, according to the command:

```
python scripts/data/create_alignment_data.py --summarizer gpt3_summarizer --gpt3-model text-curie-001 --dataset writing_prompts --data-dir emnlp22_re3_data/data/writing_prompts --limit 2000 --save-csv $PATH_TO_SAVE_ALIGNMENT_DATA
```

Note that the preprocessing with the WritingPrompts dataset can take a while, so feel free to try it with the folder `emnlp22_re3_data/data/writing_prompts_debug` if you just want to check if it's working first.
After getting the data, train the relevance reranker:

```
CUDA_VISIBLE_DEVICES=0 python scripts/training/train_controller.py --controller longformer_classifier --data-dir emnlp22_re3_data/data/alignment_data.csv --dataset alignment --batch-size 8 --controller-save-dir $DIR_TO_SAVE_RELEVANCE_CKPT --length-limit 2000 --controller-epochs 20 --loader alignment --controller-num-negatives 3 --controller-model-string allenai/longformer-base-4096 --controller-lr 1e-6 --coherence-negative-categories other shuffle
```

Feel free to modify batch size depending on your GPU memory. 

### Training Coherence Reranker:

```
CUDA_VISIBLE_DEVICES=0 python scripts/training/train_controller.py --controller longformer_classifier --data-dir emnlp22_re3_data/data/writing_prompts --dataset writing_prompts --batch-size 1 --controller-save-dir $DIR_TO_SAVE_COHERENCE_CKPT --length-limit 1000 --controller-epochs 20 --loader coherence --controller-num-negatives 3 --controller-model-string allenai/longformer-base-4096 --controller-lr 1e-6 --coherence-negative-categories other shuffle repeat
```

Feel free to modify batch size depending on your GPU memory. 

## Edit Module Analysis

The data we used for evaluation in Section 5.2 can be found in `emnlp22_re3_data/data/consistency_analysis_data`. The command to evaluate each method is as follows, where `$METHOD` is any of `structured`, `entailment`, or `entailment-dpr` corresponding to the methods of the same names in the paper. Note that there's some randomness involved in `structured` due to GPT3 reliance. The latter can also take a while to run. 

```
CUDA_VISIBLE_DEVICES=0 python story_generation/edit_module/evaluate_consistency.py --consistency-dataset-dir $CONSISTENCY_DATA_DIR --consistency-method $METHOD
```

We originally generated the data by generating a large number N setups according to the main story generation command above with the `--setup-only` flag, saving them as `0.pkl` up to `N-1.pkl` in your directory of choice. We then manually sampled and filtered modified versions via `scripts/data/resample_character_descriptions.py` to resample character descriptions until we got contradictory descriptions compared to the original setup. Finally, we used `scripts/data/generate_contradictory_stories.py` to generate story beginnings for the two contradictory setups until the contradiction manifests in the story. (Both scripts rely on the indexing 0 to N-1 to track what setups have already been processed, in case one needs to restart the script. At any point one can backtrack to add more data in previous steps if needed.) See the individual files if you're actually interested in running these scripts, but for a larger-scale analysis in the future it might be better to integrate this pipeline with a crowdsourcing platform like MTurk to avoid the need for manually annotating. 