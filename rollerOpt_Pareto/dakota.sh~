#!/bin/bash

#Adapted from tutorial located at: http://www.holzmann-cfd.de/index.php/en/
# $1 change from DAKOTA to OF file
# $2 pipe OF output to DAKOTA

# Update OF temperature through DAKOTA
# ------------------------------------------------------------------------------
dprepro $1 system/dakotaParameters.orig system/dakotaParameters

# Run simulation with new parameter set
# ------------------------------------------------------------------------------
    
    # Get parameters and remove possible scientific notation
    #---------------------------------------------------------------------------
    burnerT=`head -5 system/dakotaParameters | tail -1 | cut -d'=' -f2`
    rollerOmega=`head -6 system/dakotaParameters | tail -1 | cut -d'=' -f2`
    burnerUy=`head -7 system/dakotaParameters | tail -1 | cut -d'=' -f2`
    rollerDist=`head -8 system/dakotaParameters | tail -1 | cut -d'=' -f2`

    burnerT=`echo $burnerT | sed -e 's/[eE]+*/\*10\^/'`
    rollerOmega=`echo $rollerOmega | sed -e 's/[eE]+*/\*10\^/'`
    burnerUy=`echo $burnerUy | sed -e 's/[eE]+*/\*10\^/'`
    rollerDist=`echo $rollerDist | sed -e 's/[eE]+*/\*10\^/'`

    # Rescale parameters to real values
    #---------------------------------------------------------------------------
    burnerTscaled=`echo "$burnerT*1500 + 300" | bc -l`
    rollerOmegaScaled=`echo "$rollerOmega*17 + 3" | bc -l`
    burnerUyScaled=`echo "$burnerUy" | bc -l`
    rollerDistScaled=`echo "$rollerDist*0.04 + 0.135" | bc -l`

    # Print current loop number and parameter values
    #---------------------------------------------------------------------------
    loopNumber=`cat .optimizationLoop`
    >&2 echo -e ""
    >&2 echo -e "Optimization Iteration $loopNumber"
    >&2 echo -e "Inlet Temperature = $burnerTscaled"
    >&2 echo -e "Roller Angular Velocity = $rollerOmegaScaled"
    >&2 echo -e "Inlet Vertical Velocity = $burnerUyScaled"
    >&2 echo -e "Roller Vertical Separation = $rollerDistScaled"
    >&2 echo -e ""

    echo "$burnerT $rollerOmega $burnerUy $rollerDist" >> .parameterFile
    
    # Set new parameter values
    #---------------------------------------------------------------------------
    cp -rf 0_backup 0

    sed -i "40s/.*/	value		uniform $burnerTscaled;/" 0/T
    sed -i "29s/.*/	omega		$rollerOmegaScaled;/" 0/U
    sed -i "36s/.*/	value		uniform (0 $burnerUyScaled 0);/" 0/U
    
    cp system/moveRollerDict system/moveRoller
    sed "s/rollerDist/$rollerDistScaled/" system/moveRoller -i
    sed -i "27s/.*/	origin		(0 $rollerDistScaled 0.05);/" 0/U
    
    # Create mesh
    #---------------------------------------------------------------------------
    source createMesh

    # Make folder for log files
    #---------------------------------------------------------------------------
    logFolder="runLogs/Optimization"$loopNumber
    mkdir -p $logFolder

    # Run OpenFOAM
    #---------------------------------------------------------------------------
    source runSolver

    # Get Desired Quantities
    #---------------------------------------------------------------------------

    meanT=`grep -r "Patch Mean : " log.fireDyMFoam | awk '{print $4}'`
    gradT=`grep -r "Patch Gradient : " log.fireDyMFoam | awk '{print $4}'`

    varT=`grep -r "Patch Variance : " log.fireDyMFoam | awk '{print $4}'`
    varT=`echo ${varT} | sed -e 's/[eE]+*/\\*10\\^/'`
    rollerOmega=`echo $rollerOmega | sed -e 's/[eE]+*/\*10\^/'`

    # Update cost function
    #---------------------------------------------------------------------------

    func=`echo "$varT - $rollerOmega - $gradT" | bc -l`

    echo $func > .dakotaInput.dak
    echo -e "$func" >> .costFunc.dat

    >&2 echo -e ""
    >&2 echo "Roller Temperature Mean : $meanT"
    >&2 echo "Roller Temperature Variance : $varT"
    >&2 echo "Cost Function : $func"
    >&2 echo -e ""
    
    mv log* $logFolder
    finalTime=$(foamListTimes | tail -1)
    
    >&2 echo "Final Time : " $finalTime
    
    mv $finalTime $logFolder
    cp -rf constant $logFolder
    cp -rf system $logFolder

    # Clean Case
    #---------------------------------------------------------------------------
    source Allclean

    # Increase the loop number and store in dummy file
    #--------------------------------------------------------------------------
    echo $((loopNumber+1)) > .optimizationLoop

# Pipe output file to DAKOTA
#------------------------------------------------------------------------------
cp .dakotaInput.dak $2

sleep 1

#------------------------------------------------------------------------------
