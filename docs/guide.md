# User Guide

RGFROSH has two main applications:

=== "Analysis"
    
    Calculating shock conditions from the initial conditions and experimentally measured shock velocity:
    
    ```py hl_lines="4"
    from rgfrosh import FrozenShock
    import cantera as ct
    
    shock = FrozenShock(ct.CarbonDioxide(), 1000, 295, 101325)
    print(f"T5 = {shock.T5:.1f} K, P5 = {shock.P5 / 101325:.2f} atm")
    ```

    which gives the post-reflected-shock temperature (`T5`) and pressure (`P5`):
    
    ```
    T5 = 1222.3 K, P5 = 115.82 atm
    ```

=== "Planning"
    
    Calculating initial conditions for target post-reflected-shock conditions:
    
    ```py hl_lines="4"
    from rgfrosh import FrozenShock
    import cantera as ct
    
    shock = FrozenShock.target_conditions(ct.CarbonDioxide(), 1100, 200e5)
    print(f"P1 = {shock.P1 / 133.322:.0f} torr, u1 = {shock.u1:.1f} m/s")
    ```
    
    which gives the required fill pressure (`P1`) and incident shock velocity (`u1`):
    
    ```
    P1 = 1687 torr, u1 = 918.9 m/s
    ```

## Thermodynamic Interfaces 

### Cantera

RGFROSH natively supports `cantera.ThermoPhase` objects, as shown in the
above examples.

!!! warning
    The implementations of the Redlich-Kwong (RK) and Peng-Robinson (PR) EOS
    in Cantera do not yet include definitions for `isothermal_compressibility` and 
    `thermal_expansion_coeff` 
    (see [Cantera/enhancements/issues/#122](https://github.com/Cantera/enhancements/issues/122)).

### User-Defined Interfaces

Custom thermodynamic routines are supported using the 
[`ThermoInterface`][rgfrosh.ThermoInterface] class, which utilizes structural 
subtyping [(PEP 544)](https://www.python.org/dev/peps/pep-0544/) through the 
built-in [`typing.Protocol`](https://docs.python.org/3/library/typing.html#typing.Protocol)
base class. Thus, any class that implements the required methods is considered a 
subtype, even if it is not an explicit subclass of `ThermoInterface`; however, 
explicitly subclassing `ThermoInterface` is recommended for user-defined 
interfaces to ensure all required methods are defined.

### CoolProp (Example)

An interface for `CoolProp.AbstractState`, which accounts for differences 
in method names as well as units, is shown in the following example:

```py
from rgfrosh import ThermoInterface, FrozenShock
import CoolProp as CP


class CPInterface(ThermoInterface):
    def __init__(self, state: CP.AbstractState):
        self.state = state

    @property
    def mean_molecular_weight(self):
        return self.state.molar_mass() * 1e3  # [kg/mol] to [kg/kmol]

    @property
    def TP(self):
        return self.state.T(), self.state.p()

    @TP.setter
    def TP(self, value):
        T, P = value
        self.state.update(CP.PT_INPUTS, P, T)

    @property
    def density_mass(self):
        return self.state.rhomass()

    @property
    def cp_mass(self):
        return self.state.cpmass()

    @property
    def enthalpy_mass(self):
        return self.state.hmass()

    @property
    def isothermal_compressibility(self):
        return self.state.isothermal_compressibility()

    @property
    def thermal_expansion_coeff(self):
        return self.state.isobaric_expansion_coefficient()
```

The interface can then be used to wrap a `CoolProp.AbstractState` object, allowing:

```py hl_lines="4"
state = CP.AbstractState("SRK", "CarbonDioxide")
state.specify_phase(CP.iphase_gas)

shock = FrozenShock(CPInterface(state), 1000, 295, 101325)
print(f"T5 = {shock.T5:.1f} K, P5 = {shock.P5 / 101325:.2f} atm")
```

which outputs the following solution of the post-reflected-shock conditions 
for the Soave-Redlich-Kwong (SRK) EOS:

```
T5 = 1216.8 K, P5 = 115.45 atm
```


*[EOS]: Equation of State
