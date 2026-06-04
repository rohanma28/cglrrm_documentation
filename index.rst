.. Great Lakes Copula and Routing Model Usage Documentation documentation master file, created by
   sphinx-quickstart on Thu Mar 12 20:16:53 2026.
   You can adapt this file completely to your liking, but it should at least
   contain the root `toctree` directive.

Welcome to the Great Lakes Copula and Routing Model's documentation!
====================================================================================

Disclaimers
-----------------------------------------------
This documentation page is meant to be accessed alongside the GitHub repository that contains all of the codefiles that I will be referencing, which is located at
`this link <https://github.com/rohanma28/cglrrm>`_. This repository is currently set to private due to concerns surrounding data privacy, but if you need access, 
**please do not hesitate to email me at rathreya@umich.edu!** 

The hardcoded simulation names (Ctrl+F "INSERT_SIM_NAME_HERE" in each of the codefiles to find all of them) must be changed to the simulation you want to analyze in order for the files
to run and produce output as expected.

This page does not discuss the statistical background behind the copula model besides what is necessary to run it and craft simulations.
Alex Vandeweghe, who was part of SEAS-Hydro from 2019 to 2024 and received a BS and MS in Environmental Engineering along the way, was the principal creator
of the model, and you can view his manuscript that explains the copula model in depth `here <_static/Manuscript.pdf>`_. Note that this manuscript was written alongside
the codefiles present in the "Alex_Original_Codefiles/DS_S1" folder in the GitHub repository. The similarly named codefiles in the "Copula" folder have been slightly 
tweaked to enable us to run the output of the Copula through the CGLRRM.

Contents:

.. toctree::
   :maxdepth: 2

   running_model
