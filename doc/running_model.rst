Running the Model
=====

In this section, I will explain the steps to create future water level forecasts.

Step 1: Formatting the L2SWBM Data
-----

Within the "Copula" folder, there is a folder titled "L2SWBM_med_1950-2022" that contains the output of running the 
Large Lake Statistical Water Balance (L2SWBM, L2S for short) model on the Great Lakes. This model takes multiple reliable
estimates for water balance components across the Great Lakes and reconciles them into confidence intervals for each
components. If you are interested in recreating the output yourself, you can run one of the models found 
`here <https://github.com/luo-yifan/L2SWBM>`_.

The output of the L2S consists of multiple CSV files, each corresponding to a water balance component (precipitation, evaporation, 
runoff, etc.) for one lake, whose rows include a confidence interval for what the true component value was for a specific month. 
These confidence intervals are what we use to create the multivariate distribution that is the copula model. These values are fed 
into the "InputDataFormatting.R" file to create a csv file with the medians found in the "formatted_input" folder and an R object 
of the 95% confidence intervals for each component for each month (averaged through the years) on each lake.

On inspection of the file in the "formatted_input" folder, one may be confused regarding the present of three extra
antecedent months prior to the regular 12 months of data. From my prior reading, this is because during the development 
phase of the copula model the developers wanted to see if chaining together simulations by matching the final three months 
of one simulation to the antecedent three months of another simulation (that's why L_ant is 3) would result in simulations 
that were significantly more true to historical observations which was found to not be the case. The code is still structured 
in a way to leave the door open in case the "matching" idea was to be explored later down the line but from my experience it 
wasn't something I looked into.

Step 2a: Creating and Sampling From the Copula Model
-----

Once the L2S input is formatted, we pass it into the "FittingAndSampling.R" file in order to create the copula using the
Vine Copula library of R. A high level overview of the copula model is that it is a multivariate distribution that describes
how the random variables used to create it are related (the dependence structure) but has uniformly distributed marginal 
distributions for each variable. We generate pseudo-random samples from the Copula in which the values for each component will range 
from 0 to 1, and then we create marginal distributions of the component values for each month for each lake using the medians from 
the L2S data and pass the pseudo-random samples through these marginal distributions to get real values for the components. The 
marginal distribution of precipitation is gamma, that of evaporation is normal, and that of runoff is log-normal.

The following code blocks are from the "2a_FittingAndSampling.R" file. This first snippet is part of a for loop that loops through
each combination of component-lake-month for the LS2WBM outputs (stored in vals) and stores the relevant parameters of the observed
distribution of the values based on the component that is being analyzed.

   .. code-block:: R

        if (comp=="P"){
            # Define gamma parameters
            marginal_pars[[col]]$x.bar_s <- mean(vals, na.rm=T)
            marginal_pars[[col]]$theta_s <- log(marginal_pars[[col]]$x.bar_s)-mean(log(vals),na.rm = TRUE)
            marginal_pars[[col]]$shape   <- 1/(4*marginal_pars[[col]]$theta_s)*(1+sqrt(1+4*marginal_pars[[col]]$theta_s/3))
            marginal_pars[[col]]$rate    <- marginal_pars[[col]]$shape/marginal_pars[[col]]$x.bar_s
            marginal_pars[[col]]$scale   <- 1/marginal_pars[[col]]$rate
            marginal_pars[[col]]$thres   <- 0
        }

This second code snippet takes the aforementioned pseudorandom samples from the copula distribution (stored in samp_mat) which are from from 0 to 1, and passes them 
through the marginal distributions that we just created, which returns the "actual" values for the components (actual in quotations because these
are still simulated values).

   .. code-block:: R

        if(comp=="P"){
            samp_mat[,col] <- qgamma3(pvals,shape=marginal_pars[[col]]$shape,
                                            scale=marginal_pars[[col]]$scale,
                                            thres=marginal_pars[[col]]$thres)
        }

Step 2b: Sampling From the Copula Model Under Future Climate Scenarios
-----

Under future climate scenarios, we expect the distributions of the component values to change from their historical trend.
As such, in order to create component values under different climate scenarios, we should take the pseudo-random samples 
and pass them through adjusted marginal distributions. Note that this assumes that our dependency structure remains the same 
no matter what the underlying climate is, which may not be a reasonable assumption. This is definitely an area for future
exploration!

The following code block is from the "2b_FitSampSimulation.R" file and details how we can shift the means of precipitation for
each month (note that this code ignores interlake variation - one must quintuple the weight vector to apply different weights for
each). The pweights vector is the percent shift in the mean, so to bring the mean from 90 to 100, we would apply a weight of 1.1111.
The means should be adjusted such as to keep the coefficient of variation of the distribution, which is the standard deviation divided
by the mean, the same as it is in the original distribution. For a gamma distribution, the coefficient of variation is only affected by 
the shape parameter, so we must keep the shape parameters the same as they originally were and change the scale parameter to reflect the
adjusted means. You can do similar calculations to determine how to change the means for evaporation and runoff.

One can also read in the csv files in the "2b_input_changefiles" folder instead of constructing the weight vector manually.

    .. code-block:: R
        
        pweights <- c(1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1)

        for (col in c(1, 46, 91, 136, 181)) {
            meanvec <- as.numeric(unlist(sapply(marginal_pars, "[", 1)[col:(col+14)]))
            smoothmeans <- loess.smooth(c(1:15), meanvec * pweights, evaluation = 15)[[2]]
            shapevec <- as.numeric(unlist(sapply(marginal_pars, "[", 3)[col:(col+14)]))
            newscales <- smoothmeans / shapevec

            for (subcol in seq(col, col + 14, by = 1)) {
                comp  <- substr(colnames(psamp_mat)[subcol],start=nchar(colnames(psamp_mat)[subcol]),stop=nchar(colnames(psamp_mat)[subcol]))
                pvals <- psamp_mat[,subcol]

                samp_mat[,subcol] <- qgamma3(pvals,shape=marginal_pars[[subcol]]$shape,
                                            scale=newscales[subcol %% 45],
                                            thres=marginal_pars[[subcol]]$thres)
            }
        }

Step 3: Creating the Forecasted NBS Values and Running the CGLRRM
-----

In "3_MatchingAndPlotting.R", we take the R object with the "actual" component values that we created in the previous step and sample random
rows of the component value matrix. Then, we calculate the net basin supply (P - E + R) for each month for each lake and store the NBS values in
individual files for each lake. These output files are found in "3_output". The "3_plots" folder also contains plots generated from this code file,
although I have never referenced them.

The next step is to run the Coordinated Great Lakes Regulation and Routing Model, which determines the extent to which water flows between the rivers 
such as to determine the lakes' water levels. Note that the model that we are using is the most easily accessible model but was coded in Fortran decades
ago and is thus operationally outdated; researchers at GLERL are working on their own Python/R version of the CGLRRM.

The way that the MatchingAndPlotting file is currently configured, running the file also formats the files such that they can be fed into the CGLRRM by
running the "input_file_structure.R" file and then also runs the CGLRRM itself with the "system("cglrrm_test.exe") file. However, this will only work if
you have created the executable. You need to make sure that your current working device has a Fortran compiler: on Windows, download the Intel Fortran
Compiler (ifort), and on MacOS, download gfortran via Homebrew. Once this is installed, on Windows, run "ifort CGLRRM.FOR CALCUTL.FOR CONTROL.FOR CUSTOM.FOR 
INIT.FOR MIDLAKES.FOR MISCUTL.FOR TSINPUT.FOR SUPERIOR.FOR Preproject.for SuperiorRegPlans.for ontario.for ontarioinit.for -o cglrrm_test". This code is the same
for MacOS except that "ifort" must be replaced by "gfortran". This will create an executable called cglrrm_test.exe. The raw outputs of the CGLRRM after we run
the executable will be found in the "CGLRRM_Rohan/output" folder, in which the four files with format ****lv.test are the relevant water level files. The
"3_MatchingAndPlotting.R" codefile contains the code that turns these test files into clean csv files within the "output/{sim_name}/{sim_number}"" directory,
where you hardcode in the sim_name, and the sim_number is generated in a for loop. Finally, the code runs the "plotting.R" file, which creates a water level
plot for each simulation, as such.

Note that you can also change the parameters present in CGLRRM_params.2008 such as the locations of the input files, the initial water levels of each lake, and the
start and end dates of the simulations, however the model should still run with the current configuration of the parameters.

    .. image:: /src/_static/waterLevels.png
     :width: 600px
     :align: center

Step 4: Analyzing the Output
-----

Now that you have the output from the routing model, you can run the fullplot.R codefile in the "r_code" directory to showcase all of the simulated water levels
together in a water level plot, and you can compare forecasted water levels under the current climate to your new simulation under a different climate change scenarios
in terms of variables like average water level, minimum water level, maximum water level, and variance. These plots will be stored in the corresponding directory for the 
simulation. Below is an example with the average water levels using a changed precipitation simulation. The top set of plots shows the true historical water levels in 
red and a collection of simulations of forecasted water levels in black. The bottom set of plots compares the distribution of average water levels for the baseline 
simulations (forecasted simulations under no climate alterations) in red, and the distribution of average water levels under the altered evaporation simulation in black. 
The red and black bars show the medians of the distributions.

    .. image:: /src/_static/avgchangeprecip.png
     :width: 600px
     :align: center

In the "threedplot.R" file, I created a three-dimension plot with precipitation on the x-axis, evaporation on the y-axis, and water level on the z-axis and plotted what
would happen if I shifted precipitation and evaporation by incremental percent changes for each month for each lake. The fact that I was able to do this is an amazing
capability that being able to run the three aforementioned models in ensemble can bring you!

In order to create the data that "threedplot.R" takes in, uncomment the three-dimensional plot part of "2b_FitSampSimulation.R" and edit "3_MatchingAndPlotting.R" to include
the following code block before the for loop across all of the simulations start.

   .. code-block:: R

        supmeans <- rep(0, n)
        stcmeans <- rep(0, n)
        mihmeans <- rep(0, n)
        erimeans <- rep(0, n)

Further, edit "3_MatchingAndPlotting.R" to include the following code block after the for loop ends.

   .. code-block:: R

        sup_sim <- read.table("/raw_output/spmmlv.test", skip = 18)
        stc_sim <- read.table("/raw_output/scmmlv.test", skip = 19)
        mih_sim <- read.table("/raw_output/mhmmlv.test", skip = 19)
        eri_sim <- read.table("/raw_output/ermmlv.test", skip = 19)

        supmeans[sim] <- mean(as.matrix(sup_sim[61:70,2:13]), na.rm = TRUE)
        stcmeans[sim] <- mean(as.matrix(stc_sim[61:70,2:13]), na.rm = TRUE)
        mihmeans[sim] <- mean(as.matrix(mih_sim[61:70,2:13]), na.rm = TRUE)
        erimeans[sim] <- mean(as.matrix(eri_sim[61:70,2:13]), na.rm = TRUE)

Then, run "2b_FitSampSimulation.R", and the necessary data files will be in the output folder.