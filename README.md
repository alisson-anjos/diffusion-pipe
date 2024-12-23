# diffusion-pipe
A pipeline parallel training script for diffusion models.

Currently supports Flux, LTX-Video, and HunyuanVideo.

**Work in progress and highly experimental.** It is unstable and not well tested. Things might not work right.

## Features
- Pipeline parallelism, for training models larger than can fit on a single GPU
- Full fine tune support for:
    - Flux
- LoRA support for:
    - Flux, LTX-Video, HunyuanVideo
- Useful metrics logged to Tensorboard
- Compute metrics on a held-out eval set, for measuring generalization
- Training state checkpointing and resuming from checkpoint
- Efficient multi-process, multi-GPU pre-caching of latents and text embeddings
- Easily add support for new models by implementing a single subclass


## Windows support
There are reports that it doesn't work on Windows. This is because Deepspeed only has [partial Windows support](https://github.com/microsoft/DeepSpeed/blob/master/blogs/windows/08-2024/README.md). However, at least one user was able to get it running and training successfully on Windows Subsystem for Linux, specifically WSL 2. If you must use Windows I recommend trying WSL 2.


## Installing
Clone the repository:
```
git clone --recurse-submodules https://github.com/tdrussell/diffusion-pipe
```

If you alread cloned it and forgot to do --recurse-submodules:
```
git submodule init
git submodule update
```

Install Miniconda: https://docs.anaconda.com/miniconda/

Create the environment:
```
conda create -n diffusion-pipe python=3.12
conda activate diffusion-pipe
```

Install nvcc: https://anaconda.org/nvidia/cuda-nvcc. Probably try to make it match the CUDA version that was installed on your system with PyTorch.

Install the dependencies:
```
pip install -r requirements.txt
```

## Training
**Start by reading through the config files in the examples directory.** Almost everything is commented, explaining what each setting does.

Once you've familiarized yourself with the config file format, go ahead and make a copy and edit to your liking. At minimum, change all the paths to conform to your setup, including the paths in the dataset config file.

Launch training like this:
```
NCCL_P2P_DISABLE="1" NCCL_IB_DISABLE="1" deepspeed --num_gpus=1 train.py --deepspeed --config examples/hunyuan_video.toml
```
RTX 4000 series needs those 2 environment variables set. Other GPUs may not need them. You can try without them, Deepspeed will complain if it's wrong.

If you enabled checkpointing, you can resume training from the latest checkpoint by simply re-running the exact same command but with the ```--resume_from_checkpoint``` flag.

## Output files
A new directory will be created in ```output_dir``` for each training run. This contains the checkpoints, saved models, and Tensorboard metrics. Saved models/LoRAs will be in directories named like epoch1, epoch2, etc. Deepspeed checkpoints are in directories named like global_step1234. These checkpoints contain all training state, including weights, optimizer, and dataloader state, but can't be used directly for inference. The saved model directory will have the safetensors weights, PEFT adapter config JSON, as well as the diffusion-pipe config file for easier tracking of training run settings.

## VRAM requirements
### Flux
Flux doesn't currently support training a LoRA on a fp8 base model (if you want this, PRs are welcome :) ). So you need to use a >24GB GPU, or use pipeline_stages=2 or higher with multiple 24GB cards. With four 24GB GPUs, you can even full finetune Flux with the right techniques (see the train.py code about gradient release and the custom AdamW8bitKahan optimizer).

### HunyuanVideo
HunyuanVideo supports fp8 transformer. The example config file will train a HunyuanVideo LoRA, on images only, in well under 24GB of VRAM. You can probably bump the resolution to 1024x1024 or higher.

Video uses A LOT more memory. I was able to train a rank 32 LoRA on 512x512x33 sized videos in just under 23GB VRAM usage. Pipeline parallelism will help a bit if you have multiple GPUs, since the model weights will be further divided among them (but it doesn't help with the huge activation memory use of videos). Long term I want to eventually implement ring attention and/or Deepspeed Ulysses for parallelizing the sequence dimension across GPUs, which should greatly help for training on videos.

### LTX-Video
I've barely done any training on LTX-Video. The model is much lighter than Hunyuan, and the latent space more compressed, so it uses less memory. You can train loras even on video at a reasonable length (I forgot exactly what it was) on 24GB.

## Parallelism
This code uses hybrid data- and pipeline-parallelism. Set the ```--num_gpus``` flag appropriately for your setup. Set ```pipeline_stages``` in the config file to control the degree of pipeline parallelism. Then the data parallelism degree will automatically be set to use all GPUs (number of GPUs must be divisible by pipeline_stages). For example, with 4 GPUs and pipeline_stages=2, you will run two instances of the model, each divided across two GPUs. Note that due to a weird bug I'm still investigating, pipeline_stages>1 doesn't work with HunyuanVideo.

## Pre-caching
Latents and text embeddings are cached to disk before training happens. This way, the VAE and text encoders don't need to be kept loaded during training. The Huggingface Datasets library is used for all the caching. Cache files are reused between training runs if they exist. All cache files are written into a directory named "cache" inside each dataset directory.

This caching also means that training LoRAs for text encoders is not currently supported.

Two flags are relevant for caching. ```--cache_only``` does the caching flow, then exits without training anything. ```--regenerate_cache``` forces cache regeneration. If you edit the dataset in-place (like changing a caption), you need to force regenerate the cache (or delete the cache dir) for the changes to be picked up.

## HunyuanVideo LoRAs
HunyuanVideo doesn't have an official Diffusers integration yet, and as such it doesn't have an official LoRA format. This script outputs the LoRA using the typical Diffusers convention, i.e. the state_dict keys are all prefixed with "transformer.". This will work with the latest version of the ComfyUI HunyuanVideoWrapper extension, and is likely to work with Diffusers whenever HunyuanVideo is officially integrated. Make sure the HunyuanVideoWrapper extension is fully updated, and use the "HunyuanVideo Lora Select" node.

## Gradio Interface

![preview gradio](/preview-gradio.png)

### Features

- Web UI (Gradio) for configuring and executing LoRA training.
- Optional NVIDIA GPU support for accelerated training.
- Ability to map model and output directories from the host system into the container.
- Optional automatic download of required models upon first initialization.

### Prerequisites

- **Docker:**  
  Install Docker for your platform by following the official documentation:  
  [Get Docker](https://docs.docker.com/get-docker/)

- **GPU Support (optional):**  
  To utilize GPU acceleration (NVIDIA):
  - **Linux:** Install the [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html) and ensure your NVIDIA drivers are set up.
  - **Windows/macOS:** Check respective Docker and NVIDIA documentation for GPU passthrough (e.g., WSL2 on Windows). If GPU support is not available, you can run the container without `--gpus all`.

### How to Run

#### Basic Run Command

```bash
docker run --gpus all -it -p 7860:7860 -p 8888:8888 alissonpereiraanjos/diffusion-pipe-interface:latest
```

- `--gpus all`: Enables GPU support if configured.  
- `-p 7860:7860`: Exposes port 7860 so you can access the Gradio UI at `http://localhost:7860`.
- `-p 8888:8888`: (optional) Exposes port 8888 so you can access the Jupyter Lab UI at `http://localhost:8888`.

If you do not have or do not want GPU support, omit `--gpus all`.

#### Mapping Directories for Models and Output

You can mount host directories to store models and training outputs outside the container:

```bash
docker run --gpus all -it \
   -v /path/to/models:/workspace/models \
  -v /path/to/output:/workspace/output \
  -v /path/to/datasets:/workspace/datasets \
  -v /path/to/configs:/workspace/configs \
  -p 8888:8888 \
  -p 7860:7860 \
  alissonpereiraanjos/diffusion-pipe-interface:latest
```

- Replace `/path/to/models` and `/path/to/output` with your desired host directories.
- On Windows, for example:
  ```bash
  docker run --gpus all -it \
    -v D:\AI\hunyuan\models:/workspace/models \
    -v D:\AI\hunyuan\output:/workspace/output \
    -v D:\AI\hunyuan\datasets:/workspace/datasets \
    -v D:\AI\hunyuan\configs:/workspace/configs \
    -p 8888:8888 \
    -p 7860:7860 \
    alissonpereiraanjos/diffusion-pipe-interface:latest
  ```

#### Controlling Model Downloads

By default, the container downloads the required models during the first initialization. If you already have the models in `/models` and want to skip automatic downloads, set the `DOWNLOAD_MODELS` environment variable to `false`:

```bash
docker run --gpus all -it \
  -v /path/to/models:/workspace/models \
  -v /path/to/output:/workspace/output \
  -v /path/to/datasets:/workspace/datasets \
  -v /path/to/configs:/workspace/configs \
  -p 8888:8888 \
  -p 7860:7860 \
  -e DOWNLOAD_MODELS=false \
  alissonpereiraanjos/diffusion-pipe-interface:latest
```

#### Running in Detached Mode

If you prefer to run the container in the background without an interactive terminal, use `-d`:

```bash
docker run --gpus all -d \
  -v /path/to/models:/workspace/models \
  -v /path/to/output:/workspace/output \
  -v /path/to/datasets:/workspace/datasets \
  -v /path/to/configs:/workspace/configs \
  -p 8888:8888 \
  -p 7860:7860 \
  -e DOWNLOAD_MODELS=false \
  alissonpereiraanjos/diffusion-pipe-interface:latest
```

Access the UI (gradio) at `http://localhost:7860`.
Access the Jupiter lab UI at `http://localhost:8888`.

#### Summary of Options

- `-v /host/path:/container/path`: Mount host directories into the container.
- `-p host_port:container_port`: Map container ports to host ports.
- `-e VARIABLE=value`: Set environment variables.
  - `DOWNLOAD_MODELS=false`: Skips downloading models inside the container.
- `--gpus all`: Enables GPU support if available.
- `-it`: Start in interactive mode (useful for debugging).
- `-d`: Start in detached mode (runs in the background).

Use these options to tailor the setup to your environment and requirements.

### Running on RunPod

If you prefer to use [RunPod](https://runpod.io/), you can quickly deploy an instance based on this image by using the following template link:

[Deploy on RunPod](https://runpod.io/console/deploy?template=t46lnd7p4b&ref=8t518hht)

This link takes you to the RunPod console, allowing you to set up a machine directly with the provided image. Just configure your GPU, volume mounts, and environment variables as needed.

## Extra
You can check out my [qlora-pipe](https://github.com/tdrussell/qlora-pipe) project, which is basically the same thing as this but for LLMs.

