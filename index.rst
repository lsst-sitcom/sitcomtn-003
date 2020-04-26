..
  Technote content.

  See https://developer.lsst.io/restructuredtext/style.html
  for a guide to reStructuredText writing.

  Do not put the title, authors or other metadata in this document;
  those are automatically added.

  Use the following syntax for sections:

  Sections
  ========

  and

  Subsections
  -----------

  and

  Subsubsections
  ^^^^^^^^^^^^^^

  To add images, add the image file (png, svg or jpeg preferred) to the
  _static/ directory. The reST syntax for adding the image is

  .. figure:: /_static/filename.ext
     :name: fig-label

     Caption text.

   Run: ``make html`` and ``open _build/html/index.html`` to preview your work.
   See the README at https://github.com/lsst-sqre/lsst-technote-bootstrap or
   this repo's README for more info.

   Feel free to delete this instructional comment.

:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. TODO: Delete the note below before merging new content to the master branch.



.. Add content here.
.. Do not include the document title (it's automatically added from metadata.yaml).

.. note::

   The objective of this technote is to clarify all the different coordinate systems that are relevant to the operations of the Rubin Observatory Active Optics System, and the transformations between them, which is crucial for the Active Optics Control to work in a coherent way.

############
Introduction
############

The operations of the Rubin Observatory Active Optics System (AOS) involve the following components

- M1M3
- M2
- M2 hexapod
- Camera hexapod
- Camera rotator
- ComCam (during commissioning)
- LSSTCam (during commissioning and operations)

Over the years, each component has been under development under their own coordinate system (CS).
Sometimes the situation can be even more complicated - A single vendor may be using multiple coordinate systems for different sub-components (for example, mechanical versus optical). There are also inconsistencies between various engineering models and drawings.

If we include simulations and control algorithm work, there are additional coordinate systems that are relevant to the AOS

- Zemax - as defined in `the project official optical model <https://confluence.lsstcorp.org/display/SYSENG/As-built+optical+model>`__ (This is used to calculate the optical sensitivity matrix)
- PhoSim :cite:`2015ApJS..218...14P` (This is used for closed-loop simulations)

In order for the AOS to work as one system, a clean understanding of these coordinate systems and how they fit together is required. This is the major objective of this technical note. We will also discuss how to transformations between various coordinate systems, and more importantly, how to transform everything into a common coordinate system - the Optical Coordinate System.

.. note::

   Document-18499 :cite:`Document-18499` defines a set of coordinate systems and the transformations between them. However, it doesn't go into all the levels of details that are relevant to the AOS operations.

.. _section-ocs:

#############################
The Optical Coordinate System
#############################

Our AOS system-wide CS is the Optical Coordinate System (OCS). The defition of the OCS is the same as defined in LTS-136 :cite:`LTS-136`, Section 2.2. In summary

- The origin of the OCS is at the theoretical vertex of M1.
- +z axis points along the optical axis, toward the sky
- +x axis goes in parallel with the elevation axis. When look from the sky, +x points to the right.
- +y axis can then be determined using the right hand rule. At horizon pointing, +y goes up.
- The azimuth angle is zero when the telescope points toward northern horizon. Looking from the sky, it increase clockwise.
- The zenith angle is zero when the telescope points at zenith. It increase to 90 degrees for horizon pointing.

.. figure:: /_static/ocs.png
   :name: fig-ocs
   :target: ../_images/ocs.png
   :alt: The Optical Coordinate System (OCS)

Optical Coordinate System (OCS). Left: view from the sky; Right: side view from the -X axis.

The idea is that the OCS is fixed to M1, which defines the reference for the optical system.
Relative to the ground, the OCS rotates with the azimuth angle and the zenith angle.
An easy to identify the +x/+y/+z while looking at a drawing or the actual hardware is that, the walkway on the TMA is in the +y direction. Otherwise when we go toward horizon it is going to bump into the ground.

.. note::
   The Telescope and Site top-level CS document is LTS-136 :cite:`LTS-136`, and the Camera top-level CS document is LCA-280 :cite:`LCA-280`. Unlike those two documents, we do not define the Observatory Mount CS or the Azimuth CS, because these CS, as defined by LTS-136 :cite:`LTS-136` and LCA-280 :cite:`LCA-280`, actually contradict each other, in terms of which axis points north, which points west etc. We also avoid the use of terms such as ``elevation angle`` and ``altitude``. Instead, we always use ``zenith angle``, as defined above.


################
Zemax and PhoSim
################

We discuss the Zemax and PhoSim CS first, because these are relatively easier to define -
they are self-consistent within their own framework.
When we say Zemax CS, we refer to the global CS as used by
`the official Rubin Observatory optical model <https://confluence.lsstcorp.org/display/SYSENG/As-built+optical+model>`__. The good news here is that when we worked with the PhoSim team at Purdue on the AOS simulations, we made sure that the PhoSim coordinate system conform to the project standard, at least externally, to the level that we care about while exercising AOS control.
The Zemax/PhoSim CS is defined as

- The origin of Zemax/PhoSim CS overlaps with OCS origin, i.e., at the theoretical vertex of M1.
- The +z axis of Zemax/PhoSim CS points from the sky to M1M3. It follows the direction of the incoming on-axis rays. This is opposite of the OCS +z axis.
- The +y axis is the same as OCS +y axis.
- The +x axis is the opposite of OCS +x axis

.. figure:: /_static/zcs.png
   :name: fig-zcs
   :target: ../_images/zcs.png
   :alt: The Zemax/PhoSim Coordinate System (ZCS)

The Zemax/PhoSim Coordinate System (ZCS)

.. code-block:: py

   def zcs2ocs(x,y,z):
       return -x,y,-z

   def ocs2zcs(x,y,z):
       return -x,y,-z

The optical sensitivity matrix (senM) is derived using the Zemax optical model.
Therefore, everything about the senM follows the ZCS. We were able to close the simulation loop with PhoSim, because we made PhoSim consistent with Zemax.
With the actual hardware, we will need to convert all commands returned the AOS control into the proper CS of each component before they are applied.

.. note::

    Note that we apply the decenters and tilts in Zemax via ``Coordinate Breaks``. Mathematically the order of decenters and tilts matter. In Zemax, there is a ``order flag``. When it is set to 0, Zemax does the decenters first, then x-tilt, y-tilt, z-rotation. When the ``order flag`` is set to 1, Zemax does these in exact opposite order, so that users can easily go back to the original CS :cite:`Zemax13manual`. However, in the AOS context, we don't really care about these because the tilts are always small enough for the order not to make a difference. If this is not true, then the basic approach of taking the decenters and tilts of the hexapods as independent variable wouldn't be correct.

####
M1M3
####

The M1M3 glass mirror was fabricated and polished at the University of Arizona Richard F. Caris Mirror Lab (RFCML).
The mirror cell was made by CAID Industriess, and software is designed and written by the Rubin Obs. team.

When looking at M1M3 drawings and data, be wary that there are multiple versions  of CS around. In particular, mechanical folks look at the actuators from inside the M1M3 cell a lot, so they tend to define +z as pointing down from M2. While optical people always look at the M1M3 surface from outside, so they tend to define +z as pointing to the sky. People also flip the +x around the +y axes sometimes. We define M1M3 CS as the following -

- The origin of M1M3 CS overlaps with OCS origin, i.e., at the theoretical vertex of M1.
- +x points toward actuator 106.
- +y points toward actuator 441, which is close to the M1M3 mirror cell door.
- +z points toward the sky.
- **When mounted on the TMA, M1M3 CS is the same as OCS.**


.. figure:: /_static/m1m3.png
   :name: fig-m1m3
   :target: ../_images/m1m3.png
   :alt: The M1M3 CS

The M1M3 CS.

Our goal here is not to change all the engineering drawings to be in this CS. Instead, the goal is to make sure that for anything that is being used by the AOS, we can put them into M1M3 CS or OCS correctly.

Note that M3 vertex is at (0, 0, -233.8)mm in the OCS.

The Rubin Obs. official M1M3 Finte Element Model (FEM), as provided by Doug Neil and Ed Hileman, uses the M1M3 CS.
`The bending mode shapes and forces derived using this FEM
<https://github.com/lsst-sitcom/M1M3_ML/blob/master/data/M1M3_1um_156_README.txt>`__
use the M1M3 CS as well.

- When the force on an single-axis actuator or the primary cylinder of a lateral or bilateral actuator is positive, it pushes M1M3 toward the sky, along +z axis. The bending mode forces are given `here <https://github.com/lsst-sitcom/M1M3_ML/blob/master/data/M1M3_1um_156_force.txt>`__.
- For bending modes, there are two variaties. The `surface normal bending modes <https://github.com/lsst-sitcom/M1M3_ML/blob/master/data/M1M3_1um_156_grid.txt>`__ are those that were directly measured in the RFCML. Here the displacement vectors of the Finite Element nodes point toward the center of curvature, and are normal to the M1M3 surface. For use in an optical raytrace program like Zemax or PhoSim, and for deriving the senM, we need the `surface sag bending modes <https://github.com/lsst-sitcom/M1M3_ML/blob/master/data/M1M3_1um_156_sag.txt>`__. These displacement vectors point along +z axis of the OCS or M1M3 CS.

Like other components of the AOS, M1M3 operates mostely off its Look-Up Table (LUT), which contains our best knowledge of the forces as functions as gravity (or zenith angle) and temperature profiles on and around the mirror surfaces. The current M1M3 LUT can be found `here <https://github.com/lsst-sitcom/M1M3_ML/blob/master/data/FLUT.yaml>`__.

- The zenith angle, as the primary input to the M1M3 LUT, is defined the same way as the OCS zenith angle as defined in Sec. :ref:`section-ocs`.
- Unrelated to the bending modes, but relevant to the LUT, are the forces on the secondary cylinders of the lateral actuators and bilateral actuators. When the force on the secondary cylinder of an lateral actuator is positive, it pushes M1M3 in the y-z plane, along 45 degrees between +y and +z axes. When the force on the secondar cylinder of a bilateral actuator is positive, it pushes M1M3 in the x-z plane, either along the 45 degree line between the +z and +x or +z and -x direction, depending on the location of the bilateral actuator.

The M1M3 control software uses the M1M3 CS as well (see `here <https://github.com/lsst-ts/ts_m1m3support/blob/master/Controller/SettingFiles/Tables/ForceActuatorTable.csv>`__). When we reposition the M1M3 mirror relative to its cell, that is in referece to the M1M3 CS.

##
M2
##

The M2 mirror substrate was manufactured by Corning Inc. M2 mirror polishing, mirror cell and control software production were all done at Harris Corporation.

The M2 system as a whole, especially on the software side, leaves a lot to be desired. For example, with regard to the CS, the `M2 control software in LabView <https://github.com/lsst-ts/ts_mtm2>`__ uses a different CS than the `Matlab tools <https://github.com/lsst-ts/ts_mtm2_matlab_tools>`__ used for generating the configurations.

We define the M2 CS as the following -

- The origin of the M2 CS is on the +z axis of the OCS, and at M2 vertex (6156.201mm from M1 vertex, based on `v3.3 optical design <https://confluence.lsstcorp.org/display/SYSENG/As-built+optical+model>`__).
- The +x axis points toward actuators B8/B9.
- The +y axis points toward tangent link A1 and actuator B1.
- The +z axis points toward the sky.
- **When mounted on the TMA, M2 CS has its 3 axes parallel to those of the OCS, all in the same direction. The coordinates of M2 CS origin in the OCS is (0, 0, 6156.201)mm.**

.. figure:: /_static/m2.png
   :name: fig-m2
   :target: ../_images/m2.png
   :alt: The M2 CS

The M2 CS and M2 FEA CS.

Our goal here is not to change all the engineering drawings to be in this CS. Instead, the goal is to make sure that for anything that is being used by the AOS, we can put them into M2 CS or OCS correctly.

Harris derived a set of M2 bending modes prior to M2 cell and mirror delivery, but those made no sense to us at all. The `M2 bending modes that we use now <https://github.com/lsst-sitcom/M2_FEA/blob/master/data/M2_1um_72_README.txt>`__ have been calculated by us using the final FEM as delivered by Harris. This FEM uses a different CS. Since this same CS is also used by the Harris Matlab code which is used for regenerating M2 configuration, we are going to continue to use this CS, and call it M2 FEA CS -

- The origin of the M2 FEA CS overlaps with the M2 CS (at M2 vertex)
- The +y axis points toward actuators B23/B24.
- The +x axis points toward tangent link A4 and actuator B16.
- The +z axis points toward M1M3.

.. code-block:: py

   #all units are in mm
   def m2fea2m2cs(x,y,z):
       return -y,-x,-z
   def m2cs2m2fea(x,y,z):
       return -y,-x,-z
   def ocs2m2cs(x,y,z):
       return x,y,z-6156.201

The M2 LabView control software uses M2 CS (most likely by coincidence). See
`here <https://github.com/lsst-ts/ts_mtm2/blob/master/doc/project/CellConfiguration.xlsx>`__.
The M2 Matlab tools which are used to generate the configuration files uses the M2 FEA CS. See
`here <https://github.com/lsst-ts/ts_mtm2_matlab_tools/blob/master/ReferenceFiles/AxialActuatorLocations.csv>`__.
The configuration file thus generated are usable by the LabView software because
all the configuration files refer to actuator ID instead of their coordinates \ [#label1]_.
When we reposition the M2 mirror relative to its cell, that is in referece to the M2 CS.
The axial actuator force distribution found on the M2 Engineering User Interface (EUI) uses the M2 CS.

So, on the bending modes -

- `the bending mode forces <https://github.com/lsst-sitcom/M2_FEA/blob/master/data/M2_1um_72_force.txt>`__ calculated are in the M2 FEA CS. At zenith pointing, a positive bending force means that the actuator is pushing down. However, while applying the forces to the control system, the forces are in the M2 CS, where a positive force means pulling the mirror toward the cell, as evidenced in the `LUT test <https://github.com/lsst-sitcom/M2_summit_2003/blob/master/a17_LUT_cart_rotation.ipynb>`__.
- To be consistent with M1M3, M2 bending mode shapes also come with two variaties. The `surface normal bending modes <https://github.com/lsst-sitcom/M2_FEA/blob/master/data/M2_1um_72_grid.txt>`__ has the displacement vectors pointing away from the center of curvature of M2, and are normal to the M2 surface. The `surface sag bending modes <https://github.com/lsst-sitcom/M2_FEA/blob/master/data/M2_1um_72_sag.txt>`__ have the displacement vectors along +z axis in the M2 FEA CS.

Some clarifications on the M2 LUT -

- As we discussed above, for axial actuators, a positve force always pulls the M2 mirror. That is why during the `LUT test <https://github.com/lsst-sitcom/M2_summit_2003/blob/master/a17_LUT_cart_rotation.ipynb>`__, the axial forces went negative when the mirror faced up. The same applies to the tangent link, i.e., the tangent forces are positive when the tangent links pull. That is why during the `LUT test <https://github.com/lsst-sitcom/M2_summit_2003/blob/master/a17_LUT_cart_rotation.ipynb>`__, when tangent link A4 was going toward the ceiling, forces on A2 and A3 were positive.

- The angle which the M2 software uses for looking up forces in the LUT is defined in the range (-270, +90) degrees. We use ``M2 LUT angle`` to refer to this angel. This is built into the M2 software, so there is no plan too change it. The M2 `LUT test <https://github.com/lsst-sitcom/M2_summit_2003/blob/master/a17_LUT_cart_rotation.ipynb>`__ revealed the relation between the M2 LUT angle and the OCS zenith angle is

  .. math::
    M2 LUT\ angle = -(zenith\ angle) + 90 degrees

  Therefore, when the telescope moves from zenith point to horizon point, the M2 LUT angle goes from 90 degrees to 0.
- The M2 inclinometer read out obeys the same definition as the M2 LUT angle.

.. [#label1] Te-Wei needs to look at the code and confirm this.

.. note::
   To calculate the senM using Zemax, the M2 bending modes need to be properly converted into Zemax CS before used. The current senM did not handle this part correctly, by misinterpretting the M2 FEA CS. It hasn't showed up as a problem, because that misinterpretation was passed onto PhoSim as well. This will need to be fixed before the hardware are mounted on the TMA. Otherwise things will not converge.

##########
M2 Hexapod
##########

The M2 and camera hexapods, and the camera rotator were manufactured by Moog CSA Engineering.

The M2 hexapod uses M2 CS. A few additional notes -

- The center of rotation (COR) can be reconfigured with the software, we will set the COR at M2 vertex for AOS operations, so that it overlaps with the origin of M2 CS.
- The +x axis points toward actuator 6.
- The +y axis points toward actuator 1.
- The +z axis points away from the M2 mounting surface.
- Strictly speaking, the order of decenters and rotations matter. However, in the AOS context, we don't really care about these because the tilts are always small enough for the order not to make a difference.
- **When mounted on the TMA, actuator 1 is in the OCS +y direction, actuator 6 is in the OCS +x direction.**

.. figure:: /_static/m2hex.png
   :name: fig-m2hex
   :target: ../_images/m2hex.png
   :alt: The M2 Hexapod

The M2 hexapod in the M2 CS.

The M2 hexapod LUT angle is defined the same way as the OCS zenith angle \ [#label2]_, ranging between 0 and 90 degrees.

.. [#label2] Need to talk to Doug about exactly how the LUT works for the hexapods


#######
LSSTCam
#######


The Camera Coordinate System (CCS) has been widely used by the Camera team.
As pointed out in Sec. :ref:`section-ocs`, LTS-136 :cite:`LTS-136` and LCA-280 :cite:`LCA-280` actually contradict each other in some aspects. But the good news is that both the Azimuth CS defined by LTS-136 :cite:`LTS-136` and the Telescope CS defined by LCA-280 :cite:`LCA-280` have +z pointing toward the sky, and +y pointing at zenith when telescope points at horizon. So the CCS as defined by LCA-280 :cite:`LCA-280` can be made consistent with our OCS, if we forget about its orientation relative to the earth. (Yes, LCA-280 :cite:`LCA-280` explicitly uses north and west to define the CCS.)

In the AOS context, we define the CCS as the following (We believe this is the same as the CCS used by the camera team; We redefine it here simply because we find the definition of CCS in LCA-280 :cite:`LCA-280` rather confusing.)

- The origin of the CCS is at L1S1 vertex. L1S1 is the first surface, i.e., out-facing (away from the rest of the camera) surface of L1. This is about 3397mm from the M1 vertex, based on `v3.3 optical design <https://confluence.lsstcorp.org/display/SYSENG/As-built+optical+model>`__. Note that this distance also varies with the filter band.
- The +z axis points from L1S1 into the camera body, along the optical axis, so that most of the camera components have positive z.
- The +x axis points toward raft R42, along the parallel transfer direction of the individual segments. The segments are roughly 500 by 2000 pixels. The parallel transfer direction is along the 2000-pixel side.
- The +y axis points toward raft R24, along the serial register. The serial register is along the 500-pixel side of the CCD segments.
- **When mounted on the telescope mount, with the rotator angle at zero, the x/y/z axes of the OCS are in parallel with the x/y/z axes of the OCS, and points in the same directions.**

The CCS is fixed to the camera body; we use the focal plane to define the CCS because that is the only camera component that is relevant to the AOS, CS-wise. The lens surfaces do change under different gravity and thermal profile, and even the camera rotator angle. But the AOS do not actively control any camera internal components for image quality improvements.

.. figure:: /_static/ccs.png
   :name: fig-ccs
   :target: ../_images/ccs.png
   :alt: The CCS

The Camera CS.

For the wavefront sensors, the split between the intra- and extra-focal chips are parallel to the CCS y-axis on R00 and R44, and parallel to the CCS x-axis on R40 and R04. Here we refer to each 2k by 4k as one chip. Sometimes we see them refered to as half-chips as well. The one closer to the field center is always the extra-focal chip, which has larger z-coordinate in the CCS. The camera team refers to the extra-focal chip as low chip sometimes, because it is lower than the focal plane when looked through the L3 lens. For the same reason, the intr-focal chips are refered to as high chips.

Two out of the four wavefront sensors (R00 and R44) have their CCD segments aligned the same way as the science sensors. Most of the science sensor segments, as seen in the CCS, has the parallel transfer direction parallel the x-axis. However, astronomers are much more used to seeing images with the parallel transfer direction going vertically, and serial register going horizontally. LSE-349 :cite:`LSE-349` defines the project's official Data Visualization CS (DVCS) as a x-y transpose of the CCS. We should be aware that most of the time when we see a visualization of certain quantities over the entire focal plane, a raft, or a single CCD, if CS is not explicitly given, the assumption should be that it is in DVCS.

.. code-block:: py

   # all units are mm
   def ocs2ccs(x,y,z, d_L1_M1):
       '''
       d_L1_M1 is the distance between L1S1 vertex and M1 vertex.
            it is approximately 3397mm, but varies with camera hexapod positioning and filter band.
       '''
       return x,y,z-d_L1_M1
   def ccs2ocs(x,y,z, d_L1_M1):
       return x,y,z+d_L1_M1
   def dvcs2ccs(x,y,z):
       return y,x,z
   def ccs2dvcs(x,y,z):
       return y,x,z

##############
Camera Hexapod
##############

The M2 and camera hexapods, and the camera rotator were manufactured by Moog CSA Engineering.

The Camera hexapod uses the CCS. A few additional notes -

- The center of rotation (COR) can be reconfigured with the software, we will set the COR at L1S1 vertex for AOS operations, so that it overlaps with the origin of the CCS.
- The +x axis points toward actuator 6.
- The +y axis points toward the mid-point between actuators 1 and 2.
- The +z axis points away from the camera mounting surface.
- Strictly speaking, the order of decenters and rotations matter. However, in the AOS context, we don't really care about these because the tilts are always small enough for the order not to make a difference.
- **When mounted on the TMA, actuators 1 and 2 are in the OCS +y direction, actuator 6 is in the OCS +x direction.**

.. figure:: /_static/camhex.png
   :name: fig-camhex
   :target: ../_images/camhex.png
   :alt: The Camera Hexapod

The Camera hexapod in the CCS.

The camera hexapod LUT angle is defined the same way as the OCS zenith angle \ [#label3]_, ranging between 0 and 90 degrees.

.. [#label3] Need to talk to Doug about exactly how the LUT works for the hexapods


##############
Camera Rotator
##############


The M2 and camera hexapods, and the camera rotator were manufactured by Moog CSA Engineering.

.. figure:: /_static/rot.png
   :name: fig-rot
   :target: ../_images/rot.png
   :alt: The Camera Rotator

The Camera roator uses the CCS. This is looking at the camera mounting surface from the M1M3.

According to the rotator operator's manul :cite:`rotatorManual`, while looking from the sky, a positive rotation angle is counterclockwise. This is opposite of the azimuth angle as defined in the OCS.

######
ComCam
######

The ComCam is a one-raft camera that will be used in commissioning.
ComCam also uses the CCS as its standard CS. As the hardware is a bit different from the LSSTCam, we make some clarification here.

Just like the LSST Cam, we map the ComCam sensor orientation to the CCS.

- The +x points toward sensor S21, along the parallel transfer direction of the individual segments. The segments are roughly 500 by 2000 pixels. The parallel transfer direction is along the 2000-pixel side.
- The +y axis points toward sensor S12, along the serial register. The serial register is along the 500-pixel side of the CCD segments.

L1 vertex??? check comcam zemax model

.. figure:: /_static/comcam.png
   :name: fig-comcam
   :target: ../_images/comcam.png
   :alt: The ComCam

The ComCam also uses the CCS.   \ [#label4]_.

.. [#label4] Need to check with Brian.

#######
Summary
#######

Unfortunately, we will have to continue to use the following CS

- Zemax CS
- OCS
- M1M3 CS
- M2 CS
- M2 FEA CS

and the following angles

- zenith angle, and azimuth angle, as defined in the OCS
- M2 LUT angle

The transformations between them are -

.. code-block:: py

   # all units are mm
   def zcs2ocs(x,y,z):
       return -x,y,-z

   def ocs2zcs(x,y,z):
       return -x,y,-z

   def ocs2m2cs(x,y,z):
       return x,y,z-6156.201

.. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
   :style: lsst_aa
