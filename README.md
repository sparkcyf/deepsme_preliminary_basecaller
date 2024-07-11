# Code for Preliminary Basecaller Training (Modified from https://github.com/marcpaga/basecalling_architectures/)

> [!NOTE]  
> This is a modified version of the code from [marcpaga/basecalling_architectures](https://github.com/marcpaga/basecalling_architectures/) to fullfill the task of training preliminary basecallers for the [DeepSME](https://github.com/sparkcyf/DeepSME).

## Installation
You need a **CUDA compatible GPU** (no ROCm / DirectML etc.) to train the models and to use them for basecalling. Install the required packages with the following command:

``` bash
conda env create -f environment.yml
```

## Training Preliminary Basecallers

``` python
python ./scripts/train_original.py \
--data-dir preliminary_datasets_dir \
--output-dir model_dir \
--model bonito \
--window-size 2000 \
--batch-size 128 \
--use-scaler \
--num-epochs 50 \
--starting-lr 0.0005 \
--overwrite
```

## Basecalling with Preliminary Basecallers

If you want to save the basecalling results to a fastq file, you can use the following command:

> [!TIP]  
> If you want to save the CTC data, you need to specify `CTC_TRACE_SAVE_FOLDER` environment variable to the folder where you want to save the CTC data.

``` python
python ./scripts/basecall_original.py \
--fast5-dir fast5_dir  \
--checkpoint model_dir/<best_checkpoint.pt> \
--model bonito \
--chunk-size 2000 \
--window-overlap 400 \
--batch-size 32 \
--output-file fastq_folder/basecalling_preliminary.fastq
```

# Original README
> # Deep learning architectures for basecalling
>
> This repository contains the code and scripts to train nanopore basecalling neural networks from R9.4.1 flow cells data.
> The code here was used on our benchmark of the latest network architectures: https://www.biorxiv.org/content/10.1101/2022.05.17.492272v2.
> In this work, we reimplemented the latest basecaller models in PyTorch.
>
> For information regarding the datasets, train/test splits and evaluation metrics please refer to: https://github.com/marcpaga/nanopore_benchmark
>
> ## Available models
>
> - Mincall: https://github.com/nmiculinic/minion-basecaller
> - CausalCall: https://github.com/scutbioinformatic/causalcall
> - URNano: https://github.com/yaozhong/URnano
> - SACAll: https://github.com/huangnengCSU/SACall-basecaller
> - CATCaller: https://github.com/lvxuan96/CATCaller
> - Bonito: https://github.com/nanoporetech/bonito
> - Halcyon: https://github.com/relastle/halcyon
>
> We also have models where we mix the CNN-RNN/Transformer-CTC/CRF between models, with the exception of Halcyon, because it is a Seq2Seq model. See below for more info.
>
> # Installation
>
> This code has been tested on python 3.7 only.
> ```
> git clone https://github.com/marcpaga/basecalling_architectures.git 
> cd basecalling_architectures
> python3 -m venv venv3
> source venv3/bin/activate
> pip install --upgrade pip
> pip install -r requirements.txt
> ```
> If you get dependency conflicts, try to manually install `seqdist==0.0.3` and the manually upgrade to `numpy==1.21.5`, as was found as a solution in this [issue](https://github.com/marcpaga/basecalling_architectures/issues/4#issuecomment-1645361245).
>
>
> # Getting started
>
> ## Data download
>
> This repository contains a very small demo dataset that can be used to check if the scripts are running correctly. The examples below use this demo data. Before downloading and processing the data, you might want to make sure that you can run all the examples without errors with the demo data.
>
> For public data downloads refer to: https://github.com/marcpaga/nanopore_benchmark.
> There you can find download links as well as train/test splits to benchmark your model with the same data as we did in our benchmark.
>
> If you have your own data, skip this step.
>
> ## Data processing
>
> There's two main data processing steps:
>
> ### Annotate the raw data with the reference/true sequence
>
> For this step we used Tombo, which models the expected average raw signal level and aligns the expected signal with the measured signal. See their [documentation](https://nanoporetech.github.io/tombo/resquiggle.html) for detailed info on how to use it.
>
> After installation of tombo (you have to install it in a different environment, as it is not compatible with the training environment), using the example data in this repository, you should be able to run the following:
>
> ```
> tombo \
> resquiggle demo_data/fast5 \
> demo_data/Lambda_phage.fna \
> --processes 2 \
> --dna \
> --num-most-common-errors 5 \
> --ignore-read-locks
> ```
>
> ### Chunk the raw signal and save it numpy arrays
>
> In this step, we take the raw signal and splice it into segments of a fixed length so that they can be fed into the neural network.
>
> This can be done by running the following script:
>
> ```
> python ./scripts/data_prepare_numpy.py \
> --fast5-dir  ./demo_data/fast5 \
> --output-dir  ./demo_data/nn_input \
> --total-files  1 \
> --window-size 2000 \
> --window-slide 0 \
> --n-cores 2 \
> --verbose
> ```
>
> ## Model training
>
> In this step we fed all the data we prepared (in numpy arrays), and train the model.
>
> We can train the models as original architectures:
>
> ```
> python ./scripts/train_original.py \
> --data-dir ./demo_data/nn_input \
> --output-dir ./demo_data/trained_model_original \
> --model causalcall \
> --window-size 2000 \
> --batch-size 64
> ```
>
> Or mix architectures together with this other script:
>
> ```
> python ./scripts/train_comb.py \
> --data-dir ./demo_data/nn_input \
> --output-dir ./demo_data/trained_model_comb \
> --cnn-type causalcall \
> --encoder-type bonitorev \
> --decoder-type crf \
> --use-connector \
> --window-size 2000 \
> --batch-size 64
> ```
>
> For additional information on other parameters check `./scripts/benchmark/train_original.py --help` or `./scripts/benchmark/train_comb.py --help`
>
> ## Model basecalling
>
> Once a model has been trained, it can be used for basecalling. Here's an example command with the demo data:
>
> ```
> python ./scripts/basecall_original.py \
> --model bonito \
> --fast5-dir ./demo_data/fast5 \
> --checkpoint ./demo_data/properly_trained_bonito/bonito_checkpoint.pt \
> --output-file ./demo_data/demo_basecalls.fastq
> ```
>
> For additional information on other parameters check `./scripts/benchmark/basecall_original.py --help` or `./scripts/benchmark/basecall_comb.py --help`
>
> ## Model evaluation
>
> For model evaluation refer to: https://github.com/marcpaga/nanopore_benchmark. There you will find information on how to evaluate the basecalls againts their reference sesquences.
>
> # Computational requirements
>
> A CUDA compatible GPU is required to train models and to use them for basecalling.