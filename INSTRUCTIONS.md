# Masterclass 22.10: Hamiltonian replica exchange with PLUMED and GROMACS

## Origin 

This masterclass was authored by Giovanni Bussi on June 21, 2022

## Aims

In this Masterclass, we will discuss how to run Hamiltonian replica exchange using PLUMED and GROMACS. We will also understand how to analyze the resulting trajectories.
It is highly recommended to follow [this](https://www.plumed.org/doc-v2.7/user-doc/html/masterclass-21-5.html) masterclass in advance.

## Objectives

Once you have completed this Masterclass you will be able to:

- Use PLUMED to manipulate GROMACS topologies and prepare a solute tempering simulation.
- Use PLUMED and GROMACS to run replica-exchange simulations with multiple topologies.
- Use WHAM to combine multiple simulations performed with different topologies and/or bias potentials.

## Setting up PLUMED

For this masterclass you will need versions of PLUMED and GROMACS that are compiled using the MPI library.
In order to obtain the correct versions, please follow the instructions at [this link](https://github.com/plumed/masterclass-2022).

## Resources

The data needed to execute the exercises of this Masterclass can be found on [GitHub](https://github.com/plumed/masterclass-22-10).
You can clone this repository locally on your machine using the following command:


````
git clone https://github.com/plumed/masterclass-22-10.git
````

**All the exercises were tested with PLUMED version 2.8.0 and GROMACS 2020.6**

Additional documentation about the replica exchange implementation discussed in this Masterclass can be found at [this page](https://www.plumed.org/doc-master/user-doc/html/hrex.html).

## Exercises

Throughout this Masterclass we will run simulations of **alanine dipeptide in water** using GROMACS and PLUMED.
Whereas this system is too simple to be considered a proper benchmark for enhanced sampling methods,
it is complex enough to be used for learning about them.
This Masterclass regards a specific implementation of Hamiltonian replica exchange that is only available when combining GROMACS and PLUMED.

Notice that simulations might take up to a couple of hours, depending on the hardware you have access to.
The version of GROMACS that we provide on our conda channel is not optimized. Compiling GROMACS directly
on your machine  might lead to much better performance. In addition, using a GPU will also make your simulations
significantly faster.

### Introduction to Hamiltonian replica exchange

In [masterclass-21-5](https://www.plumed.org/doc-v2.7/user-doc/html/masterclass-21-5.html) we learned how to run replica-exchange simulations
where each replica was possibly feeling different biasing forces. A typical example is
umbrella sampling with replica exchange, where each replica has a restraint
located in a different position. Another example is bias-exchange metadynamics, where
each replica is biased along a different collective variable. We have also seen parallel-tempering
metadynamics, where replicas are kept at a different temperature and, at the same time,
keep track of different history-dependent potentials.
All these examples are a form of Hamiltonian replica exchange, since different replicas
are subject to different potential energy functions, and thus different Hamiltonians.

In this masterclass we will consider a conceptually similar but technically different implementation of
Hamiltonian replica exchange. Here, the different replicas will be simulated using
different force field parameters. For most of the exercise, we will thus use an empty `plumed.dat` file (no bias added!).
You will however need to include this file to ensure PLUMED can be enabled. We will also spend some time in learning how to edit the GROMACS topologies,
so as to generate the modified force fields.

In order to use multiple-replica methods, you should run your simulation using MPI. This can be done prefixing your command
with `mpiexec -np N --oversubscribe`, where `N` is the number of processes that you want to use and the `--oversubscribe`
option is an OpenMPI option that is required to use more processes than the number of available processors. This is typically suboptimal,
but we will need it in our lectures to run, e.g., simulations with 16 replicas even if we have a computer with 4 cores.

In brief, to run a GROMACS simulation where the individual replicas are in directories names `dir0`, `dir1`, etc
and the (possibly empty) `plumed.dat` file is in the parent directory you will need a command such as

```
mpiexec -np 16 --oversubscribe gmx_mpi mdrun -multidir dir? dir?? -plumed ../plumed.dat -replex 200 -hrex
```

The option `-replex 200` enables replica exchange and ensures exchanges are attempted every 200 steps.
The option `-hrex` is specific for the implemenation discussed in this Masterclass, and informs GROMACS and PLUMED
that there might be different force fields used in different replicas.


*If you forget the `-hrex` flag, no error will be issued, but the acceptance for exchanges will be incorrectly calculated.*

If you have random crashes on MacOS, try to set this environemnt variable:

````
export OMPI_MCA_btl="self,tcp"
````

In the root directory of this Masterclass you will find a `conf.gro` file, that can be used as a starting configuration
for your simulations, a `topol.top` file, which contains the topology information, and a `grompp.mdp` file
with reasonable simulation parameters. You have also a `ala.pdb` file, that is basically the `conf.gro` file
converted to PDB format, and can be used for the [MOLINFO](https://www.plumed.org/doc-master/user-doc/html/_m_o_l_i_n_f_o.html) action so as to facilitate atom selections in analysis.

### Exercise 1a: Test with different temperatures

Before swithing to solute tempering, we will play a bit with parallel tempering.
Parallel tempering simulations require the highest replica to be hot enough for important energy barriers to be crossed
in the simulation. We will first estimate how much we should raise the temperature so as to see forward and backward transitions
between the two metastable states of our system.

To do so:
- Prepare an array of simulations, with temperatures equal to 300, 400, 500, 600, 700, 800, 900, 1000
- Run each of them for 1 ns and check at which temperature you have at least one transition from the initial basin to the other one and one backward transition

These are serial simulations, so you can run them with this command:

```
# first edit grompp.mdp setting the temperature (both solute and solvent groups!)
# then create the topol.tpr file:
gmx_mpi grompp
# then run your simulation:
gmx_mpi mdrun -nsteps 500000
```
though it is recommended to use a script to run your simulations.

To analyze the resulting trajectories you can use PLUMED [driver](https://www.plumed.org/doc-master/user-doc/html/driver.html)
```
plumed driver --ixtc traj_comp.xtc --plumed plumed.dat
```
with the following `plumed.dat` file:

```plumed
MOLINFO STRUCTURE=ala.pdb
phi: TORSION ATOMS=@phi-2 
PRINT ARG=phi FILE=COLVAR
```

You can then plot the second column of the resulting `COLVAR` file and see if there is a transition from the first basin (phi in range (-3,-1))
to the second basin (phi in range (0.5,1.5)), and back. Answer the following question:

- How much should you increase the temperature to see a transition back-and-forth between the two basins?

The result will depend on stochastic factors, but also on the length of the simulation.
I recommend running a 1 ns long simulation (500000 steps), but you can try with a longer or shorter trajectory.

### Exercise 1b: Run a parallel tempering simulation

Once you have identified this temperature, you can run a parallel tempering simulation. You will need a number of replicas to bridge from T=300
to the temperature you have identified. As a first guess, you can place them in a geometric series (that's the allocation that leads to
uniform acceptance in a system with temperature-independent specific heat).

Let's say that you want to try with 16 replicas. You should create 16 directories named `dir0`, `dir1`, ... `dir15` and place a file named `topol.tpr` in each.
Each `topol.tpr` files will be created setting a different temperature in the `grompp.mdp` file.
We will not need PLUMED for this exercise,
so the simulation can be run with a command like this one:

````
mpiexec -np 32 --oversubscribe gmx_mpi mdrun -multidir dir? dir?? -replex 200 -nsteps 500000
````

The `-replex` option tells to GROMACS how frequently exchanges should be attempted.
The average acceptance will be reported at the end of the `md.log` file.

The hottest replica will display transitions between the two metastable states. Thanks to the exchanges, both states will be observed also in the replica that is kept as T=300K.

Now answer the following questions:
- How many replicas do you need to have an acceptance that is at least 30%? (the answer will depend on the maximum temperature you have chosen)
- How much is the relative population of the two metastable states at T=300? Plot this population as a function of the temperature (you can just extract results from the different replicas).

### Exercise 2a: Setting up scaled Hamiltonians

In order to run Hamiltonian replica exchange simulations with multiple topologies, we will have first to learn how to generate
the multiple topologies. Different tools could be used (including editing the `topol.top` file by hand!) but we will now learn
how to use the [partial_tempering](https://www.plumed.org/doc-master/user-doc/html/partial_tempering.html) tool available in PLUMED. This tool
basically allows you to generate topologies where the energy of a subset of the atom have be scaled by a chosen factor. Remember that dividing the energy by a factor 2 is equivalent to multiplying the temperature by a factor 2.

Let's first have a look at the `topol.top` file. In this file there are a number of lines that look like `#include ...`, so that this file
is not self-contained. The first thing that we have to do is to generate a self-contained topology file:

```
gmx_mpi grompp -p topol.top -pp processed.top
```

Have a look at the resulting `processed.top` file with a text editor.
This file does not contain only information about alanine dipeptide, but also about the generic force-field parameters.
It is thus self-contained.

Then you should edit the `processed.top` file to indicate which atoms you want to scale. To do so you have to add an underscore (`_`)
to the atom name of the selected atoms. For instance, this line:
```
     1         HC     1    ACE   HH31      1     0.1123      1.008   ; qtot 0.1123
```
should be modified to this line:
```
     1         HC_    1    ACE   HH31      1     0.1123      1.008   ; qtot 0.1123
```
To perform a solute tempering simulation, you should add the underscore to all the solute atoms (look in the `[ atoms ]` section). Once this is done you can use the following command:
```
plumed partial_tempering 1.0 < processed.top  > scaled.top
```
This will scale the Hamiltonian of the selected atoms by a factor 1.0 (which means: no change!).
Have a look at the resulting `scaled.top` file and find out what has changed.
You can also try to apply two different scaling factors and check the difference:
```
plumed partial_tempering 1.0 < processed.top  > scaled0.top
plumed partial_tempering 0.5 < processed.top  > scaled1.top
diff scaled0.top scaled1.top
```
### Exercise 2b: Sanity check on generated topologies

Notice that these scaled topologies can be used to run GROMACS simulations. The `parial_tempering` script is far from perfect. We will now make some sanity check. To do these checks, we will either generate a new trajectory or simply take one of the trajectories that we generated in the previous exercise. We will then used a GROMACS tool named `rerun`, which allows recomputing the energy along a trajectory using a new topology.

First, create an energy file corresponding to the original topology:
```
# it is better to do this in a separate directory and rename the
# trajectory file, or gromacs will complain:
gmx_mpi mdrun -rerun traj.xtc -s topol.tpr
```
The resulting `ener.edr` file can be converted to a text file with the `gmx_mpi energy` command.

Then, generate a new topology with scaling factor 1.0 and compute energies again:
```
plumed partial_tempering 1.0 < topol_selected.top > topol_scaled.top
gmx_mpi grompp -p topol_scaled.top -o topol_scaled.tpr
gmx_mpi mdrun -rerun traj.xtc -s topol_scaled.tpr -e ener_scaled.edr
```

The resulting energies should be identical!

As a second check,
generate a new topology where you scaled *all* atoms (make sure to also select water!) by a factor 0.5:
```
plumed partial_tempering 0.5 < topol_all_selected.top > topol_all_scaled.top
gmx_mpi grompp -p topol_all_scaled.top -o topol_all_scaled.tpr
gmx_mpi mdrun -rerun traj.xtc -s topol_all_scaled.tpr -e ener_all_scaled.edr
```

This time, dihedral angles, LJ, and Coulomb energies should be multiplied by 0.5.

This is a very important check. The `partial_tempering` script is not compatible with some specific functional forms,
e.g., CHARMM CMAP (search PLUMED mailing list for a fix).
In case you are using an incompatible force field, you will find inconsistencies in these sanity checks.
For the provided `topol.top` file, everything should work.

<b> These tests are very important, please only proceed if you managed to pass them.</b>

### Exercise 2c: Sanity check on replica-exchange implementation

Another thing that we will have to check now is if acceptance is computed correctly.
The code inserted in GROMACS to implement this calculation is non trivial and quite fragile.
To do this check, you should run a short Hamiltonian replica exchange with two equivalent topology files.
For instance, you can use the original `topol.tpr` file and the one that you obtained with scaling factor 1.0. Make sure that the two tpr files are using a different seed for randomizing the initial velocities or, even better, use two different `conf.gro` files to initialize the two simulations.
For everything else, use the same settings you will use in production (ideally, same number of processes per replica, same GPU settings, etc).

Now run a short replica exchange simulation:
```
mpiexec -np 2 --oversubscribe gmx_mpi mdrun -multidir dir0 dir1 -replex 200 -nsteps 10000 -hrex -plumed plumed.dat
```

As written above, `plumed.dat` can be just an empty file.
Check the resulting acceptance. Since the Hamiltonians are identical, the acceptance **should be exactly 1.0**.

Notice that whereas this is formally true there might be numerical issues that make this test fail.
Specifically:

- Dynamic load balancing (can be switched off with `mdrun -dlb no`)
- When using GPUs or a large number of MPI processes, GROMACS might also optimize the PME calculation dynamically, altering the acceptacte in this case (can be switched off with `mdrun -notunepme`).

See also [here](https://github.com/plumed/plumed2/issues/1177) on GitHub for a discussion of this issue.

### Exercise 2d: Find minimum scaling factor needed to cross the barrier

You are now able to generate tpr files where the energy of a subset of the atoms is scaled. We will do something similar to exercise 1a above, but playing with scaling factor instead of temperature. Make sure that you select all (and only) the atoms belonging to the solute (alanine dipeptide), so as to implement solute tempering.
Try to run a set of 1 ns long simulations, with scaling factors decreasing (e.g., 1.0, 0.9, 0.8, etc).
Now answer the following question:
- How much should you decrease the scaling factor to see transitions between the two metastable states in the first nanosecond?


### Exercise 3: Run Hamiltonian replica exchange simulations

Now run a Hamiltonian replica exchange simulation with multiple replicas, bridging from lambda=1.0 to the minimun value identified in the previous point. I would suggest using a linear distribution in lambda rather than a geometric one, but you can experiment.
Similarly to parallel tempering, we can for now just analyze the reference replica (at lambda=1.0).

Now answer the following questions:
- How many replicas (and which is the optimal distribution) you need to have an acceptance that is at least 30%? (the answer will depend on the minimum lambda you have chosen). Is this smaller or larger than the number of replicas that you used for exercise 1b above?
- How much is the relative population of the two metastable states at T=300? Plot this population as a function of lambda (you can just extract results from the different replicas). Is there any relationship between the dependence on lambda and the dependence on T that we have seen in exercise 1b above?


### Exercise 4: Analyze Hamiltonian replica exchange simulations with WHAM

So far we only analized the reference replica (lambda=1.0). We can however do better and combine all replicas with WHAM. To this aim you should:
- Concatenate all trajectories in a single file (`gmx_mpi trjcat -cat -f dir?/traj_comp.xtc dir??/traj_comp.xtc -o traj_multi.xtc`).
- Recompute the potential energy according to each of the tpr files (for each replica)
  (`gmx_mpi mdrun -rerun traj_multi.xtc -s dir0/topol.tpr -e energy0.edr`,
  `gmx_mpi mdrun -rerun traj_multi.xtc -s dir1/topol.tpr -e ener1.edr`, etc.).
- Convert the energies to text files
  (`echo 11 | gmx_mpi energy -f ener0.edr -xvg no -e energy0.xvg -xvg no`, etc.).
  11 should select the potential energy.

Then you can use the provided python wham script to compute the weight of all frames with the following python commands:

```python
import wham
print(wham.__file__) # make sure you are using the wham script provided with this masterclass
energies=[]
with cd("3/6reps"):
    for i in range(len(lambdas)):
        energies.append(np.loadtxt("energy{}.xvg".format(i),usecols=1))
energies=np.array(energies).T
# energies[i,j] is the energy of frame i according to the j-th Hamiltonian
kBT=0.00831446261815324*300
w=wham.wham(energies,T=kBT)
# w["logW"] are the Boltzmann factors
# notice that we did not specify yet to which ensemble we are reweighting
# this can be done with the following line:
logW=w["logW"]-energies[:,0]/kBT
# logW are the logaritm weights to obtain properties corresponding to replica 0
logW-=np.max(logW) # avoid numerical errors in exp
weights=np.exp(logW)
weights/=np.sum(weights) # normalize weights
# these weights can be used to compute weighted averages
```

Now plot the `weights` array and answer the following questions:
- How does the weight depend on the frame index? Remember that the replica exchange trajectories were concatenated.
  Can you recognize those frames coming from the first, second, etc replica?
- Compute the population of the two states by summing the weight of all states in the two basins.
- Modify the script above to compute weights corresponding to replicas other than the first one, and compute 
  the population of the two states as a function of lambda.

### Exercise 5: Optimize the lambda ladder

The aim of this exercise is to optimize the list of values of lambda. As a target, we will try to make the
product of the acceptances as large as possible.
We will do this *without* running new simulations, i.e. just analyzing the simulation above.

This is a difficult exercise. It will be solved in the solution, but it is **not** necessary to complete it so as to proceed to the next step.
I write some hints here.
- In solute tempering, the energy is a quadratic function of `sqrt(lambda)`. In other words, for each frame, you could
  write the energy as `A*lambda+B*sqrt(lambda)+C`, where `A`, `B`, and `C` are to be recomputed at each frame.
  Analyze the `ene` array that we generated in the last exercise so as to obtain these coefficients.
  The goal is to have a function that, for any frame, and for any value of lambda (including values not yet simulated!),
  returns the energy of the system.
- Using these functions, you will be able to compute weights corresponding to arbitrary values of lambda. You could for instance use these
  weights to generate a smooth version of the population vs lambda plot that we did in the last exercise.
- Given two values of lambda, you can compute the average acceptance by doing ensemble averages.
  You can easily test this function: use it to predict the acceptance associated to the neighboring replicas
  simulated above. Can you predict the acceptances seen during the simulation?

Once you have this function (let's call it `predict_acceptance(lambda1,lambda2)`) you can use the following script
to find the ladder that maximizes the product of the acceptances. Let's say that the largest lambda is fixed to 1.0
and the lowest is fixed to `lambda_min`

```python
# lambda is the array with the replicas we used for out simulation (a starting point(
# first and last replicas are fixed (1.0 and lambda_min)
# we thus optimize all elements excluding the first and the last (lambda[1:-1])

# this function compute the acceptances given the three intermediate values of
# lambda
def predict_all_acceptances(x):
    x=np.array(x)
    acc=[]
    # lambdas should not be negative
    if np.any(x<=0.0):
        return np.zeros(len(x)+1)
    # nor larger than 1
    if np.any(x>=1.0):
        return np.zeros(len(x)+1)
    # first replica has lambda=1.0
    acc.append(predict_acceptance(1.0,x[0]))
    for i in range(len(x)-1):
        acc.append(predict_acceptance(x[i],x[i+1]))
    # last replica has lambda=lambda_min
    acc.append(predict_acceptance(x[-1],lambda_min))
    return acc

# this is function to be minimized
def func(x):
    acc=predict_all_acceptances(x)
    return -np.prod(acc) # negative, since we minimize the result

from scipy.optimize import minimize
res=minimize(func,lambdas[1:-1]) # starting values

print("optimal lambdas: ",1.0,res.x,lambda_min)
print(predict_all_acceptances(res.x))

```

You can then run a new simulation with the optimal lambdas and check if the predicted acceptances correspond to the observed ones!


### Exercise 6: Combine with metadynamics on psi

As a very last point, repeat exercise the 5th exercise in [this masterclass](https://www.plumed.org/doc-v2.7/user-doc/html/masterclass-21-5.html) 
using alanine dipeptide in explicit solvent and solute tempering with the lambda values optimized above.
No further guidance is provided for this exercise, but it should not be difficult following
the suggestions given in [this masterclass](https://www.plumed.org/doc-v2.7/user-doc/html/masterclass-21-5.html).
