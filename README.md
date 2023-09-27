# DDPM & DDIM PyTorch
### DDPM & DDIM re-implementation with various 

This code is the implementation code of the papers DDPM ([Denoising Diffusion Probabilistic Models](https://arxiv.org/abs/2006.11239)) 
and DDIM ([Denoising Diffusion Implicit Models](https://arxiv.org/abs/2010.02502)).

---
## Objective

Our code is mainly based on [denoising-diffusion-pytorch](https://github.com/lucidrains/denoising-diffusion-pytorch)
repository, which is a Pytorch implementation of the official Tensorflow code
[Denoising Diffusion Probabilistic Models](https://github.com/hojonathanho/diffusion). But we find that there are some
differences in the implementation especially in **U-net** structure. And further we find that Pytorch implementation 
version lacks some functionality for monitoring training and does not have inference code. So we decided to
re-implement the code such that it can be helpful for someone who is first to **Diffusion** models.

---
## Training

To train the diffusion model, first thing you have to do is to configure your training settings by making configuration
file. You can find some example inside the folder ```./config```. I will explain how to configure your training using
```./config/cifar10_example.yaml``` file and ```./config/cifar10_torch_example.yaml``` file.
Inside the ```cifar10_example.yaml``` you may find 4 primary section, ```type, unet, ddim, trainer```. 

We will first look at ```trainer``` section which is configured as follows.

```yaml
dataset: cifar10
batch_size: 128
lr: 0.0002
total_step: 600000
save_and_sample_every: 2500
num_samples: 64
fid_estimate_batch_size: 128
ddpm_fid_score_estimate_every: null
ddpm_num_fid_samples: null
tensorboard: true
clip: both
```
- ```dataset```: You can give Dataset name which is available by ```torchvision.datasets```. You can find some Datasets provided
by torchvision in this [website](https://pytorch.org/vision/stable/datasets.html). If you want to use torchvision's
Dataset just provide the dataset name, for example ```cifar10```. Currently, tested datasets are ```cifar10```. 
Or if you want to use custom datasets which you have prepared, you have to pass the path to the folder which is containing
images 


- ```batch_size, lr, total_step```: You can find the values used by DDPM author in the [DDPM paper](https://arxiv.org/abs/2006.11239)
Appendix B. total_step means total training step, for example DDPM author trained cifar10 model with 800K steps.


- ```save_and_sample_every```: The interval to which save the model and generated samples. For example in this case, 
for every 2500 steps trainer will save the model weights and also generate some sample images to visualize the training progress.


- ```num_samples```: When sampling the images evey ```save_and_sample_every``` steps, trainer will sample total ```num_samples``` 
images and save it to one large image containing each sampled images where one large image have (num_samples)**0.5 rows and
columns. So ```num_samples``` must be square number ex) 25, 36, 49, 64, ...


- ```fid_estimate_batch_size```: Batch size for sampling images for FID calculation. This batch size will be applied to
DDPM sampler as well as DDIM samplers. If you cannot decide the value, just setting this value equal to ```batch_size``` will
be fine.


- ```ddpm_fid_score_estimate_every```: Step interval for FID calculation using DDPM sampler. If set to null, FID score
will not be calculated with DDPM sampling. If you use DDPM sampling for FID calculation, i.e. setting this value other than null, 
it can be very time-consuming, ***so it is wise to set this value to null***, and use DDIM sampler for 
FID calculation (Using DDIM sampler is explained below). But anyway you can calculate FID score with DDPM sampler if you **insist**.


- ```ddpm_num_fid_samples```: Number of sampling images for FID calculation using DDPM sampler. If you set 
```ddpm_fid_score_estimate_every``` to null, i.e. not using DDPM sampler for FID calculation, then this value will
be just ignored.


- ```tensorboard```: If set to true, then you can monitor training progress such as loss, FID value, and sampled images,
during training, with the tensorboard.


- ```clip```: It must be one of [both, true, false]. 

---

Now we will look at ```type, unet``` section which is configured as follows.

```yaml
# ```./config/cifar10_example.yaml```
type: original
unet:
  dim: 64
  image_size: 32
  dim_multiply:
  - 1
  - 2
  - 2
  - 2
  attn_resolutions:
  - 16
  dropout: 0.1
  num_res_blocks: 2
```

```yaml
# ```./config/cifar10_torch_example.yaml```
type: torch
unet:
  dim: 64
  image_size: 32
  dim_multiply:
  - 1
  - 2
  - 2
  - 2
  full_attn:
  - false
  - true
  - false
  - false
  attn_heads: 4
  attn_head_dim: 32
```

-```type```: It must be one of [original, torch]. ***original*** will use U-net structure which was originally suggested by
Jonathan Ho. So it's structure will be the one used in [Denoising Diffusion Probabilistic Models](https://github.com/hojonathanho/diffusion)
which is an official version written in Tensorflow. ***torch*** will use U-net structure which was suggested by 
[denoising-diffusion-pytorch](https://github.com/lucidrains/denoising-diffusion-pytorch) which is a transcribed version of
official Tensorflow version. 

I have separated those two because there structure differs significantly. To name a few, following is the difference of
those two U-net structure.
1. official version use self Attention where the feature map resolution at each U-net level 
   is in ```attn_resolutions```. In the DDPM paper you can find that they used self Attention at the 16X16 resolution, 
   and this is why ```attn_resolutions``` is by default ```[16, ]```

   On the other hand, Pytorch transcribed version use Linear Attention and multi-head self Attention. They use
   multi-head self Attention at the U-net level where ```full_attn``` is true and Linear Attention at the 
   rest of the U-net level. So in this particular case, they used multi-head self Attention at the U-net level 1 (I will 
denote U-net level as 0, 1, 2, 3, ...) and the Linear Attention at the U-net level 0, 2, 3.