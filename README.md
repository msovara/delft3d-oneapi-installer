# Delft3D installation on LENGAU runnning CentOS-7


Delft3D is Open Source Software and facilitates the hydrodynamic (Delft3D-FLOW module), morphodynamic (Delft3D-MOR module), waves (Delft3D-WAVE module), water quality (Delft3D-WAQ module including the DELWAQ kernel) and particle (Delft3D-PART module) modelling. The source code can be downloaded here: https://oss.deltares.nl/web/delft3d. This repo is based on the installation tutorial here: https://gist.github.com/H0R5E/c4af6db788b227de702a12e01b64cf46 and https://www.youtube.com/watch?v=o1iQvO3x8A0. 

2018 Intel MPI compilation on CentOS-7

Using Tmux inside an interactive session,

## Set compile and runtime environment variables for Delf3D in a script to be sourced during compilation and execution
**The set script:** Run the bash file that sets the delft3d environment. The file needs to be made executable and sourced to apply the changes. 
```
#!/bin/bash

#===============================
# Delft3D Environment Setup
#===============================

# Function to check if directory exists
check_dir() {
    if [ ! -d "$1" ]; then
        echo "Error: Directory $1 does not exist!"
        return 1
    fi
}

# Function to validate compiler availability
check_compiler() {
    if ! command -v "$1" &> /dev/null; then
        echo "Error: Compiler $1 not found!"
        return 1
    fi
}

#---------------
# Module Setup
#---------------
echo "Setting up modules..."
module purge

# Load required modules
source /home/apps/chpc/compmech/compilers/intel/oneapi/setvars.sh || exit 1
module load chpc/compmech/mpich/4.2.2/oneapi2023-ssh
module load chpc/BIOMODULES
module add curl/7.50.0

#---------------
# Directory Setup
#---------------
export DelftDIR=/home/apps/chpc/earth/delft3d_mpich_oneapi
export DIR=${DelftDIR}/LIBRARIES
export NCDIR=${DIR}/netcdf-c-4.6.1
export NCFDIR=${DIR}/netcdf-fortran-4.5.0
export HDF5_DIR=${DIR}/hdf5-1.10.6

# Validate directories
for d in "${DelftDIR}" "${DIR}" "${HDF5_DIR}"; do
    check_dir "$d" || exit 1
done

#---------------
# Compiler Setup
#---------------
export I_MPI_ROOT=/home/apps/chpc/compmech/mpich-4.2.2-oneapi2023
export CC=mpicc
export CXX=mpicc
export FC=mpif90
export F77=mpif77

# Validate compilers
for compiler in mpicc mpif90 mpif77; do
    check_compiler "$compiler" || exit 1
done

#---------------
# Library Paths
#---------------
# Intel compiler libraries for libstdc++ compatibility
export LD_LIBRARY_PATH="\
/opt/intel/oneapi/compiler/latest/linux/compiler/lib/intel64_lin:\
${NCDIR}/lib:\
${HDF5_DIR}/lib:\
${LD_LIBRARY_PATH}"

#---------------
# Compiler Flags
#---------------
export CPPFLAGS="-I${NCDIR}/include -I${HDF5_DIR}/include"
export LDFLAGS="-L${NCDIR}/lib -L${HDF5_DIR}/lib"
export NETCDF_CFLAGS="-I${NCDIR}/include"
export NETCDF_LIBS="-L${NCDIR}/lib -lnetcdf"
export FCFLAGS="-O2 -fPIC -I${NCDIR}/include -I${HDF5_DIR}/include"
export FFLAGS=${FCFLAGS}
export CFLAGS=${FCFLAGS}
export CXXFLAGS="-O2 -fPIC"
export AM_FFLAGS='-lifcoremt'
export AM_LDFLAGS='-lifcoremt'

# Set unlimited stack size
ulimit -s unlimited

#---------------
# Verification
#---------------
echo "=== Environment Setup Summary ==="
echo "1. Loaded Modules:"
module list
echo -e "\n2. Critical Environment Variables:"
env | grep -E "DelftDIR|DIR|NCDIR|HDF5_DIR|LD_LIBRARY_PATH|MPICC|MPIF" | sort

echo -e "\nEnvironment setup completed successfully."
```

## Compile HDF5
```
# In the dtn node get the download the source code and decompress the file 
wget https://support.hdfgroup.org/ftp/HDF5/releases/hdf5-1.10/hdf5-1.10.6/src/hdf5-1.10.6.tar.bz2
tar -xf hdf5-1.10.6.tar.bz2

# Build and compile hdf5 with the appropriate flags and compiler choices
./configure --enable-parallel --enable-shared --prefix="/home/apps/chpc/earth/delft3d_mpich_oneapi/LIBRARIES/hdf5-1.10.6"
make
make install
cd ..
```

When running into ```Make``` errors, something to do with ```H5lib_settings.c```, generated during the HDF5 build process, it contains invalid syntax and conflicting information due to a problem in the environment or build configuration. The fix below seems to clear it:
```
export MXM_LOG_LEVEL=error
export UCX_LOG_LEVEL=error

# Rebuild HDF5
make clean
make -j4   # assign 4 processors to the task
make install
```

## Compile netcdf-c
```
# In the dtn node get the download the source code and decompress the file 
wget https://github.com/Unidata/netcdf-c/archive/refs/tags/v4.6.1.tar.gz -O netcdf-c-4.6.1.tar.gz
tar -xf netcdf-c-4.6.1.tar.gz

# Configure NetCDF with HDF5 and MPICH

./configure CC="${D3D_MPICC}" \
    --disable-dap-remote-tests \
    --with-hdf5=/home/apps/chpc/earth/delft3d_mpich_oneapi/LIBRARIES/hdf5-1.10.6 \
    --prefix=/home/apps/chpc/earth/delft3d_mpich_oneapi/LIBRARIES/netcdf-c-4.6.1 \   
```

## Compile netcdf-fortran
```
wget https://github.com/Unidata/netcdf-fortran/archive/refs/tags/v4.5.0.tar.gz -O netcdf-fortran-4.5.0.tar.gz
tar -xf netcdf-fortran-4.5.0.tar.gz
cd netcdf-fortran-4.5.0

./configure \
    --prefix=${NCFDIR} \
    --enable-shared \
    CC=${CC} \
    FC=${FC} \
    F77=${F77} \
    CPPFLAGS="-I${NCDIR}/include -I${HDF5_DIR}/include" \
    LDFLAGS="-L${NCDIR}/lib -L${HDF5_DIR}/lib" \
    LIBS="-lnetcdf -lhdf5_hl -lhdf5 -lz"
make clean
make -j4
make install
```


export DelftDIR=/home/apps/chpc/earth/delft3d
export DIR=$DelftDIR/LIBRARIES

export I_MPI_SHM="off"

export CC=/apps/compilers/intel/parallel_studio_xe_2018_update2/compilers_and_libraries_2018.2.199/linux/mpi/intel64/bin/mpicc
export CXX=icc
export FC=ifort
export FCFLAGS="-m64 -I$DIR/netcdf/include -I$DIR/grib2/include -I$DIR/hdf5-1.10.6/include"
export F77=ifort
export FFLAGS=$FCFLAGS
export CFLAGS=$FCFLAGS

export MPICC=/apps/compilers/intel/parallel_studio_xe_2018_update2/compilers_and_libraries/linux/mpi/intel64/bin/mpiicc
export MPIF90=/apps/compilers/intel/parallel_studio_xe_2018_update2/compilers_and_libraries/linux/mpi/intel64/bin/mpiifort
export MPIF77=/apps/compilers/intel/parallel_studio_xe_2018_update2/compilers_and_libraries/linux/mpi/intel64/bin/mpiifort

# HDF5 Paths
export HDF5_DIR="$DIR/hdf5-1.10.6"
export LDFLAGS="-L$HDF5_DIR/lib -L$DIR/netcdf-c-4.6.1/lib"
export CPPFLAGS="-I$HDF5_DIR/include -I$DIR/netcdf-c-4.6.1/include"
export LD_LIBRARY_PATH="$HDF5_DIR/lib:$DIR/netcdf-c-4.6.1/lib:$LD_LIBRARY_PATH"

# NetCDF Paths
export NCDIR="$DIR/netcdf-c-4.6.1"
export NETCDF_CFLAGS="-I$NCDIR/include -I$DIR/netcdf-c-4.6.1/include"
export NETCDF_LIBS="-L$NCDIR/lib -lnetcdf"

CFLAGS='-O2 ' CXXFLAGS='-O2 ' AM_FFLAGS='-lifcoremt ' FFLAGS='-O1 ' AM_FCFLAGS='-lifcoremt ' FCFLAGS='-O1 ' AM_LDFLAGS='-lifcoremt '

ulimit -s unlimited

```

# LIBRARIES

**hdf5**
```
wget https://support.hdfgroup.org/ftp/HDF5/releases/hdf5-1.10/hdf5-1.10.6/src/hdf5-1.10.6.tar.bz2
tar -xf hdf5-1.10.6.tar.bz2
cd hdf5-1.10.6
./configure --prefix=/home/apps/chpc/earth/delft3d/LIBRARIES/hdf5-1.10.6 --enable-parallel --enable-shared
make
make check
make install
cd ..
```

**netcdf-c**
```
wget https://github.com/Unidata/netcdf-c/archive/refs/tags/v4.6.1.tar.gz -O netcdf-c-4.6.1.tar.gz
tar -xf netcdf-c-4.6.1.tar.gz
cd netcdf-c-4.6.1
./configure --prefix=/home/apps/chpc/earth/delft3d/LIBRARIES/netcdf-c-4.6.1 --disable-dap-remote-tests
make 
make check
make install
cd ..
```
**netcdf-fortran**
```
wget https://github.com/Unidata/netcdf-fortran/archive/refs/tags/v4.5.0.tar.gz -O netcdf-fortran-4.5.0.tar.gz
tar -xf netcdf-fortran-4.5.0.tar.gz
cd netcdf-fortran-4.5.0
./configure --prefix=/home/apps/chpc/earth/delft3d/LIBRARIES/netcdf-fortran-4.5.0
make 
make check 
make install 
cd ..
```
 - If the **netcdf-fortran** build fails, regenerate the build files and run ./configure again:
```
aclocal
autoconf
automake --add-missing
```
**Install Delft3D**: 
With all the necessary dependencies installed, we can now download, compile and install Delft3D. Note that you must have a Deltares SVN server account to download the source code. If you need to register, see the “Steps needed to use the Delft3D source code” section of the Delft3D compilation guide. In the compilation instructions below, replace <username> and <password> with the SVN login details that Deltares sent you. The commands below will install tag 68819, the recommended tag for Delft3D-FM. This time the binaries are placed /home/apps/chpc/earth/Delft3D/bin directory.
```
svn checkout --username <username> --password <password> https://svn.oss.deltares.nl/repos/delft3d/tags/delft3dfm/68819/ delft3dfm-68819
cp delft3dfm-68819/src/third_party_open/swan/src/*.[fF]* delft3dfm-68819/src/third_party_open/swan/swan_mpi
cp delft3dfm-68819/src/third_party_open/swan/src/*.[fF]* delft3dfm-68819/src/third_party_open/swan/swan_omp
cd delft3dfm-68819/src
./autogen.sh --verbose 
./configure --prefix=/home/apps/chpc/earth/delft3d/delft3d
make ds-install 
make ds-install -C engines_gpl/dflowfm 
cd ..
```
# Test Delft3D
To test a structured mesh example, edit the run.sh file appropriately and issue the following commands:
```
pushd examples/01_standard
./run.sh
popd
```

To test a flexible mesh example, issue the following:
```
pushd examples/12_dflowfm/test_data/e02_f14_c040_westerscheldt
./run.sh
popd
```
