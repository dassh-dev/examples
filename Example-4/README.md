# Example 4: Orificing optimization demonstration
Prepared 2021-11-03 using DASSH v0.8.0

## Introduction

In this problem, we utilize the hypothetical system introduced in [Example 2](https://github.com/dassh-dev/examples/tree/master/Example-2). The system is comprised of 37 assemblies, of which the inner 19 are fuel-type assemblies and the outer 18 are reflectors. We will use this example to demonstrate the DASSH orificing optimization algorithm. The algorithm is also demonstrated in a spreadsheet title "hand_calculation.xlsx".

## Input file overview

The input file is named `input.txt`. The input is similar to that from Example 2, but features some important differences.

First, two fuel assemblies are used: one with 271 pins (`fuel`) and one with 217 pins (`fuel2`). The purpose of this distinction is to highlight that the DASSH orificing algorithm can accommodate different assembly designs. Both assemblies have the `FuelModel` block specified so that DASSH will calculate fuel pin temperatures. The reflectors are modeled using the low-fidelity model that homogenizes the assembly to a single channel.

Second, the `Orificing` input section is added to control the optimization and is shown below.

```
[Orificing]
    assemblies_to_group = fuel, fuel2
    n_groups = 3
    value_to_optimize = peak fuel temp
    bulk_coolant_temp = 773.15
    convergence_tol = 0.002
```

Here, we identify the assemblies to be included in the optimization; the desired number of orifice groups; the parameter that we are seeking to optimize (minimize); the target coolant outlet temperature to constrain coolant flow rate; and a tolerance on convergence of the iterations to distribute coolant flow among orifice groups.

It should be noted that in this example, only one timestep is utilized, but DASSH has the capability to optimize orifice groups over multiple timesteps. For that, rather than providing just one file for each of the required binary files from ARC, the user would provide a list of files; for BOC/EOC, the user would provide two CCCC files for each required file type, one for BOC and one for EOC.

Lastly, assemblies are assigned to core locations in the `Assignment` block. The inner 7 fuel assemblies are `fuel`, and the outer 12 fuel assemblies (in ring 3) are `fuel2`. The outer ring of assemblies are reflectors. For the orificing calculation, the user must still provide a condition by which to determine the coolant flow rate in each assembly, but DASSH will override it when performing iterations to distribute coolant flow among orificing groups. As such, the values shown in the input are a placeholder.

## Running DASSH

With the input file prepared and the CCCC binary files ready (in subdirectory `cccc`), we can run DASSH. Once installed, DASSH is run from the command line just like any other executable.

```
dassh input.txt

```

Inclusion of the `Orificing` section in the input triggers the orificing optimization calculation, rather than the basic DASSH sweep. The orificing procedure occurs as follows:

1. DASSH determines total power and peak-average linear power for each assembly and averages them over all timesteps. The peak-average linear power is the maximum (over axial space) of the average (over pins) linear power. Depending on the optimization variable of interest, DASSH uses one of these parameters to perform the orifice grouping. DASSH assigns the highest power assembly to Group 1. Then the second highest power assembly is assigned to Group 1, until `(max(Group 1 power) - min(Group 1 power)) / average(Group 1 power) > cutoff`. Then, the next assembly would be assigned to Group 2. Once all assemblies are assigned to groups, if the number of groups is not equal to the desired number of groups, the cutoff value is modified and the process is repeated. This creates the `_power` subdirectory. For this calculation, [this figure](https://github.com/dassh-dev/examples/tree/master/Example-2/CoreHexPlot_total_power.png) from Example 2 shows the total power in each assembly and thereby gives a strong indication of how the assemblies will be grouped.

2. Using a single assembly calculation (with no inter-assembly heat transfer), DASSH performs parametric calculations to determine the dependence of the optimization variable on the power-to-flow ratio for each assembly type included in the optimization. It was observed that the power-to-flow ratio is strongly correlated with peak coolant, peak clad, and peak fuel temperature. The results of the parametric calculation enables fast iterations. This creates the `_parametric` subdirectory

3. DASSH distributes coolant flow to the orifice groups. First, the total flow rate required to achieve the target bulk coolant outlet temperature is estimated. Then, coolant flow to each group is adjusted until the maximum value of the optimization variable (in this case, peak fuel temperature) is the same across all groups. For this first iteration, that is informed by the parametric analysis alone. DASSH is then run for each timestep and summary results are generated, producing the `_iter1` subdirectory.

4. Step (3) is repeated, but with some corrections. First, the total flow rate is adjusted based on the bulk coolant outlet temperature. Then, coolant flow is again distributed among groups until the maximum value of the optimization variable is equal across all groups. This time, the value used as an estimate for each assembly is corrected based on the ratio between the value from the parametric data and the value observed in the previous iteration. This accounts for the effects of inter-assembly heat transfer, non-symmetric power generation, and variability in material properties. DASSH is then run again for each timestep, producing the `_iter2` subdirectory.

5. Step (4) is repeated until the distribution of flow among coolant channels achieves a result within the covergence criteria specified in the input. For this calculation, only two iterations are required.

During the calculation, DASSH prints information to the terminal to indicate its status. After the title ASCII art, the terminal output for this sample problem is shown below:

```

      ......@@@@@@......@@@@......@@@@......@@@@....@@....@@......
    ........@@...@@....@@@@@@...@@@@@@@@..@@@@@@@@..@@....@@........
  ..........@@....@@..@@....@@..@@........@@........@@....@@..........
............@@....@@..@@@@@@@@...@@@@@@....@@@@@@...@@@@@@@@............
  ..........@@....@@..@@....@@........@@........@@..@@....@@..........
    ........@@...@@...@@....@@..@@@@@@@@..@@@@@@@@..@@....@@........
      ......@@@@@@....@@....@@....@@@@......@@@@....@@....@@......

DASSH....DASSH logger initialized
DASSH....Reading input: input.txt
DASSH....DASSH orificing optimization calculation
DASSH....Value to minimize: "peak fuel temp"
DASSH....Requested number of groups: 3
DASSH....Grouping assemblies
DASSH....Performing single-assembly parametric calculations
DASSH....Iter 1: Distributing coolant flow among groups
T_opt_max:  864.3143637429398
Asm per group:  [7 6 6]
Flow rates:  [29.53767165 16.17237078 13.46793451]
Total flow rate:  384.6055332626699
DASSH....Iter 1: Running DASSH with orificed assemblies
DASSH....Setting up power distribution
DASSH....Reading core power profile from VARPOW output files
DASSH....Generating Assembly objects
DASSH....    Assembly "fuel" param "number of rods in bundle" is outside the acceptable range for correlation "friction_ctd"
DASSH....    Assembly "fuel" param "number of rods in bundle" is outside the acceptable range for correlation "flowsplit_ctd"
DASSH....Generating Core object
DASSH....Total power (W): 72000000.0
DASSH....Total flow rate (kg/s): 392.8041
DASSH....Axial step size required (m): 0.005047
DASSH....800 axial steps required
DASSH....Performing temperature sweep...
DASSH....Progress: plane   50 of 800; z = 0.25 m; cumulative sweep time = 00:00:00.76
DASSH....Progress: plane  100 of 800; z = 0.50 m; cumulative sweep time = 00:00:01.63
DASSH....Progress: plane  150 of 800; z = 0.75 m; cumulative sweep time = 00:00:02.41
DASSH....Progress: plane  200 of 800; z = 1.00 m; cumulative sweep time = 00:00:03.16
DASSH....Progress: plane  250 of 800; z = 1.25 m; cumulative sweep time = 00:00:04.05
DASSH....Progress: plane  300 of 800; z = 1.50 m; cumulative sweep time = 00:00:04.90
DASSH....Progress: plane  350 of 800; z = 1.75 m; cumulative sweep time = 00:00:05.75
DASSH....Progress: plane  400 of 800; z = 2.00 m; cumulative sweep time = 00:00:06.61
DASSH....Progress: plane  450 of 800; z = 2.25 m; cumulative sweep time = 00:00:07.45
DASSH....Progress: plane  500 of 800; z = 2.50 m; cumulative sweep time = 00:00:08.29
DASSH....Progress: plane  550 of 800; z = 2.75 m; cumulative sweep time = 00:00:09.13
DASSH....Progress: plane  600 of 800; z = 3.00 m; cumulative sweep time = 00:00:09.94
DASSH....Progress: plane  650 of 800; z = 3.25 m; cumulative sweep time = 00:00:10.65
DASSH....Progress: plane  700 of 800; z = 3.50 m; cumulative sweep time = 00:00:11.34
DASSH....Progress: plane  750 of 800; z = 3.75 m; cumulative sweep time = 00:00:12.01
DASSH....Progress: plane  800 of 800; z = 4.00 m; cumulative sweep time = 00:00:12.66
DASSH....Temperature sweep complete
DASSH....Output written
[[759.37490786 769.36725861 865.61436369 872.98956559]
[787.69447411 787.69490158 861.02230229 861.02237313]
[786.81390848 786.81553537 859.81903119 859.82165656]
[772.28488108 787.69490158 863.23817825 872.98956559]]
Convergence:  0.026340904861134953
DASSH....Iter 2: Distributing coolant flow among groups
T_opt_max:  869.8222484000441
Asm per group:  [7 6 6]
Flow rates:  [30.36142425 15.44843935 12.8483735 ]
Total flow rate:  382.3108468853952
DASSH....Iter 2: Running DASSH with orificed assemblies
DASSH....Setting up power distribution
DASSH....Reading core power profile from VARPOW output files
DASSH....Generating Assembly objects
DASSH....    Assembly "fuel" param "number of rods in bundle" is outside the acceptable range for correlation "friction_ctd"
DASSH....    Assembly "fuel" param "number of rods in bundle" is outside the acceptable range for correlation "flowsplit_ctd"
DASSH....Generating Core object
DASSH....Total power (W): 72000000.0
DASSH....Total flow rate (kg/s): 390.4862
DASSH....Axial step size required (m): 0.005021
DASSH....800 axial steps required
DASSH....Performing temperature sweep...
DASSH....Progress: plane   50 of 800; z = 0.25 m; cumulative sweep time = 00:00:00.69
DASSH....Progress: plane  100 of 800; z = 0.50 m; cumulative sweep time = 00:00:01.40
DASSH....Progress: plane  150 of 800; z = 0.75 m; cumulative sweep time = 00:00:02.14
DASSH....Progress: plane  200 of 800; z = 1.00 m; cumulative sweep time = 00:00:02.87
DASSH....Progress: plane  250 of 800; z = 1.25 m; cumulative sweep time = 00:00:03.70
DASSH....Progress: plane  300 of 800; z = 1.50 m; cumulative sweep time = 00:00:04.56
DASSH....Progress: plane  350 of 800; z = 1.75 m; cumulative sweep time = 00:00:05.41
DASSH....Progress: plane  400 of 800; z = 2.00 m; cumulative sweep time = 00:00:06.27
DASSH....Progress: plane  450 of 800; z = 2.25 m; cumulative sweep time = 00:00:07.13
DASSH....Progress: plane  500 of 800; z = 2.50 m; cumulative sweep time = 00:00:07.99
DASSH....Progress: plane  550 of 800; z = 2.75 m; cumulative sweep time = 00:00:08.82
DASSH....Progress: plane  600 of 800; z = 3.00 m; cumulative sweep time = 00:00:09.64
DASSH....Progress: plane  650 of 800; z = 3.25 m; cumulative sweep time = 00:00:10.37
DASSH....Progress: plane  700 of 800; z = 3.50 m; cumulative sweep time = 00:00:11.08
DASSH....Progress: plane  750 of 800; z = 3.75 m; cumulative sweep time = 00:00:11.76
DASSH....Progress: plane  800 of 800; z = 4.00 m; cumulative sweep time = 00:00:12.42
DASSH....Temperature sweep complete
DASSH....Output written
[[756.04829929 765.52460066 862.64862105 869.88786631]
[794.82987667 794.83031813 869.88753378 869.88760991]
[794.15617703 794.15794066 869.52278815 869.52565618]
[773.13500925 794.83031813 865.78981004 869.88786631]]
Convergence:  0.00037499549795151556
DASSH....DASSH execution complete

```

Prior to each iteration, DASSH reports its guess at the minimized optimization variable, the number of assemblies in each group, the group-wise flow rates, and the total flow rate. After each iteration, DASSH reports some summary data for each of the groups. The 4x4 table columns are average bulk coolant, peak bulk coolant, average optimization variable (fuel temperature), and peak optimization variable. The rows are for Group 1, Group 2, Group 3, and core-total. This illustrates how in just two iterations, DASSH was able to distribute the coolant flow such that the peak fuel temperature in the hypothetical system is 869.9 K and roughly equal across all assemblies.

## Results

The files produced by DASSH during each iteration are located in the iteration subdirectories, `_iter1` and `_iter2`. If multiple timesteps were included, each would have it's own subdirectory, e.g. `_iter1/timestep_0`. The summary output table for the orificing calculation that details results on an assembly-by-assembly basis is written in the main directory and titled "orificing_result_assembly.csv". A subset of the data from that file is shown here.

|Asm ID |Name  |Power (W) |Peak avg-lin power (W/m) | Group | Flow rate (kg/s) |Bulk cool (K) |Peak cool (K) |Peak clad MW (K) |Peak clad ID (K) |Peak fuel CL (K) |
|-------|------|----------|-------------------------|-------|------------------|--------------|--------------|-----------------|-----------------|-----------------|
|0      |fuel  |5.36E+06  |1.38E+04                 |0      |30.36             |765.52        |791.15        |793.01           |794.08           |869.89           |
|1      |fuel  |4.83E+06  |1.25E+04                 |0      |30.36             |754.57        |783.17        |784.78           |785.87           |861.51           |
|2      |fuel  |4.83E+06  |1.25E+04                 |0      |30.36             |754.45        |783.01        |784.62           |785.71           |861.43           |
|3      |fuel  |4.83E+06  |1.25E+04                 |0      |30.36             |754.45        |783.01        |784.62           |785.71           |861.43           |
|4      |fuel  |4.83E+06  |1.25E+04                 |0      |30.36             |754.45        |783.01        |784.62           |785.71           |861.43           |
|5      |fuel  |4.83E+06  |1.25E+04                 |0      |30.36             |754.45        |783.01        |784.62           |785.71           |861.43           |
|6      |fuel  |4.83E+06  |1.25E+04                 |0      |30.36             |754.45        |783.01        |784.62           |785.71           |861.43           |
|7      |fuel2 |2.79E+06  |8.98E+03                 |2      |12.85             |794.15        |853.54        |854.25           |855.00           |869.51           |
|8      |fuel2 |3.35E+06  |1.08E+04                 |1      |15.45             |794.83        |847.75        |848.03           |848.91           |869.89           |
|9      |fuel2 |2.79E+06  |8.98E+03                 |2      |12.85             |794.16        |853.56        |854.27           |855.02           |869.53           |
|10     |fuel2 |3.35E+06  |1.08E+04                 |1      |15.45             |794.83        |847.75        |848.03           |848.91           |869.89           |
|11     |fuel2 |2.79E+06  |8.98E+03                 |2      |12.85             |794.16        |853.56        |854.27           |855.02           |869.53           |
|12     |fuel2 |3.35E+06  |1.08E+04                 |1      |15.45             |794.83        |847.75        |848.03           |848.91           |869.89           |
|13     |fuel2 |2.79E+06  |8.98E+03                 |2      |12.85             |794.16        |853.56        |854.27           |855.02           |869.53           |
|14     |fuel2 |3.35E+06  |1.08E+04                 |1      |15.45             |794.83        |847.75        |848.03           |848.91           |869.89           |
|15     |fuel2 |2.79E+06  |8.98E+03                 |2      |12.85             |794.16        |853.56        |854.27           |855.02           |869.53           |
|16     |fuel2 |3.35E+06  |1.08E+04                 |1      |15.45             |794.83        |847.75        |848.03           |848.91           |869.89           |
|17     |fuel2 |2.79E+06  |8.98E+03                 |2      |12.85             |794.16        |853.56        |854.27           |855.02           |869.53           |
|18     |fuel2 |3.35E+06  |1.08E+04                 |1      |15.45             |794.83        |847.75        |848.03           |848.91           |869.89           |

## Additional capabilities and future developments

DASSH can apply the optimization demonstrated here to optimize other parameters: peak coolant temperature, peak clad midwall temperature, peak inner clad temperature, and peak fuel centerline temperature (shown). If multiple timesteps are included, DASSH can run them in parallel. Additional documentation on these features is forthcoming. Currently, the algorithm does not include the capability to shuffle orifice groups to improve the result once they are initially set, but development of that capability is planned.
