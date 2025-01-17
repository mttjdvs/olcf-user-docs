.. _containers-on-summit:

********************
Containers on Summit
********************

Users can build container images with Podman, convert it to a SIF file, and run the
container using the Singularity runtime. This page is intended for users with some
familiarity with building and running containers.

Basic Information
=================

Users will make use of two applications on Summit - Podman and Singularity - in their
container build and run workflow. Both are available without needing to load any modules.

Podman is a container framework from Red Hat that functions as a drop in replacement for
Docker. On Summit, we will use Podman for building container images and converting them
into tar files for Singularity to use. Podman makes use of Dockerfiles to describe the
images to be built. We cannot use Podman to run
containers as it doesn't properly support MPI on Summit, and Podman does not support
storing its container images on GPFS or NFS.

Due to Podman's lack of support for storage on GPFS and NFS, container images will be
built on the login nodes using the node-local NVMe on the login node. This NVMe is mounted
in ``/tmp/containers``. Users should treat this storage as temporary. Any data (container
image layers or otherwise) in this storage will be purged if the node is ever rebooted or
when it gets full.  So any images created with Podman need to be converted to tar files
using ``podman save`` and stored elsewhere if you wish to preserve your image.

.. note::
   The NVMes on the login nodes are not shared storage across all login nodes. Each login
   node is different with an independent NVMe. So you will not find the container layers
   built on one login node to appear in another login node. Every time you login to
   Summit, the load balancer may put you on a different login node than where you were
   working on building your container. It is the user's responsibility to save any in
   progress or completed container image build as a tar file or sif file before you close
   a session.

Singularity is a container framework from Sylabs. On Summit, we will use Singularity
solely as the runtime. We will convert the tar files of the container images Podman
creates into sif files, store those sif files on GPFS, and run them with
Jsrun. Singularity also allows building images but ordinary users cannot utilize that on
Summit due to additional permissions not allowed for regular users.

Users will be building and running containers on Summit without root permissions
i.e. containers on Summit are rootless.  This means users can get the benefits of
containers without needing additional privileges. This is necessary for a shared system
like Summit. And this is part of the reason why Docker doesn't work on Summit. Podman and
Singularity provides rootless support but to different extents hence why users need to use
a combination of both.



Setup before Building
=====================

Users will need to set up a file in their home directory
``/ccs/home/<username>/.config/containers/storage.conf`` with the following content:
::

   [storage]
   driver = "overlay"
   graphroot = "/tmp/containers/<user>"
   
   [storage.options]
   additionalimagestores = [
   ]
   
   [storage.options.overlay]
   ignore_chown_errors = "true"
   mount_program = "/usr/bin/fuse-overlayfs"
   mountopt = "nodev,metacopy=on"
   
   [storage.options.thinpool]

``<user>`` in the ``graphroot = "/tmp/containers/<user>"`` in the above file should be
replaced with your username. This will ensure that Podman will use the NVMe mounted in ``/tmp/containers`` for storage during container image builds.


Build and Run Workflow 
=======================

As an example, let's build and run a very simple container image to demonstrate the workflow.

Building a Simple Image
-----------------------

- Create a directory called ``simplecontainer`` on home or GPFS and ``cd`` into it.
- Create a file named ``simple.dockerfile`` with the following contents.
  ::

     FROM quay.io/centos/centos:stream8
     RUN dnf -y install epel-release && dnf -y install fakeroot
     RUN fakeroot dnf upgrade -y && fakeroot dnf update -y
     RUN fakeroot dnf install -y wget hostname
     ENTRYPOINT ["/bin/bash"]

.. note::
   You will notice the use of the fakeroot command when doing package installs with dnf. This is necessary as some some package installations require root permissions on container which the container builder does not have. So fakeroot allows dnf to think it is running as root and allows the installation to succeed.
     
- Build the container image with ``podman build -t simple -f simple.dockerfile .``.

  * The ``-t`` flag names the container image and the ``-f`` flag indicates the file to use for building the image.

- Run ``podman image ls`` to see the list of images. ``localhost/simple`` should be among them. Any container created without an explicit url to a container registry in its name will automatically have the ``localhost`` prefix.
  ::

     $ podman image ls
     REPOSITORY             TAG      IMAGE ID      CREATED      SIZE
     localhost/simple       latest   e47dbfde3e99  3 hours ago  687 MB
     quay.io/centos/centos  stream8  ad6f8b5e7f64  8 days ago   497 MB

- Convert this Podman container image into a tar file with ``podman save -o simple.tar localhost/simple``.
- Convert the tar file into a Singularity sif file with  ``singularity build --disable-cache simple.sif docker-archive://simple.tar``


Using a Container Registry to Build and Save your Images
--------------------------------------------------------

If you are familiar with using a container registry like DockerHub, you can use that to save your Podman container images
and use Singularity to pull from the registry and build the sif file. Below, we will use DockerHub as the example but there are many
other container registries that you can use.

- Using the ``simple`` example from the previous section, build the container image with ``podman build -t docker.io/<username>/simple -f simple.dockerfile .`` where ``<username>`` is your user on DockerHub.

  - ``podman push`` uses the URL in the container image's name to push to the appropriate registry.

- Check if your image is created
  ::

     $ podman image ls
     REPOSITORY                         TAG      IMAGE ID      CREATED      SIZE
     docker.io/subilabrahamornl/simple  latest   e47dbfde3e99  3 hours ago  687 MB
     localhost/simple                   latest   e47dbfde3e99  3 hours ago  687 MB
     quay.io/centos/centos              stream8  ad6f8b5e7f64  8 days ago   497 MB
     
- Run ``podman login docker.io`` and enter your account's username and password so that Podman is logged in to the container registry before pushing.

- Push the container image to the registry with ``podman push docker.io/<username>/simple``.

-  You can now create a Singularity sif file with ``singularity build --disable-cache --docker-login simple.sif docker://docker.io/<username>/simple``.

   - This will ask you to enter your Docker username and password again for Singularity to download the image from Dockerhub and convert it to a sif file.

.. note::
   The reason we include the ``--disable-cache`` flag is because Singularity's caching can
   fill up your home directory without you realizing it. And if the home directory is
   full, Singularity builds will fail. If you wish to make use of the cache, you can set
   the environment variable
   ``SINGULARITY_CACHEDIR=/tmp/containers/<user>/singularitycache`` or something like that
   so that the NVMe storage is used as the cache.

Running a Simple Container in a Batch Job
-----------------------------------------

As a simple example, we will run ``hostname`` with the Singularity container.

- Create a file submit.lsf with the contents below.
  ::

     #!/bin/bash
     # Begin LSF Directives
     #BSUB -P STF007
     #BSUB -W 0:10
     #BSUB -q debug
     #BSUB -nnodes 1
     #BSUB -J simple_container_job
     #BSUB -o simple_container_job.%J
     #BSUB -e simple_container_job.%J

     jsrun -n2 singularity exec ./simple.sif hostname

- Submit the job with ``bsub submit.lsf``. This should produce an output that looks like:
  ::

     h41n08
     h41n08

  Here, Jsrun starts 2 separate Singularity container runtimes since we pass the -n2 flag to start two processes. Each Singularity container runtime then loads the container image simple.sif and executes the ``hostname`` command from that container. If we had requested 2 nodes in the batch script and had run ``jsrun -n2 -r1 singularity exec ./simple.sif hostname``, Jsrun would've started a Singularity runtime on each node and the output would look something like 
  ::

     h41n08
     h41n09


Running an MPI program with the OLCF MPI base image
--------------------------------------------------- 

Creating Singularity containers that run MPI programs require a few additional steps. 

OLCF provides an MPI base image that you can use for MPI programs. You can pull it with Podman with ``podman pull code.ornl.gov:4567/olcfcontainers/olcfbaseimages/mpiimage-centos-cuda``


Let's build an simple MPI example container using the prebuilt MPI base image from the repository.

- Create a new directory ``mpiexample``.
- Create a file ``mpiexample.c`` with the following contents.
  ::

     #include <stdio.h>
     #include <mpi.h>
     
     int main (int argc, char *argv[])
     {
     int rank, size;
     MPI_Comm comm;
     
     comm = MPI_COMM_WORLD;
     MPI_Init (&argc, &argv);
     MPI_Comm_rank (comm, &rank);
     MPI_Comm_size (comm, &size);
     
     printf("Hello from rank %d\n", rank);
     
     MPI_Barrier(comm);
     MPI_Finalize();
     }

- Create a file named ``mpiexample.dockerfile`` with the following contents
  ::

     FROM code.ornl.gov:4567/olcfcontainers/olcfbaseimages/mpiimage-centos-cuda:latest
     RUN mkdir /app
     COPY mpiexample.c /app
     RUN cd /app && mpicc -o mpiexample mpiexample.c

- The MPI base image only supports gcc/9.1.0 at the moment in order to be able to compile an MPI program during the container build.
  So run the following commands to build the Podman image and convert it to the Singularity format.
  ::

     module purge
     module load DefApps
     module load gcc/9.1.0
     module -t list
     podman build -v $MPI_ROOT:$MPI_ROOT -f mpiexample.dockerfile -t mpiexample:latest .;
     podman save -o mpiexampleimage.tar localhost/mpiexample:latest;
     singularity build --disable-cache mpiexampleimage.sif docker-archive://mpiexampleimage.tar;

- It's possible the singularity build step might get killed due to reaching cgroup memory limit. To get around this, you can start an interactive job and build the singularity image with
  ::

     jsrun -n1 -c42 -brs singularity build --disable-cache mpiexampleimage.sif docker-archive://mpiexampleimage.tar;


- Create the following submit script submit.lsf. Make sure you replace the ``#BSUB -P STF007`` line with your own project ID.
  ::

     #BSUB -P STF007
     #BSUB -W 0:30
     #BSUB -nnodes 2
     #BSUB -J singularity
     #BSUB -o singularity.%J
     #BSUB -e singularity.%J
     
     module purge
     module load DefApps
     module load  gcc/9.1.0
     
     source /gpfs/alpine/stf007/world-shared/containers/utils/requiredmpilibs.source
     
     jsrun -n 8 -r4  singularity exec --bind $MPI_ROOT:$MPI_ROOT,/autofs/nccs-svm1_home1,/autofs/nccs-svm1_home1:/ccs/home mpiexampleimage.sif /app/mpiexample
     
     # uncomment the below to run the preinstalled osubenchmarks from the container.
     #jsrun -n 8 -r 4 singularity exec --bind $MPI_ROOT:$MPI_ROOT,/autofs/nccs-svm1_home1,/autofs/nccs-svm1_home1:/ccs/home mpiimage.sif /osu-micro-benchmarks-5.7/mpi/collective/osu_allgather


You can view the Dockerfiles used to build the MPI base image at the `code.ornl.gov
repository <https://code.ornl.gov/olcfcontainers/olcfbaseimages>`_. These Dockerfiles are
buildable on Summit yourself by cloning the repository and running the ``./build`` in the
individual directories in the repository. This allows you the freedom to modify these base
images to your own needs if you don't need all the components in the base images. You may
run into the cgroup memory limit when building so kill the podman process, log out, and
try running the build again if that happens when building.

.. note::
   Treat your Podman storage as temporary storage. The NVMes on the login nodes
   will be purged on reboot or if it becomes too full with no warning. It is the user's
   responsibility to make sure they to save any images they have as tar files or sif files
   and load them on a new session.


Tips and Tricks
=================

- Run ``podman system prune`` and then run ``podman image rm --force $(podman image ls
  -aq)`` several times to clean out all the dangling images and layers if you want to do a
  full reset.
- Sometimes you may want to do a full purge of your container storage area. Your user
  should own all the files in your ``/tmp/containers`` location. Recursively add write
  permissions to all files by running ``chmod -R +w /tmp/containers/<username>`` and then
  run ``rm -r /tmp/containers/<username>``.
- Sometimes you may need to kill your podman process because you may have gotten killed
  due to hitting cgroup limit. You can do so with ``pkill podman``, then log out and log
  back in to reset your cgroup usage.
- If you already have a "image.tar" file created with ``podman save`` from earlier that
  you are trying to replace, you will need to delete it first before running any other
  ``podman save`` to replace it. ``podman save`` won't overwrite the tar file for you.
- Not using the ``--disable-cache`` flag in your ``singularity build`` commands could
  cause your home directory to get quickly filled by singularity caching image data. You
  can set the cache to a location in ``/tmp/containers`` with ``export
  SINGULARITY_CACHEDIR=/tmp/containers/<username>/singularitycache`` if you want to avoid
  using the ``--disable-cache`` flag.
