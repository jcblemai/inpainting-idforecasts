# UNC_IDD InfluPainting

## Instruction
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
conda create -c conda-forge -n diffusion_torch seaborn scipy numpy pandas matplotlib ipykernel xarray netcdf4 h5netcdf tqdm jupyterlab einops ipywidgets
conda activate diffusion_torch
# the next commands are inside the diffusion_torch environment
conda install torchvision -c pytorch
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

### Running on a full compute note
For the first time only, create jupyter lab password:
```bash
sh create_notebook_password.sh
```
Then launch a batch job to create a jupyter notebook:

## Run the diffusion
Create synthetic data padded from the data notebook
Run the diffusion from HF-annotated_diffusion.
### run on a Volta GPU at UNC



Launch a job for 18h:
```bash
srun --ntasks=1 --cpus-per-task=4 --mem=32G --time=18:00:00 --partition=volta-gpu --gres=gpu:1 --qos=gpu_access --output=out.out sh runjupyter.sh &
```
then `cat out.out` which shows the instructions to go and make the ssh tunnel to connect on jupyter lab.

