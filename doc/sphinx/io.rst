Input / Output
==============

No generic checkpointing!
-------------------------

One of the most asked-for feature that seems to be missing in is
*checkpointing*, a simple way to tell to store and restore the current
state of the simulation, and to be able to write this state to or read
it from a file. This would be most useful to be able to restart a
simulation from a specific point in time.

Unfortunately, it is impossible to provide a simple command
(``checkpoint``), out of two reasons. The main reason is that has no way
to determine what information constitutes the actual state of the
simulation. On the one hand, scripts sometimes use Tcl-variables that
contain essential information about a simulation, the stored values of
an observable that was computed in previous time steps, counters, etc.
These would have to be contained in a checkpoint. However, not all
Tcl-variables are of interest. For example, Tcl has a number of
automatically set variables that contain information about the hostname,
the machine type, etc. These variables should most probably *not* be
included the simulation state. has no way to distinguish between these
variables. On the other hand, the core has a number of internal
variables, the particle coordinates. While most of these are probably
good candidates for being included into a checkpoint, this is not
necessarily so. For example, when you have particles in your system that
have fixed coordinates, should these be stored in a checkpoint, or not?
If the system contains mostly fixed particles and only very few moving
particles, this would increase the memory size of a checkpoint
needlessly. And what about the interactions in the system, or the bonds?
Should these be stored in a checkpoint, or are they generated by the
script?

Another problem with a generic checkpoint would be the control flow of
the script. In principle, the checkpoint would have to store where in
the script the checkpointing function was called to be able to return
there. All this is even further complicated by the fact that is running
in parallel.

Instead, in |es| the user has to specify what information needs to be saved to a
file to be able to restore the simulation state. When floating point numbers
are stored in text files (the particle positions), there is only a limited
precision. Therefore, it is not possible to bitwise reproduce a simulation
state using this text files. When you need bitwise reproducibility, you will have
to use checkpointing , which stores positions, forces and velocities in binary
format. 

(Almost) generic checkpointing in Python
----------------------------------------

Referring to the previous section, generic checkpointing poses
difficulties in many ways. Fortunatelly, the Python checkpointing module
presented in this section provides a comfortable workflow for an almost
generic checkpointing.

The idea is to let the user initially define which data is of interest
for checkpointing and thus solve the above mentioned problem. Once this
is done, checkpoints can then be saved simply by calling one save
function.

The checkpoint data can then later be restored easily by calling one
load function that will automatically process the checkpoint data by
setting the user variables and the checkpointed properties in .

In addition, the checkpointing module is also able to catch signals that
are invoked for example when the simulation is aborted by the user or by
a timeout.

The checkpointing module can be imported with::

    from espressomd import checkpointing

    [ checkpoint_path= ]

Determines the identifier for a checkpoint. Legal characters for an id
are "0-9", "a-zA-Z", "-", "_".

Specifies the relative or absolute path where the checkpoints are
stored.

For example ``checkpoint = checkpointing.Checkpointing(checkpoint_id="mycheckpoint")``
would create the new checkpoint with id "mycheckpoint" and all the
checkpointing data will be stored in the current directory.

After the system and checkpointing user variables are set up they can be
registered for checkpointing.
Name the string of the object or user variable that should be registered for
checkpointing.

To give an example::

    myvar = "some variable value"
    skin = 0.4
    checkpoint.register("myvar")
    checkpoint.register("skin")

    system = espressomd.System()
    # ... set system properties like box_l or
    timestep here ... checkpoint.register("system")

    system.thermostat.set_langevin(kT=1.0, gamma=1.0)
    checkpoint.register("system.thermostat")

    # ... set system.non_bonded_inter here ...
    checkpoint.register("system.non_bonded_inter")

    # ... add particles to the system with system.part.add(...) here ...
    checkpoint.register("system.part")

    # ... set charges of particles here ... from espressomd import
    electrostatics p3m = electrostatics.P3M(prefactor=1.0, accuracy=1e-2)
    system.actors.add(p3m)
    checkpoint.register("p3m")

will register the user variables ``skin`` and ``myvar``, system properties, a
langevin thermostat, non-bonded interactions, particle properties and a p3m
object for checkpointing. It is important to note that the checkpointing of
|es| will only save basic system properties. This excludes for example the
system thermostat or the particle data. For this reason one has to explicitly
register and for checkpointing.

Analogous to this, objects that have been registered for checkpointing but are
no longer needed in the next checkpoints can be unregistered with ``checkpoint
unregister var``.  A list of all registered object names can be generated with
``checkpoint get_registered_objects``.  A new checkpoint with a consecutive
index that contains the latest data of the registered objects can then be
created by calling ``checkpoint save [checkpoint_index]``.

An existing checkpoint can be loaded with ``checkpoint load
[checkpoint_index]``.

If no is passed the last checkpoint will be loaded. Concerning the procedure of
registering objects for checkpointing it is good to know that all registered
objects saved in a checkpoint will be automatically re-registered after loading
this checkpoint.

In practical implementations it might come in handy to check if there are any
available checkpoints for a given checkpoint id. This can be done with
``checkpoint has_checkpoints`` which returns a bool value.

As mentioned in the introduction, the checkpointing module also enables
to catch signals in order to save a checkpoint and quit the simulation.
Therefore one has to register the signal which should be caught with
``checkpoint register_signal signum=int_number``.

The registered signals are associated with the checkpoint id and will be automatically
re-registered when the same checkpoint id is used later.

Following the example above, the next example loads the last checkpoint,
restores the state of all checkpointed objects and registers a signal.

.. code::

    import espressomd from espressomd import checkpointing import signal

    checkpoint = checkpointing.Checkpointing(checkpoint_id="mycheckpoint")
    checkpoint.load()

    system = espressomd.System()
    system.cell_system.skin = skin
    system.actors.add(p3m)

    #signal.SIGINT: signal 2, is sent when ctrl+c is pressed
    checkpoint.register\_signal(signal.SIGINT)

    # integrate system until user presses ctrl+c while True:
    system.integrator.run(1000)

The above example runs as long as the user interrupts by pressing
ctrl+c. In this case a new checkpoint is written and the simulation
quits.

It is perhaps surprising that one has to explicitly create ``system`` again.
But this is necessary as not all |es| modules like ``cell_system`` or
``actors`` have implementations for checkpointing yet. By calling ``System()`` these modules
are created and can be easily initialized with checkpointed user variables
(like ``skin``) or checkpointed submodules (like ``p3m``).

.. _Writing H5MD-Files:

Writing H5MD-files
------------------

For large amounts of data it’s a good idea to store it in the hdf5 (H5MD
is based on hdf5) file format (see https://www.hdfgroup.org/ for
details). Currently |es| supports some basic functions for writing simulation
data to H5MD files. The implementation is MPI-parallelized and is capable
of dealing with varying numbers of particles.

To write data in a hdf5-file according to the H5MD proposal (see
http://nongnu.org/h5md/), first an object of the class
:class:`espressomd.io.writer.h5md.H5md` has to be created and linked to the
respective hdf5-file. This may, for example, look like:

.. code:: python

    from espressomd.io.writer import h5md
    system = espressomd.System()
    # ... add particles here
    h5 = h5md.H5md(filename="trajectory.h5", write_pos=True, write_vel=True)

If a file with the given filename exists and has a valid H5MD structures
it will be backed up to a file with suffix ".bak". This file will be
removed by the close() method of the class which has to be called at the
end of the simulation to close the file. The current implementation
allows to write the following properties: positions, velocities, forces,
species (|es| types), and masses of the particles. In order to write any property, you
have to set the respective boolean flag as an option to the H5md class.
Currently available:

    - write_pos: particle positions

    - write_vel: particle velocities

    - write_force: particle forces

    - write_species: particle types

    - write_mass: particle masses

    - write_ordered: if particles should be written ordered according to their
      id (implies serial write). 



In simulations with varying numbers of particles (MC or reactions), the
size of the dataset will be adapted if the maximum number of particles
increases but will not be decreased. Instead a negative fill value will
be written to the trajectory for the id. If you have a parallel
simulation please keep in mind that the sequence of particles in general
changes from timestep to timestep. Therefore you have to always use the
dataset for the ids to track which position/velocity/force/type/mass
entry belongs to which particle. To write data to the hdf5 file, simply
call the H5md objects write method without any arguments.

h5.write()

After the last write call, you have to call the close() method to remove
the backup file and to close the datasets etc.

Writing VTF files
-----------------

The formats VTF (**V**\ TF **T**\ rajectory **F**\ ormat), VSF
(**V**\ TF **S**\ tructure **F**\ ormat) and VCF (**V**\ TF
**C**\ oordinate **F**\ ormat) are formats for the visualization
software VMD:raw-latex:`\cite{humphrey96a}`. They are intended to
be human-readable and easy to produce automatically and modify.

The format distinguishes between *structure blocks* that contain the
topological information of the system (the system size, particle names,
types, radii and bonding information, amongst others), while *coordinate
blocks* (a.k.a. as *timestep blocks*) contain the coordinates for the
particles at a single timestep. For a visualization with VMD, one
structure block and at least one coordinate block is required.

Files in the VSF format contain a single structure block, files in the
VCF format contain at least one coordinate block, while files in the VTF
format contain a single structure block first and an arbitrary number of
coordinate blocks afterwards, thus allowing to store all information for
a whole simulation in a single file. For more details on the format,
refer to the homepage of the format .

Creating files in these formats from within is supported by the commands
and , that write a structure respectively a coordinate block to the
given Tcl channel. To create a VTF file, first use at the beginning of
the simulation, and then ``writevcf`` after each timestep to generate a
trajectory of the whole simulation.

The structure definitions in the VTF/VSF formats are incremental, a user
can easily add further structure lines to the VTF/VSF file after a
structure block has been written to specify further particle properties
for visualization.

Note that the ids of the particles in and VMD may differ. VMD requires
the particle ids to be enumerated continuously without any holes, while
this is not required in . When using and , the particle ids are
automatically translated into VMD particle ids. The function allows the
user to get the VMD particle id for a given particle id.

Also note, that these formats can not be used to write trajectories
where the number of particles or their types varies between the
timesteps. This is a restriction of VMD itself, not of the format.

``writevsf``: Writing the topology
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

writevsf(fp,types)


Writes a structure block describing the system’s structure to the
channel given by `fp`. `fp` must be an identifier for an open channel such as the
return value of an invocation of `open`. The output of this command can be
used for a standalone VSF file, or at the beginning of a VTF file that
contains a trajectory of a whole simulation.


Specify the coordinates of which particles should be written. If `types` is
used, all coordinates will be written (in the ordered timestep format).
Otherwise, has to be a Tcl-list specifying the pids of the particles.
The default is `types="all"`. 
Example
`pids =[0, 23, 42]`
`pids="all"`

``writevcf``: Writing the coordinates
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``writevcf(fp, types)``

Writes a coordinate (or timestep) block that contains all coordinates of
the system’s particles to the channel given by ``fp``. ``fp`` must be an identifier
for an open channel such as the return value of an invocation of ``open``.

.. todo:: NOT IMPLEMENTED

Specify, whether the output is in a human-readable, but somewhat longer
format (), or in a more compact form (). The default is .

.. todo:: NOT IMPLEMENTED

Specify whether the particle positions are written in absolute
coordinates () or folded into the central image of a periodic system ().
The default is .

Specify the coordinates of which particles should be written. If ``types`` is
used, all coordinates will be written (in the ordered timestep format).
Otherwise, has to be a Tcl-list specifying the pids of the particles.
The default is ``types="all"``. 
Example::

    pids =[0, 23, 42]
    pids="all"

.. todo:: NOT IMPLEMENTED

Specify arbitrary user data for the particles. has to be a Tcl list
containing the user data for every particle. The user data is appended
to the coordinate line and can be read into VMD via the VMD plugin
``VTFTools``. The default is to provide no userdata.
``userdata {"red" "blue" "green"}``

``vtfpid``: Translating particles ids to VMD particle ids
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

vtfpid

.. todo:: NOT IMPLEMENTED

If is the id of a particle as used in , this command returns the atom id
used in the VTF, VSF or VCF formats.

.. _MDAnalysis:

Writing various formats using MDAnalysis
----------------------------------------

If the MDAnalysis package (http://mdanalysis.org) is installed, it
is possible to use it to convert frames to any of the supported
configuration/trajectory formats, including PDB, GROMACS, GROMOS,
CHARMM/NAMD, AMBER, LAMMPS, ...)

To use MDAnalysis to write in any of these formats, one has first to prepare a stream from
the |es| particle data using the class :class:`espressomd.MDA_ESP`, and then read from it
using MDAnalysis. A simple example is the following:

.. code:: python

    import espressomd
    import MDAnalysis as mda
    from espressomd import MDA_ESP
    system = espressomd.System()
    # ... add particles here
    eos = MDA_ESP.Stream(system) # create the stream
    u =  mda.Universe( eos.topology, eos.trajectory ) # create the MDA universe

    # example: write a single frame to PDB
    u.atoms.write("system.pdb")

    # example: save the trajectory to GROMACS format
    from MDAnalysis.coordinates.TRR import TRRWriter
    W = TRRWriter("traj.trr",n_atoms=len(system.part)) # open the trajectory file
    for i in range(100):
        system.integrator.run(1)
        u.load_new(eos.trajectory) # load the frame to the MDA universe
        W.write_next_timestep(u.trajectory.ts) # append it to the trajectory

For other examples see samples/python/MDAnalysisIntegration.py

Online-visualization with Mayavi or OpenGL
------------------------------------------

With the python interface, |es| features two possibilities for
online-visualization:

#. Using the mlab module to drive *Mayavi, a "3D scientific data
   visualization and plotting in Python"*. Mayavi has a user-friendly
   GUI to specify the appearance of the output.
   Additional requirements:
   python module *mayavi*, VTK (package *python-vtk* for Debian/Ubuntu).
   Note that only VTK from version 7.0.0 and higher has Python 3
   support.

#. A direct rendering engine based on *pyopengl*. As it is developed for |es|, 
   it supports the visualization of several specific features like
   external forces or constraints. It has no GUI to setup the
   appearance, but can be adjusted by a large set of parameters.
   Additional requirements:
   python module *PyOpenGL*.

Both are not meant to produce high quality renderings, but rather to
debug your setup and equilibration process.

General usage
~~~~~~~~~~~~~

The recommended usage of both tools is similar: Create the visualizer of
your choice and pass it the ``espressomd.System()`` object. Then write
your integration loop in a seperate function, which is started in a
non-blocking thread. Whenever needed, call ``update()`` to synchronize
the renderer with your system. Finally start the blocking visualization
window with ``start()``. See the following minimal code example::

    import espressomd 
    from espressomd import visualization 
    from threading import Thread

    system = espressomd.System() 
    system.cell_system.skin = 0.4
    system.time_step = 0.01
    system.box_l = [10,10,10]

    system.part.add(pos = [1,1,1]) 
    system.part.add(pos = [9,9,9])

    #visualizer = visualization.mayaviLive(system) 
    visualizer = visualization.openGLLive(system)

    def main_thread(): 
        while True: 
            system.integrator.run(1)
            visualizer.update()

    t = Thread(target=main_thread) 
    t.daemon = True 
    t.start()
    visualizer.start()

Common methods for openGL and mayavi
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

| :meth:`espressomd.visualization.mayaviLive.update()` 
| :meth:`espressomd.visualization.openGLLive.update()`

``update()`` synchonizes system and visualizer, handles keyboard events for
openGLLive.

| :meth:`espressomd.visualization.mayaviLive.start()` 
| :meth:`espressomd.visualization.openGLLive.start()`

``start()`` starts the blocking visualizer window. 
Should be called after a seperate thread containing ``update()`` has been started.

| :meth:`espressomd.visualization.mayaviLive.register_callback()`
| :meth:`espressomd.visualization.openGLLive.register_callback()`

Registers the method ``callback()``, which is called every ``interval`` milliseconds. Useful for
live plotting (see sample script samples/python/visualization.py).

Mayavi visualizer
~~~~~~~~~~~~~~~~~

The mayavi visualizer is created with the following syntax:

:class:`espressomd.visualization.mayaviLive()`

Required paramters:
    * `system`: The espressomd.System() object.
Optional keywords:
    * `particle_sizes`:
        * `"auto"` (default)`: The Lennard-Jones sigma value of the self-interaction is used for the particle diameter.
        * `callable`: A lambda function with one argument. Internally, the numerical particle type is passed to the lambda function to determine the particle radius.
        * `list`: A list of particle radii, indexed by the particle type.

OpenGL visualizer
~~~~~~~~~~~~~~~~~

| :meth:`espressomd.visualization.openGLLive.run()` 

To visually debug your simulation, ``run()`` can be used to conveniently start 
an integration loop in a seperate thread once the visualizer is initialized::

    import espressomd 
    from espressomd import visualization 

    system = espressomd.System() 
    system.cell_system.skin = 0.4
    system.time_step = 0.00001
    system.box_l = [10,10,10]

    system.part.add(pos = [1,1,1], v = [1,0,0]) 
    system.part.add(pos = [9,9,9], v = [0,1,0])

    visualizer = visualization.openGLLive(system, background_color = [1,1,1])
    visualizer.run(1)

:class:`espressomd.visualization.openGLLive()`

The optional keywords in ``**kwargs`` are used to adjust the appearance of the visualization.
The parameters have suitable default values for most simulations. 

Required paramters:
    * `system`: The espressomd.System() object.
Optional keywords:
    * `window_size`: Size of the visualizer window in pixels.
    * `name`: The name of the visualizer window.
    * `background_color`: RGB of the background.
    * `periodic_images`: Periodic repetitions on both sides of the box in xyzdirection.
    * `draw_box`: Draw wireframe boundaries.
    * `draw_axis`: Draws xyz system axes.
    * `quality_particles`: The number of subdivisions for particle spheres.
    * `quality_bonds`: The number of subdivisions for cylindrical bonds.
    * `quality_arrows`: The number of subdivisions for external force arrows.
    * `quality_constraints`: The number of subdivisions for primitive constraints.
    * `close_cut_distance`: The distance from the viewer to the near clipping plane.
    * `far_cut_distance`: The distance from the viewer to the far clipping plane.
    * `camera_position`: Initial camera position. `auto` (default) for shiftet position in z-direction. 
    * `camera_target`: Initial camera target. `auto` (default) to look towards the system center.
    * `camera_right`: Camera right vector in system coordinates. Default is [1, 0, 0] 
    * `particle_sizes`:     
        * `auto` (default)`: The Lennard-Jones sigma value of the self-interaction is used for the particle diameter.
        * `callable`: A lambda function with one argument. Internally, the numerical particle type is passed to the lambda function to determine the particle radius.
        * `list`: A list of particle radii, indexed by the particle type.
    * `particle_coloring`:  
        * `auto` (default)`: Colors of charged particles are specified by particle_charge_colors, neutral particles by particle_type_colors
        * `charge`: Minimum and maximum charge of all particles is determined by the visualizer. All particles are colored by a linear interpolation of the two colors given by particle_charge_colors according to their charge.
        * `type`: Particle colors are specified by particle_type_colors, indexed by their numerical particle type.
    * `particle_type_colors`: Colors for particle types.
    * `particle_type_materials`: Materials of the particle types.
    * `particle_charge_colors`: Two colors for min/max charged particles.
    * `draw_constraints`: Enables constraint visualization. For simple constraints (planes, spheres and cylinders), OpenGL primitives are used. Otherwise, visualization by rasterization is used.
    * `rasterize_pointsize`: Point size for the rasterization dots.
    * `rasterize_resolution`: Accuracy of the rasterization.
    * `quality_constraints`: The number of subdivisions for primitive constraints.
    * `constraint_type_colors`: Colors of the constaints by type.
    * `constraint_type_materials`: Materials of the constraints by type.
    * `draw_bonds`: Enables bond visualization.
    * `bond_type_radius`: Radii of bonds by type.
    * `bond_type_colors`: Color of bonds by type.
    * `bond_type_materials`: Materials of bonds by type.
    * `ext_force_arrows`: Enables external force visualization.
    * `ext_force_arrows_scale`: Scale factor for external force arrows.
    * `drag_enabled`: Enables mouse-controlled particles dragging (Default`: False)
    * `drag_force`: Factor for particle dragging
    * `light_pos`: If `auto` (default) is used, the light is placed dynamically in the particle barycenter of the system. Otherwise, a fixed coordinate can be set.
    * `light_colors`: Three lists to specify ambient, diffuse and specular light colors.
    * `light_brightness`: Brightness (inverse constant attenuation) of the light. 
    * `light_size`: Size (inverse linear attenuation) of the light. If `auto` (default) is used, the light size will be set to a reasonable value according to the box size at start.
    * `spotlight_enabled`: If set to ``True`` (default), it enables a spotlight on the camera position pointing in look direction.
    * `spotlight_colors`: Three lists to specify ambient, diffuse and specular spotlight colors.
    * `spotlight_angle`: The spread angle of the spotlight in degrees (from 0 to 90).
    * `spotlight_brightness`: Brightness (inverse constant attenuation) of the spotlight. 
    * `spotlight_focus`: Focus (spot exponent) for the spotlight from 0 (uniform) to 128. 

Colors and Materials
^^^^^^^^^^^^^^^^^^^^

Colors for particles, bonds and constraints are specified by RGBA arrays.
Materials by an array for the ambient, diffuse, specular and shininess (ADSS)
components. To distinguish particle groups, arrays of RGBA or ADSS entries are
used, which are indexed circularly by the numerical particle type::

    # Particle type 0 is red, type 1 is blue (type 2 is red etc)..
    visualizer = visualization.openGLLive(system, 
                                          particle_coloring = 'type',
                                          particle_type_colors = [[1, 0, 0, 1],[0, 0, 1, 1]])

`particle_type_materials` lists the materials by type::

    # Particle type 0 is gold, type 1 is blue (type 2 is gold again etc).
    visualizer = visualization.openGLLive(system, 
                                          particle_coloring = 'type',
                                          particle_type_colors = [[1, 1, 1, 1],[0, 0, 1, 1]],
                                          particle_type_materials = [gold, bright])

Materials are stored in :attr:`espressomd.visualization.openGLLive().materials`. 

Controls
^^^^^^^^

The camera can be controlled via mouse and keyboard:

    * hold left button: rotate the system
    * hold right button: translate the system
    * hold middle button: zoom / roll
    * mouse wheel / key pair TG: zoom
    * WASD-Keyboard control (WS: move forwards/backwards, AD: move sidewards)
    * Key pairs QE, RF, ZC: rotate the system 

Additional input functionality for mouse and keyboard is possible by assigning
callbacks to specified keyboard or mouse buttons. This may be useful for
realtime adjustment of system parameters (temperature, interactions, particle
properties etc) of for demonstration purposes. The callbacks can be triggered
by a timer or keyboard input:: 

    def foo():
        print "foo"

    #Registers timed calls of foo()
    visualizer.register_callback(foo,interval=500)

    #Callbacks to control temperature 
    temperature = 1.0
    def increaseTemp():
            global temperature
            temperature += 0.1
            system.thermostat.set_langevin(kT=temperature, gamma=1.0)
            print "T =",system.thermostat.get_state()[0]['kT']

    def decreaseTemp():
        global temperature
        temperature -= 0.1

        if temperature > 0:
            system.thermostat.set_langevin(kT=temperature, gamma=1.0)
            print "T =",system.thermostat.get_state()[0]['kT']
        else:
            temperature = 0
            system.thermostat.turn_off()
            print "T = 0"

    #Registers input-based calls
    visualizer.keyboardManager.registerButton(KeyboardButtonEvent('t',KeyboardFireEvent.Hold,increaseTemp))
    visualizer.keyboardManager.registerButton(KeyboardButtonEvent('g',KeyboardFireEvent.Hold,decreaseTemp))

Further examples can be found in samples/python/billard.py or samples/python/visualization\_openGL.py.

Dragging particles
^^^^^^^^^^^^^^^^^^

With the keyword ``drag_enabled`` set to ``True``, the mouse can be used to
exert a force on particles in drag direction (scaled by ``drag_force`` and the
distance of particle and mouse cursor). 

Visualization example scripts
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Various example scripts can be found in the samples/python folder or in
some tutorials:

-  samples/python/visualization.py: LJ-Liquid with live plotting.

-  samples/python/visualization\_bonded.py: Sample for bond
   visualization.

-  samples/python/billard.py: Simple billard game including many
   features of the openGL visualizer.

-  samples/python/visualization\_openGL.py: Timer and keyboard callbacks
   for the openGL visualizer.

-  doc/tutorials/python/02-charged\_system/scripts/nacl\_units\_vis.py:
   Periodic NaCl crystal, see tutorial “Charged Systems”.

-  doc/tutorials/python/02-charged\_system/scripts/nacl\_units\_confined\_vis.py:
   Confined NaCl with interactively adjustable electric field, see
   tutorial “Charged Systems”.

-  doc/tutorials/python/08-visualization/scripts/visualization.py:
   LJ-Liquid visualization along with tutorial “Visualization”.

Finally, it is recommended to go through tutorial “Visualization” for
further code explanations. Also, the tutorial “Charged Systems” has two
visualization examples.
