# Tencent Advertising Algorithm Competition Platform Skill

This skill summarizes the required workflow and file conventions for the Tencent Advertising Algorithm Competition training, model release, and evaluation platform. Use it when writing, checking, or modifying competition code for Codex.

## 1. Overall Workflow

The platform workflow is:

```text
Create training task
→ Platform runs run.sh
→ Training script saves checkpoints under TRAIN_CKPT_PATH/global_step...
→ Publish one checkpoint as a released model
→ Create evaluation task
→ Platform runs infer.py:main()
→ infer.py loads model from MODEL_OUTPUT_PATH
→ infer.py reads test data from EVAL_DATA_PATH
→ infer.py writes predictions.json to EVAL_RESULT_PATH
→ Platform scores predictions.json
```

Priority rule: first make the full platform pipeline run successfully, then optimize the model.

## 2. Training Task Requirements

### Required Entry File

A training task must include a shell entry script named exactly:

```bash
run.sh
```

The platform automatically executes `run.sh` when the training task starts.

Example:

```bash
#!/bin/bash
python train.py
```

### Training Task Creation Steps

1. Fill in `Job Name`.
2. Fill in `Job Description`.
3. Upload local scripts via `Local Upload`, or create scripts online via `New Script`.
4. Confirm and click `Submit`.

## 3. Training Environment Variables

The platform passes training paths into the container as environment variables.

| Variable | Meaning |
|---|---|
| `USER_CACHE_PATH` | User cache path, quota 20GB. Available in both training and evaluation. Can be used to share files between stages. |
| `TRAIN_DATA_PATH` | Path to training dataset. |
| `TRAIN_CKPT_PATH` | Path where model checkpoints must be saved. |
| `TRAIN_TF_EVENTS_PATH` | Path for TensorBoard event files. |

Read environment variables in shell:

```bash
${TRAIN_DATA_PATH}
```

Read environment variables in Python:

```python
import os

train_data_path = os.environ.get("TRAIN_DATA_PATH")
ckpt_path = os.environ.get("TRAIN_CKPT_PATH")
tf_events_path = os.environ.get("TRAIN_TF_EVENTS_PATH")
```

Do not hardcode local paths such as `./data/train.csv` unless they are derived from the platform-provided paths.

## 4. Training Output Requirements

Model weights must be saved under the directory specified by:

```text
TRAIN_CKPT_PATH
```

Checkpoint directories must be prefixed with:

```text
global_step
```

Otherwise, the platform may not recognize the checkpoint.

Valid examples:

```text
global_step1
global_step10
global_step20.lr=0.001.layer=2.head=1.hidden=50.maxlen=200
```

Directory name rules:

- Must not exceed 300 characters.
- May only contain letters `a-z`, `A-Z`, numbers `0-9`, underscores `_`, hyphens `-`, equal signs `=`, and periods `.`.
- Avoid Chinese characters, spaces, slashes, colons, brackets, and other special symbols.

## 5. TensorBoard Metrics

The platform supports TensorBoard scalar metrics only.

Examples of supported scalar metrics:

```text
loss
auc
accuracy
```

For PyTorch:

```python
import os
from torch.utils.tensorboard import SummaryWriter

writer = SummaryWriter(os.environ.get("TRAIN_TF_EVENTS_PATH"))
writer.add_scalar("loss/train", loss_value, global_step)
writer.add_scalar("auc/val", auc_value, global_step)
```

## 6. Model Release Workflow

After training is complete:

1. Go to `Model Training`.
2. Open the training task.
3. Click `Instances`.
4. Click `Output`.
5. Select the checkpoint to release.
6. Click `Publish`.
7. Fill in `Model Name` and `Model Description`.
8. Click `Submit`.
9. Confirm that `Publish Status` changes to `Released`.

Only released models can be selected for evaluation.

## 7. Evaluation Task Requirements

### Required Entry File

The evaluation task must include an inference entry script named exactly:

```python
infer.py
```

`infer.py` must contain a no-argument `main()` function:

```python
def main():
    ...
```

Do not define it as:

```python
def main(args):
    ...
```

The platform executes `infer.py` and expects `main()` to run without arguments.

### Supporting Files

You may upload supporting modules imported by `infer.py`, such as:

```text
dataset.py
model.py
utils.py
```

All uploaded inference files will be placed under:

```text
EVAL_INFER_PATH
```

### Script Size Limit

The total size of all uploaded scripts is limited to:

```text
100 MB
```

### Daily Evaluation Limit

Each team may submit up to 3 evaluation tasks per AOE day.

Failed or stopped tasks are not counted.

## 8. Installing Evaluation Dependencies

If dependencies need to be installed before inference starts, upload a script named exactly:

```bash
prepare.sh
```

`prepare.sh` runs automatically before inference starts, inside the activated conda environment.

Example:

```bash
#!/bin/bash

# This script runs before inference starts.
# It runs inside the activated conda environment.
# Install only necessary dependencies.

# Example:
# pip install some_package
# conda install -y some_package
```

Recommendation: avoid extra dependency installation when possible. Platform dependency installation can be slow or unstable.

## 9. Evaluation Environment Variables

The platform passes evaluation paths into the container as environment variables.

| Variable | Meaning |
|---|---|
| `USER_CACHE_PATH` | User cache path, quota 20GB. Available in both training and evaluation. |
| `MODEL_OUTPUT_PATH` | Path to the released model output. Load model/checkpoint files from here. |
| `EVAL_DATA_PATH` | Path to test data directory for inference. |
| `EVAL_RESULT_PATH` | Output path for intermediate results and final `predictions.json`. |
| `EVAL_INFER_PATH` | Directory containing user-uploaded inference scripts. |

Read them in Python:

```python
import os

model_output_path = os.environ.get("MODEL_OUTPUT_PATH")
eval_data_path = os.environ.get("EVAL_DATA_PATH")
eval_result_path = os.environ.get("EVAL_RESULT_PATH")
eval_infer_path = os.environ.get("EVAL_INFER_PATH")
```

## 10. Evaluation Output Requirements

`infer.py` must generate a file named exactly:

```text
predictions.json
```

The file must be saved under:

```text
EVAL_RESULT_PATH
```

Example save path:

```python
import os
import json

result_path = os.environ.get("EVAL_RESULT_PATH")
save_path = os.path.join(result_path, "predictions.json")

with open(save_path, "w", encoding="utf-8") as f:
    json.dump(result_dict, f, ensure_ascii=False)
```

## 11. predictions.json Format

The JSON file must contain a top-level field named:

```json
"predictions"
```

`predictions` must be a mapping from `user_id` string to predicted conversion probability.

Expected format:

```json
{
  "predictions": {
    "user_001": 0.8732,
    "user_002": 0.1245,
    "user_003": 0.5621
  }
}
```

Rules:

- Each key must be a valid `user_id` from the test dataset.
- Each key must be a string.
- Each value must be a float.
- Each value must be in the range `[0, 1]`.
- Values should represent predicted conversion probability.
- Missing or extra `user_id`s may affect the final score.
- Do not output logits, labels, ranks, or raw scores outside `[0, 1]`.

Bad example:

```json
{
  "predictions": {
    "user_001": 3.8
  }
}
```

Good example:

```json
{
  "predictions": {
    "user_001": 0.8732
  }
}
```

## 12. Evaluation Status Meanings

| Status | Meaning |
|---|---|
| `Pending` | Task has been submitted and is queued. |
| `Waiting for Inference Resources` | Waiting for available compute resources. |
| `Inference Running` | The platform is executing `infer.py`. |
| `Waiting for Evaluation Resources` | Inference completed; waiting for scoring resources. |
| `Evaluation Running` | The platform is scoring predictions. |
| `Success` | Evaluation completed; score can be viewed. |
| `Failed` | Evaluation failed; check logs. |

## 13. Platform Hardware Environment

| Resource | Specification |
|---|---|
| Computing Power | 20% of a single GPU |
| GPU Memory | 19 GiB |
| CPU Cores | 9 |
| Memory | 55 GiB |

The platform does not provide a full exclusive GPU. Optimize memory and runtime accordingly.

## 14. Platform Software Environment

| Software | Version |
|---|---|
| OS | Ubuntu 22.04 |
| CUDA | 12.6 |
| cuDNN | 9.5.1 |
| cuBLAS | 12.6.3.3 |
| NCCL | 2.26.2 + CUDA 12.6 |
| conda | 26.1.1 |
| Python | 3.10.20 |

Code should be compatible with Python 3.10 and the available CUDA environment.

## 15. Minimal Recommended File Structure

### Training Upload

```text
run.sh
train.py
model.py
dataset.py
utils.py
```

### Evaluation Upload

```text
infer.py
model.py
dataset.py
utils.py
prepare.sh    # optional
```

## 16. Minimal Training Skeleton

```bash
# run.sh
#!/bin/bash
python train.py
```

```python
# train.py
import os


def main():
    train_data_path = os.environ.get("TRAIN_DATA_PATH")
    train_ckpt_path = os.environ.get("TRAIN_CKPT_PATH")

    # TODO: load training data from train_data_path
    # TODO: train model

    # Save checkpoint under a directory starting with global_step
    save_dir = os.path.join(train_ckpt_path, "global_step1")
    os.makedirs(save_dir, exist_ok=True)

    # TODO: save model weights/files into save_dir


if __name__ == "__main__":
    main()
```

## 17. Minimal Evaluation Skeleton

```python
# infer.py
import os
import json


def main():
    model_output_path = os.environ.get("MODEL_OUTPUT_PATH")
    eval_data_path = os.environ.get("EVAL_DATA_PATH")
    eval_result_path = os.environ.get("EVAL_RESULT_PATH")

    # TODO: load released model from model_output_path
    # TODO: load test data from eval_data_path
    # TODO: produce probabilities for every valid user_id in test data

    predictions = {
        "user_001": 0.5
    }

    output = {"predictions": predictions}

    os.makedirs(eval_result_path, exist_ok=True)
    save_path = os.path.join(eval_result_path, "predictions.json")

    with open(save_path, "w", encoding="utf-8") as f:
        json.dump(output, f, ensure_ascii=False)


if __name__ == "__main__":
    main()
```

## 18. Pre-submit Checklist

Before training submission:

- [ ] `run.sh` exists and is named exactly `run.sh`.
- [ ] `run.sh` can start training correctly.
- [ ] Training reads data from `TRAIN_DATA_PATH`.
- [ ] Checkpoints are saved under `TRAIN_CKPT_PATH`.
- [ ] Checkpoint directory starts with `global_step`.
- [ ] Checkpoint directory name uses only allowed characters.
- [ ] TensorBoard metrics, if used, are scalar metrics only.

Before model release:

- [ ] Training task has produced checkpoint output.
- [ ] Target checkpoint directory starts with `global_step`.
- [ ] Model is published successfully.
- [ ] Publish status is `Released`.

Before evaluation submission:

- [ ] `infer.py` exists and is named exactly `infer.py`.
- [ ] `infer.py` contains `def main():` with no arguments.
- [ ] `infer.py` loads model files from `MODEL_OUTPUT_PATH`.
- [ ] `infer.py` reads test data from `EVAL_DATA_PATH`.
- [ ] `infer.py` writes `predictions.json` to `EVAL_RESULT_PATH`.
- [ ] `predictions.json` has top-level key `predictions`.
- [ ] Each prediction key is a valid test `user_id` string.
- [ ] Each prediction value is a float probability in `[0, 1]`.
- [ ] No missing or extra `user_id`s if the test user list is available.
- [ ] Total uploaded inference scripts are under 100 MB.

## 19. Common Failure Causes

- `run.sh` missing or incorrectly named.
- `infer.py` missing or incorrectly named.
- `main()` in `infer.py` has arguments.
- `predictions.json` is saved to the wrong directory.
- `predictions.json` has the wrong filename.
- JSON top-level key is not `predictions`.
- Prediction values are logits or labels instead of probabilities.
- Prediction values are outside `[0, 1]`.
- User IDs are missing, duplicated, invalid, or extra.
- Checkpoint directory does not start with `global_step`.
- Model was trained but not published.
- Published model status is not `Released`.
- Extra dependencies fail to install in `prepare.sh`.

## 20. Codex Instruction Summary

When generating or modifying code for this platform, Codex should:

1. Respect exact required filenames: `run.sh`, `infer.py`, and optionally `prepare.sh`.
2. Use environment variables instead of hardcoded absolute paths.
3. Save training checkpoints under `TRAIN_CKPT_PATH/global_step...`.
4. Make evaluation entrypoint `infer.py` expose a no-argument `main()` function.
5. Save final evaluation output to `EVAL_RESULT_PATH/predictions.json`.
6. Ensure `predictions.json` follows `{ "predictions": { user_id: probability } }`.
7. Keep probabilities in `[0, 1]`.
8. Avoid unnecessary dependencies and large uploaded files.
9. Prefer simple, robust baseline code before complex model optimization.
