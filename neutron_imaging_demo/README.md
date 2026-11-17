# One wedge, two radiographs: X-rays and neutrons with gVXR and NCrystal

How different does the same polyethylene step look under X-rays and neutrons? This [Jupyter notebook](build_neutron_wedge.ipynb) builds one instrument in gVXR, shows its geometry in OpenGL, and produces two radiographs: one for **80 keV X-rays** and one for **1.8 Å neutrons**. The sample, aperture and detector stay in place, so you can compare transmission through each step on the same grayscale.

## Run the notebook

Download [the notebook](build_neutron_wedge.ipynb), [the wedge mesh](polyethylene_step_wedge.stl) and [the Conda environment file](environment.yml) into the same folder. The STL coordinates are in millimetres. You need Conda and a graphical session that can open an OpenGL 4.3 window.

From that folder, run:

```bash
conda env create -f environment.yml       # first time only
conda activate gvxr-wedge
python -m jupyterlab build_neutron_wedge.ipynb
```

Select the Python kernel from the **gvxr-wedge** environment and run the cells in order. Part 5 displays a snapshot of the instrument. The optional `gvxr.renderLoop()` cell lets you rotate the scene in a separate window; press **Q** in that window to return to the notebook and continue the acquisition. You can skip this cell to go straight to the radiographs.

**WSL2 note:** The Conda file supplies the Python software. The OpenGL window also needs working graphics in WSLg. If rendering is slow, check `glxinfo -B` in the same WSL terminal. On the NVIDIA laptop used for this project, the following launch selected the GPU when the default renderer was `llvmpipe`:

```bash
GALLIUM_DRIVER=d3d12 MESA_D3D12_DEFAULT_ADAPTER_NAME=NVIDIA python -m jupyterlab build_neutron_wedge.ipynb
```

## What the notebook simulates

| Setting | Default |
|---|---|
| Source | Uniform circular aperture, 30 mm diameter, represented by 258 equally weighted positions |
| Geometry | Beam along +z; 3 m from aperture to the downstream face of the wedge (L/D = 100); 10 mm from that face to detector |
| Sample | Polyethylene (CH₂); both calculations use the density from NCrystal's Polyethylene_CH2.ncmat |
| Detector | Ideal count detector; 512 × 512 pixels over 150 × 150 mm |
| Beams | Monochromatic: 80 keV photons and 1.8 Å neutrons |

For X-rays, [gVXR](https://gvirtualxray.sourceforge.io/) calculates attenuation from the photon energy, composition and density. For neutrons, [NCrystal](https://github.com/mctools/ncrystal) supplies the scattering and absorption cross sections at the selected wavelength. The notebook adds them and multiplies by atomic number density to obtain a linear attenuation coefficient in cm⁻¹. It passes this coefficient to gVXR from Python; no changes to gVXR's C++ code are needed. The photon energy still present in the gVXR source is a placeholder during the neutron calculation: the neutron wavelength enters through NCrystal.

For each beam, gVXR computes one projection per aperture position and averages the counts. The notebook then divides each image by a shared open-beam exposure, calculated with the wedge made transparent. The plotted quantity is transmission, with **white = 1** and **black = 0**. Compare the interior of the same step in both panels: the thicker polyethylene steps should look much darker for neutrons at these settings.

## Try a different material

The STL defines the wedge shape. To keep that shape but simulate aluminium instead of polyethylene, make two edits:

1. In **Part 3**, replace `material = "Polyethylene_CH2.ncmat"` with `material = "Al_sg225.ncmat"`. This **NCrystal configuration string** selects the material used to calculate neutron scattering and absorption. It can also take parameters after a semicolon: for example, `"Al_sg225.ncmat;temp=300K"` specifies aluminium at 300 K.
2. In **Parts 4d and 6b**, replace each `gvxr.setCompound("wedge", "CH2")` with `gvxr.setElement("wedge", "Al")`. Keep the following density assignment: the notebook already reads the selected material’s density from NCrystal. This edit makes the X-ray panel represent aluminium too.

Restart the kernel and run the notebook again to compare the same aluminium wedge in both panels. For other materials and options, see the [NCrystal user guide](https://github.com/mctools/ncrystal/wiki/Using-NCrystal) and the [configuration-string parameter reference](https://github.com/mctools/ncrystal/wiki/CfgRefDoc).

## What to explore

Change the neutron wavelength in Part 3 and rerun the notebook from a fresh kernel. The X-ray panel provides a fixed reference while the neutron coefficient and radiograph change. To explore geometric blur instead, keep both beams fixed and increase the sample-to-detector gap in Part 4. For a quick setup check, reduce the aperture sampling in Part 2 to a smaller even number; restore 258 for the default comparison.

Both images describe the **direct beam**. Scattering contributes to removal from that beam, but scattered particles are not followed into other detector pixels. Softening at a step boundary can therefore arise from the finite aperture; this model does not predict a scattering halo. The comparison also assumes monochromatic beams and an ideal detector, so it predicts normalized sample contrast rather than absolute counts on a particular instrument.
