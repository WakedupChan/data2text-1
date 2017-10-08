# data2text

Code for [Challenges in Data-to-Document Generation](https://arxiv.org/abs/1707.08052) (Wiseman, Shieber, Rush; EMNLP 2017); much of this code is adapted from an earlier fork of [OpenNMT](https://github.com/OpenNMT/OpenNMT).

The boxscore-data associated with the above paper can be downloaded from the [boxscore-data repo](https://github.com/harvardnlp/boxscore-data), and this README will go over running experiments on the RotoWire portion of the data; running on the SBNation data (or other data) is quite similar.


## Preprocessing
Before training models, you must preprocess the data. Assuming the RotoWire json files reside at `~/Documents/code/boxscore-data/rotowire`, the following command will preprocess the data

```
th box_preprocess.lua -json_data_dir ~/Documents/code/boxscore-data/rotowire -save_data roto
```

and write files called roto-train.t7, roto.src.dict, and roto.tgt.dict to your local directory.

### Incorporating Pointer Information
For the "conditional copy" model, it is necessary to know where in the source table each target word may have been copied from.

This pointer information can be incorporated into the preprocessing by running:

```
th box_preprocess.lua -json_data_dir ~/Documents/code/boxscore-data/rotowire -save_data roto -ptr_fi "roto-ptrs.txt"
```

The file roto-ptrs.txt has been included in the repo.


## Training (and Downloading Trained Models)
The command for training the Joint Copy + Rec + TVD model is as follows:

```
th box_train.lua -data roto-train.t7 -save_model roto_jc_rec_tvd -rnn_size 600 -word_vec_size 600 -enc_emb_size 600 -max_batch_size 16 -dropout 0.5 -feat_merge concat -pool mean -enc_layers 1 -enc_relu -report_every 50 -gpuid 1 -epochs 50 -learning_rate 1 -enc_dropout 0 -decay_update2 -layers 2 -copy_generate -tanh_query -max_bptt 100 -discrec -rho 1 -partition_feats -recembsize 600 -discdist 1
```

A model trained in this way can be downloaded from  https://drive.google.com/file/d/0B1ytQXPDuw7ONlZOQ2R3UWxmZ2s/view?usp=sharing


The command for training the Conditional Copy model is as follows:

```
th box_train.lua -data roto-train.t7 -save_model roto_cc -rnn_size 600 -word_vec_size 600 -enc_emb_size 600 -max_batch_size 16 -dropout 0.5 -feat_merge concat -pool mean -enc_layers 1 -enc_relu -report_every 50 -gpuid 1 -epochs 100 -learning_rate 1 -enc_dropout 0 -decay_update2 -layers 2 -copy_generate -tanh_query -max_bptt 100 -switch -multilabel
```

A model trained in this way can be downloaded from https://drive.google.com/file/d/0B1ytQXPDuw7OaHZJZjVWd2N6R2M/view?usp=sharing

## Generation
Use the following commands to generate from the above models:

```
th box_train.lua -data roto-train.t7 -save_model roto_jc_rec_tvd -rnn_size 600 -word_vec_size 600 -enc_emb_size 600 -max_batch_size 16 -dropout 0.5 -feat_merge concat -pool mean -enc_layers 1 -enc_relu -report_every 50 -gpuid 1 -epochs 50 -learning_rate 1 -enc_dropout 0 -decay_update2 -layers 2 -copy_generate -tanh_query -max_bptt 100 -discrec -rho 1 -partition_feats -recembsize 600 -discdist 1 -train_from roto_jc_rec_tvd_epoch45_7.22.t7 -just_gen -beam_size 5 -gen_file roto_jc_rec_tvd-beam5_gens.txt
```

```
th box_train.lua -data roto-train.t7 -save_model roto_cc -rnn_size 600 -word_vec_size 600 -enc_emb_size 600 -max_batch_size 16 -dropout 0.5 -feat_merge concat -pool mean -enc_layers 1 -enc_relu -report_every 50 -gpuid 1 -epochs 100 -learning_rate 1 -enc_dropout 0 -decay_update2 -layers 2 -copy_generate -tanh_query -max_bptt 100 -switch -multilabel -train_from roto_cc_epoch34_7.44.t7 -just_gen -beam_size 5 -gen_file roto_cc-beam5_gens.txt
 ```

The beam size used in generation can be adjusted with the `-beam_size` argument. You can generate on the test data by supplying the `-test` flag.

## Misc/Utils
You can regenerate a pointer file with

```
python data_utils.py -mode ptrs -input_path ~/Documents/code/boxscore-data/rotowire/train.json -output_fi "my-roto-ptrs.txt"
```

## Information/Relation Extraction

### Creating Training/Validation Data
You can create a dataset for training or evaluating the relation extraction system as follows:

```
python data_utils.py -mode make_ie_data -input_path "../boxscore-data/rotowire" -output_fi "roto-ie.h5"
```

This will create files `roto-ie.h5`, `roto-ie.dict`, and `roto-ie.labels`.

### Evaluating Generated summaries
1. You can download the extraction models we ensemble to do the evaluation from this [link](https://drive.google.com/drive/u/1/folders/0B1ytQXPDuw7OdjBCUW50S2VIdDQ). There are six models in total, with the name pattern `*ie-ep*.t7`. Put these extraction models in the same directory as `extractor.lua`. (Note that `extractor.lua` hard-codes the paths to these saved models, so you'll need to change this if you want to substitute in new models.)

2. Once you've generated summaries, you can put them into a format the extraction system can consume as follows:

```
python data_utils.py -mode prep_gen_data -gen_fi roto_cc-beam5_gens.txt -dict_pfx "roto-ie" -output_fi roto_cc-beam5_gens.h5 -input_path "../boxscore-data/rotowire"
```

where the file you've generated is called `roto_cc-beam5_gens.txt` and the dictionary and labels files are in `roto-ie.dict` and `roto-ie.labels` respectively (as above). This will create a file called `roto_cc-beam5_gens.h5`, which can be consumed by the extraction system.

3. The extraction system can then be run as follows:

```
th extractor.lua -gpuid 1 -datafile roto-ie.h5 -preddata roto_cc-beam5_gens.h5 -dict_pfx "roto-ie" -just_eval
```

This will print out the **RG** metric numbers. (For the recall number, divide the 'nodup correct' number by the total number of generated summaries, e.g., 727). It will also generate a file called `roto_cc-beam5_gens.h5-tuples.txt`, which contains the extracted relations, which can be compared to the gold extracted relations.

4. We now need the tuples from the gold summaries. `roto-gold-val.h5-tuples.txt` and `roto-gold-test.h5-tuples.txt` have been included in the repo, but they can be recreated by repeating steps 2 and 3 using the gold summaries (with one gold summary per-line, as usual).

5. The remaining metrics can now be obtained by running:

```
python non_rg_metrics.py roto-gold-val.h5-tuples.txt roto_cc-beam5_gens.h5-tuples.txt
```
