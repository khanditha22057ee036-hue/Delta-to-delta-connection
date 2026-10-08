"""
Delta-Delta (Δ-Δ) Connection Calculator
---------------------------------------
A menu-driven program for engineering students to analyse a three-phase
system with a DELTA-connected source feeding a DELTA-connected load.

Balanced delta-delta system (convert the load to an equivalent star):
    Equivalent star load    Z_Y = Z_delta / 3
    Equivalent source phase Vph = VL / sqrt(3)
    Line current            IL = Vph / |Zline + Z_delta/3|
    Load phase current      Iph = IL / sqrt(3)
    Load phase voltage      V_load = Iph * |Z_delta|  (= load line voltage)
    Load power              P = 3 * Iph^2 * R_delta
    Line loss               Ploss = 3 * IL^2 * R_line

Unbalanced delta-delta system (phasor analysis):
    1. Convert the load delta to an equivalent star:
           Za = Zab*Zca / (Zab+Zbc+Zca)    Zb = Zab*Zbc / (Zab+Zbc+Zca)
           Zc = Zbc*Zca / (Zab+Zbc+Zca)
    2. Source star-equivalent voltages: Van = (Vab - Vca) / 3, etc.
    3. Find the neutral shift with Millman's theorem and the line currents.
    4. Load phase currents: Iab = V_AB(load) / Zab, etc.

Delta-delta three-phase transformer:
    VL2 = VL1 / a,   IL2 = a * IL1,   no phase shift between primary and secondary
    Each transformer carries 1/3 of the total load.

Open-delta (V-V) connection (one transformer removed):
    Capacity = sqrt(3) * (rating of one transformer)
    = 57.7 % of the closed-delta capacity
"""

import cmath
import math

SQRT3 = math.sqrt(3)


# ---------------------------------------------------------------- input helpers
def get_positive_float(prompt):
    """Keep asking until the user enters a valid positive number."""
    while True:
        try:
            value = float(input(prompt))
            if value <= 0:
                print("  Please enter a value greater than zero.")
                continue
            return value
        except ValueError:
            print("  Invalid input. Please enter a number.")


def get_nonneg_float(prompt):
    """Ask for a number that may be zero but not negative."""
    while True:
        try:
            value = float(input(prompt))
            if value < 0:
                print("  Please enter zero or a positive value.")
                continue
            return value
        except ValueError:
            print("  Invalid input. Please enter a number.")


def get_float(prompt):
    """Ask for any number (zero and negative values allowed)."""
    while True:
        try:
            return float(input(prompt))
        except ValueError:
            print("  Invalid input. Please enter a number.")


def get_load_impedance(label):
    """Ask for resistance and reactance of a load phase; return complex Z.
    Reactance: positive for inductive, negative for capacitive."""
    r = get_positive_float(f"  Load {label}: resistance R (ohm): ")
    x = get_float(f"  Load {label}: reactance X (ohm, - for capacitive): ")
    return complex(r, x)


def get_line_impedance():
    """Ask for the impedance of each line conductor (may be zero)."""
    print("  Line (cable) impedance per phase - enter 0 and 0 if ignored:")
    r = get_nonneg_float("  Line resistance (ohm): ")
    x = get_nonneg_float("  Line reactance (ohm): ")
    return complex(r, x)


# ------------------------------------------------------------------ core maths
def delta_to_star_unbalanced(zab, zbc, zca):
    """Equivalent star impedances (Za, Zb, Zc) of an unbalanced delta."""
    total = zab + zbc + zca
    return zab * zca / total, zab * zbc / total, zbc * zca / total


def neutral_shift(voltages, impedances):
    """Millman's theorem: voltage between load neutral and source neutral."""
    admittances = [1 / z for z in impedances]
    numerator = sum(v * y for v, y in zip(voltages, admittances))
    return numerator / sum(admittances)


# ------------------------------------------------------------------- display
def show_polar(name, value, unit):
    """Print a complex quantity in polar form."""
    mag, ang = cmath.polar(value)
    print(f"  {name} = {mag:.3f} < {math.degrees(ang):.2f} deg {unit}")


def balanced_delta_delta():
    v_l = get_positive_float("Source line voltage VL (V): ")
    z_line = get_line_impedance()
    z_delta = get_load_impedance("(each delta phase)")

    v_ph_eq = v_l / SQRT3
    z_total = z_line + z_delta / 3
    i_l = v_ph_eq / abs(z_total)
    i_ph = i_l / SQRT3
    v_load = i_ph * abs(z_delta)
    pf = z_delta.real / abs(z_delta)
    s_load = 3 * i_ph ** 2 * abs(z_delta)
    p_load = 3 * i_ph ** 2 * z_delta.real
    q_load = 3 * i_ph ** 2 * z_delta.imag
    p_loss = 3 * i_l ** 2 * z_line.real
    drop = (v_l - v_load) / v_l * 100
    eff = p_load / (p_load + p_loss) * 100

    print("\n  ----- Balanced Delta-Delta System -----")
    print(f"  Source line voltage     = {v_l:.3f} V (= source phase voltage)")
    print(f"  Line current IL         = {i_l:.4f} A")
    print(f"  Load phase current Iph  = {i_ph:.4f} A")
    print(f"  Load phase/line voltage = {v_load:.3f} V")
    print(f"  Voltage drop in lines   = {drop:.2f} %")
    print(f"  Load power factor       = {pf:.4f} {'lagging' if z_delta.imag > 0 else 'leading' if z_delta.imag < 0 else '(unity)'}")
    print(f"  Load apparent power S   = {s_load:,.2f} VA")
    print(f"  Load active power P     = {p_load:,.2f} W")
    print(f"  Load reactive power Q   = {q_load:,.2f} VAR")
    print(f"  Line power loss         = {p_loss:,.2f} W")
    print(f"  Transmission efficiency = {eff:.2f} %")


def unbalanced_delta_delta():
    v_l = get_positive_float("Source line voltage VL (V): ")
    z_line = get_line_impedance()
    print("\n  Enter the impedance of each delta phase of the load:")
    zab, zbc, zca = get_load_impedance("AB"), get_load_impedance("BC"), get_load_impedance("CA")

    # Source line voltages (ABC sequence, Vab as reference)
    vab = cmath.rect(v_l, 0)
    vbc = cmath.rect(v_l, math.radians(-120))
    vca = cmath.rect(v_l, math.radians(120))

    # Equivalent star source and equivalent star load
    van, vbn, vcn = (vab - vca) / 3, (vbc - vab) / 3, (vca - vbc) / 3
    za, zb, zc = delta_to_star_unbalanced(zab, zbc, zca)
    totals = [z_line + za, z_line + zb, z_line + zc]

    v_nn = neutral_shift([van, vbn, vcn], totals)
    ia, ib, ic = [(v - v_nn) / zt for v, zt in zip([van, vbn, vcn], totals)]

    # Load terminal line voltages and delta phase currents
    v_ab_load = ia * za - ib * zb
    v_bc_load = ib * zb - ic * zc
    v_ca_load = ic * zc - ia * za
    iab, ibc, ica = v_ab_load / zab, v_bc_load / zbc, v_ca_load / zca

    p_total = (abs(iab) ** 2 * zab.real + abs(ibc) ** 2 * zbc.real + abs(ica) ** 2 * zca.real)
    p_loss = (abs(ia) ** 2 + abs(ib) ** 2 + abs(ic) ** 2) * z_line.real

    print("\n  ----- Unbalanced Delta-Delta System -----")
    show_polar("Line current Ia ", ia, "A")
    show_polar("Line current Ib ", ib, "A")
    show_polar("Line current Ic ", ic, "A")
    show_polar("Load phase current Iab", iab, "A")
    show_polar("Load phase current Ibc", ibc, "A")
    show_polar("Load phase current Ica", ica, "A")
    show_polar("Load voltage Vab", v_ab_load, "V")
    show_polar("Load voltage Vbc", v_bc_load, "V")
    show_polar("Load voltage Vca", v_ca_load, "V")
    print(f"  Total load active power P = {p_total:,.2f} W")
    print(f"  Total line loss           = {p_loss:,.2f} W")
    print(f"  Check: Ia + Ib + Ic       = {abs(ia + ib + ic):.6f} A (must be zero)")


def delta_delta_transformer():
    v1 = get_positive_float("Primary line voltage VL1 (V): ")
    turns = get_positive_float("Primary turns per phase N1: ")
    turns2 = get_positive_float("Secondary turns per phase N2: ")
    kva = get_positive_float("Total three-phase rating (kVA): ")

    a = turns / turns2
    v2 = v1 / a
    il1 = kva * 1000 / (SQRT3 * v1)
    il2 = a * il1

    print("\n  ----- Delta-Delta Transformer -----")
    print(f"  Turns ratio a = N1/N2        = {a:.4f}")
    print(f"  Primary phase voltage        = {v1:.3f} V (= line voltage)")
    print(f"  Secondary line voltage       = {v2:.3f} V")
    print(f"  Primary line current         = {il1:.3f} A")
    print(f"  Secondary line current       = {il2:.3f} A")
    print(f"  Primary winding current      = {il1 / SQRT3:.3f} A")
    print(f"  Secondary winding current    = {il2 / SQRT3:.3f} A")
    print(f"  Rating of each transformer   = {kva / 3:.3f} kVA")
    print(f"  Type                         = {'Step-down' if a > 1 else 'Step-up' if a < 1 else 'Isolation'} transformer")
    print("  No phase shift between primary and secondary line voltages.")


def open_delta():
    kva_one = get_positive_float("Rating of ONE transformer (kVA): ")
    closed = 3 * kva_one
    open_cap = SQRT3 * kva_one

    print("\n  ----- Open-Delta (V-V) Connection -----")
    print(f"  Closed delta capacity (3 units) = {closed:.2f} kVA")
    print(f"  Open delta capacity (2 units)   = {open_cap:.2f} kVA")
    print(f"  Open delta / closed delta       = {open_cap / closed * 100:.1f} %")
    print(f"  Open delta / two-unit rating    = {open_cap / (2 * kva_one) * 100:.1f} % (utilisation of the 2 units)")
    print("  Open delta can keep supplying a three-phase load if one transformer fails.")


def menu():
    print("\n" + "=" * 54)
    print("       DELTA-DELTA CONNECTION CALCULATOR")
    print("=" * 54)
    print(" 1. Balanced delta-delta system (with line impedance)")
    print(" 2. Unbalanced delta-delta system (phasor analysis)")
    print(" 3. Delta-delta three-phase transformer")
    print(" 4. Open-delta (V-V) capacity")
    print(" 0. Exit")
    print("-" * 54)


def main():
    while True:
        menu()
        choice = input("Enter your choice: ").strip()

        if choice == "1":
            balanced_delta_delta()
        elif choice == "2":
            unbalanced_delta_delta()
        elif choice == "3":
            delta_delta_transformer()
        elif choice == "4":
            open_delta()
        elif choice == "0":
            print("\nThank you for using the calculator. Goodbye!")
            break
        else:
            print("  Invalid choice. Please select from the menu.")


if __name__ == "__main__":
    main()
