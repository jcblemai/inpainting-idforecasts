# Influpaint : Inpainting denoising diffusion probabilistic models for infectious disease (influenza) forecasting
* **authors** Joseph Lemaitre, Justin Lessler
* **affiliation** The University of North Carolina at Chapel Hill




**⚠️⚠️⚠️ The description below is now outdated, please wait for the new one (we use CoPaint instead of REpaint for inpainting) ⚠️⚠️⚠️**

volta-gpu (16G) from longleaf don't have enough gpu-mem neede a100-gpu

## Introduction

Denoising Diffusion Probabilistic Models (DDPM) are generative models like generative adversarial networks, autoregressive models, and variational auto-encoder. While slow, they can generate high quality samples with good diversity. They work wonderfully well for image synthesis, see [openAI DALL-E 2](https://openai.com/dall-e-2/). These model has been described in [1] and [2].

Here we treat influenza epidemic curves as images (axes are time and location, a pixel value is e.g incident hospitalization), and we generate synthetic epidemic trajectories using “stock” DDPM algorithms.

### Denoising Diffusion Probabilistic Models (DDPM)
DDPM consists of two transforms, backward and forward, in diffusion time:
* Forward: Markov chain that gradually adds noise to the data until the signal is destroyed;
* Backward: A trained neural network that denoises the image step by step.

We train the neural network using forward transform samples from the dataset, and we sample by transforming random gaussian noise with the backward transform.

<img width="1345" alt="image" src="https://user-images.githubusercontent.com/7485811/233663274-fb44d45c-52f2-4b3f-82f2-5b596c180e99.png">

### Neural Net Architecture

_(note that this section is updated less often than the code and might be outdated as we frequently improve the model)_

This architecture was built from trial & error (e.g, ConvNext blocks yield unsatisfying results) and is empirically satisfying. We use as a Forward process (or noise scheduler) a linear beta-schedule with 200 time-steps. The neural network for the backward transform is a  U-net with a Wide ResNet block, Attention module, Group normalization, and Residual connection. 

We use a Sinusoidal time-step embedding to integrate time-step information into the sample (as the neural network weights are the same for any diffusion time).

This architecture is directly taken from image generation DDPM. Treating forecasts as images has many caveats as the locality in the incidence and time axes have very different semantics (e.g human mobility between states is not properly captured by the small convolution kernels). We're exploring custom architecture but found that the present one performs empirically well. The code is directly inspired from PyTorch implementations found, especially the [Hugging Face Annotated Diffusion model](https://huggingface.co/blog/annotated-diffusion), but also [Keras docs](https://keras.io/examples/generative/ddim/) and this [Colab notebook](https://colab.research.google.com/drive/1sjy9odlSSy0RBVgMTgP7s99NXsqglsUL?usp=sharing#scrollTo=Rj17psVw7Shg).

### Inpainting
With our DDPM, we can generate epidemiological curves of Flu Hospitalization, i.e full season forecasts. However, after the first data point are reported, we condition our forecasts on the evidence using inpainting. Inpainting alter the reverse diffusion iterations by sampling the unmasked regions using ground-truth. We use the REpaint algorithm [3], which outperform SOTA (GAN, Auto-regressive) for diverse, high-quality inpainting. 

<img width="670" alt="image" src="https://user-images.githubusercontent.com/7485811/219053390-b0aecee5-db2d-431e-8a7b-3a771f18d396.png">

The Repaint algorithm has important parameters that influence global harmonization: the number of resampling steps and possible jumps in diffusion time. We use repaint for forecasts with past incidence as ground truth:

<img width="725" alt="image" src="https://user-images.githubusercontent.com/7485811/219055262-69e3c059-176e-4311-9aa7-8a98846933ea.png">

In Feb 2023, Rout et al. showed that the RePaint algorithm has some theoretical justification for sample recovery. But they also observed and corrected a misalignment error between the inpainted areas and the ground-truth. We have been observing this as well (see image below) and we have been working hard to correct it [4]

### Training Data
No dataset corresponds to what we want to model (New Hospital Admission at the state level). We use either data generated by a mechanistic influenza transmission model (taken from Round 1 of the [US Flu Scenario Modeling Hub](https://fluscenariomodelinghub.org)) or reported US influenza data ([FluView](https://gis.cdc.gov/grasp/fluview/fluportaldashboard.html), [FluSurv](https://www.cdc.gov/flu/weekly/influenza-hospitalization-surveillance.htm)) at different locations, or more generally a mix of the above in certain proportions. We augment the dataset with random transform.

### FluSight
We submit weekly Flu hospitalization forecasts for each U.S. state to the FluSight challenge, organized by the CDC. We have found that InfluPaint is been plug-n-play, but good performance required a bit of “care”: mainly applying the right transforms to enrich the dataset, having the right diffusion timing and the right inpainting parameters, such as resampling and jumps. It is not too computationally intensive (only 30 min on a Tesla V100-16GB, training included). 

A benefit of this approach is the nice diversity in forecasts compared to mechanistic models (double peaks and single peaks, little bumps …), but our performance is inequal as we iteratively improved the algorithm for this Flu season.

<img width="988" alt="image" src="https://user-images.githubusercontent.com/7485811/233660673-b415dd62-fd5a-4097-b8ce-ef698202269d.png">


### Current focus
- Use other variables (humidity, Flu A, and Flu B proportions) as additional image channels.
- Application to other diseases and validation
- Design of ID modeling specific neural architectures

[1] Ho, Jonathan, Ajay Jain, and Pieter Abbeel. “Denoising Diffusion Probabilistic Models.” arXiv, December 16, 2020. https://doi.org/10.48550/arXiv.2006.11239.

[2] Dhariwal, Prafulla, and Alex Nichol. “Diffusion Models Beat GANs on Image Synthesis.” arXiv, June 1, 2021. https://doi.org/10.48550/arXiv.2105.05233.

[3] Lugmayr, Andreas, Martin Danelljan, Andres Romero, Fisher Yu, Radu Timofte, and Luc Van Gool. “RePaint: Inpainting Using Denoising Diffusion Probabilistic Models.” arXiv, August 31, 2022. https://doi.org/10.48550/arXiv.2201.09865.

[4] Rout, Parulekar, aramanis, Shakkottai. “A Theoretical Justification for Image Inpainting Using Denoising Diffusion Probabilistic Models.” arXiv, February 2, 2023. https://arxiv.org/abs/2302.01217

## Log

2023-04-11 Our model showed some bias in the CDC's plots that we did not observe on our side, and we have found a bug where the post-processing rescaling (due to the misalignement observed in Rout et al.) has not been applied in the quantiles sent to FluSight. This mainy affected the submissions of the last few weeks, where our model consistently scaled lower than reported hospital admission. It is now corrected.

2023-02-13: Before: more resampling steps gives smaller confidence interval, because more certainty in what matches the curve. Now that I added some random noise perturbation of the training set, more resampling steps gives larger confidence interval under certain conditions.

## Instruction
The main notebook either run on google Colab or on UNC HPC cluster, longleaf.

### Building the conda environment
If on UNC HPC cluster longleaf, just ssh into longleaf longing node: `ssh chadi@longleaf.unc.edu`.

Build conda environment, do just once:
```bash
## Only on UNC Longleaf
module purge
module load anaconda

# initialized conda in your .bashrc:
conda init
```

then disconnect & reconnect to you shell for the changes to be taken into account. You should see `(base)` on the left of the prompt, then:

```bash
conda create -c conda-forge -n diffusion_torch seaborn scipy numpy pandas matplotlib ipykernel xarray netcdf4 h5netcdf tqdm  einops tenacity aiohttp ipywidgets jupyterlab # (if not on longleaf, you don't have to install the last two packages)
conda activate diffusion_torch
# the next commands are inside the diffusion_torch environment
conda install torchvision -c pytorch
conda install -c bioconda epiweeks
# install a jupyter kernel for this environment
python -m ipykernel install --user --name diffusion_torch --display-name "Python (diffusion_torch)"
```

Keep in mind that on longeaf one cannot modify the base enviroment (located /nas/longleaf/rhel8/apps/anaconda/2021.11) but can create new enviroment with everything needed in these.

### Running for UNC OpenOndemand
Now you can run on [UNC open Ondemand (OOD)](https://ondemand.rc.unc.edu), which is also a very convienient way to download data or to view figures outputed by the model. Just run a juypter notebook with request-gpu option selected and the following *Jupyter startup directory*
```
"/nas/longleaf/home/chadi/inpainting-idforecasts"
```
and the following *Additional Job Submission Arguments*:
```
--mem=32gb -p volta-gpu --qos=gpu_access --gres=gpu:1
```
(I don't the above arguments are really necessary, because on OOD you won't get a full volta gpu anway, but an A100 divided into small MIG 1g.5gb.

Then go to to run diffusion once your job is allocated.

### Running on a full compute node with Volta GUP or on UNC-IDD patron node
For the first time only, create jupyter lab password:
```bash
sh create_notebook_password.sh
```
Then launch a batch job to create a jupyter notebook server you can connect to (here requests one volta-gpu for 18 hours)

Launch a job for 18h on volta-gpu
```bash
srun --ntasks=1 --cpus-per-task=4 --mem=32G --time=18:00:00 --partition=volta-gpu --gres=gpu:4 --qos=gpu_access --output=out.out sh runjupyter.sh &
```
or on UNC-IDD patron node:
```bash
srun --ntasks=1 --time=18:00:00 -p jlessler --gres=gpu:1 --output=out.out sh runjupyter.sh &
```
UNC-IDD specs are:
- 512GB ram
- 56 physical CPU cores - can be 112 vCores if you decide to enable Hyperthreading
- [Quantity 4 of Nvidia L40, 48GB](https://www.nvidia.com/en-us/data-center/l40/)

where I request 4 GPUs here; you will see
```
run: job 56345284 queued and waiting for resources
```
and after some time:
```
srun: job 56345284 has been allocated resources
```

then `cat out.out` which shows the instructions to go and make the ssh tunnel to connect on jupyter lab.




## Run the diffusion
Make sure on the upper right corner, that the conda enviroment kernel `Python (diffusion_torch)` is activated.

Create synthetic data from the `dataset_builder.ipynb` notebook, and run the inpainting forecast from `inpaintingFluForecasts.ipynb`


## Useful repo
```bash
git clone https://github.com/andreas128/RePaint.git referenceimplementations/RePaint
git clone https://github.com/openai/guided-diffusion.git referenceimplementations/guided-diffusion
git clone https://github.com/cmu-delphi/delphi-epidata.git Flusight/flu-datasets/delphi-epidata¨


git clone https://github.com/jcblemai/Flusight-forecast-data.git Flusight/2022-2023/FluSight-forecast-hub-official
git clone https://github.com/cdcepi/FluSight-forecast-hub Flusight/2023-2024/FluSight-forecast-hub-official
git clone https://github.com/midas-network/flu-scenario-modeling-hub.git Flusight/Flu-SMH
```


# WIS by Adrian Lison
git clone https://github.com/adrian-lison/interval-scoring.git interval_scoring
```
then to update your repository, type:
```
./update_data.sh
```


## Installing git lfs on longleaf
```bash
https://github.com/git-lfs/git-lfs/releases/download/v3.2.0/git-lfs-linux-amd64-v3.2.0.tar.gz
tar -xf git-lfs-linux-amd64-v3.2.0.tar.gz
cd git-lfs-3.2.0
export PREFIX=$HOME/bin
./install.sh
```
Make sure it is rightly installed & in the path. If needed edit `.profile` as
```
if [ -d "$HOME/bin/bin" ] ; then
  PATH="$PATH:$HOME/bin/bin"
fi
```

```bash
git lfs install
git lfs pull
```
