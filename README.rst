carsons
=======

.. image:: https://badge.fury.io/py/carsons.svg
   :target: https://badge.fury.io/py/carsons
   :alt: latest release on pypi
.. image:: https://img.shields.io/pypi/pyversions/carsons.svg
   :target: https://pypi.python.org/pypi/carsons
   :alt: versons of python supported by carsons
.. image:: https://img.shields.io/github/license/opusonesolutions/carsons.svg
   :alt: GitHub license
   :target: https://github.com/opusonesolutions/carsons/blob/master/LICENSE.txt
.. image:: https://travis-ci.org/opusonesolutions/carsons.svg?branch=master
   :target: https://travis-ci.org/opusonesolutions/carsons
   :alt: build passing or failing
.. image:: https://coveralls.io/repos/github/opusonesolutions/carsons/badge.svg?branch=master
   :target: https://coveralls.io/github/opusonesolutions/carsons?branch=master
   :alt: test coverage
.. image:: https://api.codeclimate.com/v1/badges/22cfed180fd6032fe29b/maintainability
   :target: https://codeclimate.com/github/opusonesolutions/carsons/maintainability
   :alt: Maintainability

This is an implementation of Carson's Equations, a mathematical model for
deriving the equivalent impedance of an AC transmission or distribution line.

Implementation
--------------

``carsons`` is developed using python 3.6 support for
unicode characters like π, ƒ, ρ, μ, ω etc. This feature allows us to avoid
translating the problem into a more typical programming syntax, so the code
is dense and can easily be compared to published formulations of the problem.

For example, we implement the kron reduction, a matrix decomposition step,
using unicode notation to indicate the slightly different meaning of impedance
values before and after a kron reduction:


.. code:: python

   def perform_kron_reduction(z_primitive):
        Ẑpp, Ẑpn = z_primitive[0:3, 0:3], z_primitive[0:3, 3:]
        Ẑnp, Ẑnn = z_primitive[3:,  0:3], z_primitive[3:,  3:]
        Z_abc = Ẑpp - Ẑpn @ inv(Ẑnn) @ Ẑnp
        return Z_abc


Take a look at the `source code <https://github.com/opusonesolutions/carsons/blob/add-documentation/carsons/carsons.py>`_ to see more cool unicode
tricks!

Installation
------------

.. code:: bash

    ~/$ pip install carsons

Usage
-----

Carsons model requires a line model object that maps each phase to properties
of the conductor for that phase.

.. code:: python


   from carsons import CarsonsEquations, perform_kron_reduction

    class Line:
       gmr: {
           'A': geometric_mean_radius_A
           ...
       }
       r: {
            'A' => per-length resistance of conductor A in ohms
            ...
       }
       phase_positions: {
            'A' => (x, y) cross-sectional position of the conductor in meters
            ...
       }
       phases: {'A', ... }
         # map of phases 'A', 'B', 'C' and 'N<>' which are described in the
         # gmr, r and phase_positions attributes


    z_primitive = CarsonsEquations(Line()).build_z_primitive()
    z_abc = perform_kron_reduction(z_primitive)


The model supports any combination of ABC phasings (for example BC, BCN etc...)
including systems with multiple neutral cables; any phases that are not present
in the model will have zeros in the columns and rows corresponding to that
phase.

Multiple neutrals are supported, as long as they have unique labels starting
with ``N`` (e.g. ``Neutral1``, ``Neutral2``).

For examples of how to use the model, see the `tests <https://github.com/opusonesolutions/carsons/blob/master/tests/test_carsons.py>`_.

``carsons`` is tested against several cable configurations from the
`IEEE 4-bus test network <http://sites.ieee.org/pes-testfeeders/resources/>`_.

Problem Description
-------------------

Carsons equations model an AC transmission or distribution line into an
equivalent set of phase-phase impedances, which can be used to model the line
in a power flow analysis.

For example, say we have a 4-wire system on a utility pole, with ``A``,
``B``, ``C`` phase conductors as well as a neutral cable N. We know that when
conductors carry electrical current, they exhibit a magnetic field --- so its
pretty easy to imagine that, e.g., the magnetic field produced by ``A`` would
interact with the ``B``, ``C``, and ``N`` conductors.

::

                             B
                               O
                               |
                               |
                   A        N  |       C
                     O        O|         O
                     ----------|-----------
                               |
                               |
                               |
                               |
                               |
                               |
                               |
                               |
                               |
                               |
                               |
                               |
                               |
         ==============[Ground]============================
         /     /     /     /     /     /     /     /     /
              /     /     /     /     /     /     /
                   /     /     /     /     /
      
      
      
      
      
      
      
      
      
      
                      A*       N*          C*
                        0        0           0
      
                                B*
                                  0

     Figure: Cross-section of a 4-wire distribution line, with
             ground return.


However, each conductor also has a ground return path (or 'image') --- shown as
``A*``, ``B*``, ``C*``, and ``N*`` in the figure above --- which is a magnetically induced
current path in the ground. When `A` produceds a magnetic field, that field
*also* interacts with ``B*``, ``C*``, ``N*``, *and* ``A*``. Carsons equations model all
these interactions and reduce them to an equivalent impedance matrix that makes
it much easier to model this system.


In addition ``carsons`` implements the kron reduction, a conversion that
approximates the impedances caused by neutral cables by incorporating them into
the impedances for phase ``A``, ``B``, and ``C``. Since most AC and DC powerflow
formulations don't model the neutral cable, this is a valuable simplification.

References
----------

The following works were used to produce this formulation:

* `Leonard L. Grigsby - Electrical Power Generation, Transmission and Distribution <https://books.google.ca/books?id=XMl8OU4wIEQC&lpg=SA21-PA4&dq=kron%20reduction%20carson%27s%20equation&pg=SA21-PA4#v=onepage&q=kron%20reduction%20carson's%20equation&f=true>`__
* `William H. Kersting -- Distribution System Modelling and Analysis 2e <https://books.google.ca/books?id=1R2OsUGSw_8C&lpg=PA84&dq=carson%27s%20equations&pg=PA85#v=onepage&q=carson's%20equations&f=false>`__
* `Timothy Vismore -- The Vismor Milieu <https://vismor.com/documents/power_systems/transmission_lines/S2.SS1.php>`__
* `Daniel Van Dommelen, Albert Van Ranst, Robert Poncelet -- GIC Influence on Power Systems calculated by Carson's method <https://core.ac.uk/download/pdf/34634673.pdf>`__
