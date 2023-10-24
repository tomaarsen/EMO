# EMO
<p align="center">
<img src="./emo_alpaca_bg.png" width="200" height="200">
</p>

This is the public codebase for arxiv paper: [EMO: Earth Mover Distance Optimization for Auto-regressive Language Modeling](https://arxiv.org/abs/2310.04691).

## Abstract
Neural language models are probabilistic models of human text. They are predominantly trained using maximum likelihood estimation (MLE), which is equivalent
to minimizing the forward cross-entropy between the empirical data distribution and the model distribution. However, various degeneration phenomena are still
widely observed when decoding from the distributions learned by such models. We establish that the forward cross-entropy is suboptimal as a distance metric for aligning human and model distribution due to its (1) recall-prioritization (2) negative diversity ignorance and (3) train-test mismatch. In this paper, we propose Earth Mover Distance Optimization (EMO) for auto-regressive language modeling. EMO capitalizes on the inherent properties of earth mover distance to address the aforementioned challenges. Due to the high complexity of direct computation, we further introduce a feasible upper bound for EMO to ease end-to-end training. Upon extensive evaluation of language models trained using EMO and MLE. We find that EMO demonstrates a consistently better language modeling performance than MLE across domains. Moreover, EMO demonstrates noteworthy enhancements in downstream performance with minimal fine-tuning on merely 25,000 sentences. This highlights the tremendous potential of EMO as a lightweight calibration method for enhancing large-scale pre-trained language models.

# Usage
## Standalone Package
We provide PyPi package of EMO as a easy-to-use loss function. Before install EMO, make sure you have installed `torch`.
```bash
pip install EMOLoss==0.0.1
```
### Examples
EMO requires three input fields, namely logits, labels, and cost_embedding:
```python
import torch
from emo import EMOLoss
"""
Signature of EMOLoss
Args:
    logits (Tensor, requried): the output logits after lm_head, before applying softmax
    labels (Tensor, required): the ground truth next tokens
    cost_embedding (Tensor, required): the cost embedding used to compute the transport cost between individual pairs of tokens
    ignore_index (Tensor, optional): usually set to -100 as in nn.CrossEntropyLoss
    mode (Int, optional): 1 by default, it means putting more weight on the MLE loss. Setting mode=2 will put more emphasis on EMO loss. 
Shape:
    - logits: (batch_size, seq_len, vocab_size) 
    - labels: (batch_size, seq_len)
    - cost_embedding: (vocab_size, hidden_size)
"""
logits = torch.rand(32, 1024, 32000)
labels = torch.ones(32, 1024)
cost_embedding = torch.rand(32000, 4096)
emo_loss = EMOLoss(logits, labels, cost_embedding, ignore_index=-100)
```
The `cost_embedding` must share the same vocabulary size as `logits`, e.g., 32000 for LLaMa. However, the hidden size of `cost_embedding` is not required to be identical to the model you want to train.
## Setup
We recommend using `python>=3.10.0`, `torch>=2.0.1`, `transformers>=4.30.0`.
```bash
git clone https://github.com/DRSY/EMO.git
cd EMO
pip install -r requirements.txt
```
## Code Structure
This repository provide training scripts for three different scenarios, i.e., language modeling, continual fine-tuning, and instruction tuning, as discussed in the paper. Detailed instructions for each scenario are described in the following sections.
```
.
├── README.md
├── accelerate_configs
│   └── accelerate_config_0.yaml
├── continual_finetuning
│   ├── emo_llama.py
│   ├── finetune_fsdp.sh
│   ├── finetune_lora.sh
│   ├── icl.py
│   ├── llama_flash_attn_monkey_patch.py
│   ├── merge.sh
│   ├── merge_lora.py
│   ├── run_clm_trainer_emo.py
│   └── run_clm_trainer_emo_fsdp.py
├── instruction_tuning
│   ├── alpaca_gpt4_data.json
│   ├── emo_llama.py
│   ├── flash_attention_patch.py
│   ├── train.py
│   └── train_emo_alpaca_gpt4.sh
├── language_modeling
│   ├── gpt2.py
│   ├── run_lm.py
│   ├── run_lm_gpt2.sh
│   └── test_utils.py
```
## 🏫 Language Modeling Experiments
The core code and scripts for language modeling experiments in the paper are located at [language_modeling](./language_modeling/). Model file that implements various training objective can be found at [gpt2.py](./language_modeling/gpt2.py).Training hyper-parameters can be adjusted in [run_lm_gpt2.sh](./language_modeling/run_lm_gpt2.sh). The argument "mode" specifies the training objective(`mle|mixce|tvd|emo`).
```bash
cd language_modeling
bash run_lm_gpt2.sh
```
We use [Mauve](https://github.com/krishnap25/mauve) as the primary evaluation metrics, make sure you install it before running the above script.

## 📑 NLU Experiments
Scripts related to continual fine-tuning and downstream NLU evaluations are located under [continual_finetuning](./continual_finetuning/).
```bash
cd continual_finetuning
```
### Run continual fine-tuning on WikiText-103
The core script for lightweight continual fine-tuning on a single GPU using LoRA is named [finetune_lora.sh](./finetune_lora.sh). Training hyper-parameters are defined in the script and can be adjusted as needed.
```bash
bash finetune_lora.sh MODEL_PATH OUTPUT_PATH
```
MODEL_PATH points to the model name on HuggingFace or path to a local directory. OUTPUT_PATH specifies the output directory.
if the model is fine-tuned using LoRA, we need to first merge the trained LoRA weights into the original model checkpoint.
```bash
bash merge.sh OUTPUT_PATH MERGED_PATH
```
Specify your desired path for saving the merged model checkpoint at MERGED_PATH.

The core script for lightweight continual fine-tuning in a distributed setting using FSDP with FP16 mixed-precision training is named [finetune_fsdp.sh](./finetune_fsdp.sh). Training hyper-parameters are defined in the script and can be adjusted as needed.
```bash
bash finetune_fsdp.sh MODEL_PATH OUTPUT_PATH
```
### Run downstream tasks using few-shot in-context learning
The fine-tuned model can be evaluated on downstream natural language understanding tasks using few-shot in-context learning. Before running evaluation, make sure you have installed OpenICL:
```bash
git clone https://github.com/Shark-NLP/OpenICL
cd OpenICL
pip install -e .
```
Afterwards, we can run evaluation using the following command:
```bash
CUDA_VISIBLE_DEVICES=0, python icl.py --model_path OUTPUT_PATH/MERGED_PATH
```
> **Note**
> you may have to modify the model initialization part of OpenICL in order to run inference in torch.float16 data type.

## 📚 Instruction-Tuning
EMO is also applicable in supervised instruction-tuning stage. We provide distributed training script(FSDP full fine-tuning using 4 GPUs) in [instruction_tuning](./instruction_tuning/) folder. Run the following command to launch training of specified model using the alpaca-gpt4 dataset:
```bash
cd instruction_tuning
bash train_emo_alpaca_gpt4.sh MODEL_PATH OUTPUT_DIR
```
Training hyper-parameters such as training objective(`mle|emo`), training epochs, and global batch size are defined in [train_emo_alpaca_gpt4.sh](./instruction_tuning/train_emo_alpaca_gpt4.sh) and are kept the same as in Stanford Alpaca codebase, feel free to adjust them as needed.

## 🌐 Acknowledgements
+ Evaluation on NLU tasks is implemented using [OpenICL](https://github.com/Shark-NLP/OpenICL).
+ Instruction-tuning code is adapted from [Stanford Alpaca](https://github.com/tatsu-lab/stanford_alpaca).
+ Implementation of baselines are based on:
  + [TaiLr](https://github.com/thu-coai/TaiLr)
  + [MixCE](https://github.com/bloomberg/mixce-acl2023)


## Citation
If you find that our paper or code useful, please cite the paper as follows:
```latex
@misc{ren2023emo,
      title={EMO: Earth Mover Distance Optimization for Auto-Regressive Language Modeling}, 
      author={Siyu Ren and Zhiyong Wu and Kenny Q. Zhu},
      year={2023},
      eprint={2310.04691},
      archivePrefix={arXiv},
      primaryClass={cs.CL}
}
```
