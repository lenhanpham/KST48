# KST48: A Powerful Tool for MECP locating

KST48 is a purely-python program to locate Minimum Energy Crossing Points (MECPs), based on the searching algorithm by Bearpark, Robb and Schlegal in 1994 (Chemical physics letters, **1994**, 223(3): 269-274.).  An early implement of this algorithm is Harvey's pioneering work in 1998 (Theoretical Chemistry Accounts, **1998**, 99(2): 95-99.), which remains the most popular program to look for MECPs. Harvey's Fortran-based MECP program works in combination with Gaussian: by invoking Gaussian to calculate the forces and energies of the two states, a BFGS minimization algorithm was performed and gives an MECP starting from the input geometry.

KST48 provides a much more user-friendly and extensible solution for MECP locating, with a much more simplified preparation procedure. The optimization is based on the GDIIS algorithm, and is generally as effective. In addition to the locating of MECPs between different spin states by ground-state DFT methods, as implemented in Harvey's program, KST48 also supports the crossing between excited states with TD-DFT methods, or any other situations where the gradient can be read from a standard quantum chemical program.

In addition to the locating of MECP without any geometrical constrains, KST48 supports constrained optimization of bond lengths or angles, as well as the 1D- or 2D- scanning in the 3N-7 dimension of space with E1=E2 satisfied.

![logo](https://github.com/RimoAccelerator/KST48/blob/main/logo_KST48.png)

# Requirements
Python 3

Numpy
(KST48 is written with Numpy 1.19.3. However, it only ultilizes the most basic features of Python and Numpy, and should be compatible with any popular versions)

A quantum chemical program (currently supported are Gaussian, BAGEL and ORCA)

# Usage
1. Put kst48.py in any folder you want, together with a folder names JOBS. The task jobs will be placed here.
2. Prepare an input file:
```
#This subset is required. It controls your quantum chemistry tasks.
nprocs = 8 #processors
mem = 8GB # memory to be used. change this into the maxcore value if you want to use ORCA
method = wb97xd def2sv # your keywords line. It will be presented in the job files. Don't write guess=mix or stable=opt; they will be added automatically.
td1 =  # keywords for TD-DFT of state A (only for Gaussian; please write it in the tail part for ORCA)
td2 = # keywords for TD-DFT of state B (only for Gaussian; please write it in the tail part for ORCA)
mp2 = false #set true for MP2 or doubly hybrid calculation in Gaussian
charge = 0
mult1 = 1 # multiplicity of state A
mult2 = 3 # multiplicity of state B
mode = stable #normal; stable; read; inter_read; noread

#This subset is optional. It controls the convergence threshols, and the details of the GDIIS algorithm. Shown here are the default values.
dE_thresh = 0.000050
rms_thresh = 0.0025
max_dis_thresh = 0.004
max_g_thresh = 0.0007
rms_g_thresh = 0.0005
max_steps = 100
max_step_size = 0.1
reduced_factor = 0.5 # the gdiis stepsize will be reduced by this factor when rms_gradient is close to converge

# This subset controls which program you are using, and how to call them
program = gaussian  #gaussian or orca
gau_comm = 'g16'
orca_comm = '/opt/orca5/orca'
xtb_comm = 'xtb'

#Between *geom and *, write the cartesian coordinate of your initial geometry (in angstrom)
*geom
 C                 -0.03266394   -2.07149616    0.00000000
 H                  0.76176530   -1.26928217    0.00000000
 H                 -0.91673236   -1.36926867    0.00000000
 *
#If you have anything to be put at the end of the input file, write it here. This part is especially useful for ORCA: you can add anything related to your keywords here.
*tail1
#-C -H 0
#6-31G(d)
#****
*
*tail2
*

#This subset controls the constraints. R 1 2 1.0 means to fix distance between atom 1 and 2 (start from 1) to be 1.0 angstrom.
*constr
#R 1 2 1.0
#A 1 2 3 100.0 # to fix angle 1-2-3 to be 100 degree
#S R 1 2 1.0 10 0.1 # to run a scan of R(1,2) starting from 1.0 angstrom, with 10 steps of 0.1 angstrom
#S R 2 3 1.5 10 0.1 # you can at most set a 2D-scan
*
```
3. If the input file is called inp, then simply run `python3 kst48.py inp`. The output will be printed to the screen.
You will see the convergence of each step. After the MECP is converged, you will find its input and output files at current folder.

# Modes
There are several modes you can choose:
1. normal
  No special treatment.
2. stable
  Sometimes you might concern whether you are using stable wavefunctions in the MECP locating procedure. By setting mode=stable, KST48 will automatically run a wavefunction stability calculation of the initial geometry, and the so-generated wavefunction will be read for the following optimization.
  Note: please explicitly write UKS when you are using ORCA with the stable mode, to calculate a singlet state.
3. read
  If you have already had the wavefunction file (.chk for Gaussian or .gbw for ORCA), you could put them together with kst48.py, named as a.chk and b.chk (or .gbw for ORCA) respectively. By setting mode=read, KST48 will read them and check the stability, then do the optimization.
4. inter_read
  In some very hard cases, a stable wavefunction may not be correct. For example, for some molecule, you can find a stable RHF wavefunction for its singlet state, but there is a UHF wavefunction significantly lower in energy, and only can be obtained by SCF reading the triplet wavefunction (together with careful convergence controlling, such as guess=mix or even scf=vshift=-1000 in Gaussian). By setting mode=inter_read, state B will firstly be calculated in the first step, and then state A reads its wavefunction. Stability will be automatically checked. It is your work to write any convergence controlling keywords.
5. noread
  In normal mode, each step in the optimization reads the wavefunction from the previous step. In noread mode, no wavefunction is read.

# BAGEL Support
Since Feb. 2022, interface to BAGEL has been added to KST48. In order to invoke BAGEL, you need to set three options in the input file:
```
program = bagel  
bagel_comm = mpirun -np 36 /opt/bagel/bin/BAGEL
bagel_model = model.inp 
state1 = 0 #only set it for the multireference calculation using BAGEL
state2 = 1 #only set it for the multireference calculation using BAGEL
```
As you see, now a input template (here named model.inp) is required. Necessary keywords for a force calculation in BAGEL should be properly set in this file, and leave the geometry part as *geometry*. Then KST48 will automatically change the spin multiplicity and "target" option according to your mult1, mult2, state1 and state2. An example model.inp is attached along with the code (example_bagel_model_file.inp).

# Structure Interpolation
Since Jun. 2022, an **experimental** feature has been added to do an interpolation between two structures. In some cases, this could aid you on finding a good guess for the MECP between two structures. By using this feature, set the "\*LST1" and "\*LST2" part in the input file. Put the geometries like this:
```
*lst1
 C                 -2.05168000   -3.89619700   -2.03479500
...
*
*lst2
 C                 -4.08538500    4.08668300   -0.76532000
...
*
```
Once "lst" part is detected, KST48 will ask you how many intermediate structures you are going to use for interpolation. The two geometries will be aligned, and the linear interpolation will be performed to generate several intermediate structures between lst1 and lst2. At present, only Gaussian is supported. Once the intermediate structures are approved, KST48 will invoke Gaussian to calculate the energies of both states for each structure, and give a list of energies as the output. You can use the one with minimal A-B energy difference as an initial guess for crossing point locating.


# Citation
Any use of this code MUST cite this Github Page (https://github.com/RimoAccelerator/KST48/), and it is encouraged to cite the author's first article using KST48. The citation is listed as the following:

1. Y. Ma, A. A. Hussein, ChemistrySelect 2022, 7, e202202354.

2. Yumiao Ma.  KST48: A Powerful Tool for MECP locating. https://github.com/RimoAccelerator/KST48, accessed on xxxx.xx.xx

The citation will be updated once KST48 is published in a journal or as preprint.

# Bug Report
You are welcomed to report any bug or problems to ymma@bsj-institute.top.

