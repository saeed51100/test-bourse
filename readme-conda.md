###  Why included conda-forge in environment.yml?

* **TensorFlow with/without CUDA:** This is the most important reason to include conda-forge in environment.yml.  conda-forge provides TensorFlow builds that are specifically compiled for CUDA (e.g., tensorflow-gpu) and also CPU-only builds (tensorflow).


* **CUDA Machine:** On your machine with CUDA, conda will automatically install the CUDA-enabled TensorFlow build when you just specify tensorflow. You don't need to specify tensorflow[and-cuda] explicitly; conda will handle that for you.  Make sure you have the correct CUDA drivers and toolkit installed on your system.


* **Non-CUDA Machine:** On your machine without CUDA, conda will automatically install the CPU-only version of TensorFlow when you specify tensorflow.  Again, you don't need to do anything special.


* **Consistency:** Using conda-forge ensures that both your development and deployment environments remain consistent, regardless of the hardware differences between your machines.

* **Future Flexibility:** If you need to add more packages in the future (or adjust for specific CUDA versions), the conda-forge ecosystem typically provides a flexible and up-to-date collection.

### Commands:
```
Install cuda for tensorflow gpu
```
```
set the pycharm terminal in your project folder.
```
```
conda activate base
```
```
Set pycharm interpreter to base:  
/home/saeed/miniconda3/bin/python
```
```
conda env create --prefix ./envs -f environment.yml
```
```
conda env update -f environment.yml --prune
```
```
Set pycharm interpreter to:  
./envs/bin/python
```

---
     conda activate ./envs
     conda activate ./bade
     conda deactivate
---

```
for remove environments:
first deactivate all environments check:
conda env list
then:
conda env remove --prefix ./envs
if not remove completelly you should remove folder manually:
sudo rm -rf envs
```
---
```
Remove - default channel:
rm /home/saeed/miniconda3/.condarc
```
---
```
conda list
```
```
conda env list
```
```
conda info --envs
```