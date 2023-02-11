# Masterclass 22.10: Hamiltonian replica exchange with PLUMED and GROMACS

This lesson was given as part of the PLUMED masterclass series in 2022.  It includes:

* A video that explain the theory covered and a second video which shows you how the exercises should be completed.
* A series of exercises that you should try to complete yourself.
* Some supplementary python notebooks that provide further background information on the exercise.

The flow chart shown below indicates the order in which you should consult the resources.  You can click on the nodes to access the various resources.  Follow the thick black lines for the best results.  The resources that are connected by dashed lines are supplmentary resources that you may find useful when completing the exercise.

This lesson was the first masterclass in the 2021 series.

```mermaid
flowchart TB;
  A[ref1] ==> C[Lecture I];
  B[ref2] ==> C;
  A ==> D[Slides];
  B ==> D;
  C ==> E[Instructions];
  D ==> E;
  E ==> F[Lecture II];
  F ==> G[Solution];
  click A "ref1" "Instructions for how to install the version of PLUMED that you will need for this exercise";
  click B "ref2" "This lesson introduces shows you how to run replica exchange simulations. To get the most out of this exercise you should complete this earlier lesson first";
  click C "video1" "A lecture that was given on June 21st 2022 as part of the plumed masterclass series that introduces you to the exercises in this lesson";
  click D "slides.pdf" "The slides for the lecture that introduces the exercises";
  click E "INSTRUCTIONS.md" "The instructions for the exercises";
  click F "video2" "A lecture that was given on June 27th 2022 as part of the plumed masterclass series that goes through the solutions to the exercises in the lesson";
  click G "Solution.ipynb" "A python notebook that contains a full set of soluations to the exercises that are discussed in the masterclass.  This notebook is the one that was editted during the second video lecture of the masterclass";
```
