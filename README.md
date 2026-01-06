# $B^\ast \to tW$ Background Estimate Exercise - CMSDAS @FNAL LPC 2025
Background estimation for the 2025 CMSDAS $b^\ast \to tW$ exercise, using the updated version of 2DAlphabet and the latest version of Combine (v10.0.1).

## Getting started (in bash shell)

First, ensure that you have [SSH keys tied to your github account](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent) and that they've been added to the ssh-agent:
```
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_xyz
```
This step is necessary for cloning some of the tools used in the Combine and 2DAlphabet installation.

### Setup CMSSW and 2DAlphabet environment:
Ensure that you are connecting to an LPC node with AlmaLinux 9, e.g. `cmslpc-el9.fnal.gov`.
Then naviate to wherever you would like to install the exercise. Note that this should be *outside* of the CMSSW release of the previous exercise.
Somewhere like `~/nobackup/CMSDAS2026` is a good choice. 
Then, intialize the CMSSW release we need. (Note that this is a different release version than the TIMBER setup!) 
```
cmsrel CMSSW_14_1_0_pre4
cd CMSSW_14_1_0_pre4/src
cmsenv
```

Now set up 2DAlphabet:
```
git clone https://github.com/cms-analysis/HiggsAnalysis-CombinedLimit.git HiggsAnalysis/CombinedLimit
cd HiggsAnalysis/CombinedLimit
git fetch origin
git checkout v10.0.1
cd ../../
git clone --branch CMSWW_14_1_0_pre4 git@github.com:JHU-Tools/CombineHarvester.git
scramv1 b clean
scramv1 b -j 16
git clone git@github.com:JHU-Tools/2DAlphabet.git
python3 -m virtualenv twoD-env
source twoD-env/bin/activate
cd 2DAlphabet/
python setup.py develop
```

Then, check that the 2DAlphabet installation worked by typing the following in your bash shell:
```
python -c "import ROOT; r=ROOT.RooParametricHist()"
```
If the installation succeeded, you should see no output from the above command.

### Finally, clone this repo to the `src` directory as well:
```
cd ..
git clone --branch fnal-2026 https://github.com/ozamram/BstarToTW_CMSDAS2024_BackgroundEstimation.git
```
OR fork the code onto your own personal space and set the upstream:
```
https://github.com/<USERNAME>/BstarToTW_CMSDAS2024_BackgroundEstimation.git
cd BstarToTW_CMSDAS2024_BackgroundEstimation
git remote add upstream https://github.com/ozamram/BstarToTW_CMSDAS2024_BackgroundEstimation.git
git remote -v
git checkout -b fnal-2025
```

## Every time you reconnect to the LPC:
Go back to the directory where you installed 2DAlphabet and where the virtual environment resides:
```
ssh -XY USERNAME@cmslpc-el9.fnal.gov
cd ~/nobackup/CMSDAS2025/CMSSW_14_1_0_pre4/src/
cmsenv
source twoD-env/bin/activate
```
Then you should be good to go!

## Background estimate
For this exercise we will use the [`2DAlphabet`](https://github.com/ammitra/2DAlphabet) github package. This package uses `.json` configuration files to specify the input histograms (to perform the fit) and the uncertainties. These uncertainties will be used inside of the `Higgs Combine` backend, the fitting package used widely within CMS. The 2DAlphabet package serves as a nice interface with Combine to allow the user to use the 2DAlphabet method without having to create their own custom version of combine. 

# Running the exercises 

First create a 2DAlphabet workspace for a given signal mass point. Pass the mass (in GeV) as the argument `--sig` and pass the `--make` flag to make the workspace.
```
python bstar.py -s [mass] --make
```
From here, there are several Combine methods you can run:

## The maximum likelihood fit
You can run the maximum likelihood (ML) fit using Combine's [`FitDiagnostics`](https://cms-analysis.github.io/HiggsAnalysis-CombinedLimit/latest/part3/nonstandard/?h=fitdiagn) method. This method runs the background-only and signal-plus-background fits and produces post-fit shape distributions of the backgrounds and signal. 
```
python bstar.py -s [mass] --fit
```
This requires that a workspace has been generated for the given signal mass point. 

## Plotting the ML fit results
You can plot the pots-fit distributions and the nuisance parameter pulls using the following command:
```
python bstar.py -s [mass] --plot
```

**Note:** you might see this output displayed on the screen for a minute or so:
```
Executing: PostFit2DShapesFromWorkspace -w higgsCombineTest.FitDiagnostics.mH120.root --output postfitshapes_b.root -f fitDiagnosticsTest.root:fit_b --postfit --samples 100 --print > PostFitShapes2D_stderr_b.txt
Executing: PostFit2DShapesFromWorkspace -w higgsCombineTest.FitDiagnostics.mH120.root --output postfitshapes_s.root -f fitDiagnosticsTest.root:fit_s --postfit --samples 100 --print > PostFitShapes2D_stderr_s.txt
```
The code hasn't frozen, it just takes a while for the `PostFit2DShapesFromWorkspace` method to create the post-fit distributions from the B-only and S+B fits. The code isn't frozen!

## Calculating nuisance paramater impacts
You can calculate the nuisance parameter [impacts](https://cms-analysis.github.io/HiggsAnalysis-CombinedLimit/latest/tutorial2023/parametric_exercise/?h=impact#impacts) on the signal strength by running the following command:
```
python bstar.py -s [mass] --impacts
```
A plot `impacts.pdf` will be generated in the signal workspace directory.

## Calculating the asymptotic limits
The asymptotic Frequentist limits can be calculated for a given signal by running the following command:
```
python bstar.py -s [mass] --limit
```


# 2DAlphabet configuration file explained

The configuration file that you will be using is called `bstar.json`, located in this repository. Let's take a look at this file and see the various parts:

* `GLOBAL`
  - This section contains meta information regarding the location (`path`), filenames (`FILE`), and input histogram names (`HIST`) for all ROOT files used in the background estimation procedure.
  - Everything in this section will be used in a file-wide find-and-replace. So wherever you see the name of the sub-objects in this file, it will be expanded by the value assigned to it in this section. 
  - Additionally, the `SIGNAME` list should include the name(s) of all signals you wish to investigate, so that they are added to the workspace when you run the python script.
    - If you wanted to investigate limits for only three signals, for example, you'd just add their names as given in the ROOT files to this list. 
    - For this exercise, the value is `signalLHSIGMASS`. This is a placeholder, and is overwritten by the `findreplace` argument to the TwoDAlphabet constructor. By passing in the signal mass to the `make_workspace()` function, we overwrite this value with the desired signal mass.

* `REGIONS`
  - This section contains the various regions we are interested in transferring between.
  - Each region contains a `PROCESSES` object, listing the signals and backgrounds to be included in the fit, as well as  `BINNING` object, which is defined elsewhere in the config file.
  - The name of each region in `REGIONS` is dependent on the input histogram name, as well as your choice of `HIST` name in the `GLOBAL` section above
    - For instance, in this file we declared `HIST = MtwvMt$region`, where `$region` will be expanded as the name given in `REGIONS`. 
    - We chose this name because the input histograms are titled `MtwvMtPass` and `MtwvMtFail` for the Pass and Fail regions, respectively. 

* `PROCESSES`
  - In this section we define all of the various process ROOT files that will be used to produce the fit. These include data, signals, and backgrounds.
  - Each process contains its own set of options:
    - `SYSTEMATICS`: a list of systematic uncertainties, whose properties are defined elsewhere in the config file
    - `SCALE`: how much to scale this process by in the fit
    - `COLOR`: color to plot in the fit (ROOT color schema)
    - `TYPE`: `DATA`, `BKG`, `SIGNAL`
    - `TITLE`: label in the plot legend (LaTeX compatible)
    - `ALIAS`: if the process has a different filename than standard, this will be what replaces `$process` in the `GLOBAL` section's `FILE` option, so that this process gets picked up properly
    - `LOC`: the location of the file, using the definitions laid out in `GLOBAL`

* `SYSTEMATICS`
  - This contains the names of all systematic uncertainties you want to apply to the various processes.
  - The `CODE` key describes the type of systematic that will be used in Combine.
  - The `VAL` key is how we assign the value of that uncertainty. For instance, a `VAL` of `1.018` in the `lumi` (luminosity) means that this systematic has a 1.8% uncertainty on the yield.

* `BINNING`
  - This section allows us to name and define custom binning schema. After naming the schema, one would define several variables for both `X` and `Y`:
    - `NAME`: allows us to denote what is being plotted on the given axis
    - `TITLE`: the axis label for the plot (LaTeX enabled)
    - `BINS`: a list of bins
    - `SIGSTART`, `SIGEND`: the bins defining a window `[SIGSTART, SIGEND]` around which to blind (if the blinded option is selected)

* `OPTIONS`
  - A list of boolean and other options to be considered when generating the fit
  - (explanation WIP)

# Running the ML fit
By default, the `bstar.py` python API should set up a workspace, perform the ML fit, and plot the distributions. 

```
python bstar.py
```

The output is stored in the `tWfits/` output directory by default.

# Systematics
Systematic uncertainties were described in the config file section above. Add the Top pT uncertainties to the appropriate processes in the config file, then re-run the fit after having copied the old Combine card somewhere safe. Compare the pre- and post-Top pT Combine cards using `diff`.

# Limit setting
WIP
